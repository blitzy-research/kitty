# How kitty Regulates Terminal Data Flow (Flow Control & Backpressure)

> **Question answered.** As terminal output — particularly kitty graphics-protocol data — arrives faster than kitty can comfortably process and respond to, how does kitty decide whether to **buffer** it, **pause** intake, or **throttle** (slow) processing? What happens when **responses must be written back** to the program while the **output path is already under pressure**? **Where** in the code do these decisions live? How do they **show up at runtime** under overload, and does kitty **quietly adapt** or produce **visible signs**?

**Subject.** kitty terminal emulator, branch `kitty_815df1e210e0`, HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` ("Wire up applying of font config").

**Method — run first, write from what was observed.** kitty was compiled from this checkout and driven through its **real PTY entry point** under overload. Each behavioral claim sits next to the **complete, unedited** command output that produced it, with a `file:line` citation and cause→effect reasoning. Evidence is labelled:

- **observed (canonical)** — default-configuration kitty driven through a real PTY, or default compiled constants read from the built extension;
- **observed (debug print-only build)** — a build with extra `printf`/`fprintf` diagnostics injected via `CC=...`; the print statements do not change flow-control logic, so behavior is identical to canonical;
- **[non-canonical]** — real kitty C logic exercised through in-process `kitty_tests` parse hooks (no real PTY) and/or with a deliberately reduced limit to make an eviction observable in-process;
- **[inferred]** — derived from reading code, not separately instrumented.

---

## 0. Executive answer

Under overload kitty applies flow control on **two independent axes** in the same poll-based event loop, plus a set of **graphics-specific** memory/response limits.

**(a) Buffer vs. pause vs. throttle (inbound / read path).**
- **Buffer:** inbound bytes from the child are read into a **fixed 1 MiB** VT-parser buffer (`BUF_SZ (1024u*1024u)`, `kitty/vt-parser.c:18`). This is the only inbound buffer; it does not grow.
- **Pause:** once that buffer is full, `vt_parser_has_space_for_input()` returns false and the event loop **stops registering the child fd for `POLLIN`** (`children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(...) ? POLLIN : 0;`, `kitty/child-monitor.c:1501`). kitty simply **stops reading**; the kernel PTY buffer then fills and the child's next `write()` **blocks**. This is OS-level backpressure and it is **silent**.
- **Throttle:** parsing/rendering are **batched** by two small delays — `input_delay` (default **3 ms**) and `repaint_delay` (default **10 ms**). `run_worker` defers parsing until `flush || time_since_new_input >= OPT(input_delay) || read.sz + 16*1024 > BUF_SZ` (`kitty/vt-parser.c:1425`); i.e. it batches by 3 ms but **abandons batching and force-parses when the 1 MiB buffer is nearly full**.

**(b) Response write-back under output pressure (outbound / write path).**
Responses to the child (e.g. cursor-position/device-attributes replies, graphics `OK`/error replies) are queued into a **growable `write_buf`** via `schedule_write_to_child`. They are drained only when the child fd signals writable — the loop requests `POLLOUT` only while data is queued (`events |= (screen->write_buf_used ? POLLOUT : 0);`, `kitty/child-monitor.c:1503`). If the actual `write()` returns `EAGAIN/EWOULDBLOCK`, the data is **kept buffered and retried on the next `POLLOUT`** (`kitty/child-monitor.c:1463`). If the queue would exceed **100 MiB**, the new data is **discarded and an error is logged** (`kitty/child-monitor.c:341-344`, message at `:342`).

**(c) Where in the code.** See the reference map in §6; the mechanisms live in `kitty/vt-parser.c`, `kitty/child-monitor.c`, `kitty/graphics.c`, and `kitty/screen.c`, with defaults in `kitty/options/definition.py`.

**(d) Runtime manifestation.** Under a flooding child kitty **paces the producer** (a child writing to kitty runs ~3× slower than the same child writing to `/dev/null`; §2.3) and, if reading is stopped, the child **freezes blocked in `write()`** (§2.2, Evidence E2). Under a response flood the outbound queue hits 100 MiB and kitty logs `Too much data being sent to child with id: 1, ignoring it` millions of times (§3.3, Evidence F). Graphics overload surfaces as LRU eviction, `ENOSPC`, and `EINVAL`/`EFBIG` APC error replies (§4).

**(e) Silent vs. visible.**
- **Silent adaptation:** the inbound `POLLIN` pause and PTY backpressure (no log; the child just blocks), routine buffering, and client-requested response suppression (`q=`).
- **Visible signs:** the outbound `Too much data being sent to child...` overflow log (`kitty/child-monitor.c:342`); graphics error APC replies (`EINVAL`, `ENOSPC`, `EFBIG`); the pending-mode `Pending mode change to already current mode...` log (`kitty/screen.c:1176`); and the parser `escape code too long` log (`kitty/vt-parser.c:419`).

**(f) Investigation hygiene.** All observation scripts were kept **outside** the repository (under `/tmp`) and removed afterwards; build artifacts (`build/`, `kitty/fast_data_types.so`, `kitty/launcher/kitty`) are git-ignored. The source tree is unchanged apart from this document.

---

## 1. How the investigation was run (exact commands)

kitty ships **without** a pre-built C extension or launcher in this checkout, so it was compiled first (Rule 1).

**Canonical build** (default configuration; produces `kitty/fast_data_types.so` and `kitty/launcher/kitty`):

```
python3 setup.py build --debug --extra-logging=event-loop --ignore-compiler-warnings
```

- `--extra-logging=event-loop` enables the `DEBUG_EVENT_LOOP` trace (`loop tick`, `pollForEvents final timeout`, …). It is the only `--extra-logging` choice (`setup.py` `choices=('event-loop',)`).
- `--ignore-compiler-warnings` (`setup.py:2002`, sets `werror=''`) is required **only** because the environment's `wayland-protocols` is newer than kitty@815df1e21 expects, which makes `glfw/wl_window.c` trip `-Werror=switch`. It changes **no source** and does **not** affect flow-control behavior. (On the canonical Docker image this would not be needed.)

**Debug print-only build** (adds diagnostics; flow-control logic unchanged):

```
CC='cc -DDEBUG_POLL_EVENTS -DKITTY_PRINT_BYTES_SENT_TO_CHILD' \
  python3 setup.py build --debug --extra-logging=event-loop --ignore-compiler-warnings
