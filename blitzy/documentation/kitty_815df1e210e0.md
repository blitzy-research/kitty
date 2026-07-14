# How kitty Regulates Terminal Data Flow (Flow Control & Backpressure)

> **Question answered.** As terminal output — particularly kitty graphics-protocol data — arrives faster than kitty can comfortably process and respond to, how does kitty decide whether to **buffer** it, **pause** intake, or **throttle** (slow) processing? What happens when **responses must be written back** to the program while the **output path is already under pressure**? **Where** in the code do these decisions live? How do they **show up at runtime** under overload, and does kitty **quietly adapt** or produce **visible signs**?

**Subject.** kitty terminal emulator.

- **Investigated source baseline:** HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` ("Wire up applying of font config"). Every `file:line` citation and every observation in this document is against that baseline.
- **Delivery vs. baseline (see §9):** this document is the **only** content added relative to the baseline. It is introduced by a Blitzy delivery commit layered on top of the baseline — the initial add was commit `3ee38c66df78f704971598ee3002223730a1fe5d`, and any later revision of this same file (including this one) is a further descendant commit. **No existing repository file is modified.** The baseline hash and the delivery hash are therefore distinct; the delivery HEAD is a descendant of the baseline, not equal to it.

**Method — run first, write from what was observed (Rule 1).** kitty was compiled from this checkout and driven through its **real PTY entry point** under overload. Each behavioral claim sits next to the **complete, unedited** command output that produced it (or, where a stream is unbounded, a **contiguous unedited window** plus a counted total — labelled as such), with a `file:line` citation and cause→effect reasoning.

**Evidence labels used throughout.**

- **observed (canonical)** — the **default release build** (`kitty/fast_data_types.so` = **1,253,792 bytes**, compiled `-O3 -DNDEBUG`, **no** debug macros) driven through a **real PTY**, or a default compiled constant read from that extension. All throughput / timing / magnitude / count numbers come only from this build.
- **observed (instrumented debug build)** — a separate **debug** build (`kitty/fast_data_types.so` = **6,287,848 bytes**, compiled `-Og -DDEBUG` plus `-DDEBUG_EVENT_LOOP -DDEBUG_POLL_EVENTS -DKITTY_PRINT_BYTES_SENT_TO_CHILD`) used **only** to expose event-loop poll timeouts, per-fd returned `revents`, and per-`write()` byte counts. The injected diagnostics are print-only and change no flow-control logic, but no throughput/timing magnitude is reported from this build.
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

A headless X server is needed because kitty is a GUI app:

```
Xvfb :99 -screen 0 1280x800x24 +extension GLX +render &
```

### 1.1 Build A — canonical (default release build)

This is the build used for **every** throughput/timing/magnitude/count measurement below.

```
CI=true python3 setup.py build --ignore-compiler-warnings
```

- Produces `kitty/fast_data_types.so` (**1,253,792 bytes**), `kitty/launcher/kitty`, `kitty/launcher/kitten`.
- Compiler flags for the C sources are the default release flags `-O3 -DNDEBUG`; **no** `DEBUG_EVENT_LOOP`, `DEBUG_POLL_EVENTS`, or `KITTY_PRINT_BYTES_SENT_TO_CHILD` macros are defined.
- `--ignore-compiler-warnings` sets `werror=''` at `setup.py:491` (the C-extension build) and `setup.py:1231` (the kittens/launcher build); the flag is declared at `setup.py:2003-2004`. It is required **only** because the environment's `wayland-protocols` is newer than kitty@`815df1e21` expects, which makes the **out-of-scope** file `glfw/wl_window.c` trip `-Werror`. It changes **no source** and does **not** affect flow-control behavior. On the canonical Docker image (matched deps) it is unnecessary.

### 1.2 Build B — instrumented debug build (diagnostics only)

Used **only** to expose poll timeouts (T), per-fd returned `revents`, and per-`write()` byte counts (F slow-drain / EAGAIN). No magnitude/throughput number is read from this build.

```
CI=true CC='cc -DDEBUG_POLL_EVENTS -DKITTY_PRINT_BYTES_SENT_TO_CHILD' \
  python3 setup.py build --debug --extra-logging=event-loop --ignore-compiler-warnings
