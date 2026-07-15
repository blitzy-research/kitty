# How kitty Regulates Terminal Data Flow (Flow Control & Backpressure)

> **Question answered.** As terminal output — particularly kitty graphics-protocol data — arrives faster than kitty can comfortably process and respond to, how does kitty decide whether to **buffer** it, **pause** intake, or **throttle** (slow) processing? What happens when **responses must be written back** to the program while the **output path is already under pressure**? **Where** in the code do these decisions live? How do they **show up at runtime** under overload, and does kitty **quietly adapt** or produce **visible signs**?

**Subject.** kitty terminal emulator.

- **Investigated source baseline:** HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` ("Wire up applying of font config"). Every `file:line` citation and every observation in this document is against that baseline.
- **Delivery vs. baseline (see §9):** this document is the **only** content added relative to the baseline. It is introduced by a Blitzy delivery commit layered on top of the baseline — the initial add was commit `3ee38c66df78f704971598ee3002223730a1fe5d`, and any later revision of this same file (including this one) is a further descendant commit. **No existing repository file is modified.** The baseline hash and the delivery hash are therefore distinct; the delivery HEAD is a descendant of the baseline, not equal to it.

**Method — run first, write from what was observed (Rule 1).** kitty was compiled from this checkout and driven through its **real PTY entry point** under overload. Each behavioral claim sits next to the **complete, unedited** command output that produced it (or, where a stream is unbounded, a **contiguous unedited window** plus a counted total — labelled as such), with a `file:line` citation and cause→effect reasoning.

**Evidence labels used throughout.**

- **observed (canonical)** — the **default release build** (`kitty/fast_data_types.so` = **1,253,792 bytes**, compiled `-O3 -DNDEBUG`, **no** debug macros) driven through a **real PTY**, or a default compiled constant read from that extension. All **throughput, wall-clock timing, and resource-magnitude** numbers (bytes/s, MiB/s, buffer/quota sizes in bytes, elapsed seconds) come only from this build. It is *not* the source of any **count of diagnostic poll/write events** or **per-`write()` byte size** — those come from the instrumented build below and are labelled as such.
- **observed (instrumented debug build)** — a separate **debug** build (`kitty/fast_data_types.so` = **6,287,848 bytes**, compiled `-Og -DDEBUG` plus `-DDEBUG_EVENT_LOOP -DDEBUG_POLL_EVENTS -DKITTY_PRINT_BYTES_SENT_TO_CHILD`) used **only** to expose event-loop poll timeouts, per-fd returned `revents`, io-loop wakeup events, and per-`write()` byte counts. The injected diagnostics are print-only and change no flow-control logic. Where this document quotes a **count of diagnostic events** (e.g. numbers of `POLLIN` reads or io-loop wakeups) or a **per-event byte size** (e.g. `Wrote: N bytes`), that value is an **(instrumented diagnostic)** taken from *this* build; **no throughput, wall-clock, or resource-magnitude number is reported from this build** — those come only from the canonical build above. (The `DEBUG_EVENT_LOOP` event-loop trace — `loop tick`, `handleEvents`, `pollForEvents` — is emitted by the `glfw/` code compiled into the launcher; the `revents` printer and `Wrote:` lines are emitted by `kitty/fast_data_types.so`.)
- **[non-canonical in-process]** — the real compiled C logic of `graphics.c` / `screen.c` / `vt-parser.c` exercised through the `kitty_tests` parse hooks (`test_create_write_buffer` / `test_commit_write_buffer` / `test_parse_written_data`) with **no live PTY** — the same entry the kitty test-suite uses — sometimes with a **deliberately reduced limit** so a quota/eviction is observable without transmitting hundreds of MiB. The guard/quota code executed is identical to canonical; the PTY entry and the limit value are not.
- **[non-canonical stimulus]** — a real canonical kitty driven through a real PTY, but overload is induced by an **external stand-in** (`SIGSTOP`) rather than by the internal trigger. It reproduces the same OS primitive (PTY backpressure) and is labelled as a stand-in.
- **[inferred]** — read from code, not separately instrumented.

---

## 0. Executive answer

Under overload kitty applies flow control on **two independent axes** in the same poll-based event loop, plus a set of **graphics-specific** memory/response limits.

**(a) Buffer vs. pause vs. throttle (inbound / read path).**
- **Buffer:** inbound bytes from the child are read into a **fixed 1 MiB** VT-parser buffer (`#define BUF_SZ (1024u*1024u)`, `kitty/vt-parser.c:18`). This is the only inbound buffer; it does not grow. A single escape code is separately bounded to **256 KiB** (`MAX_ESCAPE_CODE_LENGTH = BUF_SZ/4u`, `kitty/vt-parser.c:21`) — but that bound is **not** an unconditional per-escape cap (see §2.4).
- **Pause:** once the buffer is full, `vt_parser_has_space_for_input()` returns false and the I/O loop **stops registering the child fd for `POLLIN`** (`children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(...) ? POLLIN : 0;`, `kitty/child-monitor.c:1501`). kitty simply **stops reading**; the kernel PTY buffer then fills and the child's next `write()` **blocks**. This is OS-level backpressure and it is **silent** (no in-band signal, no log).
- **Throttle:** parsing/rendering are **batched** by two small delays — `input_delay` (default **3 ms**, `kitty/options/definition.py:878`) and `repaint_delay` (default **10 ms**, `:866`). `run_worker` defers parsing until `flush || pd->time_since_new_input >= OPT(input_delay) || self->read.sz + 16 * 1024 > BUF_SZ` (`kitty/vt-parser.c:1425`) — it batches by 3 ms but **abandons batching and force-parses when the 1 MiB buffer is nearly full**; likewise `render()` skips a repaint if none is due and no input was read (`kitty/child-monitor.c:875-878`).

**(b) Response write-back under output pressure (outbound / write path).**
Responses to the child (cursor/device-attribute replies, graphics `OK`/error replies) are appended to a **growable per-screen `write_buf`** by the `schedule_write_to_child` machinery. They are drained only when the child fd signals writable — the loop requests `POLLOUT` **only while data is queued** (`events |= (screen->write_buf_used ? POLLOUT : 0);`, `kitty/child-monitor.c:1503`). In `write_to_child`, a partial `write()` advances; `EAGAIN/EWOULDBLOCK` **breaks and keeps the data buffered for the next `POLLOUT`** (`:1463`); `ret == 0` also breaks and retains (`:1458-1460`); only a genuine error discards, with a `perror` (`:1464`). If enqueuing would push the queue **strictly over 100 MiB**, the new data is **discarded and an error is logged** (`kitty/child-monitor.c:341-342`).

**(c) Where in the code.** See the reference map in §6. The mechanisms live in `kitty/vt-parser.c`, `kitty/child-monitor.c`, `kitty/graphics.c`, and `kitty/screen.c`, with defaults in `kitty/options/definition.py` / `types.py`.

**(d) Runtime manifestation.** A child flooding plain bytes into kitty is **paced to kitty's consume rate** — ~112 MiB/s through kitty vs ~108 GiB/s to `/dev/null`, a ~10³× reduction (§2.3, E1). If reading is stopped, the child **freezes blocked in `write()`** on the full PTY (§2.2, E2). Under a response flood the outbound queue reaches the 100 MiB cap and kitty logs `Too much data being sent to child with id: 1, ignoring it` en masse (§3.3, F). Graphics overload surfaces as LRU eviction, `ENOSPC`, and `EINVAL`/`EFBIG` APC error replies (§4). A synchronized-update DECSET `2026` pauses rendering with a 2000 ms default timeout (§5).

**(e) Silent vs. visible.**
- **Silent adaptation:** the inbound `POLLIN` pause and PTY backpressure (no log; the child just blocks); routine buffering; outbound `EAGAIN` retention; graphics LRU eviction; the synchronized-update pause and its 2000 ms auto-unpause; and client-requested response suppression (`q=`).
- **Visible signs:** the outbound overflow log `Too much data being sent to child with id: 1, ignoring it` (`kitty/child-monitor.c:342`); graphics error APC replies (`EINVAL` / `ENOSPC` / `EFBIG`); the pending-mode `Pending mode change to already current mode...` log (`kitty/screen.c:1176`); and the parser `escape code too long` logs (`kitty/vt-parser.c:419`, `:831`).

**(f) Investigation hygiene.** All observation scripts were kept **outside** the repository (under `/tmp/kitty_obs`) and removed afterwards; build artifacts (`build/`, `kitty/fast_data_types.so`, `kitty/launcher/kitty`, `kitty/launcher/kitten`) are git-ignored. The source tree is unchanged apart from this document (§9).

---

## 1. How the investigation was run (exact commands)

**Environment.** Canonical build/run container image `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`). kitty ships **without** a pre-built C extension or launcher in this checkout, so it was compiled first.

A headless X server is needed because kitty is a GUI app. Start it once, capture its PID, and register a `trap` so it is always torn down (on normal exit, `Ctrl-C`, or error); then wait until the display answers before launching kitty:

```
# start Xvfb in the background and remember its PID
Xvfb :99 -screen 0 1280x800x24 +extension GLX +render &
XVFB_PID=$!
export DISPLAY=:99
# always stop it on exit (covers normal exit, Ctrl-C, and errors)
trap 'kill "$XVFB_PID" 2>/dev/null' EXIT INT TERM
# wait (bounded) for the X11 socket before launching kitty
for _ in $(seq 1 50); do [ -S /tmp/.X11-unix/X99 ] && break; sleep 0.1; done
```

This exact lifecycle was exercised: the socket-readiness loop returned `display :99 ready`, and the `trap` reliably reaps `XVFB_PID` at shell exit. (`xvfb-run -a --server-args="-screen 0 1280x800x24 +extension GLX +render"` is an equivalent one-shot wrapper that manages the same lifecycle automatically.)

### 1.1 Build A — canonical (default release build)

This is the build used for **every** throughput/timing/magnitude/count measurement below.

```
CI=true python3 setup.py build --ignore-compiler-warnings
```

Complete tail of the build output and its exit status (observed):

```
[122/122] Compiling kitty/gl-wrapper.c ...
 done
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
# echo "BUILD_A_EXIT=$?"  ->  BUILD_A_EXIT=0
```

- Produces `kitty/fast_data_types.so` (**1,253,792 bytes**), `kitty/launcher/kitty`, `kitty/launcher/kitten`.
- **Full artifact set produced by this build** (all free of debug macros — verified with `strings … | grep`): `kitty/fast_data_types.so` = **1,253,792**, `kitty/glfw-x11.so` = **373,896**, `kitty/glfw-wayland.so` = **451,016**, `kitty/launcher/kitty` = **40,384** bytes. This full set matters: kitty loads `glfw-x11.so` under X11, so a *canonical* run requires **all** of these to be canonical, not just `fast_data_types.so` (a mixed set — canonical `fast_data_types.so` beside an event-loop `glfw-x11.so` — silently injects the `DEBUG_EVENT_LOOP` main-loop trace).
- Compiler flags for the C sources are the default release flags `-O3 -DNDEBUG`; **no** `DEBUG_EVENT_LOOP`, `DEBUG_POLL_EVENTS`, or `KITTY_PRINT_BYTES_SENT_TO_CHILD` macros are defined.
- `--ignore-compiler-warnings` sets `werror=''` at `setup.py:491` (the C-extension build) and `setup.py:1231` (the kittens/launcher build); the flag is declared at `setup.py:2003-2004`. It is required **only** because the environment's `wayland-protocols` is newer than kitty@`815df1e21` expects, which makes the **out-of-scope** file `glfw/wl_window.c` trip `-Werror`. It changes **no source** and does **not** affect flow-control behavior. On the canonical Docker image (matched deps) it is unnecessary.

### 1.2 Build B — instrumented debug build (diagnostics only)

Used **only** to expose poll timeouts (T), per-fd returned `revents`, and per-`write()` byte counts (F slow-drain / EAGAIN). No magnitude/throughput number is read from this build.

```
CI=true CC='cc -DDEBUG_POLL_EVENTS -DKITTY_PRINT_BYTES_SENT_TO_CHILD' \
  python3 setup.py build --debug --extra-logging=event-loop --ignore-compiler-warnings
```

Complete tail of the build output and its exit status (observed):

```
[122/122] Compiling kitty/gl-wrapper.c ...
 done
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
# echo "BUILD_B_EXIT=$?"  ->  BUILD_B_EXIT=0
```

- Produces `kitty/fast_data_types.so` (**6,287,848 bytes**); flags `-Og -DDEBUG -DKITTY_DEBUG_BUILD` plus `-DDEBUG_EVENT_LOOP` (from `--extra-logging=event-loop`), `-DDEBUG_POLL_EVENTS`, and `-DKITTY_PRINT_BYTES_SENT_TO_CHILD` (from `CC`).
- **Instrumented artifact set** (verified with `strings … | grep`): `kitty/fast_data_types.so` = **6,287,848** (contains the `i:%lu` `revents` printer and the `Wrote: %zd` printer), `kitty/glfw-x11.so` = **1,640,256** (contains the `loop tick`/`pollForEvents` event-loop trace), `kitty/launcher/kitty` = **281,392** bytes. **Restoring the canonical build therefore means re-running Build A** (which rebuilds the whole set) — copying back only `fast_data_types.so` leaves the instrumented `glfw-x11.so` in place.
- CC-injection works because `setup.py` reads `CC` via `shlex.split`; **no source file is edited**.
- `DEBUG_EVENT_LOOP` enables the GLFW main-loop trace (`pollForEvents final timeout: ...`, `glfw/backend_utils.c:298`, gated by `glfw/internal.h:837-838`). `DEBUG_POLL_EVENTS` enables the per-fd `revents` printer (`kitty/child-monitor.c:1550-1556`). `KITTY_PRINT_BYTES_SENT_TO_CHILD` enables `Wrote: %zd bytes:` (`kitty/child-monitor.c:1449-1451`).

> **AAP corrections (observed).** `KITTY_PRINT_BYTES_SENT_TO_CHILD` and `DEBUG_POLL_EVENTS` are **compile-time `#ifdef` macros**, not runtime environment variables, and `DEBUG_POLL_EVENTS` is **not** enabled by `--extra-logging`. `DEBUG_POLL_EVENTS` prints `revents` (what `poll()` **returned**), not the gated `.events` request mask (see §3.3 / F6).

### 1.3 Headless run template (real kitty app + real `child-monitor.c` I/O loop)

`PYTHONPATH=.` is the repository root; child scripts live under `/tmp/kitty_obs`. The template (with `<child_script>` / `<args>` as the two slots to fill) is:

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  python3 /tmp/kitty_obs/<child_script>.py <args>
```

A concrete, copy-paste-runnable instantiation (writes the full E2 harness from §10, then runs it — produces `/tmp/kitty_obs/e2_prog.txt`; exits 0):

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 9 \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  python3 /tmp/kitty_obs/flood_plain.py 7 /tmp/kitty_obs/e2_prog.txt
```

Every evidence block below fills the two slots with a specific harness from the §10 appendix and (for real-PTY children whose fd 1/2 *is* the PTY) an output side-file path as `<args>`; the exact instantiation is shown with each block.

### 1.4 In-process harness template ([non-canonical in-process])

Real compiled `graphics.c`/`screen.c`/`vt-parser.c` logic through the `kitty_tests` parse hooks; the `Screen` is built exactly as `kitty_tests` does — the **same** `Callbacks` instance is passed twice (`Screen(c, lines, cols, scrollback, cw, ch, 0, c)`):

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. python3 /tmp/kitty_obs/<harness>.py
```

A concrete, copy-paste-runnable instantiation (writes the full B2 harness from §10, then runs it — prints the storage-LRU table to stdout; exits 0):

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. \
  python3 /tmp/kitty_obs/graphics_b2_storage.py
```

These harnesses run **standalone** (not under the launcher): they import `kitty.fast_data_types` directly and drive the compiled `graphics.c`/`screen.c`/`vt-parser.c` through the `kitty_tests` parse hooks, so no PTY, no `DISPLAY`, and no child process is involved (the `DISPLAY=:99` prefix is harmless and kept only for a uniform command line).

### 1.5 Canonical defaults confirmed from the built (canonical) extension

```
input_delay   = 3 ms    [kitty/options/definition.py:878; kitty/options/types.py:536]
repaint_delay = 10 ms   [kitty/options/definition.py:866; kitty/options/types.py:567]
sync_to_monitor = True  [kitty/options/types.py:586]
VT_PARSER_BUFFER_SIZE          = 1048576 bytes ( 1 MiB )   [kitty/vt-parser.c:18  BUF_SZ]
VT_PARSER_MAX_ESCAPE_CODE_SIZE =  262144 bytes ( 256 KiB ) [kitty/vt-parser.c:21  MAX_ESCAPE_CODE_LENGTH]
GraphicsManager default storage_limit = 335544320 bytes ( 320 MiB ) [kitty/graphics.c:25/:78]
```

### 1.6 Repository test corroboration (canonical build)

The repository's own unit tests were run against the **canonical** build via the launcher (`+launch test.py` is required so the Go-backed `sys.kitty_run_data` is present; plain `python3 test.py` cannot). Exact commands and complete result lines (each exits 0):

```
DISPLAY=:99 ./kitty/launcher/kitty +launch test.py --module parser
#   test_utf8_simd_decode (kitty_tests.parser.TestParser.test_utf8_simd_decode) ... ok
#   ----------------------------------------------------------------------
#   Ran 16 tests in 0.056s
#   OK                                    (exit 0)

DISPLAY=:99 ./kitty/launcher/kitty +launch test.py --module screen
#   test_zwj (kitty_tests.screen.TestScreen.test_zwj) ... ok
#   ----------------------------------------------------------------------
#   Ran 36 tests in 0.082s
#   OK                                    (exit 0)

DISPLAY=:99 ./kitty/launcher/kitty +launch test.py --module graphics
#   test_xor_data (kitty_tests.graphics.TestGraphics.test_xor_data) ... ok
#   ----------------------------------------------------------------------
#   Ran 19 tests in 0.208s
#   OK                                    (exit 0)
```

These three modules cover the parser (`parser` — 16 tests), the screen/pending-mode state machine (`screen` — 36 tests), and the graphics manager (`graphics` — 19 tests) that this document's claims rest on; `kitty_tests/graphics.py::test_graphics_quota_enforcement` in particular independently exercises the storage-LRU and 5×-frame-cache paths of §4.

---

## 2. Inbound (read) path — buffer, pause, throttle

### 2.1 Buffer — a fixed 1 MiB parser buffer