```

- This injects two **compile-time** diagnostics: `DEBUG_POLL_EVENTS` (`kitty/child-monitor.c:1550-1554`, prints per-fd `revents`) and `KITTY_PRINT_BYTES_SENT_TO_CHILD` (`kitty/child-monitor.c:1449`, prints each `write()` to the child). CC-injection works because `setup.py` reads `CC` via `shlex.split` (`setup.py:300-312`); **no source file is edited**. Instrumented `fast_data_types.so` = 6145784 B vs canonical 6143160 B (extra print code only).

**Headless run** (real kitty app + real `child-monitor.c` event loop; Xvfb for GL):

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 PYTHONPATH=<repo> \
  kitty/launcher/kitty --config NONE -o confirm_os_window_close=0 <child-command>
```

**In-process harness** (real compiled `graphics.c`/`screen.c`/`vt-parser.c` logic driven through `kitty_tests` parse hooks — labelled **[non-canonical]** delivery because it bypasses the real PTY, though the C logic executed is identical):

```
PYTHONPATH=<repo> python3 /tmp/obs_harness.py
```

> **AAP corrections (observed).** (1) `KITTY_PRINT_BYTES_SENT_TO_CHILD` is a **compile-time `#ifdef` macro**, not a runtime environment variable. (2) `DEBUG_POLL_EVENTS` is a **compile-time macro**, not enabled by `--extra-logging`. (3) `DEBUG_POLL_EVENTS` prints `revents` (what `poll()` returned), not the gated `.events` request mask; the `POLLIN`-gating *decision* (`kitty/child-monitor.c:1501`) is confirmed by code inspection and its *effect* is directly observed in Evidence E2.

Canonical defaults confirmed from the built extension (observed, canonical):

```
input_delay   = 3 ms   [options/definition.py:878]
repaint_delay = 10 ms  [options/definition.py:866]
sync_to_monitor = True
VT_PARSER_BUFFER_SIZE          = 1048576 bytes ( 1024 KiB ) [vt-parser.c:18 BUF_SZ]
VT_PARSER_MAX_ESCAPE_CODE_SIZE = 262144 bytes ( 256 KiB ) [vt-parser.c:21]
GraphicsManager default storage_limit = 335544320 bytes ( 320 MiB ) [graphics.c:25]
```

---

## 2. Inbound (read) path — buffer, pause, throttle

### 2.1 Buffer — a fixed 1 MiB parser buffer

All child output flows through the VT parser, whose input buffer is a compile-time constant:

```c
// kitty/vt-parser.c:18
#define BUF_SZ (1024u*1024u)
// kitty/vt-parser.c:21
#define MAX_ESCAPE_CODE_LENGTH (BUF_SZ / 4u)   // 256 KiB
```

`read_bytes()` (`kitty/child-monitor.c`) obtains a write region via `vt_parser_create_write_buffer` (`kitty/vt-parser.c:1451`), whose available space is `BUF_SZ − offset`; if space is `0` it returns **without reading**. So inbound data is bounded by this **1 MiB** buffer — it does not grow. A single escape code is separately bounded to **256 KiB** (`MAX_ESCAPE_CODE_LENGTH`); an escape code that never terminates is dropped with a visible log (§2.4/Evidence D).

### 2.2 Pause — stop registering POLLIN → OS PTY backpressure (silent)

On every loop iteration the child fd's polled events are recomputed. Reading is gated on parser space:

```c
// kitty/child-monitor.c:1501
children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;
```

```c
// kitty/vt-parser.c:1477  (has-space predicate)
vt_parser_has_space_for_input(const Parser *p) { ... ans = self->read.sz + self->write.pending < BUF_SZ; ... }
```