```

- Produces `kitty/fast_data_types.so` (**6,287,848 bytes**); flags `-Og -DDEBUG -DKITTY_DEBUG_BUILD` plus `-DDEBUG_EVENT_LOOP` (from `--extra-logging=event-loop`), `-DDEBUG_POLL_EVENTS`, and `-DKITTY_PRINT_BYTES_SENT_TO_CHILD` (from `CC`).
- CC-injection works because `setup.py` reads `CC` via `shlex.split`; **no source file is edited**.
- `DEBUG_EVENT_LOOP` enables the GLFW main-loop trace (`pollForEvents final timeout: ...`, `glfw/backend_utils.c:298`, gated by `glfw/internal.h:837-838`). `DEBUG_POLL_EVENTS` enables the per-fd `revents` printer (`kitty/child-monitor.c:1550-1556`). `KITTY_PRINT_BYTES_SENT_TO_CHILD` enables `Wrote: %zd bytes:` (`kitty/child-monitor.c:1449-1451`).

> **AAP corrections (observed).** `KITTY_PRINT_BYTES_SENT_TO_CHILD` and `DEBUG_POLL_EVENTS` are **compile-time `#ifdef` macros**, not runtime environment variables, and `DEBUG_POLL_EVENTS` is **not** enabled by `--extra-logging`. `DEBUG_POLL_EVENTS` prints `revents` (what `poll()` **returned**), not the gated `.events` request mask (see §3.3 / F6).

### 1.3 Headless run template (real kitty app + real `child-monitor.c` I/O loop)

Exact, runnable (no placeholders). `PYTHONPATH=.` is the repository root; child scripts live under `/tmp/kitty_obs`:

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 \
  python3 /tmp/kitty_obs/<child_script>.py <args>
```

### 1.4 In-process harness template ([non-canonical in-process])

Real compiled `graphics.c`/`screen.c`/`vt-parser.c` logic through the `kitty_tests` parse hooks; the `Screen` is built exactly as `kitty_tests` does — the **same** `Callbacks` instance is passed twice (`Screen(c, lines, cols, scrollback, cw, ch, 0, c)`):

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=. python3 /tmp/kitty_obs/<harness>.py
```

### 1.5 Canonical defaults confirmed from the built (canonical) extension

```
input_delay   = 3 ms    [kitty/options/definition.py:878; kitty/options/types.py:536]
repaint_delay = 10 ms   [kitty/options/definition.py:866; kitty/options/types.py:567]
sync_to_monitor = True  [kitty/options/types.py:586]
VT_PARSER_BUFFER_SIZE          = 1048576 bytes ( 1 MiB )   [kitty/vt-parser.c:18  BUF_SZ]
VT_PARSER_MAX_ESCAPE_CODE_SIZE =  262144 bytes ( 256 KiB ) [kitty/vt-parser.c:21  MAX_ESCAPE_CODE_LENGTH]
GraphicsManager default storage_limit = 335544320 bytes ( 320 MiB ) [kitty/graphics.c:25/:78]
```

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
- **`ret == 0`** — `break`, retaining the queued data (`:1458-1460`). **[inferred]** from code (non-deterministic to force).
- **`EINTR`** — `continue` and retry (`:1462`). **[inferred]** from code.
- **`EWOULDBLOCK/EAGAIN`** — `break`, **keep buffered, retry on next `POLLOUT`** (`:1463`). **Observed** (F EAGAIN below).
- **other `errno`** — `perror("Call to write() to child fd failed, discarding data.")` then discard the whole buffer (`:1464-1465`). **[inferred]** from code; the `perror` is a documented visible sign.

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
[0.160] Failed to open systemd user bus with error: Connection refused
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
[5.590] Too much data being sent to child with id: 1, ignoring it
TOTAL_OVERFLOW_OCCURRENCES=845788
```

Run 2 — first-crossing line + total (stability, F7): `[5.581] Too much data being sent to child with id: 1, ignoring it` … `TOTAL_OVERFLOW_OCCURRENCES=845826`.

**Reading the evidence:** the **visible** overflow log (`kitty/child-monitor.c:342`) first fires at **t≈5.58–5.59 s** and then repeats **845,788 / 845,826** times in a 7 s run — two-run stable, and with **no rate-limiting** (every discarded enqueue logs). The message reads `id: 1` (the first child). This log appears on the **canonical** build (no debug macros), confirming it is canonical, not debug-gated.

> **Operational safety (F21).** This experiment is intentionally abusive: an earlier ~16 s probe produced **6,875,319** overflow lines totalling **458 MB** in a single log. Reproduce it **only** in a disposable, resource-bounded environment, and always bound it (short `timeout`, and a streaming counter/rolling-window capture as above) — never redirect the raw stream to disk unbounded.

**F — slow-drain (observed, instrumented debug build).** To watch draining, the child enqueues 200,000 DA2 replies (2.6 MiB) then reads 4 KiB every 60 ms. Counts are of **returned** `revents` (what `poll()` returned), tallied over the whole run by `grep | sort | uniq -c` (a count, not the raw stream):

```
child bytes actually drained (FINAL): read_total 2600000        # = 200000 x 13
returned revents (DEBUG_POLL_EVENTS, kitty/child-monitor.c:1550-1556):
  i:0 POLLIN  ~12519     (wakeup fd)
  i:1 POLLIN       1     (signal fd)
  i:2 POLLIN     130     (the query burst)
  i:2 POLLOUT ~12511     (child fd RETURNED writable)
  i:2 POLLHUP      1     (child exit)