All child output flows through the VT parser, whose input buffer is a compile-time constant:

```c
// kitty/vt-parser.c:18
#define BUF_SZ (1024u*1024u)
// kitty/vt-parser.c:21
#define MAX_ESCAPE_CODE_LENGTH (BUF_SZ / 4u)
```

`read_bytes()` (`kitty/child-monitor.c`) obtains a write region via `vt_parser_create_write_buffer` (`kitty/vt-parser.c:1451`), whose available space is `BUF_SZ − offset`; if space is `0` it returns **without reading**. So inbound data is bounded by this **1 MiB** buffer — it does not grow.

### 2.2 Pause — stop registering POLLIN → OS PTY backpressure (silent)

On every loop iteration the child fd's polled events are recomputed. Reading is gated on parser space, and the has-space predicate is the complete short function below (no elisions):

```c
// kitty/child-monitor.c:1501
children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;
```

```c
// kitty/vt-parser.c:1477-1484  (complete function)
vt_parser_has_space_for_input(const Parser *p) {
    PS *self = (PS*)p->state;
    bool ans;
    with_lock {
        ans = self->read.sz + self->write.pending < BUF_SZ;
    } end_with_lock;
    return ans;
}
```

**Cause→effect:** when `read.sz + write.pending` reaches the 1 MiB `BUF_SZ`, `vt_parser_has_space_for_input` returns false → the child fd is **not** registered for `POLLIN` → kitty stops reading → the kernel PTY buffer fills → the child's next `write()` **blocks**. kitty sends **no** in-band throttle to the child; the pause is realized purely by *not reading*, so it is **silent**.

**Evidence E2 — child-stall [non-canonical stimulus] (canonical build, real PTY).** A child (`/tmp/kitty_obs/flood_plain.py`) writes `b'x'*65536` to fd 1 in a loop and logs `(cumulative_bytes, elapsed)` to a side file after every `write()`; `write()` **blocks** when the PTY buffer is full. Child body (inline):

```python
# /tmp/kitty_obs/flood_plain.py
import os, sys, time
duration = float(sys.argv[1]); progress_path = sys.argv[2]
chunk = b'x' * 65536
start = time.monotonic(); written = 0; last_log = 0.0
with open(progress_path, 'w', buffering=1) as pf:
    while time.monotonic() - start < duration:
        written += os.write(1, chunk)          # blocks when the PTY buffer is full
        now = time.monotonic()
        if now - last_log >= 0.2:
            pf.write("%d %.3f\n" % (written, now - start)); pf.flush(); last_log = now
    pf.write("FINAL %d %.3f\n" % (written, time.monotonic() - start)); pf.flush()
```

Mid-flood the **kitty reader process** (selected by `comm=kitty` — not the `timeout` wrapper) is `SIGSTOP`-ed, then `SIGCONT`-ed after ~2.6 s:

```
kitty PID selector:  ps -eo pid,comm | awk '$2=="kitty"{print $1}'
kill -STOP <pid>   ;  sleep 2.6  ;  kill -CONT <pid>
```

Complete progress file, run 1 (unedited); samples are ~0.2 s apart until the freeze, then jump:

```
65536 0.000
22085632 0.202
44892160 0.403
66322432 0.603
88211456 0.803
110886912 1.003
132186112 1.208
154468352 1.408
176947200 1.608
199294976 1.813
221642752 2.013
243859456 2.213
265355264 2.413
287834112 2.614
308871168 5.422
330432512 5.622
352387072 5.823
374407168 6.025
396689408 6.225
418447360 6.425
440532992 6.625
462487552 6.825
484507648 7.027
506789888 7.228
528547840 7.428
```

**Reading the evidence:** the samples advance every ~0.2 s until `287834112 2.614`, then the **next** sample is `308871168 5.422` — a **2.808 s wall-clock gap** during which the byte counter barely moved (only ~20 MiB, the drain right after resume). The child could not even log during the gap because it was blocked **inside `os.write()`** (the `pf.write` runs only after `write()` returns). After `SIGCONT` the 0.2 s cadence resumes. Run 2 was identical in shape: `294518784 2.643` → `313393152 5.430` (gap 2.787 s). There is **no log line** for this pause — the only signal is the blocking `write()`, confirming it is **silent**.