**Cause→effect:** when the 1 MiB buffer is full, `vt_parser_has_space_for_input` returns false → the child fd is **not** registered for `POLLIN` → kitty stops reading → the kernel PTY buffer fills → the child's next `write()` **blocks**. kitty sends **no** in-band throttle signal to the child; the pause is realized purely by *not reading*, so it is **silent**.

**Evidence E2 (observed, canonical).** A child (`/tmp/flood_plain.py`) writes `'x'*65536` in a loop to its stdout (the PTY) and records `(bytes_written, elapsed)` to a side file after each `write()`. Mid-flood, the kitty reader process is `SIGSTOP`-ed (an external stimulus that reproduces the *effect* of kitty ceasing to read) and later `SIGCONT`-ed:

```
kitty(reader)=40161  child(python)=40229
t~3s RUNNING  : bytes=266403840 2.815 | child_stat=Rs+
>>> SIGSTOP kitty 40161 (stat=Tl) -- kitty stops reading the PTY
   stop+1s : bytes=267190272 2.819 | child_stat=Ss+
   stop+2s : bytes=267190272 2.819 | child_stat=Ss+
   stop+3s : bytes=267190272 2.819 | child_stat=Ss+
   stop+4s : bytes=267190272 2.819 | child_stat=Ss+
   stop+5s : bytes=267190272 2.819 | child_stat=Ss+
>>> SIGCONT kitty 40161 (stat=Rl) -- kitty resumes reading
   cont+1s : bytes=361758720 8.855 | child_stat=Rs+
   cont+2s : bytes=457179136 9.863 | child_stat=Ss+
   cont+3s : bytes=552599552 10.870 | child_stat=Ss+
cleanup done
```

**Reading the evidence:** while kitty is stopped the child's byte counter is **frozen** for the full 5 s (`267190272`, one unique value) and the child sits in state `S` (sleeping, blocked in `write()`); on `SIGCONT` it immediately resumes. Between the last running sample and the freeze the child advanced only `267190272 − 266403840 = 786432 B = 768 KiB` — i.e. it filled the kernel PTY buffer plus the partial 1 MiB parser buffer (under ~1 MiB of headroom) and then blocked. A second run was identical (frozen at `269877248`). There is **no log line** for this pause — the only signal is the blocking `write()`, confirming it is **silent**.

> **Faithful labelling.** `SIGSTOP` is an external stand-in that forces kitty to stop reading; it reproduces the *effect* of the internal pause (`POLLIN` gated off when the parser buffer is full, `kitty/child-monitor.c:1501` + `kitty/vt-parser.c:1477`). Both produce identical OS-level PTY backpressure that blocks the child's `write()`.

### 2.3 Throttle — pacing under sustained flood

Even without a hard stop, a child cannot outrun kitty's read+parse+render rate. Throughput pacing (observed, canonical):

```
--- E1: THROUGHPUT PACING (producer rate is bounded by kitty's consume rate) ---
Command (baseline):  timeout 5 python3 /tmp/flood_plain.py 3 /tmp/base_prog.txt >/dev/null
  child -> /dev/null : 779.7 MiB in 3.00s = 259.9 MiB/s
Command (via kitty): DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 kitty/launcher/kitty --config NONE \
                     -o confirm_os_window_close=0 python3 /tmp/flood_plain.py 15 /tmp/flood_plain_progress.txt
  child -> real PTY  : 1310.1 MiB in 15.00s = 87.3 MiB/s
  => kitty paces the producer ~3x slower than /dev/null: the child cannot outrun kitty's
     read+parse+render rate (inbound backpressure). Plain bytes parse cheaply, so pacing is modest.
```

The batching that produces this pacing is the **`input_delay`** window. `run_worker` defers parsing until:

```c
// kitty/vt-parser.c:1425
if (flush || pd->time_since_new_input >= OPT(input_delay) || self->read.sz + 16 * 1024 > BUF_SZ) {
```

and the I/O loop waits up to `input_delay` before waking the render thread:

```c
// kitty/child-monitor.c:1506-1509
if (has_pending_wakeups) {
    now = monotonic();
    monotonic_t time_delta = OPT(input_delay) - (now - last_main_loop_wakeup_at);
    if (time_delta >= 0) ret = poll(children_fds, self->count + EXTRA_FDS, monotonic_t_to_ms(time_delta));
    else ret = 0;
```

**Evidence T (observed, canonical).** A modest periodic producer (`printf` a line, `sleep 0.1`, ×30) run under the debug event-loop trace shows the poll timeout settling on the 3 ms window:

```
Observed DEBUG_EVENT_LOOP poll timeouts (top):
     29 pollForEvents final timeout: 0.003     <-- 0.003s = 3 ms = OPT(input_delay) default
      6 pollForEvents final timeout: 0.498
      3 pollForEvents final timeout: 0.000
      ... (longer idle/cursor-blink timeouts)
```

**Cause→effect:** when input has arrived and a wakeup is pending, the loop waits up to `input_delay` (3 ms) before rendering, batching bursty input into fewer render passes; `repaint_delay` (10 ms) throttles rendering similarly. **[inferred]** The `self->read.sz + 16*1024 > BUF_SZ` clause at `kitty/vt-parser.c:1425` (abandon batching and force-parse when the 1 MiB buffer is nearly full) is confirmed by reading the code but was not separately instrumented.