distinct 'Wrote:' sizes (KITTY_PRINT_BYTES_SENT_TO_CHILD, all whole multiples of 13):
  156 (=12x13)  169 (=13x13)  247 (=19x13)  260 (=20x13)  234 (=18x13)  182 (=14x13)
```

The child drained **exactly 2,600,000 bytes** (= 200,000 × 13), and in this run every `write()` size happened to be a whole multiple of the 13-byte reply (e.g. `156 = 12×13`, `247 = 19×13`) — the drain proceeds in whatever chunks the PTY accepts as the slow reader frees space (which can also stop mid-reply, as the EAGAIN capture below shows).

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
#   -o confirm_os_window_close=0 python3 /tmp/kitty_obs/fill_eagain.py /tmp/kitty_obs/eagain_prog.txt 2>eagain_stderr.log
# child result file:  first_read 4095
```

The run produced exactly **84** `Wrote:` diagnostic entries (bounded); the first 83 are positive writes (every one a whole multiple of the 13-byte reply, e.g. `156 = 12×13`, `247 = 19×13`, `377 = 29×13`), and the **last two entries are the EAGAIN transition**. Because the `EWOULDBLOCK/EAGAIN` `break` (`kitty/child-monitor.c:1463`) skips `print_text`, the `Wrote: -1 bytes:` line has **no trailing content and no newline before it**, and the final positive write is cut **mid-reply**. Complete, unedited transition tail (the end of the `Wrote: 5632 bytes` payload directly followed by the `Wrote: -1 bytes` line):

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
- **Raw/direct `d` over-budget** → `EFBIG:Too much data`, `:533` (fires only for direct, non-PNG data whose accumulated size exceeds `MAX_DATA_SZ`; **[inferred]** — requires ~400 MB of actual payload, impractical in a bounded environment).

**Over-dimension response is state-dependent (F11).** The transmit reply is built from `lg = &self->currently_loading.start_command` (`kitty/graphics.c:2177`, used at `:2180`), **not** from the command's own `g`. The over-dimension `ABRT` at `:695` fires **before** `initialize_load_data` sets `start_command = *g` (`:634`, which also sets `start_command.id`, `:717`), and `free_load_data` (`:103`) does **not** clear `start_command`. So the id on an over-dimension reply is whatever the **last** command that reached `:634` left behind. Complete harness output, each case a **fresh process**, stable across two runs:

```
A fresh over-dim i=1 ONLY cmd            -> b''   (start_command.id=0 -> :765 gate -> empty)
C zero-dim i=1 (fails AFTER :634 sets start_command.id=1) -> b'\x1b_Gi=1;EINVAL:Zero width/height not allowed\x1b\\'
C then over-dim i=1 (ABRT :695 before :634; reuses start_command.id=1) -> b'\x1b_Gi=1;EINVAL:Image too large\x1b\\'
E zero-dim i=9 (sets start_command.id=9)  -> b'\x1b_Gi=9;EINVAL:Zero width/height not allowed\x1b\\'
E then over-dim NO id -> id LEAKS from start_command: b'\x1b_Gi=9;EINVAL:Image too large\x1b\\'
```

**Reading the evidence:** on a **fresh** screen an over-dimension-only command returns an **empty** response (`b''`) — the `ABRT` fires before any id is set, so `finish_command_response` sees `g->id == 0 && g->image_number == 0` and emits nothing (`:765`). But after a **prior** command left a `start_command.id` (Case C: id 1; Case E: id 9 — note the over-dimension command in E carried **no** id yet still replied with `i=9`), the over-dimension reply **inherits that stale id** and is **visible** (`EINVAL:Image too large`). The empty-response result is therefore **only** valid in a fresh, isolated harness. (The `ImportError: sys.meta_path is None` printed after each case is benign Python interpreter-shutdown noise, not part of the observation.)

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