> **Faithful labelling (F5).** `SIGSTOP` is an **external stand-in** that forces kitty to stop reading; it reproduces the *effect* of the internal pause (`POLLIN` gated off when the parser buffer is full, `kitty/child-monitor.c:1501` + `kitty/vt-parser.c:1477`) via the identical OS primitive (a full PTY blocking the child's `write()`). The internal parser-full trigger itself is **[inferred]** from the gating code; E2 observes only its OS-level *effect*.

**Evidence E2b — natural-pressure parser-full stall (observed, canonical) + buffer-full/resume cycles (observed, instrumented).** The `SIGSTOP` of E2 is only a stand-in; the parser buffer can also be filled **naturally**, with no external signal, by a producer fast enough to outrun kitty's read+parse loop. A ~GB/s C producer (`/tmp/kitty_obs/flood.c`, 4 MiB per `write()`) run as a real child of a normally-built kitty does exactly this:

```c
#define _GNU_SOURCE
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <time.h>
#include <stdlib.h>
static double now(void){struct timespec ts;clock_gettime(CLOCK_MONOTONIC,&ts);return ts.tv_sec+ts.tv_nsec/1e9;}
int main(int argc,char**argv){
    size_t CH=4u*1024u*1024u;            // 4 MiB per write
    long target_mib = argc>1?atol(argv[1]):512;   // total MiB
    char *buf=malloc(CH);
    for(size_t i=0;i<CH;i++) buf[i]= (i%80==79)?'\n':'X';
    FILE*prog=fopen("/tmp/kitty_obs/cflood_progress.txt","w");
    double t0=now(); long long total=0; long long target=(long long)target_mib*1024*1024;
    int nblock=0; double maxb=0.0;
    while(total<target){
        double a=now();
        ssize_t n=write(1,buf,CH);
        double b=now();
        if(n<0){ fprintf(prog,"WRITE_ERR errno-based\n"); break; }
        total+=n; double dt=b-a;
        if(dt>0.05){ nblock++; if(dt>maxb)maxb=dt; }
        fprintf(prog,"t=%.4f total=%lld write_dt=%.5f n=%zd\n",b-t0,total,dt,n); fflush(prog);
    }
    fprintf(prog,"DONE t=%.4f total=%lld blocked(>50ms)=%d max_block=%.4fs\n",now()-t0,total,nblock,maxb); fflush(prog);
    struct timespec s={0,200000000}; nanosleep(&s,NULL);
    return 0;
}
```

Command (build the producer, then run it under the **canonical** launcher; it logs every `write()` to a side file):

```
cc -O2 -o /tmp/kitty_obs/flood_bin /tmp/kitty_obs/flood.c
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 timeout 15 \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  /tmp/kitty_obs/flood_bin 64      # 64 = total MiB to push
```

Canonical, two runs (unedited `DONE` lines; every 4 MiB `write()` that blocked >50 ms is counted):

```
run 1: DONE t=1.4909 total=67108864 blocked(>50ms)=16 max_block=0.1052s
run 2: DONE t=1.5471 total=67108864 blocked(>50ms)=16 max_block=0.1390s
```

All **16 of 16** writes (64 MiB / 4 MiB) blocked >50 ms in both runs, and the child is paced to ~43 MiB/s (64 MiB in ~1.5 s) purely by the full PTY — kitty emits no throttle and no log. This is the same silent OS-level backpressure as E2, but reached through **genuine parser-buffer pressure** rather than an external `SIGSTOP`.

To confirm the buffer actually reaches `BUF_SZ` (rather than kitty simply keeping pace), the identical flood was run against an **instrumented** build (`DEBUG_POLL_EVENTS` compiled into `fast_data_types.so`; glfw left canonical) that prints each descriptor's `revents`. Whenever the parser buffer fills, the **main** thread force-parses, frees space (`write_space_created = read.sz >= BUF_SZ`, `kitty/vt-parser.c:1438`) and calls `wakeup_io_loop` (`kitty/child-monitor.c:442`), which pokes the io-loop's wakeup fd (`i:0`). The count of `i:0 POLLIN` wakeups is therefore a positive proxy for buffer-full -> resume cycles:

```
run 1: i:0 POLLIN wakeups = 20   i:2 POLLIN reads = 160094
run 2: i:0 POLLIN wakeups = 13   i:2 POLLIN reads = 155426  (+ i:2 POLLHUP = 1 at child exit)
```

Both runs show wakeups **> 0** (run-variable instrumented-diagnostic counts): the 1 MiB ceiling was hit repeatedly — the exact condition under which `vt_parser_has_space_for_input` returns false so `:1501` sets the child's polled `events` to `0` (read paused), **and** under which `run_worker`'s force-parse branch `self->read.sz + 16*1024 > BUF_SZ` (`kitty/vt-parser.c:1425`) fires (see §2.3).

> **Faithful labelling (F5b).** The child-stall (canonical) and the buffer-full wakeup cycles (instrumented) are **observed**. The literal act of writing `events = 0` to drop `POLLIN` remains **[inferred from `:1501`]**: `DEBUG_POLL_EVENTS` prints only the `revents` that are **present**, so a descriptor whose `events` has been gated to `0` produces **no** line — the drop manifests as the *absence* of `i:2 POLLIN` reads during the stall, not as a positive event. Genuine attempts to surface the drop directly are reported above (GB/s producer, ~155-160 k reads/run, and the `i:0` wakeup proxy); the register write itself stays inferred from the one-line gate.

### 2.3 Throttle — pacing under sustained flood, and the delay batching

**Evidence E1 — throughput pacing (observed, canonical).** The same child, first to `/dev/null` (measures the raw producer rate), then through a real kitty PTY (bounded by kitty's read+parse+render rate).

Baseline (`child → /dev/null`), complete progress file, run 1 (unedited):

```
# CI=true PYTHONPATH=. timeout 5 python3 /tmp/kitty_obs/flood_plain.py 3 /tmp/kitty_obs/e1_base_r1.txt >/dev/null
65536 0.000
22823108608 0.200
46040678400 0.400
69387681792 0.600
92712927232 0.800
115950813184 1.000
139233591296 1.200
162229256192 1.400
185472974848 1.600
208511107072 1.800
231777763328 2.000
255080857600 2.200
278388736000 2.400
301608796160 2.600
324884627456 2.800
FINAL 347981873152 3.000
```

Through kitty (`child → real PTY`), complete progress file, run 1 (all 75 lines, unedited):

```
# DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 18 kitty/launcher/kitty --config NONE \
#   -o confirm_os_window_close=0 python3 /tmp/kitty_obs/flood_plain.py 15 /tmp/kitty_obs/e1_kitty_r1.txt
65536 0.000
23134208 0.201
47251456 0.404
71368704 0.609
95485952 0.814
119603200 1.019
143589376 1.219
166461440 1.419
189988864 1.619
213975040 1.821
237043712 2.023
261160960 2.227
285278208 2.432
309395456 2.636
333512704 2.839
357629952 3.043
381747200 3.248
405864448 3.452
429981696 3.657
454098944 3.862
477757440 4.062
501284864 4.266
525402112 4.468
549519360 4.672
573636608 4.875
595656704 5.079
618725376 5.283
641990656 5.483
661716992 5.691
684785664 5.892
708902912 6.094
733020160 6.297
757137408 6.499
781254656 6.701
805371904 6.902
829489152 7.105
853606400 7.306
877723648 7.509
901840896 7.711
925958144 7.913
950075392 8.115
974192640 8.319
998309888 8.522
1022427136 8.725
1046544384 8.929
1070661632 9.132
1094778880 9.334
1118896128 9.538
1143013376 9.739
1167130624 9.940
1191247872 10.145
1215365120 10.348
1239482368 10.552
1263599616 10.754
1287716864 10.956
1311834112 11.158
1335951360 11.360
1360068608 11.565
1383727104 11.765
1407254528 11.966
1431371776 12.168
1454964736 12.368
1478557696 12.571
1502674944 12.777
1526792192 12.979
1550909440 13.182
1575026688 13.385
1599143936 13.590
1622933504 13.790
1646329856 13.994
1670447104 14.199
1693974528 14.399
1717633024 14.602
1741750272 14.806
FINAL 1765015552 15.000
```

Both runs, derived from the `FINAL` lines (two-run stability, F7):

| condition | run | FINAL bytes / elapsed | rate |
|---|---|---|---|
| child → /dev/null | 1 | 347,981,873,152 / 3.000 s | ~108.0 GiB/s |
| child → /dev/null | 2 | 352,471,810,048 / 3.000 s | ~109.4 GiB/s |
| child → kitty (real PTY) | 1 | 1,765,015,552 / 15.000 s | ~112.2 MiB/s |
| child → kitty (real PTY) | 2 | 1,752,236,032 / 15.005 s | ~111.4 MiB/s |

**Cause→effect:** the producer alone reaches ~108 GiB/s (the kernel discards instantly). Through kitty it is paced to ~112 MiB/s — a **~10³× reduction** — because the child cannot outrun kitty's read+parse+render loop; the child completes its loop (reaches `FINAL`), i.e. it is paced, not killed. This aggregate pacing is the *net* effect of the 1 MiB buffer + `POLLIN` gating + delay batching; it does **not**, by itself, isolate `input_delay` as the sole cause.

**The delay-batching mechanism** behind the pacing. `run_worker` defers parsing until:

```c
// kitty/vt-parser.c:1425
if (flush || pd->time_since_new_input >= OPT(input_delay) || self->read.sz + 16 * 1024 > BUF_SZ) {
```

This condition has **two force arms**. The `pd->time_since_new_input >= OPT(input_delay)` arm is the ordinary **3 ms batching**, **observed** in Evidence T below (a modest producer whose buffer never approaches full). The `self->read.sz + 16 * 1024 > BUF_SZ` arm is the **near-full override** that abandons batching to drain a nearly-full buffer immediately; it is **observed (instrumented)** in Evidence E2b (§2.2): every `i:0 POLLIN` wakeup there follows a force-parse that drained a *full* buffer (`write_space_created = read.sz >= BUF_SZ`, `kitty/vt-parser.c:1438`), and `read.sz >= BUF_SZ` implies `read.sz + 16*1024 > BUF_SZ`, so the near-full arm fired **at least once per wakeup** — i.e. ≥20 (run 1) and ≥13 (run 2) times under the GB/s flood. (The exact per-cycle boundary value of `read.sz` is not printed, so the precise trip point stays **[inferred from `:1425`]**; that the arm *fires* is observed.)

and there are **two distinct poll loops**, each using `input_delay`:

1. The **child-PTY I/O poll** runs on a **dedicated thread** `io_loop()` (`pthread_create(..., io_loop, ...)`, `kitty/child-monitor.c:291`; entry `:1481`). Its `poll()` timeout is `OPT(input_delay) − (now − last_main_loop_wakeup_at)` when wakeups are pending (`:1506-1509`), else it blocks (`:1512`). This loop applies the `POLLIN`/`POLLOUT` gates (`:1501`, `:1503`).
2. The **GLFW main/display loop** computes its wait from `set_maximum_wait(...)` accumulations — after input is read, `do_parse` calls `set_maximum_wait(OPT(input_delay) − pd.time_since_new_input)` (`:445`), fed into `update_main_loop_timer(state_check_timer, MAX(0, maximum_wait), ...)` (`:1255`).

**Evidence T — poll timeouts (observed, instrumented debug build).** A modest periodic producer (`tiny_input.py`: 5 lines then idle) traced with `DEBUG_EVENT_LOOP`. Contiguous unedited head of the trace (the **GLFW main loop** — the string is defined only at `glfw/backend_utils.c:298`):

```
[0.156] Failed to open systemd user bus with error: Connection refused
[0.160] starting handleEvents(0.00)
[0.160] pollForEvents final timeout: 0.000
[0.160] State check timer firedProcessing global stateinput_read: 0, check_for_active_animated_images: 1[0.174] display_read_ok: 0
[0.174] other dispatch done
[0.174] --------- loop tick, wakeups_happened: 1 ----------
Processing global stateinput_read: 0, check_for_active_animated_images: 0[0.174] starting handleEvents(0.00)
[0.174] pollForEvents final timeout: 0.000
[0.174] display_read_ok: 0
[0.174] other dispatch done
[0.174] --------- loop tick, wakeups_happened: 0 ----------
[0.174] starting handleEvents(-0.00)
[0.174] pollForEvents final timeout: 0.001
State check timer firedProcessing global stateinput_read: 1, check_for_active_animated_images: 0[0.177] display_read_ok: 0
[0.178] other dispatch done
[0.178] --------- loop tick, wakeups_happened: 0 ----------
[0.178] starting handleEvents(-0.00)
[0.178] pollForEvents final timeout: 0.484
[0.222] display_read_ok: 0
[0.222] other dispatch done
[0.222] --------- loop tick, wakeups_happened: 1 ----------
Processing global stateinput_read: 0, check_for_active_animated_images: 0[0.222] starting handleEvents(-0.00)
[0.222] pollForEvents final timeout: 0.003
State check timer firedProcessing global stateinput_read: 1, check_for_active_animated_images: 0[0.228] display_read_ok: 0
[0.228] other dispatch done
[0.228] --------- loop tick, wakeups_happened: 0 ----------
[0.228] starting handleEvents(-0.00)
[0.228] pollForEvents final timeout: 0.434
```

Contiguous unedited tail of the same trace (the **child-PTY `io_loop` thread**'s per-fd `revents`, flushed at exit because it is a separate thread):

```
[2.425] other dispatch done
[2.425] --------- loop tick, wakeups_happened: 1 ----------
Processing global stateinput_read: 0, check_for_active_animated_images: 0[2.433] main loop exiting
i:0 POLLIN
i:2 POLLIN
i:2 POLLIN
i:2 POLLIN
i:2 POLLIN
i:2 POLLIN
i:1 POLLIN
i:2 POLLHUP
i:0 POLLIN
```

**Cause→effect (F9):** `pollForEvents final timeout: 0.003` is the **GLFW main loop** waiting `0.003 s = 3 ms = OPT(input_delay)` (set via `set_maximum_wait` at `:445` → `:1255`), **not** the child I/O poll. The `0.484 → 0.434 → …` values count down toward a scheduled repaint/animation; idle waits settle near `0.497 s` (the ~500 ms state-check). The **child-PTY** poll is the separate `io_loop` thread whose `i:N` `revents` appear at the tail (`i:0`/`i:1` are the wakeup/signal fds since `EXTRA_FDS=2`; `i:2` is the first child; `i:2 POLLHUP` = child exit). `render()` additionally **skips a repaint** when nothing is due and no input was read — `if (!input_read && time_since_last_render < OPT(repaint_delay)) { set_maximum_wait(...); return; }` (`kitty/child-monitor.c:875-878`) — so `repaint_delay` is a render throttle that is *bypassed* while input is pending.

### 2.4 Escape-code length handling — the 256 KiB bound is conditional (visible)

The 256 KiB `MAX_ESCAPE_CODE_LENGTH` is **not** an unconditional per-escape cap. The relevant logic (quoted without elision):

```c
// kitty/vt-parser.c:394-420  accumulate_st_terminated_esc_code (ST-terminated: OSC/APC/DCS/PM/SOS)
static bool
accumulate_st_terminated_esc_code(PS *self, void(dispatch)(PS*, uint8_t*, size_t, bool)) {
    size_t pos;
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
            self->buf[self->read.pos] = before;
            continue_osc_52(self);
            return accumulate_st_terminated_esc_code(self, dispatch);
        }
        REPORT_ERROR("%s escape code too long (%zu bytes), ignoring it", vte_state_name(self->vte_state), pos);
        return true;
```

CSI has a **separate** byte-length truncation path and a **separate** parameter-count guard:

```c
// kitty/vt-parser.c:830-832
    if (UNLIKELY(*pos - start > MAX_ESCAPE_CODE_LENGTH)) {
        REPORT_ERROR("CSI escape too long ignoring and truncating");
        return true;
```

```c
// kitty/vt-parser.c:719-721   (MAX_CSI_PARAMS = 256u, kitty/vt-parser.c:22)
    if (csi->num_params >= MAX_CSI_PARAMS) {
        REPORT_ERROR("CSI escape code has too many parameters, ignoring it");
        return false;
```

**Evidence D (observed, canonical, real PTY).** A child emits four ~300 KB constructs. `REPORT_ERROR` resolves to `log_error` (`kitty/vt-parser.c:125`) → stderr with a `[PARSE ERROR]` prefix. Complete stderr, run 1 (5 lines, unedited; first two are benign headless-DBUS noise):

```
[0.154] Failed to open systemd user bus with error: Connection refused
[0.174] [glfw error 65544]: Failed to connect to DBUS session bus. DBUS error: Address does not contain a colon
[0.575] [PARSE ERROR] VTE_OSC escape code too long (300002 bytes), ignoring it
[0.977] [PARSE ERROR] Unknown char after ESC: 0x5c
[2.178] [PARSE ERROR] CSI escape too long ignoring and truncating
```

Run 2 was identical (timestamps `0.580` / `0.981` / `2.183`). Reading the four cases against the source:

- **Case A — complete OSC + ST** (`ESC ] 9 ; <300000 bytes> ST`): **no error** — a *complete* ST-terminated sequence is dispatched **without** the length check (the "lets be generous" branch, `vt-parser.c:398-404`). Observable by the **absence** of any error line.
- **Case B — incomplete OSC 9** (no ST): `VTE_OSC escape code too long (300002 bytes), ignoring it` (`vt-parser.c:419`; `300002 = 300000 + "9;"`). After the discard the parser resets to `NORMAL`, so the trailing `ESC \` (`0x5c`) is seen as a stray → `Unknown char after ESC: 0x5c`.
- **Case C — incomplete OSC 52** (no ST): **no "too long" error and no stray char** — the special OSC-52 branch **partially dispatches and continues** (`dispatch(..., true)` then `continue_osc_52`, `vt-parser.c:407-417`), and the closing `ESC \` is absorbed as the ST of the continuing OSC-52. The runtime contrast with Case B directly demonstrates the OSC-52 special-casing.
- **Case D — one giant CSI parameter** (`ESC [` + `'1'*300000`, no final byte): `CSI escape too long ignoring and truncating` (`vt-parser.c:831`). (A `';'`-separated payload instead hits the **different** guard `MAX_CSI_PARAMS=256` at `:720` → `CSI escape code has too many parameters, ignoring it`; `csi_add_digit` caps at 16 stored digits but still *consumes* extra digit bytes, so one parameter can exceed 256 KiB by bytes.)

---

## 3. Outbound (write-to-child) path — responses under output pressure

### 3.1 Queue + 100 MiB cap (the clearest visible sign)

Responses are appended to the per-screen `write_buf` by the `schedule_write_to_child` machinery, which grows the buffer up to a hard ceiling:

```c
// kitty/child-monitor.c:339-345
            size_t space_left = screen->write_buf_sz - screen->write_buf_used; \
            if (space_left < sz) { \
                if (screen->write_buf_used + sz > 100 * 1024 * 1024) { \
                    log_error("Too much data being sent to child with id: %lu, ignoring it", id); \
                    screen_mutex(unlock, write); \
                    break; \
                } \
                screen->write_buf_sz = screen->write_buf_used + sz; \
```

**Boundary (F16):** the cap is checked **only when the free space is insufficient** (`space_left < sz`) and rejects **only when the scheduled total would be strictly greater than 100 MiB** (`write_buf_used + sz > 100*1024*1024`). On rejection the branch logs and `break`s **without** marking the data written, so it is **discarded**; there is **no rate-limiting** — every over-cap enqueue attempt is discarded and logged. `log_error` here is **unconditionally compiled** (not behind any `#ifdef`), so this line is the canonical *visible sign* of outbound overflow.

### 3.2 POLLOUT-driven draining + full `write_to_child` branch ladder

The child fd is registered for `POLLOUT` **only while** output is queued:

```c
// kitty/child-monitor.c:1503
children_fds[EXTRA_FDS + i].events |= (screen->write_buf_used ? POLLOUT  : 0);
```

`write_to_child` drains when the fd is writable; the complete branch ladder (quoted without elision):

```c
// kitty/child-monitor.c:1447-1466
    while (written < screen->write_buf_used) {
        ret = write(fd, screen->write_buf + written, screen->write_buf_used - written);
#ifdef KITTY_PRINT_BYTES_SENT_TO_CHILD
        fprintf(stderr, "Wrote: %zd bytes: ", ret);
#endif
        if (ret > 0) {
#ifdef KITTY_PRINT_BYTES_SENT_TO_CHILD
            print_text(screen->write_buf + written, ret);
#endif
            written += ret;
        }
        else if (ret == 0) {
            // could mean anything, ignore
            break;
        } else {
            if (errno == EINTR) continue;
            if (errno == EWOULDBLOCK || errno == EAGAIN) break;
            perror("Call to write() to child fd failed, discarding data.");
            written = screen->write_buf_used;
        }
```

Branch coverage (observed vs. inferred):
- **`ret > 0`** — advance `written` (`:1452-1456`). **Observed** (every positive `Wrote:` sample below).
- **`ret == 0`** — `break`, retaining the queued data (`:1458-1460`). **Observed [non-canonical fault injection]** (Evidence W below): the reply is retained and re-sent on the next drain — no data lost.
- **`EINTR`** — `continue` and retry (`:1462`). **Observed [non-canonical fault injection]** (Evidence W): the reply is retried in-loop — no data lost.
- **`EWOULDBLOCK/EAGAIN`** — `break`, **keep buffered, retry on next `POLLOUT`** (`:1463`). **Observed** (F EAGAIN below).
- **other `errno`** — `perror("Call to write() to child fd failed, discarding data.")` then discard the whole buffer (`:1464-1465`). **Observed [non-canonical fault injection]** (Evidence W): the buffer is discarded (one reply lost) and the `perror` is printed — a visible sign.

**Evidence W — the three rare write branches, forced by fault injection [non-canonical fault injection].** `ret == 0`, `EINTR`, and a hard `write()` error cannot be produced on demand through a real PTY, so — after genuine attempts via the real path failed to force them — a minimal `LD_PRELOAD` shim intercepts the **single** `write()` that carries the DA2 reply prefix `ESC [ > 1` (kitty -> child) and injects one fault. It matches only the reply prefix, never the child's `ESC [ > c` query, so the child (which inherits `LD_PRELOAD`) is unaffected. The child sends **6** DA2 queries and counts replies received; **6 received = retained/retried, 5 received = one discarded** — which cleanly separates the branches.

Injector (`/tmp/kitty_obs/faultwrite.c`):

```c
#define _GNU_SOURCE
#include <unistd.h>
#include <errno.h>
#include <dlfcn.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>
typedef ssize_t (*write_fn)(int, const void*, size_t);
static write_fn real_write = NULL;
static int fired = 0;
ssize_t write(int fd, const void *buf, size_t n) {
    if (!real_write) real_write = (write_fn)dlsym(RTLD_NEXT, "write");
    const char *mode = getenv("FAULT_MODE");
    const unsigned char *b = (const unsigned char*)buf;
    // Match ONLY the DA2 reply prefix ESC [ > 1  (kitty->child). The DA2 query ESC [ > c does NOT match.
    if (mode && !fired && n >= 4 && b[0]==0x1b && b[1]=='[' && b[2]=='>' && b[3]=='1') {
        fired = 1;
        fprintf(stderr, "[FAULTWRITE] injecting %s on fd=%d n=%zu (DA2 reply ESC[>1)\n", mode, fd, n);
        if (strcmp(mode, "zero") == 0) return 0;
        if (strcmp(mode, "eintr") == 0) { errno = EINTR; return -1; }
        if (strcmp(mode, "eio")  == 0) { errno = EIO;  return -1; }
    }
    return real_write(fd, buf, n);
}
```

Child (`/tmp/kitty_obs/multi_da2.py`):

```python
import os, sys, time, tty, select
tty.setraw(0)
prog = sys.argv[1]
time.sleep(0.4)
buf=b''; sent=0
t0=time.monotonic(); next_send=0.0
while time.monotonic()-t0 < 4.0:
    now=time.monotonic()-t0
    if sent < 6 and now >= next_send:
        os.write(1, b'\x1b[>c'); sent+=1; next_send = now + 0.4
    r,_,_=select.select([0],[],[],0.1)
    if r:
        d=os.read(0,65536)
        if d: buf+=d
n_replies = buf.count(b'\x1b[>1;4000;35c')
with open(prog,'w') as f:
    f.write("sent_queries=%d received_bytes=%d received_replies=%d\n" % (sent, len(buf), n_replies))
time.sleep(0.2)
```

Build + run (kitty built `-DKITTY_PRINT_BYTES_SENT_TO_CHILD` so it prints `Wrote: N bytes` = Build B fdt):

```
cc -O2 -shared -fPIC -o /tmp/kitty_obs/faultwrite.so /tmp/kitty_obs/faultwrite.c -ldl
for mode in eintr zero eio; do
  LD_PRELOAD=/tmp/kitty_obs/faultwrite.so FAULT_MODE=$mode DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 timeout 12 \
    kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
    python3 /tmp/kitty_obs/multi_da2.py /tmp/kitty_obs/fault_prog.txt 2>fault_$mode.log
  cat /tmp/kitty_obs/fault_prog.txt
done
```

Observed (each mode **byte-identical across two runs**; the relevant `FAULTWRITE`/`Wrote:` lines and the child's reply tally):

```
# EINTR  (:1462 continue) -- retried in-loop, nothing lost
child: sent_queries=6 received_bytes=78 received_replies=6
[FAULTWRITE] injecting eintr on fd=8 n=13 (DA2 reply ESC[>1)
Wrote: -1 bytes: Wrote: 13 bytes: \x1b[>1;4000;35c

# ret==0  (:1458-1460 break) -- retained, re-sent on next drain, nothing lost
child: sent_queries=6 received_bytes=78 received_replies=6
[FAULTWRITE] injecting zero on fd=8 n=13 (DA2 reply ESC[>1)
Wrote: 0 bytes: Wrote: 13 bytes: \x1b[>1;4000;35c

# hard error EIO  (:1464-1465 perror + discard) -- WHOLE buffer discarded, one reply lost
child: sent_queries=6 received_bytes=65 received_replies=5
[FAULTWRITE] injecting eio on fd=8 n=13 (DA2 reply ESC[>1)
Wrote: -1 bytes: Call to write() to child fd failed, discarding data.: Input/output error
```

**Cause->effect.** `EINTR` (`:1462`) `continue`s the drain loop and the same reply is rewritten -> the child still receives **6/6**. `ret == 0` (`:1458-1460`) `break`s but **keeps** `write_buf` intact, so the reply is re-sent on the next `POLLOUT` -> **6/6**. A hard error (here `EIO`, `:1464-1465`) runs `perror("Call to write() to child fd failed, discarding data.")` and sets `written = write_buf_used`, discarding the **entire** queued buffer -> the child receives **5/6** (one reply permanently lost). Only this last branch loses data, and it is the only one that emits a **visible** sign (the `perror`); the first two are silent and lossless. The injection is a **[non-canonical fault injection]** stand-in for the `write()` return value; the *branch logic that consumes that return value* is kitty's own, exercised through the normal `write_to_child` drain.

### 3.3 Evidence F — driving the outbound path to the cap and observing draining

**Stimulus.** A child floods **DA2 requests** `ESC [ > c` (4 bytes → 13-byte reply `ESC[>1;4000;35c`; a **3.25×** amplifier, confirmed with `od -c`; reply body `>1;` + `PRIMARY_VERSION=4000` + `;` + `SECONDARY_VERSION=35` + `c` at `kitty/screen.c:2128`), puts its tty in **raw mode** (`tty.setraw(0)`, so kitty's replies are **not** echoed back into kitty's input), and **never reads its stdin** so replies pile into `write_buf`. Child body (inline):

```python
# /tmp/kitty_obs/fill_writebuf.py
import os, sys, time, tty
tty.setraw(0)
dur = float(sys.argv[1]); prog = sys.argv[2]
batch = b'\x1b[>c' * 65536       # 65536 DA2 queries per write (256 KiB)
start = time.monotonic(); n = 0
with open(prog, 'w', buffering=1) as pf:
    while time.monotonic() - start < dur:
        n += os.write(1, batch)
        pf.write("%d %.3f\n" % (n, time.monotonic() - start)); pf.flush()
    pf.write("FINAL %d %.3f\n" % (n, time.monotonic() - start)); pf.flush()
```

**F — overflow (observed, canonical).** The overflow stream is unbounded (millions of lines), so kitty's stderr was piped through an `awk` filter that emits a **contiguous unedited window** around the first crossing (8 preceding lines + the match + the next 30) and **counts every occurrence** without materializing the multi-hundred-MB log:

```
# DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 9 kitty/launcher/kitty --config NONE \
#   -o confirm_os_window_close=0 python3 /tmp/kitty_obs/fill_writebuf.py 7 /tmp/kitty_obs/F_prog_r1.txt \
#   2>&1 | awk -f /tmp/kitty_obs/f_capture.awk      # rolling 8-line pre-context; window + total count
```

Run 1 — complete captured window (1 pre-context line + 31 identical overflow lines) and total:

```
[0.158] Failed to open systemd user bus with error: Connection refused
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
[5.536] Too much data being sent to child with id: 1, ignoring it
TOTAL_OVERFLOW_OCCURRENCES=1107992
```

(Run 1 child made `FINAL 36962304 7.507` — i.e. the child kept running to the end of the 7 s window; it was **paced, not killed**.)

Run 2 — first-crossing line + total (stability, F7): `[6.708] Too much data being sent to child with id: 1, ignoring it` … `TOTAL_OVERFLOW_OCCURRENCES=321546` (child `FINAL 33816576 7.160`).

**Reading the evidence:** the **visible** overflow log (`kitty/child-monitor.c:342`) first fires at **t≈5.5–6.7 s** and then repeats **hundreds of thousands to over a million** times before the 7 s window closes (Run 1: 1,107,992; Run 2: 321,546) — with **no rate-limiting** (every discarded enqueue logs). The message reads `id: 1` (the first child). This log appears on the **canonical** build (no debug macros — the pre-context line is the ordinary `systemd user bus` warning, *not* an event-loop trace), confirming it is canonical, not debug-gated.

> **Portability note (magnitude vs. mechanism).** The **absolute occurrence count and the sub-second first-crossing timestamp are load-sensitive, host-dependent magnitudes** that vary substantially **even run-to-run on the same host**: the two canonical runs above differ by **~3.4×** (321,546 vs 1,107,992). The reason is arithmetic, not instability of the mechanism — the total ≈ (overflow rate) × (7 s window − first-crossing time), and the first crossing itself drifts with scheduling (5.536 s vs 6.708 s here), so Run 1 had ~1.46 s of post-crossing logging while Run 2 had only ~0.29 s. What is **invariant** — and what this evidence establishes — is the *mechanism*: the strict `>` 100 MiB cap (`kitty/child-monitor.c:341`), the exact log string, the per-discarded-enqueue logging with **no rate-limiting**, and the child being **paced, not killed** (it always reaches `FINAL`). Treat the count/timing as environment characteristics, not fixed constants. (An earlier draft reported ~845 k occurrences with a tight two-run spread; that was measured on a *mixed* build whose `glfw-x11.so` still carried the `DEBUG_EVENT_LOOP` trace — its per-tick logging both slowed the main loop and made the count artificially repeatable. The figures here are from the fully canonical artifact set of §1.1.)

> **Operational safety (F21).** This experiment is intentionally abusive: an earlier ~16 s probe produced **6,875,319** overflow lines totalling **458 MB** in a single log. Reproduce it **only** in a disposable, resource-bounded environment, and always bound it (short `timeout`, and a streaming counter/rolling-window capture as above) — never redirect the raw stream to disk unbounded.

**F — slow-drain (observed, instrumented debug build; two runs).** To watch draining, the child enqueues 200,000 DA2 replies (2.6 MiB) then reads 4 KiB every 60 ms. Counts are of **returned** `revents` (what `poll()` returned) and of per-`write()` sizes (`KITTY_PRINT_BYTES_SENT_TO_CHILD`), tallied with occurrence counters over the whole run (a count, not the raw stream). Both runs drained the identical byte total; the diagnostic counts are instrumented-build magnitudes that vary run-to-run and are reported for both runs:

```
child bytes actually drained (FINAL): read_total 2600000    # = 200000 x 13  (BOTH runs, INVARIANT)
returned revents (DEBUG_POLL_EVENTS, kitty/child-monitor.c:1550-1556):        run1     run2
  i:0 POLLIN   (wakeup fd)                                                   18276    35957
  i:1 POLLIN   (signal fd)                                                       1        1
  i:2 POLLIN   (the query burst)                                               122       94
  i:2 POLLOUT  (child fd RETURNED writable)                                     630      632
  i:2 POLLHUP  (child exit)                                                       1        1
per-write() drain (Wrote: N bytes): positive writes = 631 / 638 ; sum = 2600000 (both) ; EAGAIN(Wrote:-1) = 555 / 555
most frequent 'Wrote:' sizes are PTY-granularity chunks (multiples of 512), NOT reply-aligned:
  3584 (=512x7)   5632 (=512x11)   7680 (=512x15)    [556/631 = 88.1% and 561/638 = 87.9% are exact x512]
  only 74 / 76 of the 631 / 638 positive writes are whole multiples of the 13-byte reply
```

The child drained **exactly 2,600,000 bytes** (= 200,000 × 13) in both runs. The drain is **`POLLOUT`-gated**: the count of returned-writable events (`i:2 POLLOUT` = 630 / 632) matches the count of positive `write()`s (631 / 638) almost one-to-one. Crucially the `write()` sizes are **not** reply-aligned — **~88%** are exact multiples of **512** (the kernel PTY chunk granularity: 3584, 5632, 7680), so most drains stop **mid-reply**; each time the PTY then fills, the very next `write()` returns `EAGAIN` (**555 times per run**), taking the retain-and-retry branch (`kitty/child-monitor.c:1463`). (Correction: an earlier draft asserted the sizes were "all whole multiples of 13" and equated `i:0 POLLIN` with `i:2 POLLOUT` at ~12,511 each. Both were single-run mis-tallies. The authoritative two-run occurrence tally above shows the opposite — the sizes are overwhelmingly 512-aligned mid-reply cuts, `i:2 POLLOUT` is ~630 (≈ the number of drain writes), and `i:0 POLLIN` is a much larger, run-variable busy-wakeup counter that must not be conflated with it. The mid-reply cut is shown byte-for-byte in the EAGAIN capture below.)

> **Observability correction (F6).** `DEBUG_POLL_EVENTS` prints `children_fds[i].revents & w` (`kitty/child-monitor.c:1553`) — the **returned readiness**. This is **distinct** from the `POLLOUT` **request** `events |= POLLOUT` (`:1503`, set only while `write_buf_used > 0`). The `i:2 POLLOUT` tallies above are *returned-writable* events, not request counts, and positive `Wrote:` samples do not by themselves establish an `EAGAIN` frequency (which is why the EAGAIN branch is shown directly below).

**F — EAGAIN branch (observed, instrumented debug build).** The child enqueues ~200,000 DA2 queries, sleeps 1.5 s reading nothing (so `write_buf` builds while the fd is unwritable), then does **one** large read to free PTY space at once. Child body (inline):

```python
# /tmp/kitty_obs/fill_eagain.py
import os, sys, time, termios, tty
tty.setraw(0)
prog = sys.argv[1]
os.write(1, b'\x1b[>c' * 200000)   # enqueue ~2.6 MiB of replies into write_buf
time.sleep(1.5)                     # DO NOT read: write_buf builds; POLLOUT requested but fd unwritable
data = os.read(0, 1 << 20)          # free a big chunk at once -> kitty writes until EAGAIN
open(prog, 'w').write("first_read %d\n" % len(data))
time.sleep(0.6)
```

```
# DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 10 kitty/launcher/kitty --config NONE \
#   -o confirm_os_window_close=0 python3 /tmp/kitty_obs/fill_eagain.py /tmp/kitty_obs/eagain_prog.txt 2>/tmp/kitty_obs/eagain_stderr.log
# child result file:  first_read 4095
```

Across two runs the child freed all queued bytes in a bounded number of writes — **78** entries in run 1 and **14** in run 2 (the count depends on how the single large `read` lets kitty chunk the drain; `first_read 4095` was identical both runs). In each run **all positive writes but the final one are whole multiples of the 13-byte reply** (e.g. `156 = 12×13`, `247 = 19×13`), and the run ends with the **EAGAIN transition**: the final positive write is cut **mid-reply**, immediately followed by `Wrote: -1 bytes:`. Because the `EWOULDBLOCK/EAGAIN` `break` (`kitty/child-monitor.c:1463`) skips `print_text`, the `Wrote: -1 bytes:` line has **no trailing content and no newline before it**. In run 1 that final cut write was exactly `Wrote: 5632 bytes` (run 2's cuts were `11776` and `3584`). Complete, unedited run-1 transition tail (the end of the `Wrote: 5632 bytes` payload directly followed by the `Wrote: -1 bytes` line):

```
[>1;4000;35c\x1b[>1;4000;35c\x1b[>1;4000;35c\x1b[>
Wrote: -1 bytes: State check timer firedProcessing global statei
```

**Cause→effect:** `Wrote: 5632 bytes` is a partial drain that stops **mid-reply** — `5632 = 433×13 + 3`, i.e. 433 complete `\x1b[>1;4000;35c` replies (5629 bytes) plus a **3-byte fragment** `\x1b[>` of the 434th (visible as the truncated tail above) — because the kernel PTY buffer filled exactly there. The **next** `write()` then returns `-1` with `errno == EAGAIN`, so kitty **breaks and retains** the remaining queued bytes for the next `POLLOUT` (`:1463`). No data is lost on transient pressure — only the 100 MiB cap (§3.1) discards. (This mid-reply cut also refines the slow-drain observation above: `write()` returns whatever the kernel accepts, which is often but **not necessarily** a whole reply.)

### 3.4 Graphics responses use this SAME path (not privileged)

A graphics reply is dispatched as an APC sequence onto the shared output buffer:

```c
// kitty/screen.c:1050
if (response != NULL) write_escape_code_to_child(self, ESC_APC, response);
```

`write_escape_code_to_child` (`kitty/screen.c:979`) prepends the APC introducer (`"\033_"`, `:971`) and calls `schedule_write_to_child`, so a graphics `OK`/error reply is subject to the **same** 100 MiB cap and `POLLOUT` draining as any other output. There is **no** graphics-specific write prioritization. (Evidence F uses DA2 replies, which travel this identical path.)

---

## 4. Graphics-specific flow control & responses

The graphics subsystem adds **memory** protections (bounding what is stored) and a **response** lever (bounding what is replied), on top of the generic I/O machinery above.

### 4.1 320 MiB storage quota + LRU eviction

```c
// kitty/graphics.c:25
#define DEFAULT_STORAGE_LIMIT 320u * (1024u * 1024u)
// kitty/graphics.c:78
self->storage_limit = DEFAULT_STORAGE_LIMIT;
```

```c
// kitty/graphics.c:290-299  apply_storage_quota (invoked at :2184)
apply_storage_quota(GraphicsManager *self, size_t storage_limit, id_type currently_added_image_internal_id) {
    // First remove unreferenced images, even if they have an id
    remove_images(self, trim_predicate, currently_added_image_internal_id);
    if (self->used_storage < storage_limit) return;
    HASH_SORT(self->images, oldest_img_first);
    while (self->used_storage > storage_limit && self->images) {
        remove_image(self, self->images);
    }
```

**Evidence B2 ([non-canonical in-process]; `storage_limit` reduced to 72 bytes so eviction is observable without transmitting 320 MiB — same code path as the 320 MiB default).** Complete harness output (unedited):

```
CANONICAL default storage_limit = 335544320 bytes (= 320 MiB)
[NON-CANONICAL] reduce storage_limit to observe eviction without transmitting 320 MiB
reduced storage_limit = 72 bytes (holds two 36-byte images)

=== B2: storage quota + LRU eviction ===
BEFORE:            image_count=0  total_size=0
transmit i=1 -> OK     image_count=1  total_size=36
transmit i=2 -> OK     image_count=2  total_size=72
transmit i=3 -> OK     image_count=2  total_size=72  (limit reached: oldest evicted)
  display i=1 -> ENOENT
  display i=2 -> OK
  display i=3 -> OK
```

**Reading the evidence (before/during/after):** count climbs 0→1→2 then **stays 2** at `total_size=72` when i=3 is added. Which image was evicted is proven by attempting to display each: **i=1 → `ENOENT`** (evicted, oldest-first per `oldest_img_first`, `:295`), i=2/i=3 → `OK`. This is the **LRU** eviction of `apply_storage_quota` (`:290-299`, invoked `:2184`) — a **silent** adaptation (no log; only a later `ENOENT` if the evicted image is referenced).

**Evidence B2c (observed, canonical, real PTY, DEFAULT 320 MiB limit).** To confirm the *same* quota fires at the real default (not a reduced limit), a child in raw mode transmits four 100 MB RGBA images (5000×5000×4 = 100,000,000 bytes each ⇒ ~382 MiB total > 320 MiB) via the file medium (`t=f`), then probes which ids survive with `a=p`. Two variants isolate the eviction ordering; each was byte-identical across two runs (`diff run1 run2` → no differences).

Variant 1 — **unreferenced** transmits (`a=t`, images not placed):

```
# DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 60 kitty/launcher/kitty \
#   --config NONE -o confirm_os_window_close=0 python3 /tmp/kitty_obs/storage_canonical.py OUT
transmit i=1 -> b'\x1b_Gi=1;OK\x1b\\'
transmit i=2 -> b'\x1b_Gi=2;OK\x1b\\'
transmit i=3 -> b'\x1b_Gi=3;OK\x1b\\'
transmit i=4 -> b'\x1b_Gi=4;OK\x1b\\'
query(put) i=1 -> b'\x1b_Gi=1;ENOENT:Put command refers to non-existent image with id: 1 and number: 0\x1b\\'
query(put) i=2 -> b'\x1b_Gi=2;ENOENT:Put command refers to non-existent image with id: 2 and number: 0\x1b\\'
query(put) i=3 -> b'\x1b_Gi=3;ENOENT:Put command refers to non-existent image with id: 3 and number: 0\x1b\\'
query(put) i=4 -> b'\x1b_Gi=4;OK\x1b\\'
```

Variant 2 — **placed/referenced** transmits (`a=T`, images displayed):

```
# DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 120 kitty/launcher/kitty \
#   --config NONE -o confirm_os_window_close=0 python3 /tmp/kitty_obs/lru_canonical.py OUT
transmit+display i=1 -> b'\x1b_Gi=1;OK\x1b\\'
transmit+display i=2 -> b'\x1b_Gi=2;OK\x1b\\'
transmit+display i=3 -> b'\x1b_Gi=3;OK\x1b\\'
transmit+display i=4 -> b'\x1b_Gi=4;OK\x1b\\'
query(put) i=1 -> b'\x1b_Gi=1;ENOENT:Put command refers to non-existent image with id: 1 and number: 0\x1b\\'
query(put) i=2 -> b'\x1b_Gi=2;OK\x1b\\'
query(put) i=3 -> b'\x1b_Gi=3;OK\x1b\\'
query(put) i=4 -> b'\x1b_Gi=4;OK\x1b\\'
```

**Reading the evidence:** every transmit is accepted (`OK`) — the quota never rejects an *incoming* image; instead `apply_storage_quota` (`:290-299`) silently evicts already-stored images to make room. In **Variant 1** the four **unreferenced** images (`a=t`) are all evictable, so once the fourth pushes total past 320 MiB the three oldest are reclaimed and only **i=4 (newest) survives**. In **Variant 2** the images are **placed** (`a=T`): `remove_images(..., trim_predicate, ...)` (`:292`) first drops only *unreferenced* entries, then `HASH_SORT(oldest_img_first)` (`:295`) evicts strictly oldest-first until under quota — leaving **i=1 (oldest) evicted** and **i=2/i=3/i=4 alive**. Both variants confirm the **default 320 MiB** ceiling and the **oldest-first** ordering *canonically*, matching the reduced-limit Evidence B2 above. The eviction itself is **silent**; its only visible sign is the later `ENOENT` when an evicted id is referenced.

### 4.2 Animation frame cache — 5× quota, reclaim → recheck → ENOSPC

```c
// kitty/graphics.c:1570-1573
    if (is_new_frame && cache_size(self) + load_data->data_sz > self->storage_limit * 5) {
        remove_images(self, trim_predicate, img->internal_id);
        if (cache_size(self) + load_data->data_sz > self->storage_limit * 5)
            ABRT("ENOSPC", "Cache size exceeded cannot add new frames");
```

**Evidence B3 ([non-canonical in-process]).** Complete harness output (unedited):

```
=== B3: frame-cache 5x quota (reclaim -> recheck -> ENOSPC) ===
frame-cache limit = storage_limit*5 = 360 bytes [graphics.c:1570]
base image i=2 exists (image_count=2). Adding 36-byte animation frames (a='f' default):
  add frame 0 -> OK     total_size=108
  add frame 1 -> OK     total_size=144
  add frame 2 -> OK     total_size=180
  add frame 3 -> OK     total_size=216
  add frame 4 -> OK     total_size=252
  add frame 5 -> OK     total_size=288
  add frame 6 -> OK     total_size=324
  add frame 7 -> OK     total_size=360
  add extra frame -> ENOSPC total_size=360  (cache full even after reclaim -> ENOSPC [graphics.c:1571-1573])
  edit existing frame (r=2,s=2,v=2) -> OK     (editing does not trigger quota)
```

**Reading the evidence (F12):** for a **new** frame, kitty first attempts a **reclaim** of unreferenced images (`remove_images(...trim_predicate..., img->internal_id)`, `:1571`); it then **re-checks** the ceiling (`:1572`) and only then raises `ENOSPC` "Cache size exceeded cannot add new frames" (`:1573`). Here the reclaim frees nothing (all frames are referenced by base image i=2), so the recheck still exceeds `storage_limit*5` → `ENOSPC` (a **visible** error reply). **Editing an existing frame** is gated by `is_new_frame` (`:1570`) and therefore bypasses the quota (`OK`).

### 4.3 Per-image size / format guards (correct locations)

Each guard is a distinct `ABRT(...)` (the `ABRT` macro at `kitty/graphics.c:519` sets a failure response, frees the load data, and returns `NULL`).

**Evidence B4 ([non-canonical in-process], fresh screen).** Complete harness output (unedited):

```
=== B4: per-image size/format guards (fresh screen; in-process NON-CANONICAL) ===
zero-dim  s=0 v=1 i=1        [:646]        RESP=b'\x1b_Gi=1;EINVAL:Zero width/height not allowed\x1b\\'
unknown-format f=99 i=1      [:651]        RESP=b'\x1b_Gi=1;EINVAL:Unknown image format: 99\x1b\\'
PNG-size  S=400000001 f=100 i=1 [:638]     RESP=b'\x1b_Gi=1;EINVAL:PNG data size too large\x1b\\'
```

Mapping to source (F10 — these lines were previously misattributed):
- **Zero width/height** → `EINVAL:Zero width/height not allowed`, `kitty/graphics.c:646`.
- **Unknown format** → `EINVAL:Unknown image format: %u`, `:651`.
- **Declared PNG size too large** → `EINVAL:PNG data size too large`, `:638` (triggered by the declared `S=` exceeding `MAX_DATA_SZ = 4*100000000 = 400 MB`, `:521` — no 400 MB payload needed).
- **Over-dimension** → `EINVAL:Image too large`, `:695` (guard is inside `if (init_img)`; `MAX_IMAGE_DIMENSION = 10000`, `:674`). The animation-frame path has a **sibling** dimension guard at `:1553`.
- **Raw/direct `d` over-budget** → `EFBIG:Too much data`, `:533`. For a **direct**, **non-PNG** (`RGB`/`RGBA`) transmission kitty first allocates a load buffer of exactly **`buf_capacity = data_sz + 10`** bytes (`data_sz = width·height·(bits_per_pixel/8)`; the `+10` is compression-header slack — `+1024` when `o=z`), `kitty/graphics.c:656`. The abort fires the instant an incoming chunk would push the accumulated payload past that capacity — `buf_capacity - buf_used < payload_sz` (`:532`) combined with `data_fmt != PNG` (`:533`). So payloads **up to and including `data_sz + 10`** are accepted and the **first-failing total is `data_sz + 11`** — the boundary is the per-image capacity, **not** `MAX_DATA_SZ`. For the largest image that still clears the `MAX_IMAGE_DIMENSION = 10000` guard (`:674`, `:695`; `10000` itself passes, being not *greater than* `10000`), a `10000×10000` `RGBA` image has `data_sz = 10000·10000·4 = 400,000,000` (numerically equal to `MAX_DATA_SZ`, `:521`), giving `buf_capacity = 400,000,010` **accepted** and a **first-failing byte count of `400,000,011`**. This is **observed(canonical)** at reduced scale in **E-EFBIG** below; the `400,000,011` figure then follows from the identical code path by arithmetic.

**Over-dimension response is state-dependent (F11).** The transmit reply is built from `lg = &self->currently_loading.start_command` (`kitty/graphics.c:2177`, used at `:2180`), **not** from the command's own `g`. The over-dimension `ABRT` at `:695` fires **before** `initialize_load_data` sets `start_command = *g` (`:634`, which also sets `start_command.id`, `:717`), and `free_load_data` (`:103`) does **not** clear `start_command`. So the id on an over-dimension reply is whatever the **last** command that reached `:634` left behind. Complete harness output, each case a **fresh process**; the two runs were **byte-identical** (verified with `diff`, modulo host-dependent `[t]` timestamps):

```
A fresh over-dim i=1 ONLY cmd            -> b''   (start_command.id=0 -> :765 gate -> empty)
C zero-dim i=1 (fails AFTER :634 sets start_command.id=1) -> b'\x1b_Gi=1;EINVAL:Zero width/height not allowed\x1b\\'
C then over-dim i=1 (ABRT :695 before :634; reuses start_command.id=1) -> b'\x1b_Gi=1;EINVAL:Image too large\x1b\\'
E zero-dim i=9 (sets start_command.id=9)  -> b'\x1b_Gi=9;EINVAL:Zero width/height not allowed\x1b\\'
E then over-dim NO id -> id LEAKS from start_command: b'\x1b_Gi=9;EINVAL:Image too large\x1b\\'
```

**Reading the evidence:** on a **fresh** screen an over-dimension-only command returns an **empty** response (`b''`) — the `ABRT` fires before any id is set, so `finish_command_response` sees `g->id == 0 && g->image_number == 0` and emits nothing (`:765`). But after a **prior** command left a `start_command.id` (Case C: id 1; Case E: id 9 — note the over-dimension command in E carried **no** id yet still replied with `i=9`), the over-dimension reply **inherits that stale id** and is **visible** (`EINVAL:Image too large`). The empty-response result is therefore **only** valid in a fresh, isolated harness. (The `ImportError: sys.meta_path is None` printed after each case is benign Python interpreter-shutdown noise, not part of the observation.)

**Evidence B4c (observed, canonical, real PTY, default limits).** The same four guards fire identically when driven through a live PTY at the default configuration — and the over-dimension reply exhibits the F11 stale-id leak *canonically*. A child in raw mode sends the four guard-tripping commands in one session (`i=1` zero-dim, `i=2` unknown-format, `i=3` PNG-size, `i=4` over-dimension). Complete unedited output, byte-identical across two runs (`diff run1 run2` → no differences):

```
# DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 20 kitty/launcher/kitty \
#   --config NONE -o confirm_os_window_close=0 python3 /tmp/kitty_obs/graphics_guards.py OUT
zero-dim s=0,v=1 i=1                     -> b'\x1b_Gi=1;EINVAL:Zero width/height not allowed\x1b\\'
unknown-format f=99 i=2                  -> b'\x1b_Gi=2;EINVAL:Unknown image format: 99\x1b\\'
PNG-size S=400000001 f=100 i=3           -> b'\x1b_Gi=3;EINVAL:PNG data size too large\x1b\\'
over-dim s=20000,v=20000 i=4 (FRESH)     -> b'\x1b_Gi=3;EINVAL:Image too large\x1b\\'
```

**Reading the evidence:** the first three guards reply with the *same* `EINVAL` strings as the in-process Evidence B4 — `Zero width/height not allowed` (`:646`), `Unknown image format: 99` (`:651`), `PNG data size too large` (`:638`) — confirming those guards *canonically* at the default limits. The fourth command carries **`i=4`** but its over-dimension reply comes back as **`i=3`** (`EINVAL:Image too large`, `:695`): the `ABRT` at `:695` fires *before* `initialize_load_data` sets `start_command = *g` (`:634`), so the reply reuses the stale `start_command.id` left by the previous command (the PNG-size `i=3`). This is the **F11 stale-id leak observed canonically** — the same mechanism shown in-process above (Case E, `i=9`), now confirmed through a real PTY.

**Evidence E-EFBIG (observed, canonical, real PTY).** A child in raw mode makes three **direct** `RGBA` transmits of a **2×2** image (`data_sz = 2·2·4 = 16`, so `buf_capacity = data_sz + 10 = 26`, `graphics.c:656`), varying only the decoded payload size around that capacity. Complete captured responses (unedited), identical across two runs:

```
capacity(data_sz+10)=26 (payload=26 B) -> b'\x1b_Gi=1;OK\x1b\\'
first-failing(data_sz+11)=27 (payload=27 B) -> b'\x1b_Gi=2;EFBIG:Too much data\x1b\\'
well-over=40 (payload=40 B) -> b'\x1b_Gi=3;EFBIG:Too much data\x1b\\'
```

**Reading the evidence:** a payload of exactly `data_sz + 10` (= 26 B) is **accepted** (`OK`); the first byte beyond capacity, `data_sz + 11` (= 27 B), trips `buf_capacity - buf_used < payload_sz` (`:532`) and — because `data_fmt != PNG` — raises `EFBIG:Too much data` (`:533`); 40 B fails identically. This exercises the same `:532`/`:533`/`:656` path that governs the `10000×10000` `RGBA` case, so its `data_sz = 400,000,000` yields `buf_capacity = 400,000,010` (accepted) and a first-failing total of `400,000,011`. Generator: `python3 /tmp/kitty_obs/graphics_efbig.py` (full body in the harness appendix, §10) under a canonically-built kitty (`kitty/fast_data_types.so` = 1,253,792 bytes), driven through a real PTY. The `EFBIG` guard is therefore **observed(canonical)** (mechanism and boundary), not inferred; only the literal `400,000,011`-byte transmission was not performed (it is the identical path at full scale).

### 4.4 Response suppression — the `q=` key (client-side back-channel relief)

```c
// kitty/graphics.c:759-768  finish_command_response
finish_command_response(const GraphicsCommand *g, bool data_loaded) {
    static char rbuf[sizeof(command_response)/sizeof(command_response[0]) + 128];
    bool is_ok_response = !command_response[0];
    if (g->quiet) {
        if (is_ok_response || g->quiet > 1) return NULL;
    }
    if (g->id || g->image_number) {
        if (is_ok_response) {
            if (!data_loaded) return NULL;
            snprintf(command_response, 10, "OK");
```

The reply is built in a fixed `static char command_response[512]` (`kitty/graphics.c:302`). **Evidence B1 (observed, canonical, real PTY).** A child in raw mode transmits success (valid 1×1 RGB) and error (zero-width) commands with `q=0/1/2` and reads the APC replies. Complete captured responses (unedited); the two runs were **byte-identical** (verified with `diff run1 run2` → no differences):

```
CASE success_q0  SENT=b'\x1b_Ga=t,f=24,s=1,v=1,i=1,q=0;/wAA\x1b\\'
              RESP=b'\x1b_Gi=1;OK\x1b\\'
CASE error_q0    SENT=b'\x1b_Ga=t,f=24,s=0,v=1,i=1,q=0;/wAA\x1b\\'
              RESP=b'\x1b_Gi=1;EINVAL:Zero width/height not allowed\x1b\\'
CASE success_q1  SENT=b'\x1b_Ga=t,f=24,s=1,v=1,i=1,q=1;/wAA\x1b\\'
              RESP=b''
CASE error_q1    SENT=b'\x1b_Ga=t,f=24,s=0,v=1,i=1,q=1;/wAA\x1b\\'
              RESP=b'\x1b_Gi=1;EINVAL:Zero width/height not allowed\x1b\\'
CASE success_q2  SENT=b'\x1b_Ga=t,f=24,s=1,v=1,i=1,q=2;/wAA\x1b\\'
              RESP=b''
CASE error_q2    SENT=b'\x1b_Ga=t,f=24,s=0,v=1,i=1,q=2;/wAA\x1b\\'
              RESP=b''
```

**Reading the evidence:** `q=0` sends both `OK` and errors; **`q=1` suppresses `OK` but still sends errors**; `q=2` suppresses both — exactly matching `graphics.c:762-764` (`quiet`: suppress if `is_ok_response || quiet > 1`). This is the client's lever to relieve the **shared** 100 MiB-capped `write_buf`; it is a **silent** adaptation (the client asked for it; nothing is logged). Corroborated by `docs/graphics-protocol.rst:762-768` (the "Suppressing responses from the terminal" subsection; the `q` key sentences at `:767-768`) and `:975-984` (the "Image persistence and storage quotas" subsection; "320MB per buffer" at `:980`).

---

## 5. Synchronized-update "pending mode" render pause

A distinct, application-driven pause: the synchronized-update private mode `2026` (`PENDING_MODE`, `kitty/control-codes.h:235`) holds rendering while an app assembles a frame. `DECSET ?2026h` / `DECRST ?2026l` are handled here:

```c
// kitty/screen.c:1174-1177
        case PENDING_MODE << 5:
            if (!screen_pause_rendering(self, val, 0)) {
                log_error("Pending mode change to already current mode (%d) requested. Either pending mode expired or there is an application bug.", val);
            }
```

```c
// kitty/screen.c:2506-2522  screen_pause_rendering (excerpt of the pause path)
    if (self->paused_rendering.expires_at) return false;          // already paused -> caller logs (:1176)
    if (!self->paused_rendering.grman) self->paused_rendering.grman = grman_alloc(true);
    if (!self->paused_rendering.grman) return false;
    if (for_in_ms <= 0) for_in_ms = 2000;                          // DEFAULT 2000 ms timeout
    self->paused_rendering.expires_at = monotonic() + ms_to_monotonic_t(for_in_ms);
```

```c
// kitty/screen.c:2489-2490  auto-unpause on timeout (called from the render loop)
screen_check_pause_rendering(Screen *self, monotonic_t now) {
    if (self->paused_rendering.expires_at && now > self->paused_rendering.expires_at) screen_pause_rendering(self, false, 0);
```

The DECSET path passes `for_in_ms = 0` (`:1175`), so the default timeout is **2000 ms** (`:2521`). `screen_check_pause_rendering` is called from the render loop (`kitty/child-monitor.c:729`), so the timeout auto-unpause fires at the next render pass once `now > expires_at`. Pending state is queryable via DECRQM `ESC [ ? 2026 $ p`, which reports `1` iff `paused_rendering.expires_at` is set (`report_mode_status`, `PENDING_UPDATE` case, `kitty/screen.c:2237-2238`) → `ESC[?2026;1$y` (active) or `;2$y` (inactive).

**Evidence C — part 1: state transitions + "already current mode" ([non-canonical in-process]; real VT parser + `Screen` state machine, no live PTY; canonical DECSET/DECRST/DECRQM control codes).** Complete interleaved stdout+stderr, identical across two runs:

```
STEP 0  initial query 2026            -> b'\x1b[?2026;2$y'
STEP 1  after DECSET 2026h; query     -> b'\x1b[?2026;1$y'
STEP 2  DECSET 2026h AGAIN (expect 'already current mode' log on stderr):
[0.020] Pending mode change to already current mode (1) requested. Either pending mode expired or there is an application bug.
STEP 2  after 2nd DECSET; query       -> b'\x1b[?2026;1$y'
STEP 3  after DECRST 2026l; query     -> b'\x1b[?2026;2$y'
STEP 4  pause_rendering() 1st call    -> True
STEP 4  pause_rendering() 2nd call    -> False
STEP 4  query after hook-pause        -> b'\x1b[?2026;1$y'
```

**Reading the evidence (before/during/after):** query returns `;2$y` (inactive) → after `DECSET 2026h` it is `;1$y` (paused) → a **second** `DECSET 2026h` while still paused makes `screen_pause_rendering` return false, logging the **visible** `Pending mode change to already current mode (1)...` (`kitty/screen.c:1176`) → after `DECRST 2026l` it is `;2$y` again (explicit unpause). STEP 4 uses the test-only `pause_rendering()` hook ([non-canonical]) purely to corroborate the "return false when already paused" branch (`:2518`).

**Evidence C — part 2: the 2000 ms timeout auto-unpause (observed, canonical, real PTY).** A child sets `DECSET 2026`, forces a render pass at each checkpoint, and queries pending state, measuring elapsed from the DECSET moment. Complete progress log, identical across two runs A and B:

```
# DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 12 kitty/launcher/kitty --config NONE \
#   -o confirm_os_window_close=0 python3 /tmp/kitty_obs/pending_child.py
# DECSET 2026 pause; default timeout=2000ms (screen.c:2521). dt measured from DECSET emit.
C0-before-set  dt_from_pause=+0.000 resp=b'\x1b[?2026;2$y'
C1             dt_from_pause=+0.500 resp=b'\x1b[?2026;1$y'
C2-pre-2s      dt_from_pause=+1.700 resp=b'\x1b[?2026;1$y'
C3-post-2s     dt_from_pause=+2.600 resp=b'\x1b[?2026;2$y'
C4-post-2s     dt_from_pause=+3.000 resp=b'\x1b[?2026;2$y'
```

**Reading the evidence:** before the DECSET the mode is inactive (`;2$y`); it is **paused** (`;1$y`) at +0.500 s and still at +1.700 s (before the 2000 ms timeout); it has **auto-unpaused** (`;2$y`) by +2.600 s and remains so at +3.000 s. This brackets the **2000 ms default timeout** (`kitty/screen.c:2521`), enforced by the render-loop `screen_check_pause_rendering` (`kitty/child-monitor.c:729` → `kitty/screen.c:2490`). The auto-unpause is **silent** (no log — contrast the redundant-DECSET log in part 1, which is visible).

**Cause→effect:** pending mode is a **rendering** throttle (it coalesces mid-frame updates into one repaint); it does **not** change read/write buffering. Its only visible sign is the redundant/late-DECSET `log_error` (`:1176`); the pause itself and its 2000 ms expiry are silent.

---

## 6. Where in the code — reference map (`file:line`)

**Inbound (read) path — `kitty/vt-parser.c`, `kitty/child-monitor.c`:**
- `kitty/vt-parser.c:18` — `#define BUF_SZ (1024u*1024u)` — fixed **1 MiB** parser buffer (BUFFER).
- `kitty/vt-parser.c:21` — `#define MAX_ESCAPE_CODE_LENGTH (BUF_SZ / 4u)` — 256 KiB per escape code; `:22` — `MAX_CSI_PARAMS 256u`.
- `kitty/vt-parser.c:394-420` — `accumulate_st_terminated_esc_code`: complete-ST generous accept (`:398-404`), OSC-52 partial-dispatch/continue (`:407-417`), over-length `REPORT_ERROR` (`:419`); `:830-832` — separate CSI byte-length truncation; `:719-721` — CSI param-count guard; `:125` — `REPORT_ERROR`→`log_error`.
- `kitty/vt-parser.c:1417` — `run_worker`; `:1425` — force-parse condition (THROTTLE).
- `kitty/vt-parser.c:1451` — `vt_parser_create_write_buffer`; `:1477-1484` — `vt_parser_has_space_for_input` (`read.sz + write.pending < BUF_SZ`).
- `kitty/child-monitor.c:35` — `#define EXTRA_FDS 2`.
- `kitty/child-monitor.c:1501` — `events = vt_parser_has_space_for_input(...) ? POLLIN : 0;` (PAUSE → OS PTY backpressure).
- `kitty/child-monitor.c:291` / `:1481` — `io_loop` thread create / entry; `:1506-1509` timed `poll()` (`input_delay`), `:1512` blocking `poll()`.
- `kitty/child-monitor.c:437-446` — `do_parse` `set_maximum_wait(OPT(input_delay)…)` (`:445`); `:114-118` — `set_maximum_wait`; `:1255` — `update_main_loop_timer`; `:869-878` — `render()` repaint bypass (`:875-878`).
- `glfw/backend_utils.c:295-299` — `pollForEvents` (`EVDBG "pollForEvents final timeout: %.3f"` at `:298`); `glfw/internal.h:837-838` — `EVDBG` gated by `DEBUG_EVENT_LOOP`.

**Outbound (write-to-child) path — `kitty/child-monitor.c`:**
- `kitty/child-monitor.c:341-342` — 100 MiB `write_buf` cap (strict `>`) + `log_error("Too much data being sent to child with id: %lu, ignoring it", id)` (VISIBLE) + discard.
- `kitty/child-monitor.c:1503` — `events |= (screen->write_buf_used ? POLLOUT : 0);` (request POLLOUT only when queued).
- `kitty/child-monitor.c:1447-1466` — `write_to_child`: `ret>0` advance (`:1452-1456`), `ret==0` break (`:1458-1460`), `EINTR` continue (`:1462`), `EWOULDBLOCK/EAGAIN` break/retain (`:1463`), `perror(...)`+discard (`:1464-1465`).
- `kitty/child-monitor.c:1449-1451` — `KITTY_PRINT_BYTES_SENT_TO_CHILD` `Wrote:` (compile-time); `:1550-1556` — `DEBUG_POLL_EVENTS` per-fd `revents` (compile-time; reads `.revents`).

**Graphics — `kitty/graphics.c`, `kitty/screen.c`:**
- `kitty/graphics.c:25`, `:78` — `DEFAULT_STORAGE_LIMIT` 320 MiB → `storage_limit`; `:103` — `free_load_data` (does **not** clear `start_command`).
- `kitty/graphics.c:290-299` (invoked `:2184`) — `apply_storage_quota`: reclaim (`:292`) then LRU `oldest_img_first` (`:295`).
- `kitty/graphics.c:302` — `static char command_response[512]`; `:519` — `ABRT` macro; `:521` — `MAX_DATA_SZ` (400 MB); `:532` — direct-buffer capacity check (`buf_capacity - buf_used < payload_sz`); `:533` — raw `EFBIG:Too much data`; `:638` — PNG `EINVAL`; `:646` — zero-dim `EINVAL`; `:651` — unknown-format `EINVAL`; `:656` — direct load `buf_capacity = data_sz + (compressed ? 1024 : 10)`; `:674` — `MAX_IMAGE_DIMENSION 10000`; `:695` — over-dim `EINVAL:Image too large`; `:1553` — animation-frame sibling dimension guard.
- `kitty/graphics.c:631-634` — `initialize_load_data` sets `start_command = *g`; `:717` — `start_command.id = iid`.
- `kitty/graphics.c:759` — `finish_command_response`; `:762-764` — `q=` suppression; `:765` — needs `g->id || g->image_number`; `:2177`/`:2180` — transmit reply uses `lg = &start_command`.
- `kitty/graphics.c:1570-1573` — frame cache `storage_limit*5`: reclaim (`:1571`) → recheck (`:1572`) → `ENOSPC` (`:1573`); `:2197-2198` — `ENOENT` on an animation command for a missing image (`:2199` starts the `else`).
- `kitty/screen.c:970` `ESC_APC` case, `:971` prefix `"\033_"`; `:979` `write_escape_code_to_child`; `:1050` graphics reply onto shared output path; `:2128` DA2 reply (`>1;PRIMARY;SECONDARYc`).
- `kitty/screen.c:1174-1176` pending-mode already-current `log_error`; `:2237-2238` DECRQM `PENDING_UPDATE` status; `:2489-2490` auto-unpause; `:2506` `screen_pause_rendering`; `:2521` default 2000 ms; `kitty/child-monitor.c:729` render-loop `screen_check_pause_rendering` caller.

**Config defaults, constants, docs, tests:**
- `kitty/options/definition.py:866` `repaint_delay '10'`; `:878` `input_delay '3'`. `kitty/options/types.py:536` `input_delay=3`, `:567` `repaint_delay=10`, `:586` `sync_to_monitor=True`.
- `kitty/control-codes.h:235` `#define PENDING_MODE 2026`; `kitty/state.h:51` `monotonic_t repaint_delay, input_delay;`.
- `docs/graphics-protocol.rst:762-768` — "Suppressing responses from the terminal" subsection, with the `q` response-suppression key sentences at `:767-768`; `:975-984` — "Image persistence and storage quotas" subsection ("320MB per buffer" at `:980`; the animation frames' separate 5× quota at `:982-984`).
- `kitty_tests/screen.py:948` — 1 MiB parser-buffer boundary; `kitty_tests/parser.py:57,62` — `test_create_write_buffer` / `test_commit_write_buffer` (**non-canonical** in-process bypass); `kitty_tests/graphics.py:1189-1205` — `test_graphics_quota_enforcement` (storage `storage_limit=36*2` at `:1192`; frame `ENOSPC` at `:1205`; the edit-does-not-trigger-quota assertion at `:1207`).
- `setup.py:2003-2004` — `--ignore-compiler-warnings` declaration; effect `werror=''` at `:491` (extension) and `:1231` (kittens/launcher).

---

## 7. Runtime manifestation — silent vs. visible

| Mechanism | Runtime manifestation | Silent or visible | Evidence (build) |
|---|---|---|---|
| Inbound 1 MiB buffer full → `POLLIN` dropped | kitty stops reading; child blocks in `write()` on the full PTY; byte counter freezes / writes pace to ~43 MiB/s | **Silent** (no log; only the blocking write) | E2b §2.2 (canonical natural-pressure stall + instrumented buffer-full wakeups); E2 (`SIGSTOP` stimulus) |
| Inbound pacing (buffer + gating + delay batching) | producer bounded to ~112 MiB/s vs ~108 GiB/s to `/dev/null`; main-loop `pollForEvents` timeout settles at 0.003 s = `input_delay` | **Silent** | E1, T §2.3 (E1 canonical; T instrumented) |
| Over-long / malformed escape code | `... escape code too long (N bytes), ignoring it` / `CSI escape too long ...` logged | **Visible** (`kitty/vt-parser.c:419`, `:831`) | D §2.4 (canonical) |
| Outbound `write_buf` 100 MiB cap | `Too much data being sent to child with id: 1, ignoring it` logged en masse (1,107,992 / 321,546 in 7 s), data discarded | **Visible** (`kitty/child-monitor.c:342`) | F §3.3 (canonical) |
| Outbound `EAGAIN` retention | `write()` returns `-1`; buffer retained, retried on next `POLLOUT`; no data lost | **Silent** (visible only in debug `Wrote: -1`) | F EAGAIN §3.3 (instrumented) |
| Graphics storage quota | oldest images LRU-evicted; `image_count` capped; later `ENOENT` if referenced | **Silent** (eviction) | B2c §4.1 (**canonical, default 320 MiB**, LRU-oldest); B2 §4.1 (in-process, reduced limit) |
| Graphics frame cache 5× | reclaim → recheck → `ENOSPC` APC error reply | **Visible** (error reply) | B3 §4.2 (non-canonical in-process) |
| Graphics size/format guards | `EINVAL`/`EFBIG` APC error reply (empty only for a *fresh* over-dimension; otherwise inherits a stale id) | **Visible** (state-dependent for over-dim) | B4c §4.3 (**canonical**, incl. F11 leak); E-EFBIG §4.3 (canonical); B4, F11 §4.3 (in-process) |
| Graphics `q=` suppression | `OK` (q≥1) or `OK`+errors (q=2) withheld at client's request | **Silent** | B1 §4.4 (canonical) |
| Synchronized-update pause (DECSET 2026) | rendering held; 2000 ms default; auto-unpause at expiry or on DECRST | **Silent** (state-only; query `;1$y`/`;2$y`) | C §5 (canonical + non-canonical in-process) |
| Redundant/late DECSET 2026 while paused | `Pending mode change to already current mode...` logged | **Visible** (`kitty/screen.c:1176`) | C §5 (non-canonical in-process) |

---

## 8. Observed-vs-inferred ledger

**Observed (canonical — default-config kitty via real PTY, or default compiled constants):**
- Default option values and constants (§1.5).
- Inbound pacing E1 (~112 MiB/s via kitty vs ~108 GiB/s to `/dev/null`), **two runs** (§2.3).
- Child stall under inbound backpressure: **E2b natural-pressure canonical stall** (GB/s C producer; **16/16** 4 MiB writes block >50 ms; child paced to ~43 MiB/s), **two runs**; plus **E2** (`SIGSTOP` stand-in, **[non-canonical stimulus]**; 2.8 s progress-file gap), **two runs** — both §2.2.
- Escape-code handling D (complete-ST accept; OSC-52 continue; OSC over-length `:419`; CSI truncation `:831`), **two runs** (§2.4).
- Outbound overflow F (`Too much data...` first at ~5.5–6.7 s; 1,107,992 / 321,546 occurrences — count is a host/run-dependent magnitude, mechanism invariant), **two runs** (§3.3).
- Graphics `q=` suppression B1, **two runs** (§4.4).
- Graphics `EFBIG` over-budget boundary E-EFBIG (`data_sz + 10` accepted / `data_sz + 11` first-failing; `graphics.c:532`/`:533`/`:656`), **two runs** (§4.3).
- Graphics storage quota + LRU ordering B2c (**default 320 MiB**, real PTY: four 100 MB transmits > 320 MiB; unreferenced `a=t` → only newest `i=4` survives; placed `a=T` → strictly oldest-first, `i=1` evicted / `i=2`–`i=4` alive; `graphics.c:290-299`), byte-identical over **two runs** each variant (§4.1).
- Graphics per-image guards B4c (real PTY, default limits: zero-dim `:646` / unknown-format `:651` / PNG-size `:638` → `EINVAL`; over-dimension `:695` → `Image too large` **with the F11 stale-id leak reproduced canonically** — the `i=4` command replies `i=3`), byte-identical over **two runs** (§4.3).
- Synchronized-update 2000 ms timeout auto-unpause C-part-2, **two runs** (§5).

**Observed (instrumented debug build; diagnostics only, behavior identical to canonical):**
- Buffer-full/resume cycles under the GB/s flood: `i:0 POLLIN` io-loop wakeups (`write_space_created` = `read.sz >= BUF_SZ`, `kitty/vt-parser.c:1438` → `wakeup_io_loop`, `kitty/child-monitor.c:442`) = **20 / 13** over two runs, proving the 1 MiB ceiling is repeatedly hit (E2b, §2.2).
- GLFW main-loop poll timeouts and the separate child-`io_loop` `revents` (T, §2.3); per-fd returned `revents` and per-`write()` sizes with an exact 2,600,000-byte drain that is `POLLOUT`-gated and drains in mostly-512-aligned mid-reply chunks (≈88% ×512, not reply-aligned), with ~555 `EAGAIN` retains per run (F slow-drain, §3.3); the isolated `EAGAIN` `Wrote: -1` transition with a `5632 = 433×13+3` mid-reply cut (F EAGAIN, §3.3).

**Observed ([non-canonical in-process] — real C logic via `kitty_tests` parse hooks; B2/B3 additionally use a reduced `storage_limit`):**
- Storage LRU eviction B2, frame-cache reclaim→recheck→`ENOSPC` B3, per-image guards B4, over-dimension state-dependency F11 (**two runs**, fresh process per case), pending-mode transitions + "already current mode" log C-part-1. (B2, B4 and the F11 stale-id leak are **additionally corroborated canonically** at the default 320 MiB limit through a real PTY by **B2c/B4c** above; B3's 5× frame-cache `ENOSPC` and F11's *fresh*-process empty-reply case remain in-process only, since the former needs ~1.6 GiB of frame data to trip canonically and the latter needs a guaranteed-fresh `start_command`.)

**Observed ([non-canonical fault injection] — real branch logic, injected `write()` return value):**
- The `write_to_child` `ret == 0` break+retain (`:1458-1460`; child gets **6/6** replies), `EINTR` continue+retry (`:1462`; **6/6**), and hard-error `perror`+discard (`:1464-1465`; **5/6** — one reply lost, with a visible `perror`) branches (Evidence W, §3.2), **two runs** each.

**[inferred] (read from code, not separately instrumented):**
- The **exact per-cycle trip value** of the force-parse-near-full clause `read.sz + 16*1024 > BUF_SZ` (`kitty/vt-parser.c:1425`): that the arm *fires* is observed (instrumented; ≥20 / ≥13 times over two runs, E2b §2.2–2.3), but the precise boundary `read.sz` is not printed.
- The literal write of `events = 0` that drops `POLLIN` at the parser-full trigger (`kitty/child-monitor.c:1501`): `DEBUG_POLL_EVENTS` prints only *present* `revents`, so the drop appears as the *absence* of `i:2 POLLIN` reads during a stall. The buffer-full *condition* that triggers it **is** observed (E2b `i:0` wakeups) and the OS-level *effect* **is** observed (E2/E2b child stall); only the register write itself remains inferred from the one-line gate.

**Stability (F7).** Every **timing/magnitude** condition was run at least twice, and both runs are reported: E1 (FINAL totals table), E2 (freeze at a single value; ~2.8 s gap both runs), D (identical 5-line stderr; `0.581/0.982/2.184` vs `0.579/0.980/2.182`), F overflow (mechanism identical both runs — en masse `id: 1` logging with no rate-limit; the **count is a host/run-dependent magnitude that legitimately varies**, 1,107,992 vs 321,546, see the Portability note in §3.3), F slow-drain (exact 2,600,000-byte drain both runs; diagnostic counts vary — reported for both), B1/F11/B2–B4 (byte-identical responses, verified by `diff`), C-part-2 (identical `;1$y`/`;2$y` bracketing; `+0.502/+1.719/+2.615/+3.010`). Where a value is not from the canonical build (T, F slow-drain, F EAGAIN) it is drawn from the instrumented build and labelled accordingly, and diagnostic **counts** from that build are treated as run-dependent magnitudes, never as fixed constants.

**Exact commands.** Build A (canonical), Build B (instrumented), the headless run template, and the in-process harness template are listed verbatim in §1, and each evidence block above includes the exact command that produced it. **Every** harness used in this investigation is additionally reproduced **verbatim, with its exact copy-paste-runnable invocation, in the §10 harness appendix.** The three primary canonical harnesses also appear inline next to their evidence — `flood_plain.py` (§2.2), and `fill_writebuf.py` and `fill_eagain.py` (§3.3) — and the supplementary/non-canonical harnesses — `esc_boundaries.py` (Evidence D), `fill_slowdrain.py` (F slow-drain), `tiny_input.py` (Evidence T), `pending_child.py` (C-part-2), `graphics_b1_qsuppress.py` (B1), `graphics_efbig.py` (E-EFBIG), `graphics_guards.py` (B4c canonical guards + F11 leak), `storage_canonical.py` / `lru_canonical.py` (B2c canonical 320 MiB quota + LRU-oldest), `graphics_apc_route.py` (§3.4), the in-process graphics/pending harnesses (`graphics_b2_storage.py`, `graphics_storage_frames.py`, `graphics_b4_guards_inproc.py`, `graphics_f11_overdim.py`, `pending_transitions_inproc.py` for B2–B4, F11, C-part-1), and the `f_capture.awk` log filter — are all supplied in full in §10. Several supplementary claims are **also independently corroborated by passing repository tests**: `kitty_tests/graphics.py::test_graphics_quota_enforcement` (`graphics.py:1189`) reproduces the B2 storage-LRU and B3 5×-frame-cache claims, and the DECRQM `?2026$p` cases in `kitty_tests/parser.py` (`:456-465`) reproduce the pending-mode state transitions. A fresh investigator can therefore reproduce **every** claim in this document from the §10 appendix bodies + invocations (and, for the supplementary claims, additionally via those repository tests).

---

## 9. Investigation hygiene

- **Repository proof.** Investigated source baseline: `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` ("Wire up applying of font config"). `git diff --stat <baseline>..HEAD` shows **exactly one** changed path — `A blitzy/documentation/kitty_815df1e210e0.md` — i.e. one CREATE and zero UPDATE/DELETE of any existing file. The delivery HEAD is a **descendant** of the baseline (the initial add was `3ee38c66df78f704971598ee3002223730a1fe5d`; this revision is a further descendant); it is **not** equal to the baseline. The document does **not** claim HEAD remains the baseline.
- **No source modified.** All named source/header/config/test/build/protocol files are byte-identical to the baseline; the only addition is this document under `blitzy/documentation/`.
- **Temporary artifacts.** All observation scripts and logs lived **outside** the repository under `/tmp/kitty_obs` (`flood_plain.py`, `fill_writebuf.py`, `fill_slowdrain.py`, `fill_eagain.py`, `esc_boundaries.py`, `tiny_input.py`, `graphics_*.py`, `pending_*.py`, and `*.log`/`*.txt` scratch) and were **removed** after evidence capture.
- **Build artifacts** (`build/`, `kitty/fast_data_types.so`, `kitty/launcher/kitty`, `kitty/launcher/kitten`) are git-ignored and do not appear as tracked changes; `git status --porcelain` (tracked) is empty apart from this document.
- **Cleanup command (idempotent).** A single command removes every temporary observation script and scratch log. Because `/tmp/kitty_obs` lives entirely **outside** the repository, this can never touch a tracked file, and re-running it on an already-clean tree is a harmless no-op:

```
rm -rf /tmp/kitty_obs        # remove all harness scripts + *.log/*.txt scratch
git status --porcelain --untracked-files=all   # run from the repository root
```

  After cleanup, `git status --porcelain --untracked-files=all` reports **only** this document (untracked, since the investigation *creates* it) — the investigation added, modified, or deleted nothing else under the repository:

```
?? blitzy/documentation/kitty_815df1e210e0.md
```

---

## 10. Harness appendix - complete source of every observation script

Every script in this investigation is reproduced **verbatim** below so a fresh investigator can re-run every claim. All of them lived **only** under `/tmp/kitty_obs` (outside the repository) and were removed afterward (§9). The one-time environment setup - the Xvfb lifecycle (PID capture + `trap` cleanup) and the two build commands (Build A canonical, Build B instrumented) - is in §1 and is not repeated here. Each block gives the harness body and the exact, copy-paste-runnable invocation. Real-PTY children run **under the launcher** (their fd 1/2 *is* the PTY, so they write observations to a side file passed as `argv`); in-process harnesses run **standalone** (they import `kitty.fast_data_types` directly, no PTY/`DISPLAY` needed).

### 10.1 Canonical real-PTY children (Build A)

**`flood_plain.py`** - inbound pacing/stall (E1 throughput §2.3, E2 child-stall §2.2). Also reproduced inline at §2.2. Body:

```python
# /tmp/kitty_obs/flood_plain.py
import os, sys, time
duration = float(sys.argv[1]); progress_path = sys.argv[2]
chunk = b'x' * 65536
start = time.monotonic(); written = 0; last_log = 0.0
with open(progress_path, 'w', buffering=1) as pf:
    while time.monotonic() - start < duration:
        written += os.write(1, chunk)          # blocks when the PTY buffer is full
        now = time.monotonic()
        if now - last_log >= 0.2:
            pf.write("%d %.3f\n" % (written, now - start)); pf.flush(); last_log = now
    pf.write("FINAL %d %.3f\n" % (written, time.monotonic() - start)); pf.flush()
```
Invocation (E2 stall variant; also see §1.3):

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 9 \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  python3 /tmp/kitty_obs/flood_plain.py 7 /tmp/kitty_obs/e2_prog.txt \
#   then, mid-flood, from another shell:
#   pid=$(ps -eo pid,comm | awk '$2=="kitty"{print $1}'); kill -STOP $pid; sleep 2.6; kill -CONT $pid
```
**`flood.c`** - ~GB/s C producer (4 MiB per `write()`) that fills the 1 MiB parser buffer under **natural** pressure to force the inbound pause (E2b §2.2) and, on an instrumented build, the buffer-full/resume wakeup cycles. Also reproduced inline at §2.2. Body:

```c
#define _GNU_SOURCE
#include <unistd.h>
#include <stdio.h>
#include <string.h>
#include <time.h>
#include <stdlib.h>
static double now(void){struct timespec ts;clock_gettime(CLOCK_MONOTONIC,&ts);return ts.tv_sec+ts.tv_nsec/1e9;}
int main(int argc,char**argv){
    size_t CH=4u*1024u*1024u;            // 4 MiB per write
    long target_mib = argc>1?atol(argv[1]):512;   // total MiB
    char *buf=malloc(CH);
    for(size_t i=0;i<CH;i++) buf[i]= (i%80==79)?'\n':'X';
    FILE*prog=fopen("/tmp/kitty_obs/cflood_progress.txt","w");
    double t0=now(); long long total=0; long long target=(long long)target_mib*1024*1024;
    int nblock=0; double maxb=0.0;
    while(total<target){
        double a=now();
        ssize_t n=write(1,buf,CH);
        double b=now();
        if(n<0){ fprintf(prog,"WRITE_ERR errno-based\n"); break; }
        total+=n; double dt=b-a;
        if(dt>0.05){ nblock++; if(dt>maxb)maxb=dt; }
        fprintf(prog,"t=%.4f total=%lld write_dt=%.5f n=%zd\n",b-t0,total,dt,n); fflush(prog);
    }
    fprintf(prog,"DONE t=%.4f total=%lld blocked(>50ms)=%d max_block=%.4fs\n",now()-t0,total,nblock,maxb); fflush(prog);
    struct timespec s={0,200000000}; nanosleep(&s,NULL);
    return 0;
}
```
Invocation (build then run under the launcher; logs every `write()` to `/tmp/kitty_obs/cflood_progress.txt`):

```
cc -O2 -o /tmp/kitty_obs/flood_bin /tmp/kitty_obs/flood.c
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 15 \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  /tmp/kitty_obs/flood_bin 64
#   canonical stall:  grep DONE /tmp/kitty_obs/cflood_progress.txt
#   instrumented proxy (Build B fdt): count i:0 POLLIN wakeups / i:2 POLLIN reads on stdout
```
**`fill_writebuf.py`** - outbound 100 MiB-cap overflow (F overflow §3.3). Also inline at §3.3. Body:

```python
# /tmp/kitty_obs/fill_writebuf.py
import os, sys, time, tty
tty.setraw(0)
dur = float(sys.argv[1]); prog = sys.argv[2]
batch = b'\x1b[>c' * 65536       # 65536 DA2 queries per write (256 KiB)
start = time.monotonic(); n = 0
with open(prog, 'w', buffering=1) as pf:
    while time.monotonic() - start < dur:
        n += os.write(1, batch)
        pf.write("%d %.3f\n" % (n, time.monotonic() - start)); pf.flush()
    pf.write("FINAL %d %.3f\n" % (n, time.monotonic() - start)); pf.flush()
```
Invocation (streamed through the `f_capture.awk` filter of §10.4; two runs for stability):

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 9 \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  python3 /tmp/kitty_obs/fill_writebuf.py 7 /tmp/kitty_obs/F_prog_r1.txt \
  2>&1 | awk -f /tmp/kitty_obs/f_capture.awk
```
**`fill_eagain.py`** - isolated outbound `EAGAIN` transition (F EAGAIN §3.3; **Build B**). Also inline at §3.3. Body:

```python
# /tmp/kitty_obs/fill_eagain.py
import os, sys, time, termios, tty
tty.setraw(0)
prog = sys.argv[1]
os.write(1, b'\x1b[>c' * 200000)   # enqueue ~2.6 MiB of replies into write_buf
time.sleep(1.5)                     # DO NOT read: write_buf builds; POLLOUT requested but fd unwritable
data = os.read(0, 1 << 20)          # free a big chunk at once -> kitty writes until EAGAIN
open(prog, 'w').write("first_read %d\n" % len(data))
time.sleep(0.6)
```
Invocation (`Wrote:` lines go to **stderr**):

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 10 \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  python3 /tmp/kitty_obs/fill_eagain.py /tmp/kitty_obs/eagain_prog.txt \
  2>/tmp/kitty_obs/eagain_stderr.log
```
**`esc_boundaries.py`** - escape-code length boundaries (Evidence D §2.4): OSC-too-long, stray `ESC\`, CSI-too-long. Body:

```python
# /tmp/kitty_obs/esc_boundaries.py
# Emits four ~300 KB escape-code constructs to exercise the VT-parser length
# boundaries (MAX_ESCAPE_CODE_LENGTH = 256 KiB, kitty/vt-parser.c:21).
# Real child on a real PTY under a canonical kitty; errors surface on kitty stderr.
import os, sys, time, tty
tty.setraw(0)
ESC = b'\x1b'
big = b'x' * 300000
# Case A: COMPLETE OSC 9 + ST  -> dispatched via the "be generous" branch, NO error
os.write(1, ESC + b']9;' + big + ESC + b'\\')
time.sleep(0.4)
# Case B: INCOMPLETE OSC 9 (no ST) -> "VTE_OSC escape code too long (300002 bytes)"
os.write(1, ESC + b']9;' + big)
time.sleep(0.4)
# Case C: leading stray ESC\ (parser is NORMAL after B's discard) -> "Unknown char after ESC: 0x5c";
#         then INCOMPLETE OSC 52 (no ST) -> special-cased, NO "too long" error
os.write(1, ESC + b'\\' + ESC + b']52;' + b'A' * 300000)
time.sleep(1.2)
# Case D: leading ESC\ is ABSORBED as the ST of the continuing OSC 52 (no error);
#         then one giant CSI parameter (no final byte) -> "CSI escape too long ignoring and truncating"
os.write(1, ESC + b'\\' + ESC + b'[' + b'1' * 300000)
time.sleep(0.6)
```
Invocation (parse-error lines go to stderr):

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 8 \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  python3 /tmp/kitty_obs/esc_boundaries.py 2>&1
```
**`pending_child.py`** - synchronized-update pending mode DECSET 2026 + 2000 ms auto-unpause (C-part-2 §5). Writes to a **side file** (`/tmp/kitty_obs/pending_prog.txt`) because fd 1/2 is the PTY. Body:

```python
# /tmp/kitty_obs/pending_child.py
# Canonical real-PTY probe of synchronized-update pending mode (DECSET 2026) and
# its 2000 ms default auto-unpause (kitty/screen.c:2521). Observations are written
# to a SIDE FILE because fd 1/2 are the PTY (they would feed back into kitty).
import os, sys, time, tty, select
tty.setraw(0)
ESC = b'\x1b'
PROG = '/tmp/kitty_obs/pending_prog.txt'
time.sleep(0.4)

def query(wait=0.6):
    os.write(1, ESC + b'[?2026$p')          # DECRQM query of pending mode 2026
    buf = b''; t = time.monotonic()
    while time.monotonic() - t < wait:
        r, _, _ = select.select([0], [], [], 0.1)
        if r:
            d = os.read(0, 4096)
            if not d:
                break
            buf += d
            if buf.endswith(b'y'):           # DECRPM reply ends with '$y'
                break
    return buf

def force_render():
    os.write(1, ESC + b'[H')                 # cursor home -> dirties state -> render pass runs check

out = []
r = query()
out.append("C0-before-set  dt_from_pause=+0.000 resp=%r" % r)

pause_t = time.monotonic()
os.write(1, ESC + b'[?2026h')                # DECSET 2026 -> pause rendering (for_in_ms=0 -> 2000 ms)

for label, dt in [("C1", 0.5), ("C2-pre-2s", 1.7), ("C3-post-2s", 2.6), ("C4-post-2s", 3.0)]:
    while time.monotonic() - pause_t < dt:
        time.sleep(0.02)
    dt_reached = time.monotonic() - pause_t   # measured at checkpoint, matches schedule
    force_render()
    time.sleep(0.05)
    r = query()
    out.append("%-14s dt_from_pause=+%.3f resp=%r" % (label, dt_reached, r))

open(PROG, 'w').write("\n".join(out) + "\n")
time.sleep(0.3)
```
Invocation (read the side file afterward):

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 10 \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  python3 /tmp/kitty_obs/pending_child.py
#   cat /tmp/kitty_obs/pending_prog.txt
```
**`graphics_b1_qsuppress.py`** - graphics `q=` response suppression (B1 §4.2). Output path is `argv[1]`. Body:

```python
# /tmp/kitty_obs/graphics_b1_qsuppress.py   (observed, canonical, real PTY)
# A raw-mode child transmits success (valid 1x1 RGB) and error (zero-width) graphics
# commands with q=0/1/2 and reads the APC replies, demonstrating the q= response
# suppression lever (finish_command_response, graphics.c:759-768). Empty replies
# (b'') are captured by letting the read time out.
import os, sys, time, tty, select
tty.setraw(0); time.sleep(0.5)

def rw(payload, wait=1.0):
    os.write(1, payload)
    buf = b''; t = time.monotonic()
    while time.monotonic() - t < wait:
        r, _, _ = select.select([0], [], [], 0.1)
        if r:
            d = os.read(0, 65536)
            if not d:
                break
            buf += d
            if buf.endswith(b'\x1b\\'):
                break
    return buf

# /wAA = base64 of FF 00 00 = one RGB pixel (f=24). success: s=1,v=1 ; error: s=0,v=1 (zero width)
cases = [
    ("success_q0", b'\x1b_Ga=t,f=24,s=1,v=1,i=1,q=0;/wAA\x1b\\'),
    ("error_q0",   b'\x1b_Ga=t,f=24,s=0,v=1,i=1,q=0;/wAA\x1b\\'),
    ("success_q1", b'\x1b_Ga=t,f=24,s=1,v=1,i=1,q=1;/wAA\x1b\\'),
    ("error_q1",   b'\x1b_Ga=t,f=24,s=0,v=1,i=1,q=1;/wAA\x1b\\'),
    ("success_q2", b'\x1b_Ga=t,f=24,s=1,v=1,i=1,q=2;/wAA\x1b\\'),
    ("error_q2",   b'\x1b_Ga=t,f=24,s=0,v=1,i=1,q=2;/wAA\x1b\\'),
]
out = []
for label, payload in cases:
    r = rw(payload)
    out.append("CASE %-11s SENT=%r" % (label, payload))
    out.append("              RESP=%r" % r)
open(sys.argv[1], 'w').write("\n".join(out) + "\n")
time.sleep(0.3)
```
Invocation:

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 30 \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  python3 /tmp/kitty_obs/graphics_b1_qsuppress.py /tmp/kitty_obs/b1_out.txt
#   cat /tmp/kitty_obs/b1_out.txt
```
**`graphics_efbig.py`** - direct-transmission capacity boundary `EFBIG` (E-EFBIG §4.3), at the reduced 2x2 scale (`data_sz=16`, capacity `data_sz+10=26`). Output path is `argv[1]`. Body:

```python
import os, sys, time, tty, select, base64
tty.setraw(0); time.sleep(0.5)
def rw(payload, wait=3.0):
    os.write(1, payload)
    buf=b''; t=time.monotonic()
    while time.monotonic()-t < wait:
        r,_,_=select.select([0],[],[],0.2)
        if r:
            d=os.read(0,65536)
            if not d: break
            buf+=d
            if buf.endswith(b'\x1b\\'): break
    return buf
out=[]
# 2x2 RGBA: data_sz = 2*2*4 = 16 ; buf_capacity = data_sz+10 = 26 (graphics.c:656)
# Direct transmit (default t=d). payload_sz counted in DECODED bytes.
for (label, nbytes, iid) in [("capacity(data_sz+10)=26", 26, 1),
                              ("first-failing(data_sz+11)=27", 27, 2),
                              ("well-over=40", 40, 3)]:
    raw = b'\x00'*nbytes
    b64 = base64.standard_b64encode(raw).decode()
    r = rw(('\x1b_Ga=t,f=32,s=2,v=2,i=%d,q=0;%s\x1b\\'%(iid,b64)).encode())
    out.append("%s (payload=%d B) -> %r" % (label, nbytes, r))
open(sys.argv[1],'w').write("\n".join(out)+"\n"); time.sleep(0.3)
```
Invocation:

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 30 \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  python3 /tmp/kitty_obs/graphics_efbig.py /tmp/kitty_obs/efbig_out.txt
#   cat /tmp/kitty_obs/efbig_out.txt
```
**`graphics_apc_route.py`** - confirms a graphics `OK` reply is routed through the *same* `write_to_child`/APC output path (§3.4). Output path is `argv[1]`. Body:

```python
import os, sys, time, tty, termios, select
tty.setraw(0)
# valid 1x1 RGB image, id=7, q=0 (expect OK reply routed via write_to_child->APC)
os.write(1, b'\x1b_Ga=t,f=24,s=1,v=1,i=7,q=0;/wAA\x1b\\')
buf = b''
t = time.monotonic()
while time.monotonic() - t < 2.0:
    r,_,_ = select.select([0], [], [], 0.3)
    if r:
        d = os.read(0, 4096)
        if not d: break
        buf += d
        if b'\x1b\\' in buf: break
open(sys.argv[1], 'wb').write(b'REPLY=' + buf)
time.sleep(0.3)
```
Invocation:

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 15 \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  python3 /tmp/kitty_obs/graphics_apc_route.py /tmp/kitty_obs/apc_route_out.txt
#   cat -v /tmp/kitty_obs/apc_route_out.txt  ->  REPLY=^[_Gi=7;OK^[\
#   (the file holds RAW bytes; cat -v renders ESC as ^[, i.e. the bytes \x1b _ G i=7;OK \x1b \)
```
**`graphics_guards.py`** - canonical real-PTY per-image guards + F11 stale-id leak (Evidence B4c §4.3). Output path is `argv[1]`. Body:

```python
import os, sys, time, tty, select
tty.setraw(0)
time.sleep(0.4)
cmds = [
  ("zero-dim s=0,v=1 i=1",         b'\x1b_Ga=t,f=24,s=0,v=1,i=1,q=0;/wAA\x1b\\'),
  ("unknown-format f=99 i=2",      b'\x1b_Ga=t,f=99,s=1,v=1,i=2,q=0;/wAA\x1b\\'),
  ("PNG-size S=400000001 f=100 i=3", b'\x1b_Ga=t,f=100,S=400000001,i=3,q=0;/wAA\x1b\\'),
  ("over-dim s=20000,v=20000 i=4 (FRESH)", b'\x1b_Ga=t,f=24,s=20000,v=20000,i=4,q=0;/wAA\x1b\\'),
]
out=[]
for label, payload in cmds:
    os.write(1, payload)
    buf=b''; t=time.monotonic()
    while time.monotonic()-t < 1.2:
        r,_,_=select.select([0],[],[],0.2)
        if r:
            d=os.read(0,4096)
            if not d: break
            buf+=d
            if buf.endswith(b'\x1b\\'): break
    out.append("%-40s -> %r" % (label, buf))
open(sys.argv[1],'w').write("\n".join(out)+"\n")
time.sleep(0.2)
```
Invocation:

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 20 \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  python3 /tmp/kitty_obs/graphics_guards.py /tmp/kitty_obs/guards_c_run1.txt
```
**`storage_canonical.py`** - canonical real-PTY 320 MiB storage quota via unreferenced (`a=t`) 100 MB transmits (Evidence B2c Variant 1, §4.1). Requires the fixture `img100mb.rgba` (create once: `head -c 100000000 /dev/zero > /tmp/kitty_obs/img100mb.rgba`). Output path is `argv[1]`. Body:

```python
import os, sys, time, tty, select, base64
tty.setraw(0)
time.sleep(0.5)
path = b'/tmp/kitty_obs/img100mb.rgba'
b64path = base64.standard_b64encode(path).decode()

def rw(payload, wait=2.0):
    os.write(1, payload)
    buf=b''; t=time.monotonic()
    while time.monotonic()-t < wait:
        r,_,_=select.select([0],[],[],0.2)
        if r:
            d=os.read(0,4096)
            if not d: break
            buf+=d
            if buf.endswith(b'\x1b\\'): break
    return buf

out=[]
# transmit 4 images of 100MB each via file medium t=f, RGBA f=32, 5000x5000
for i in (1,2,3,4):
    cmd = ('\x1b_Ga=t,t=f,f=32,s=5000,v=5000,i=%d,q=0;%s\x1b\\' % (i, b64path)).encode()
    resp = rw(cmd, wait=6.0)
    out.append("transmit i=%d -> %r" % (i, resp))
# now query existence of each id via a=p (put); OK if present, ENOENT if evicted
for i in (1,2,3,4):
    cmd = ('\x1b_Ga=p,i=%d,q=0\x1b\\' % i).encode()
    resp = rw(cmd, wait=2.0)
    out.append("query(put) i=%d -> %r" % (i, resp))
open(sys.argv[1],'w').write("\n".join(out)+"\n")
time.sleep(0.3)
```
Invocation:

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 60 \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  python3 /tmp/kitty_obs/storage_canonical.py /tmp/kitty_obs/storage_c_run1.txt
```
**`lru_canonical.py`** - canonical real-PTY LRU-oldest ordering via placed (`a=T`) 100 MB images (Evidence B2c Variant 2, §4.1). Same `img100mb.rgba` fixture. Output path is `argv[1]`. Body:

```python
import os, sys, time, tty, select, base64
tty.setraw(0)
time.sleep(0.5)
path = b'/tmp/kitty_obs/img100mb.rgba'
b64 = base64.standard_b64encode(path).decode()
def rw(payload, wait):
    os.write(1, payload)
    buf=b''; t=time.monotonic()
    while time.monotonic()-t < wait:
        r,_,_=select.select([0],[],[],0.2)
        if r:
            d=os.read(0,4096)
            if not d: break
            buf+=d
            if buf.endswith(b'\x1b\\'): break
    return buf
out=[]
# place 4 referenced (a=T) 100MB images, oldest first; delay to ensure distinct atimes
for i in (1,2,3,4):
    r = rw(('\x1b_Ga=T,t=f,f=32,s=5000,v=5000,i=%d,q=0;%s\x1b\\'%(i,b64)).encode(), 25.0)
    out.append("transmit+display i=%d -> %r" % (i, r))
    time.sleep(0.6)
# query each via a=p (put) existence
for i in (1,2,3,4):
    r = rw(('\x1b_Ga=p,i=%d,q=0\x1b\\'%i).encode(), 3.0)
    out.append("query(put) i=%d -> %r" % (i, r))
open(sys.argv[1],'w').write("\n".join(out)+"\n")
time.sleep(0.3)
```
Invocation:

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 120 \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  python3 /tmp/kitty_obs/lru_canonical.py /tmp/kitty_obs/lru_c_run1.txt
```
### 10.2 Instrumented children (Build B - diagnostics only)

**`fill_slowdrain.py`** - watches `POLLOUT`-gated draining + repeated `EAGAIN` (F slow-drain §3.3). `revents` print to **stdout**, `Wrote:` to **stderr**. Body:

```python
# /tmp/kitty_obs/fill_slowdrain.py
# Enqueues 200,000 DA2 replies (~2.6 MiB) into kitty's write_buf, then drains slowly
# (4 KiB every 60 ms). Under a DEBUG_POLL_EVENTS + KITTY_PRINT_BYTES build (Build B)
# it shows returned revents and per-write() byte counts as the buffer drains.
import os, sys, time, tty
tty.setraw(0)
prog = sys.argv[1] if len(sys.argv) > 1 else '/tmp/kitty_obs/slowdrain_prog.txt'
os.write(1, b'\x1b[>c' * 200000)              # 200000 DA2 queries -> ~2.6 MiB of 13-byte replies
read_total = 0
while read_total < 200000 * 13:
    time.sleep(0.06)
    try:
        d = os.read(0, 4096)                  # slow reader: 4 KiB every 60 ms
    except OSError:
        break
    if not d:
        break
    read_total += len(d)
open(prog, 'w').write("read_total %d\n" % read_total)
```
Invocation (separate the two streams):

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 60 \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  python3 /tmp/kitty_obs/fill_slowdrain.py /tmp/kitty_obs/sd_prog.txt \
  >/tmp/kitty_obs/sd_stdout.log 2>/tmp/kitty_obs/sd_stderr.log
```
**`tiny_input.py`** - exposes GLFW main-loop poll timeouts + io-loop `revents` tail (Evidence T §2.3). Requires **Build B** (its `glfw-x11.so` carries the `DEBUG_EVENT_LOOP` trace). Body:

```python
# /tmp/kitty_obs/tiny_input.py
# A modest periodic producer: writes 5 short lines then idles. Traced under a
# DEBUG_EVENT_LOOP build (Build B), it exposes the GLFW main-loop poll timeouts
# (pollForEvents final timeout: ~0.003 = input_delay) and the io_loop revents tail.
import os, time
for i in range(5):
    os.write(1, b'line %d\r\n' % i)
    time.sleep(0.05)
time.sleep(2.2)   # idle so the trace shows the idle waits settling toward the state-check
```
Invocation (trace to stderr; `i:N` `revents` to stdout):

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. timeout 8 \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  python3 /tmp/kitty_obs/tiny_input.py 2>/tmp/kitty_obs/T.log
```
**`faultwrite.c` + `multi_da2.py`** - force the three rare `write_to_child` branches (`ret==0`, `EINTR`, hard error) via `LD_PRELOAD` fault injection (Evidence W §3.2) [non-canonical fault injection]. The shim intercepts the single `write()` carrying the DA2 reply prefix `ESC[>1`; the child sends 6 DA2 queries and tallies replies (6 = retained/retried, 5 = discarded). Bodies:

```c
#define _GNU_SOURCE
#include <unistd.h>
#include <errno.h>
#include <dlfcn.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>
typedef ssize_t (*write_fn)(int, const void*, size_t);
static write_fn real_write = NULL;
static int fired = 0;
ssize_t write(int fd, const void *buf, size_t n) {
    if (!real_write) real_write = (write_fn)dlsym(RTLD_NEXT, "write");
    const char *mode = getenv("FAULT_MODE");
    const unsigned char *b = (const unsigned char*)buf;
    // Match ONLY the DA2 reply prefix ESC [ > 1  (kitty->child). The DA2 query ESC [ > c does NOT match.
    if (mode && !fired && n >= 4 && b[0]==0x1b && b[1]=='[' && b[2]=='>' && b[3]=='1') {
        fired = 1;
        fprintf(stderr, "[FAULTWRITE] injecting %s on fd=%d n=%zu (DA2 reply ESC[>1)\n", mode, fd, n);
        if (strcmp(mode, "zero") == 0) return 0;
        if (strcmp(mode, "eintr") == 0) { errno = EINTR; return -1; }
        if (strcmp(mode, "eio")  == 0) { errno = EIO;  return -1; }
    }
    return real_write(fd, buf, n);
}
```

```python
import os, sys, time, tty, select
tty.setraw(0)
prog = sys.argv[1]
time.sleep(0.4)
buf=b''; sent=0
t0=time.monotonic(); next_send=0.0
while time.monotonic()-t0 < 4.0:
    now=time.monotonic()-t0
    if sent < 6 and now >= next_send:
        os.write(1, b'\x1b[>c'); sent+=1; next_send = now + 0.4
    r,_,_=select.select([0],[],[],0.1)
    if r:
        d=os.read(0,65536)
        if d: buf+=d
n_replies = buf.count(b'\x1b[>1;4000;35c')
with open(prog,'w') as f:
    f.write("sent_queries=%d received_bytes=%d received_replies=%d\n" % (sent, len(buf), n_replies))
time.sleep(0.2)
```

Invocation (requires the Build-B fdt for the `Wrote:` printer; run each mode; `FAULTWRITE`/`Wrote:` lines and the tally go to stderr / the side file):

```
cc -O2 -shared -fPIC -o /tmp/kitty_obs/faultwrite.so /tmp/kitty_obs/faultwrite.c -ldl
for mode in eintr zero eio; do
  LD_PRELOAD=/tmp/kitty_obs/faultwrite.so FAULT_MODE=$mode DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 timeout 12 \
    kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
    python3 /tmp/kitty_obs/multi_da2.py /tmp/kitty_obs/fault_prog.txt 2>fault_$mode.log
  cat /tmp/kitty_obs/fault_prog.txt
done
```

### 10.3 In-process harnesses ([non-canonical in-process])

These import `kitty.fast_data_types` and drive the compiled `graphics.c`/`screen.c`/`vt-parser.c` through the `kitty_tests` parse hooks (`test_create_write_buffer` / `test_commit_write_buffer` / `test_parse_written_data`), building the `Screen` exactly as `kitty_tests` does (the same `Callbacks` passed twice). Some deliberately reduce a limit so an eviction/quota is observable without transmitting hundreds of MiB - the executed guard/quota code is identical to canonical; only the limit value and the (non-PTY) entry differ. Each runs standalone:

**`graphics_b2_storage.py`** - 320 MiB storage quota + LRU eviction (B2 §4.1). Body:

```python
# /tmp/kitty_obs/graphics_b2_storage.py   [non-canonical in-process]
# Real compiled graphics.c through the kitty_tests parse hooks (no live PTY). The
# storage_limit is deliberately reduced to 72 bytes so apply_storage_quota's LRU
# eviction (graphics.c:290-299) is observable without transmitting 320 MiB. The
# guard/quota code path executed is identical to the 320 MiB canonical default.
import base64
from kitty.fast_data_types import Screen
from kitty_tests import Callbacks, parse_bytes, BaseTest

bt = BaseTest(); bt.set_options()
c = Callbacks(); s = Screen(c, 5, 5, 5, 10, 20, 0, c); g = s.grman

def cmd(cs):
    c.clear(); parse_bytes(s, cs.encode()); return c.wtcbuf
def code(res):
    return res.decode('ascii').partition(';')[2].partition(':')[0].partition('\x1b')[0] if res else '(empty)'
def place(iid, w=3, h=3):            # a=T transmit+display -> image is REFERENCED (survives trim_predicate)
    b64 = base64.standard_b64encode(b'\x00' * (w * h * 4)).decode()
    return code(cmd('\x1b_Ga=T,f=32,s=%d,v=%d,i=%d,q=0;%s\x1b\\' % (w, h, iid, b64)))
def display(iid):                    # a=p put/display; ENOENT if the image was evicted
    return code(cmd('\x1b_Ga=p,i=%d,q=0\x1b\\' % iid))

print("CANONICAL default storage_limit = %d bytes (= 320 MiB)" % Screen(Callbacks(), 5, 5, 5, 10, 20, 0, Callbacks()).grman.storage_limit)
print("[NON-CANONICAL] reduce storage_limit to observe eviction without transmitting 320 MiB")
g.storage_limit = 72
print("reduced storage_limit = %d bytes (holds two 36-byte images)" % g.storage_limit)
print()
print("=== B2: storage quota + LRU eviction ===")
print("BEFORE:            image_count=%d  total_size=%d" % (g.image_count, g.disk_cache.total_size))
print("transmit i=1 -> %s     image_count=%d  total_size=%d" % (place(1), g.image_count, g.disk_cache.total_size))
print("transmit i=2 -> %s     image_count=%d  total_size=%d" % (place(2), g.image_count, g.disk_cache.total_size))
print("transmit i=3 -> %s     image_count=%d  total_size=%d  (limit reached: oldest evicted)" % (place(3), g.image_count, g.disk_cache.total_size))
print("  display i=1 -> %s" % display(1))
print("  display i=2 -> %s" % display(2))
print("  display i=3 -> %s" % display(3))
```
Invocation:

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. \
  python3 /tmp/kitty_obs/graphics_b2_storage.py
```
**`graphics_storage_frames.py`** - B2 plus the 5x animation frame-cache quota -> `ENOSPC` (B3 §4.1). Body:

```python
# /tmp/kitty_obs/graphics_storage_frames.py   [non-canonical in-process]
# Real compiled graphics.c through the kitty_tests parse hooks (no live PTY).
# storage_limit is reduced to 72 bytes so both the 320 MiB LRU eviction path
# (B2, graphics.c:290-299) and the storage_limit*5 frame-cache path
# (B3, graphics.c:1570-1573) are observable at tiny scale. The code path is
# identical to the canonical defaults; only the limit value differs.
import base64
from kitty.fast_data_types import Screen
from kitty_tests import Callbacks, parse_bytes, BaseTest

bt = BaseTest(); bt.set_options()
c = Callbacks(); s = Screen(c, 5, 5, 5, 10, 20, 0, c); g = s.grman

def cmd(cs):
    c.clear(); parse_bytes(s, cs.encode()); return c.wtcbuf
def code(res):
    return res.decode('ascii').partition(';')[2].partition(':')[0].partition('\x1b')[0] if res else '(empty)'
def place(iid, w=3, h=3):            # a=T transmit+display -> REFERENCED image
    b64 = base64.standard_b64encode(b'\x00' * (w * h * 4)).decode()
    return code(cmd('\x1b_Ga=T,f=32,s=%d,v=%d,i=%d,q=0;%s\x1b\\' % (w, h, iid, b64)))
def display(iid):
    return code(cmd('\x1b_Ga=p,i=%d,q=0\x1b\\' % iid))
def addframe(iid, w=3, h=3):         # a=f new animation frame (36 bytes each)
    b64 = base64.standard_b64encode(b'\x11' * (w * h * 4)).decode()
    return code(cmd('\x1b_Ga=f,f=32,s=%d,v=%d,i=%d,q=0;%s\x1b\\' % (w, h, iid, b64)))
def editframe(iid, fno, w=2, h=2):   # a=f with r=<frame no> -> EDIT existing frame (is_new_frame False)
    b64 = base64.standard_b64encode(b'\x22' * (w * h * 4)).decode()
    return code(cmd('\x1b_Ga=f,f=32,r=%d,s=%d,v=%d,i=%d,q=0;%s\x1b\\' % (fno, w, h, iid, b64)))

print("CANONICAL default storage_limit = %d bytes (= 320 MiB)" % Screen(Callbacks(), 5, 5, 5, 10, 20, 0, Callbacks()).grman.storage_limit)
print("[NON-CANONICAL] reduce storage_limit to observe eviction without transmitting 320 MiB")
g.storage_limit = 72
print("reduced storage_limit = %d bytes (holds two 36-byte images)" % g.storage_limit)
print()
print("=== B2: storage quota + LRU eviction ===")
print("BEFORE:            image_count=%d  total_size=%d" % (g.image_count, g.disk_cache.total_size))
print("transmit i=1 -> %s     image_count=%d  total_size=%d" % (place(1), g.image_count, g.disk_cache.total_size))
print("transmit i=2 -> %s     image_count=%d  total_size=%d" % (place(2), g.image_count, g.disk_cache.total_size))
print("transmit i=3 -> %s     image_count=%d  total_size=%d  (limit reached: oldest evicted)" % (place(3), g.image_count, g.disk_cache.total_size))
print("  display i=1 -> %s" % display(1))
print("  display i=2 -> %s" % display(2))
print("  display i=3 -> %s" % display(3))
print()
print("=== B3: frame-cache 5x quota (reclaim -> recheck -> ENOSPC) ===")
print("frame-cache limit = storage_limit*5 = %d bytes [graphics.c:1570]" % (g.storage_limit * 5))
print("base image i=2 exists (image_count=%d). Adding 36-byte animation frames (a='f' default):" % g.image_count)
for n in range(8):
    print("  add frame %d -> %s     total_size=%d" % (n, addframe(2), g.disk_cache.total_size))
print("  add extra frame -> %s total_size=%d  (cache full even after reclaim -> ENOSPC [graphics.c:1571-1573])" % (addframe(2), g.disk_cache.total_size))
print("  edit existing frame (r=2,s=2,v=2) -> %s     (editing does not trigger quota)" % editframe(2, 2))
```
Invocation:

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. \
  python3 /tmp/kitty_obs/graphics_storage_frames.py
```
**`graphics_b4_guards_inproc.py`** - per-image zero-dim / unknown-format / PNG-size guards (B4 §4.3). Body:

```python
# /tmp/kitty_obs/graphics_b4_guards_inproc.py   [non-canonical in-process]
# Per-image size/format guards via the kitty_tests parse hooks. Each case uses a
# FRESH Screen so start_command.id state cannot leak between cases (contrast F11).
import base64
from kitty.fast_data_types import Screen
from kitty_tests import Callbacks, parse_bytes, BaseTest
bt = BaseTest(); bt.set_options()

def fresh():
    c = Callbacks(); s = Screen(c, 5, 5, 5, 10, 20, 0, c); return c, s
def run(cmdstr):
    c, s = fresh()
    c.clear(); parse_bytes(s, cmdstr.encode()); return c.wtcbuf

print("=== B4: per-image size/format guards (fresh screen; in-process NON-CANONICAL) ===")
print("zero-dim  s=0 v=1 i=1        [:646]        RESP=%r" % run('\x1b_Ga=t,f=24,s=0,v=1,i=1,q=0;/wAA\x1b\\'))
print("unknown-format f=99 i=1      [:651]        RESP=%r" % run('\x1b_Ga=t,f=99,s=1,v=1,i=1,q=0;/wAA\x1b\\'))
print("PNG-size  S=400000001 f=100 i=1 [:638]     RESP=%r" % run('\x1b_Ga=t,f=100,S=400000001,i=1,q=0;/wAA\x1b\\'))
```
Invocation:

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. \
  python3 /tmp/kitty_obs/graphics_b4_guards_inproc.py
```
**`graphics_f11_overdim.py`** - state-dependent over-dimension reply id (F11 §4.3). Body:

```python
# /tmp/kitty_obs/graphics_f11_overdim.py   [non-canonical in-process]
# Over-dimension response id is STATE-DEPENDENT. The transmit reply is built from
# self->currently_loading.start_command (graphics.c:2177/:2180), and the over-dim
# ABRT (:695) fires BEFORE initialize_load_data (called :716) assigns start_command=*g (:634) and
# BEFORE :717 sets start_command.id=iid. free_load_data (:103) does not clear
# start_command, so an over-dim reply inherits whatever id the LAST command that
# reached :634 (start_command=*g) left behind. Fresh Screen per case group.
import sys
from kitty.fast_data_types import Screen
from kitty_tests import Callbacks, parse_bytes, BaseTest
bt = BaseTest(); bt.set_options()

def fresh():
    c = Callbacks(); s = Screen(c, 5, 5, 5, 10, 20, 0, c); return c, s
def send(c, s, cmdstr):
    c.clear(); parse_bytes(s, cmdstr.encode()); return c.wtcbuf

OVERDIM_ID = '\x1b_Ga=t,f=24,s=20000,v=20000,i=%d,q=0;/wAA\x1b\\'
OVERDIM_NOID = '\x1b_Ga=t,f=24,s=20000,v=20000,q=0;/wAA\x1b\\'   # NO i= key at all
ZERODIM = '\x1b_Ga=t,f=24,s=0,v=1,i=%d,q=0;/wAA\x1b\\'

# A: fresh over-dim with id -> start_command.id still 0 (ABRT before it is set) -> :765 gate -> empty
c, s = fresh()
print("A fresh over-dim i=1 ONLY cmd            -> %r   (start_command.id=0 -> :765 gate -> empty)" % send(c, s, OVERDIM_ID % 1))
# C: zero-dim i=1 sets start_command.id=1 (fails AFTER :634); then over-dim i=1 reuses it
c, s = fresh()
print("C zero-dim i=1 (fails AFTER :634 sets start_command.id=1) -> %r" % send(c, s, ZERODIM % 1))
print("C then over-dim i=1 (ABRT :695 before :634; reuses start_command.id=1) -> %r" % send(c, s, OVERDIM_ID % 1))
# E: zero-dim i=9 sets start_command.id=9; then over-dim with NO id leaks i=9
c, s = fresh()
print("E zero-dim i=9 (sets start_command.id=9)  -> %r" % send(c, s, ZERODIM % 9))
print("E then over-dim NO id -> id LEAKS from start_command: %r" % send(c, s, OVERDIM_NOID))
```
Invocation:

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. \
  python3 /tmp/kitty_obs/graphics_f11_overdim.py
```
**`pending_transitions_inproc.py`** - pending-mode DECSET/DECRST/DECRQM transitions + "already current mode" log (C-part-1 §5). Body:

```python
# /tmp/kitty_obs/pending_transitions_inproc.py   [non-canonical in-process]
# Real VT parser + Screen state machine (no live PTY). Drives DECSET/DECRST/DECRQM
# for synchronized-update pending mode 2026 and the test-only pause_rendering()
# hook. Canonical control codes; the pause_rendering() calls in STEP 4 are the
# test hook used only to corroborate the "return False when already paused" branch
# (screen.c:2518). stdout is flushed around the redundant DECSET so the stderr
# log_error (screen.c:1176) interleaves inline. os._exit avoids a benign C-Screen
# teardown segfault at interpreter shutdown (post-observation noise).
import os, sys
from kitty.fast_data_types import Screen
from kitty_tests import Callbacks, parse_bytes, BaseTest
bt = BaseTest(); bt.set_options()
c = Callbacks(); s = Screen(c, 5, 5, 5, 10, 20, 0, c)

def pr(*a):
    print(*a); sys.stdout.flush()
def query():
    c.clear(); parse_bytes(s, b'\x1b[?2026$p'); return c.wtcbuf
def feed(b):
    c.clear(); parse_bytes(s, b); sys.stderr.flush()

pr("STEP 0  initial query 2026            -> %r" % query())
feed(b'\x1b[?2026h')
pr("STEP 1  after DECSET 2026h; query     -> %r" % query())
pr("STEP 2  DECSET 2026h AGAIN (expect 'already current mode' log on stderr):")
feed(b'\x1b[?2026h')                       # redundant DECSET -> screen.c:1176 log_error to stderr
pr("STEP 2  after 2nd DECSET; query       -> %r" % query())
feed(b'\x1b[?2026l')
pr("STEP 3  after DECRST 2026l; query     -> %r" % query())
pr("STEP 4  pause_rendering() 1st call    -> %s" % s.pause_rendering(True, 0))
pr("STEP 4  pause_rendering() 2nd call    -> %s" % s.pause_rendering(True, 0))
pr("STEP 4  query after hook-pause        -> %r" % query())
sys.stdout.flush(); sys.stderr.flush()
os._exit(0)
```
Invocation:

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. \
  python3 /tmp/kitty_obs/pending_transitions_inproc.py
```
### 10.4 Log filter

**`f_capture.awk`** - streams kitty stderr during the F overflow run, printing a contiguous unedited window (8 pre-context lines + the first overflow match + the next 30) and counting **every** occurrence, without materializing the multi-hundred-MB log. Body:

```awk
# /tmp/kitty_obs/f_capture.awk
# Streams kitty stderr and, without materializing the multi-hundred-MB log:
#   - keeps a rolling window of the last 8 non-overflow lines (pre-context);
#   - at the FIRST "Too much data..." overflow line, prints that pre-context, then
#     prints a contiguous window of the match + the next 30 overflow lines (31 total);
#   - counts EVERY overflow occurrence and prints the total at END.
/Too much data being sent to child/ {
    count++
    if (count == 1) {
        for (i = 1; i <= nbuf; i++) print buf[i]
        printing = 1
    }
    if (printing && printed < 31) { print; printed++ }
    next
}
{
    if (count == 0) {                       # maintain 8-line rolling pre-context (pre-crossing only)
        if (nbuf < 8) { nbuf++; buf[nbuf] = $0 }
        else { for (i = 1; i < 8; i++) buf[i] = buf[i+1]; buf[8] = $0 }
    }
}
END { print "TOTAL_OVERFLOW_OCCURRENCES=" count }
```
### 10.5 One-time setup & teardown recap

- **Setup:** Xvfb lifecycle with PID capture + `trap` cleanup, and the readiness wait, are in §1 (top). Build A / Build B commands are §1.1 / §1.2.
- **Teardown (idempotent):** a single `rm -rf /tmp/kitty_obs` removes **all** scripts and scratch logs; because the directory is entirely outside the repository this cannot touch tracked files. Verify with `git status --porcelain` (see §9): it lists only this document. Re-running the `rm` on an already-clean tree is a no-op (idempotent).