### 2.4 Parser escape-code / buffer boundary (visible on overflow)

An escape code that never terminates cannot grow without bound. **Evidence D [non-canonical]** (real `vt-parser.c` driven via test parse hooks):

```
[0.051] [PARSE ERROR] VTE_APC escape code too long (300001 bytes), ignoring it
[0.053] [PARSE ERROR] VTE_APC escape code too long (1048574 bytes), ignoring it
[0.053] [PARSE ERROR] Unknown char after ESC: 0x5c
```

- D1: an APC of 300000 `'a'` bytes with no ST → rejected once it exceeds the 256 KiB `MAX_ESCAPE_CODE_LENGTH` (message source `kitty/vt-parser.c:419`, guard at `:406`).
- D2: an APC of `'a'*BUF_SZ` → accumulation is bounded at ~`BUF_SZ` (1 MiB) then rejected (`1048574 bytes`), demonstrating the hard 1 MiB ceiling. The trailing `Unknown char after ESC: 0x5c` is the stray `\` (ST tail) after the discarded code.

---

## 3. Outbound (write-to-child) path — responses under output pressure

### 3.1 Queue + 100 MiB cap (the clearest visible sign)

Responses are appended to the per-screen `write_buf` by the `schedule_write_to_child` machinery, which enforces a hard ceiling:

```c
// kitty/child-monitor.c:341-344
if (screen->write_buf_used + sz > 100 * 1024 * 1024) { \
    log_error("Too much data being sent to child with id: %lu, ignoring it", id); \
    screen_mutex(unlock, write); \
    break; \
```

Below the cap the buffer is grown (`realloc`) to hold the data; at/above 100 MiB the new data is **discarded** and the error is logged. `log_error` here is **unconditionally compiled** (not behind any `#ifdef`), so this line is the canonical *visible sign* of outbound overflow.

### 3.2 POLLOUT-driven draining + EAGAIN retention

The child fd is registered for `POLLOUT` **only while** output is queued:

```c
// kitty/child-monitor.c:1503
children_fds[EXTRA_FDS + i].events |= (screen->write_buf_used ? POLLOUT  : 0);
```

`write_to_child` drains the buffer when the fd is writable, and handles a full PTY gracefully:

```c
// kitty/child-monitor.c:1461-1465
} else {
    if (errno == EINTR) continue;
    if (errno == EWOULDBLOCK || errno == EAGAIN) break;   // keep buffered, retry next POLLOUT
    perror("Call to write() to child fd failed, discarding data.");
    written = screen->write_buf_used;
```

**Cause→effect:** if the child is not reading, the kernel PTY fills and `write()` returns `EAGAIN/EWOULDBLOCK`; kitty **breaks out and keeps the data buffered**, retrying on the next `POLLOUT`. Only a genuine error (other `errno`) triggers `perror(...)` and discard. Thus outbound data is retained and retried under transient pressure, and only bounded/discarded by the 100 MiB cap.

### 3.3 Evidence F — driving the outbound path to the cap

A child (`/tmp/flood_da2.py`) floods **DA2 requests** `ESC [ > c` (`b'\x1b[>c'*16384` per `write()`; a 3.25× amplifier: 4 bytes in → 13 out, `ESC[>1;4000;35c`), **disables ECHO+ICANON** on its controlling tty (so kitty's own replies are not echoed back into its input), and **never reads its stdin** (so kitty's replies cannot drain). This makes kitty's `write_buf` grow until the 100 MiB cap discards.

**Evidence F — baseline (observed, canonical; non-instrumented `--debug` build):**

```
child wrote: 67108864 bytes DA2 written at t=19.78s
overflow log occurrences: 11855671
first occurrence at line 317; timestamp t=6.011s

--- unedited context around first occurrence (lines 315-319, timestamps intact) ---
[5.873] other dispatch done
[5.873] --------- loop tick, wakeups_happened: 1 ----------
Processing global state[6.011] Too much data being sent to child with id: 1, ignoring it
[6.011] Too much data being sent to child with id: 1, ignoring it
[6.011] Too much data being sent to child with id: 1, ignoring it
```

**Evidence F — instrumented (observed, debug print-only build; behavior identical):**

```
child wrote: 67108864 bytes DA2 written at t=19.06s

overflow log FIRST occurrence:
408:Processing global state[6.183] Too much data being sent to child with id: 1, ignoring it

overflow log unique text:  Too much data being sent to child with id: 1, ignoring it
overflow log occurrences (instrumented run): 12379950

child fd (i:2) POLLOUT requested count: 91
child fd (i:2) POLLIN  count:           18466

sample write() sizes to child (KITTY_PRINT_BYTES_SENT_TO_CHILD) - small because child never reads:
     37 Wrote: 156 bytes
     25 Wrote: 169 bytes
      6 Wrote: 182 bytes
      4 Wrote: 260 bytes
      3 Wrote: 455 bytes
      3 Wrote: 234 bytes
```

**Reading the evidence:**
- The **visible** overflow log `Too much data being sent to child with id: 1, ignoring it` (`kitty/child-monitor.c:342`) fires — ~11.86M times (baseline) / ~12.38M times (instrumented), first at **t≈6.0 s / 6.2 s**. It appears in **both** builds, proving it is canonical (not gated by any debug macro).
- `POLLOUT` is requested only when the buffer is non-empty (91 times here, `kitty/child-monitor.c:1503`), and actual `write()`s are **tiny** (156/169/182… bytes) because the child never drains its stdin → PTY full → most writes hit `EAGAIN` → the buffer grows to 100 MiB → discard.
- Stability: both builds cross the cap at ~6 s (6.011 vs 6.183), confirming the behavior is reproducible.

### 3.4 Graphics responses use this SAME path (not privileged)

A graphics reply is dispatched as an APC sequence onto the shared output buffer:

```c
// kitty/screen.c:1050
if (response != NULL) write_escape_code_to_child(self, ESC_APC, response);
```

`write_escape_code_to_child` (`kitty/screen.c:979`) ultimately calls `schedule_write_to_child`, so a graphics `OK`/error reply is subject to the **same** 100 MiB cap and `POLLOUT` draining as any other output. There is **no** graphics-specific write prioritization. (Evidence F uses DA2 replies, which travel this identical path.)

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

When stored image data exceeds `storage_limit`, `apply_storage_quota` (`kitty/graphics.c:290-296`, invoked at `:2184`) removes unreferenced images then evicts **oldest-first** (`HASH_SORT(..., oldest_img_first)` + a `remove_image` loop) until back under quota.

**Evidence B2 [non-canonical]** (real `graphics.c`; `storage_limit` deliberately reduced to 72 bytes so eviction is observable in-process — same code path as the 320 MiB default):

```
##### B2. STORAGE QUOTA + LRU EVICTION (graphics.c:290 apply_storage_quota, invoked graphics.c:2184) #####
  CANONICAL default storage_limit = 335544320 = 320 MiB
  [NON-CANONICAL] storage_limit reduced to 72 bytes so LRU eviction is observable (same code path as 320 MiB default)
  transmit i=1 (a=T): OK | image_count= 1 | present= [1]
  transmit i=2 (a=T): OK | image_count= 2 | present= [1, 2]
  transmit i=3 (a=T): OK | image_count= 2 | present= [2, 3]  <== oldest i=1 EVICTED (LRU), count capped
```

### 4.2 Animation frame cache — 5× quota → ENOSPC

```c
// kitty/graphics.c:1570
if (is_new_frame && cache_size(self) + load_data->data_sz > self->storage_limit * 5) {
// kitty/graphics.c:1573
        ABRT("ENOSPC", "Cache size exceeded cannot add new frames");
```

**Evidence B3 [non-canonical]** (frame cache ceiling = `storage_limit*5`):

```
##### B3. ANIMATION FRAME-CACHE 5x QUOTA -> ENOSPC (graphics.c:1570) #####
  frame-cache ceiling = storage_limit*5 = 360 bytes; each frame = 36 bytes
  transmit BASE image i=2 (a=T): OK
  added frames one-by-one until failure: ['OK', 'OK', 'OK', 'OK', 'OK', 'OK', 'OK', 'OK', 'OK', 'ENOSPC']
  => OK repeats then ENOSPC when cache_size + data_sz > storage_limit*5
```

(Frames require a base image first; an `a=f` frame command targeting a non-existent image returns `ENOENT`, `kitty/graphics.c:2199`.)

### 4.3 Per-image size / format guards

**Evidence B4 [non-canonical]** (each guard is a distinct `ABRT` path in `graphics.c`):

```
##### B4. PER-IMAGE SIZE/FORMAT GUARDS (graphics.c ABRT paths) #####
  unknown format f=99            [graphics.c:651]: -> b'\x1b_Gi=2;EINVAL:Unknown image format: 99\x1b\\'
  zero width s=0                 [graphics.c:646]: -> b'\x1b_Gi=10;EINVAL:Zero width/height not allowed\x1b\\'
  PNG data > MAX_DATA_SZ(~400MB) [graphics.c:521,533]: -> b'\x1b_Gi=11;EINVAL:PNG data size too large\x1b\\'
  dimension > MAX_IMAGE_DIMENSION 10000 [graphics.c:697]: -> b''
     ^ EMPTY response is EXPECTED: ABRT('EINVAL','Image too large') at graphics.c:697 fires
       BEFORE start_command.id is set at graphics.c:716, so finish_command_response (g->id==0) emits nothing.
       image rejected -> grman image_count = 0 (0 = not stored)
```

**Key nuance (observed + explained):** most guards emit a proper `EINVAL`/`EFBIG` APC error reply, but the **`Image too large`** guard (`kitty/graphics.c:697`, `MAX_IMAGE_DIMENSION = 10000`) fires **before** `start_command.id` is assigned (`kitty/graphics.c:716`); `finish_command_response` then sees `g->id == 0` and emits **nothing** — the image is still rejected (`image_count = 0`). This is why an over-dimension image returns an empty response while a zero-dimension image returns a visible `EINVAL`.

### 4.4 Response suppression — the `q=` key

```c
// kitty/graphics.c:761-764  (finish_command_response)
bool is_ok_response = !command_response[0];
if (g->quiet) {
    if (is_ok_response || g->quiet > 1) return NULL;
}
```

The reply is built in a fixed `static char command_response[512]` (`kitty/graphics.c:302`). The client-supplied `q=` key controls suppression: `q=1` suppresses **OK** replies; `q=2` suppresses **OK and error** replies.

**Evidence B1 [non-canonical]** (real `finish_command_response`):

```
##### B1. GRAPHICS q= RESPONSE SUPPRESSION (graphics.c:759 finish_command_response) #####
 SUCCESS/OK case (valid 1x1 RGB, i=1):
  q=0 (omitted): -> b'\x1b_Gi=1;OK\x1b\\'   
  q=1          : -> b''   (SUPPRESSED)
  q=2          : -> b''   (SUPPRESSED)
 ERROR case (unknown format f=99, i=2):
  q=0 (omitted): -> b'\x1b_Gi=2;EINVAL:Unknown image format: 99\x1b\\'   
  q=1          : -> b'\x1b_Gi=2;EINVAL:Unknown image format: 99\x1b\\'   
  q=2          : -> b''   (SUPPRESSED)
```

**Cause→effect:** `q=` is the graphics protocol's client-side lever to reduce return-path pressure — a well-behaved client that does not need confirmations sets `q=1`/`q=2` to keep kitty from enqueuing `OK`/error replies onto the shared 100 MiB-capped `write_buf`. This is a **silent** adaptation (the client asked for it; nothing is logged). Corroborated by `docs/graphics-protocol.rst` (the `q` key; "320MB per buffer" quota).

---

## 5. Synchronized-update "pending mode" render pause

A distinct, application-driven pause: the synchronized-update private mode `2026` (`PENDING_MODE`, `kitty/control-codes.h:235`) holds rendering while an app assembles a frame.

```c
// kitty/screen.c:2521  (screen_pause_rendering)
if (for_in_ms <= 0) for_in_ms = 2000;
```

`DECSET ?2026h` calls `screen_pause_rendering` with a default **2000 ms** timeout; `screen_check_pause_rendering` auto-unpauses on expiry or on `DECRST ?2026l` (`kitty/screen.c:2489-2490`). Re-entering pending mode while already paused logs a visible warning.

**Evidence C [non-canonical]** (real `screen.c`):

```
##### C. SYNCHRONIZED-UPDATE PENDING MODE (screen.c:1174 PENDING_MODE 2026) #####
  send DECSET 2026 (ESC [ ? 2026 h) to pause rendering (screen_pause_rendering, default 2000ms @screen.c:2521)
  send DECSET 2026 AGAIN while already paused -> 'already current mode' log_error [screen.c:1176] on stderr:
  send DECRST 2026 (ESC [ ? 2026 l) to unpause
```

stderr (unedited):

```
[0.050] Pending mode change to already current mode (1) requested. Either pending mode expired or there is an application bug.
```

**Cause→effect:** pending mode is a rendering throttle (it coalesces mid-frame updates into one repaint); it does not change read/write buffering. The `already current mode` `log_error` (`kitty/screen.c:1176`) is a **visible** sign that a synchronized update overlapped/expired.

---

## 6. Where in the code — reference map (`file:line`)

**Inbound (read) path — `kitty/vt-parser.c`, `kitty/child-monitor.c`:**
- `kitty/vt-parser.c:18` — `#define BUF_SZ (1024u*1024u)` — fixed **1 MiB** parser buffer (BUFFER).
- `kitty/vt-parser.c:21` — `#define MAX_ESCAPE_CODE_LENGTH (BUF_SZ / 4u)` — 256 KiB per escape code.
- `kitty/vt-parser.c:406`, `:419` — over-length escape-code check and `REPORT_ERROR("... escape code too long ...")`.
- `kitty/vt-parser.c:1417` — `run_worker`; `:1425` — force-parse condition `flush || time_since_new_input >= OPT(input_delay) || read.sz + 16*1024 > BUF_SZ` (THROTTLE).
- `kitty/vt-parser.c:1451` — `vt_parser_create_write_buffer`; `:1477` — `vt_parser_has_space_for_input` (`read.sz + write.pending < BUF_SZ`).
- `kitty/child-monitor.c:1501` — `events = vt_parser_has_space_for_input(...) ? POLLIN : 0;` (PAUSE: stop reading → OS PTY backpressure).
- `kitty/child-monitor.c:1506-1509` — `input_delay` poll-timeout batching (THROTTLE).

**Outbound (write-to-child) path — `kitty/child-monitor.c`:**
- `kitty/child-monitor.c:341-344` — 100 MiB `write_buf` cap; `:342` — `log_error("Too much data being sent to child with id: %lu, ignoring it", id)` (VISIBLE) + discard.
- `kitty/child-monitor.c:1503` — `events |= (screen->write_buf_used ? POLLOUT : 0);` (request POLLOUT only when queued).
- `kitty/child-monitor.c:1443` — `write_to_child`; `:1462` — `EINTR` continue; `:1463` — `EWOULDBLOCK/EAGAIN` break (retain, retry); `:1464` — `perror("Call to write() to child fd failed, discarding data.")`.
- `kitty/child-monitor.c:1449` — `KITTY_PRINT_BYTES_SENT_TO_CHILD` (`Wrote: N bytes: ...`, compile-time); `:1550-1554` — `DEBUG_POLL_EVENTS` per-fd `revents` (compile-time).

**Graphics — `kitty/graphics.c`, `kitty/screen.c`:**
- `kitty/graphics.c:25`, `:78` — `DEFAULT_STORAGE_LIMIT` 320 MiB assigned to `storage_limit`.
- `kitty/graphics.c:290-296` (invoked `:2184`) — `apply_storage_quota` LRU eviction.
- `kitty/graphics.c:302` — `static char command_response[512]`.
- `kitty/graphics.c:646` zero-dim `EINVAL`; `:651` unknown-format `EINVAL`; `:521,533` `MAX_DATA_SZ` PNG/`EFBIG`; `:695-697` `MAX_IMAGE_DIMENSION 10000` "Image too large"; `:716` `start_command.id` set.
- `kitty/graphics.c:761-764` — `finish_command_response` `q=` suppression; `:1570-1573` — frame cache `storage_limit*5` → `ENOSPC`; `:2199` — `ENOENT` frame on missing image.
- `kitty/screen.c:970` `ESC_APC` case; `:979` `write_escape_code_to_child`; `:1050` — graphics reply onto shared output path.
- `kitty/screen.c:1174-1176` — pending-mode already-current `log_error`; `:2506` `screen_pause_rendering`; `:2521` default 2000 ms; `:2489-2490` auto-unpause.

**Config defaults & constants:**
- `kitty/options/definition.py:866` `repaint_delay '10'`; `:878` `input_delay '3'`. Generated typed values in `kitty/options/types.py` (`input_delay=3`, `repaint_delay=10`, `sync_to_monitor=True`).
- `kitty/control-codes.h:235` `#define PENDING_MODE 2026`; `kitty/state.h:51` `monotonic_t repaint_delay, input_delay;`.
- `docs/graphics-protocol.rst` — `q` response-suppression key; "320MB per buffer" storage quota. Observation-design references: `kitty_tests/screen.py` (~L948, 1 MiB boundary), `kitty_tests/parser.py` (`test_create_write_buffer`/`test_commit_write_buffer` — **non-canonical** bypass), `kitty_tests/graphics.py` (~L1189+, storage eviction).

---

## 7. Runtime manifestation — silent vs. visible

| Mechanism | Runtime manifestation | Silent or visible | Evidence |
|---|---|---|---|
| Inbound 1 MiB buffer full → `POLLIN` dropped | kitty stops reading; child blocks in `write()` on the full PTY; byte counter freezes | **Silent** (no log; only the blocking write) | E2 (§2.2) |
| Inbound pacing via `input_delay` batching | producer bounded to kitty's consume rate (~3× slower than `/dev/null`); poll timeout settles at 0.003 s | **Silent** | E1, T (§2.3) |
| Outbound `write_buf` 100 MiB cap | `Too much data being sent to child with id: 1, ignoring it` logged (millions of times), data discarded | **Visible** (`kitty/child-monitor.c:342`) | F (§3.3) |
| Outbound `EAGAIN` retention | tiny `write()`s, `POLLOUT` requested while queued, data retained | Silent (visible only in debug build) | F, G (§3.3) |
| Graphics storage quota | oldest images LRU-evicted; `image_count` capped | Silent (eviction) | B2 (§4.1) |
| Graphics frame cache 5× | `ENOSPC` APC error reply | **Visible** (error reply) | B3 (§4.2) |
| Graphics size/format guards | `EINVAL`/`EFBIG` APC error reply (or empty for over-dimension) | **Visible** (mostly) | B4 (§4.3) |
| Graphics `q=` suppression | `OK`/error replies withheld at client's request | **Silent** | B1 (§4.4) |
| Pending-mode overlap/expiry | `Pending mode change to already current mode...` logged | **Visible** (`kitty/screen.c:1176`) | C (§5) |
| Over-long escape code | `... escape code too long (N bytes), ignoring it` logged | **Visible** (`kitty/vt-parser.c:419`) | D (§2.4) |

**Debug-build poll-event slice (Evidence G, observed, print-only build):**

```
child fd i:2 poll-event totals (DEBUG_POLL_EVENTS, child-monitor.c:1550-1554, stdout):
   i:2 POLLIN  = 13659   (inbound: kitty reading child output)
   i:2 POLLOUT =    86   (outbound: write_buf non-empty -> POLLOUT requested, child-monitor.c:1503)

--- DEBUG_POLL_EVENTS stdout slice (unedited): POLLIN reads interleaved with POLLOUT writes ---
i:2 POLLIN
i:2 POLLIN
i:2 POLLIN
i:2 POLLIN
i:0 POLLIN
i:0 POLLIN
i:2 POLLOUT
i:0 POLLIN
i:2 POLLOUT
i:0 POLLIN
i:2 POLLOUT

--- KITTY_PRINT_BYTES_SENT_TO_CHILD stderr slice (unedited): responses written to child ---
  (child-monitor.c:1449; DA2 replies 'ESC[>1;4000;35c'; small chunks because child never reads stdin)
Wrote: 156 bytes: \x1b[>1;4000;35c\x1b[>1;4000;35c\x1b[>1;4000;35c\x1b[>1;4000;35c\x1b[>1;...
Wrote: 143 bytes: \x1b[>1;4000;35c\x1b[>1;4000;35c\x1b[>1;4000;35c\x1b[>1;4000;35c\x1b[>1;...
Wrote: 169 bytes: \x1b[>1;4000;35c\x1b[>1;4000;35c\x1b[>1;4000;35c\x1b[>1;4000;35c\x1b[>1;...
Wrote: 130 bytes: \x1b[>1;4000;35c ... 
Wrote: 286 bytes: \x1b[>1;4000;35c ... 

NOTE: DEBUG_POLL_EVENTS prints REVENTS (what poll() returned), not the gated .events request
mask; the POLLIN-gating decision itself (child-monitor.c:1501) is confirmed by code inspection,
and its EFFECT (child stall) is directly observed in Evidence E (E2 SIGSTOP).
```

Here the child fd is poll index `i:2` (`EXTRA_FDS=2`: `i:0` wakeup, `i:1` signals, `i:2` first child). `POLLIN` (kitty reading) is interleaved with `POLLOUT` (kitty draining queued replies), and `POLLOUT` is requested far less often (86×) than `POLLIN` (13659×) — consistent with `kitty/child-monitor.c:1503` requesting `POLLOUT` only while the buffer is non-empty.

---

## 8. Observed-vs-inferred ledger

**Observed (canonical — default-config kitty via real PTY, or default compiled constants):**
- Default option values: `input_delay=3 ms`, `repaint_delay=10 ms`, `sync_to_monitor=True`; `BUF_SZ=1 MiB`, `MAX_ESCAPE_CODE_LENGTH=256 KiB`; graphics `storage_limit=320 MiB` (§1).
- Inbound pacing (~87 MiB/s via kitty vs ~260 MiB/s to `/dev/null`) and SIGSTOP-induced child stall/freeze (§2.2–2.3, Evidence E1/E2).
- Outbound 100 MiB overflow log `Too much data being sent to child with id: 1, ignoring it`, first at t≈6.0 s (baseline `--debug` build) — proving the log is canonical (§3.3, Evidence F-baseline).
- `input_delay` 3 ms poll-timeout batching (§2.3, Evidence T).

**Observed (debug print-only build; behavior identical to canonical):**
- Per-fd `POLLIN`/`POLLOUT` counts and interleaving; `Wrote: N bytes` DA2 reply chunks; overflow log in the instrumented run (§3.3/§7, Evidence F-instrumented, G).

**Observed ([non-canonical] delivery — real C logic via `kitty_tests` parse hooks, no real PTY; B2 additionally uses a reduced `storage_limit`):**
- `q=` suppression (B1), LRU eviction (B2), frame-cache `ENOSPC` (B3), size/format guards incl. the empty-response nuance (B4), pending-mode `already current mode` log (C), parser `escape code too long` at the 256 KiB/1 MiB boundaries (D).

**[inferred] (read from code, not separately instrumented):**
- The `run_worker` force-parse-near-full clause `read.sz + 16*1024 > BUF_SZ` (`kitty/vt-parser.c:1425`).
- That `DEBUG_POLL_EVENTS` shows `revents` rather than the `.events` request mask, so the `POLLIN`-gating *decision* (`kitty/child-monitor.c:1501`) is confirmed by inspection while its *effect* is what E2 observes.

**Stability.** Every observation above was reproduced across at least two runs with consistent results (frozen byte counters at a single value during SIGSTOP; overflow crossing at ~6 s in both builds; identical harness stdout/stderr modulo timestamps).

**Exact commands.** Canonical build, debug print-only build, headless run, and in-process harness commands are listed verbatim in §1.

---

## 9. Investigation hygiene

- All observation scripts lived **outside** the repository under `/tmp` (`/tmp/obs_harness.py`, `/tmp/flood_da2.py`, `/tmp/flood_plain.py`, `/tmp/e2_driver.sh`, and `/tmp/*.txt`/`/tmp/*.log` scratch) and were **removed** after evidence capture.
- Build artifacts (`build/`, `kitty/fast_data_types.so`, `kitty/launcher/kitty`) are git-ignored and do not appear as tracked changes.
- `git status --porcelain` (tracked) is empty; HEAD remains `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`. **No existing repository file was modified** — the only addition is this document under `blitzy/documentation/`.

---