The reply is built in a fixed `static char command_response[512]` (`kitty/graphics.c:302`). **Evidence B1 (observed, canonical, real PTY).** A child in raw mode transmits success (valid 1×1 RGB) and error (zero-width) commands with `q=0/1/2` and reads the APC replies. Complete captured responses (unedited), identical across two runs:

```
CASE success_q0   SENT=b'\x1b_Ga=t,f=24,s=1,v=1,i=1,q=0;/wAA\x1b\\'
              RESP=b'\x1b_Gi=1;OK\x1b\\'
CASE error_q0     SENT=b'\x1b_Ga=t,f=24,s=0,v=1,i=1,q=0;/wAA\x1b\\'
              RESP=b'\x1b_Gi=1;EINVAL:Zero width/height not allowed\x1b\\'
CASE success_q1   SENT=b'\x1b_Ga=t,f=24,s=1,v=1,i=1,q=1;/wAA\x1b\\'
              RESP=b''
CASE error_q1     SENT=b'\x1b_Ga=t,f=24,s=0,v=1,i=1,q=1;/wAA\x1b\\'
              RESP=b'\x1b_Gi=1;EINVAL:Zero width/height not allowed\x1b\\'
CASE success_q2   SENT=b'\x1b_Ga=t,f=24,s=1,v=1,i=1,q=2;/wAA\x1b\\'
              RESP=b''
CASE error_q2     SENT=b'\x1b_Ga=t,f=24,s=0,v=1,i=1,q=2;/wAA\x1b\\'
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
- `kitty/graphics.c:302` — `static char command_response[512]`; `:519` — `ABRT` macro; `:521` — `MAX_DATA_SZ` (400 MB); `:533` — raw `EFBIG`; `:638` — PNG `EINVAL`; `:646` — zero-dim `EINVAL`; `:651` — unknown-format `EINVAL`; `:674` — `MAX_IMAGE_DIMENSION 10000`; `:695` — over-dim `EINVAL:Image too large`; `:1553` — animation-frame sibling dimension guard.
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
| Inbound 1 MiB buffer full → `POLLIN` dropped | kitty stops reading; child blocks in `write()` on the full PTY; byte counter freezes for the stall window | **Silent** (no log; only the blocking write) | E2 §2.2 (canonical; stimulus non-canonical) |
| Inbound pacing (buffer + gating + delay batching) | producer bounded to ~112 MiB/s vs ~108 GiB/s to `/dev/null`; main-loop `pollForEvents` timeout settles at 0.003 s = `input_delay` | **Silent** | E1, T §2.3 (E1 canonical; T instrumented) |
| Over-long / malformed escape code | `... escape code too long (N bytes), ignoring it` / `CSI escape too long ...` logged | **Visible** (`kitty/vt-parser.c:419`, `:831`) | D §2.4 (canonical) |
| Outbound `write_buf` 100 MiB cap | `Too much data being sent to child with id: 1, ignoring it` logged en masse (845,788 / 845,826 in 7 s), data discarded | **Visible** (`kitty/child-monitor.c:342`) | F §3.3 (canonical) |
| Outbound `EAGAIN` retention | `write()` returns `-1`; buffer retained, retried on next `POLLOUT`; no data lost | **Silent** (visible only in debug `Wrote: -1`) | F EAGAIN §3.3 (instrumented) |
| Graphics storage quota | oldest images LRU-evicted; `image_count` capped; later `ENOENT` if referenced | **Silent** (eviction) | B2 §4.1 (non-canonical in-process) |
| Graphics frame cache 5× | reclaim → recheck → `ENOSPC` APC error reply | **Visible** (error reply) | B3 §4.2 (non-canonical in-process) |
| Graphics size/format guards | `EINVAL`/`EFBIG` APC error reply (empty only for a *fresh* over-dimension; otherwise inherits a stale id) | **Visible** (state-dependent for over-dim) | B4, F11 §4.3 (non-canonical in-process) |
| Graphics `q=` suppression | `OK` (q≥1) or `OK`+errors (q=2) withheld at client's request | **Silent** | B1 §4.4 (canonical) |
| Synchronized-update pause (DECSET 2026) | rendering held; 2000 ms default; auto-unpause at expiry or on DECRST | **Silent** (state-only; query `;1$y`/`;2$y`) | C §5 (canonical + non-canonical in-process) |
| Redundant/late DECSET 2026 while paused | `Pending mode change to already current mode...` logged | **Visible** (`kitty/screen.c:1176`) | C §5 (non-canonical in-process) |

---

## 8. Observed-vs-inferred ledger

**Observed (canonical — default-config kitty via real PTY, or default compiled constants):**
- Default option values and constants (§1.5).
- Inbound pacing E1 (~112 MiB/s via kitty vs ~108 GiB/s to `/dev/null`), **two runs** (§2.3).
- Child stall E2 (byte counter frozen across the stall; 2.8 s progress-file gap), **two runs** — stimulus (`SIGSTOP`) labelled **[non-canonical stimulus]** (§2.2).
- Escape-code handling D (complete-ST accept; OSC-52 continue; OSC over-length `:419`; CSI truncation `:831`), **two runs** (§2.4).
- Outbound overflow F (`Too much data...` first at ~5.58–5.59 s; 845,788 / 845,826 occurrences), **two runs** (§3.3).
- Graphics `q=` suppression B1, **two runs** (§4.4).
- Synchronized-update 2000 ms timeout auto-unpause C-part-2, **two runs** (§5).

**Observed (instrumented debug build; diagnostics only, behavior identical to canonical):**
- GLFW main-loop poll timeouts and the separate child-`io_loop` `revents` (T, §2.3); per-fd returned `revents` and whole-multiple-of-13 `Wrote:` sizes with an exact 2,600,000-byte drain (F slow-drain, §3.3); the `EAGAIN` `Wrote: -1` retain (F EAGAIN, §3.3).

**Observed ([non-canonical in-process] — real C logic via `kitty_tests` parse hooks; B2/B3 additionally use a reduced `storage_limit`):**
- Storage LRU eviction B2, frame-cache reclaim→recheck→`ENOSPC` B3, per-image guards B4, over-dimension state-dependency F11 (**two runs**, fresh process per case), pending-mode transitions + "already current mode" log C-part-1.

**[inferred] (read from code, not separately instrumented):**
- The `run_worker` force-parse-near-full clause `read.sz + 16*1024 > BUF_SZ` (`kitty/vt-parser.c:1425`).
- The `write_to_child` `ret==0` break (`:1458-1460`), `EINTR` continue (`:1462`), and hard-error `perror`+discard (`:1464-1465`).
- The internal parser-full trigger of the inbound pause (E2 observes only its OS-level effect).
- The raw/direct `EFBIG` guard (`kitty/graphics.c:533`) — needs ~400 MB of actual payload.
- That `DEBUG_POLL_EVENTS` reports `.revents`, so the `POLLIN`-gating *decision* (`:1501`) is confirmed by inspection while its *effect* is what E2 observes.

**Stability (F7).** Every **timing/magnitude** condition was run at least twice with consistent results, and both runs are reported: E1 (FINAL totals table), E2 (freeze at a single value; ~2.8 s gap both runs), D (identical 5-line stderr), F (first-crossing ~5.58–5.59 s; 845,788 vs 845,826 occurrences), B1 (identical responses), F11 (identical per-case responses), C-part-2 (identical `;1$y`/`;2$y` bracketing). Where a value is not from the canonical build (T, F slow-drain, F EAGAIN) it is drawn from the instrumented build and labelled accordingly.

**Exact commands.** Build A (canonical), Build B (instrumented), the headless run template, and the in-process harness template are listed verbatim in §1, and each evidence block above includes the exact command and inline child-script body that produced it.

---

## 9. Investigation hygiene

- **Repository proof.** Investigated source baseline: `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` ("Wire up applying of font config"). `git diff --stat <baseline>..HEAD` shows **exactly one** changed path — `A blitzy/documentation/kitty_815df1e210e0.md` — i.e. one CREATE and zero UPDATE/DELETE of any existing file. The delivery HEAD is a **descendant** of the baseline (the initial add was `3ee38c66df78f704971598ee3002223730a1fe5d`; this revision is a further descendant); it is **not** equal to the baseline. The document does **not** claim HEAD remains the baseline.
- **No source modified.** All named source/header/config/test/build/protocol files are byte-identical to the baseline; the only addition is this document under `blitzy/documentation/`.
- **Temporary artifacts.** All observation scripts and logs lived **outside** the repository under `/tmp/kitty_obs` (`flood_plain.py`, `fill_writebuf.py`, `fill_slowdrain.py`, `fill_eagain.py`, `esc_boundaries.py`, `tiny_input.py`, `graphics_*.py`, `pending_*.py`, and `*.log`/`*.txt` scratch) and were **removed** after evidence capture.
- **Build artifacts** (`build/`, `kitty/fast_data_types.so`, `kitty/launcher/kitty`, `kitty/launcher/kitten`) are git-ignored and do not appear as tracked changes; `git status --porcelain` (tracked) is empty apart from this document.
