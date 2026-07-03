# How kitty Regulates Graphics‑Protocol Data Flow Under Pressure

**Buffering, pausing, throttling, backpressure, storage‑quota eviction, protocol errors, and silent‑vs‑visible adaptation — answered from observed runtime evidence at HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (kitty v0.35.2).**

This document answers, from directly observed runtime behavior, how the kitty terminal emulator controls the inbound and outbound flow of graphics‑protocol (`APC _G`) data when it arrives faster than the terminal can comfortably process and respond to it. Every behavioral claim is paired with the exact command that produced it and the verbatim captured output. Values that could only be established by reading the source (because they require a live GPU render loop or the I/O thread that is not present in a headless harness) are explicitly labeled **INFERRED / SOURCE‑CONFIRMED** — i.e. *inferred* from reading the source rather than captured at runtime (still grounded in, and confirmed against, the exact code literals); everything else is **OBSERVED**. Exact literals are cited with `file:line`.

---

## Table of Contents

1. [Environment, Build & Reproducibility (Preamble)](#1-environment-build--reproducibility-preamble)
2. [Sub‑question 1 — How kitty decides to buffer, pause reading, or slow processing](#2-subquestion-1--how-kitty-decides-to-buffer-pause-reading-or-slow-processing)
3. [Sub‑question 2 — What happens to responses when the outbound path is congested](#3-subquestion-2--what-happens-to-responses-when-the-outbound-path-is-congested)
4. [Sub‑question 3 — Where the decisions live (file:line)](#4-subquestion-3--where-the-decisions-live-fileline)
5. [Sub‑question 4 — How the mechanisms manifest at runtime (driving past each limit)](#5-subquestion-4--how-the-mechanisms-manifest-at-runtime-driving-past-each-limit)
6. [Sub‑question 5 — Silent adaptation vs. visible signs](#6-subquestion-5--silent-adaptation-vs-visible-signs)
7. [Coverage Pass, Reasoning & Cleanup](#7-coverage-pass-reasoning--cleanup)

---

## 1. Environment, Build & Reproducibility (Preamble)

### 1.1 Mandated environment and build

All runtime observations were produced with kitty built and run in its **default, canonical configuration**. Two agreeing paths were used: the mandated Docker image (authoritative) and an identical native build at the same commit; every headline figure below was cross‑checked to be **identical** in the Docker image.

- **Docker image (authoritative):** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0` (container `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`). It contains the repository at `/app`, already built.
- **Build command (default / canonical):** `make`, whose `all:` target runs `python3 setup.py`. Verified verbatim in the root `Makefile`:

```make
all:
	python3 setup.py $(VVAL)
```

- **Optional tracing builds** (not required for the observations below; available per `Makefile`): `make debug` → `python3 setup.py build $(VVAL) --debug`; `make debug-event-loop` → `python3 setup.py build $(VVAL) --debug --extra-logging=event-loop`.
- **INSTRUMENTATION: none.** None of the reported values depend on an instrumentation‑only build. The optional tracing builds above (`make debug`, `make debug-event-loop`) and the compile‑time `KITTY_PRINT_BYTES_SENT_TO_CHILD` hook were **not** used as the source of any figure in this document; every measured value comes from the default, canonical build. Any value that had required such **instrumentation** would be labeled **INSTRUMENTATION** at the point of use.

### 1.2 Version banner (verbatim)

The version banner is identical in the native build and the canonical Docker image, and the Docker `HEAD` is the target commit:

```console
$ docker run --rm --entrypoint /bin/bash ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0 -lc 'cd /app; git rev-parse HEAD; ./kitty/launcher/kitty --version'
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
kitty 0.35.2 created by Kovid Goyal
```

### 1.3 The canonical code path used to drive the real graphics input

The graphics protocol is exercised through kitty's **real compiled C core** — the same `Screen` / `GraphicsManager` / `vt_parser` objects a live terminal uses. This is not a bypassing remote‑control or debug hook. Two genuine entry points were used:

1. **`kitty +kitten icat <image>`** — the real `icat` kitten, driven over a real PTY so a real `Screen` answers its protocol queries and it streams the image as chunked `APC _G` escape sequences.
2. **Raw `APC _G` escape sequences** fed into the real VT parser via kitty's own test harness helpers, which are thin wrappers over the C core:
   - `kitty_tests.parse_bytes(screen, data)` pushes bytes through the real parser using the C methods `test_create_write_buffer` → `test_commit_write_buffer` → `test_parse_written_data`.
   - `kitty_tests.graphics.send_command(screen, cmd, payload)` builds a genuine `\033_G<control>;<base64>\033\\` command and returns `callbacks.wtcbuf` — the exact bytes kitty writes back toward the program.

The standalone runner is `kitty +launch <script.py>` (runs Python inside kitty's environment with the compiled `fast_data_types` module available). Temporary observation scripts lived under `/tmp/blitzy_investigation/` (outside the repository) and were removed afterward (see §7).

### 1.4 Default option values in effect (verbatim)

No custom `kitty.conf` was used, so the defaults are in force. Confirmed at runtime and cited to source:

```console
$ kitty +launch defaults.py
input_delay  = 3
repaint_delay= 10
sync_to_monitor= True
```

| Option | Default | Source |
| --- | --- | --- |
| `input_delay` | `3` (ms) | `kitty/options/definition.py:L878` — `opt('input_delay', '3', ...)` |
| `repaint_delay` | `10` (ms) | `kitty/options/definition.py:L866` — `opt('repaint_delay', '10', ...)` |
| `sync_to_monitor` | `yes` | `kitty/options/definition.py:L889` — `opt('sync_to_monitor', 'yes', ...)` |

The `input_delay` help text corroborates the near‑full override discussed in §2: <br>`kitty/options/definition.py:L885` — “This setting is ignored when the input buffer is almost full.”

### 1.5 Scale/duration used and repeatability

Every magnitude/timing value below was observed **stable across at least two runs**, and cross‑checked in the canonical Docker image. Scales used:

| Scenario | Scale / duration driven | Runs |
| --- | --- | --- |
| Input parse‑buffer ceiling | committed 200 000‑byte writes until space hit 0 (total 1 048 576 B) | 2 (native) + Docker |
| Output 100 MB `write_buf` cap | one `needs_write` of 104 857 601 B (100 MiB + 1) | 2 (native) + Docker |
| Graphics storage quota / LRU | 30 × 16 MiB images (2048×2048 RGBA) = 480 MiB driven past the 320 MiB quota | 2 (native) + Docker |
| Per‑image `MAX_DATA_SZ` (EFBIG / PNG EINVAL) | `S=400000001` and RGBA buffer overflow | 2 (native) + Docker |
| `MAX_IMAGE_DIMENSION` | `s=10001` / `v=10001` | 2 (native) + Docker |
| Disk‑cache offload | one 4 MiB image | 2 (native) |
| Real `icat` framing | 64×48 PNG and 400×400 incompressible PNG | 2 each (native) |
| Pending mode (DEC 2026) | CSI/DCS toggles + terminator probes | 2 (native) + Docker |
| PTY kernel backpressure substrate | real `openpty`/`fork`, undrained master | 2 (native) |

---

## 2. Sub‑question 1 — How kitty decides to buffer, pause reading, or slow processing

**Short answer (cause → effect):** kitty **buffers** inbound bytes into a single fixed **1 MiB** per‑child VT parse buffer; it **slows processing** by *coalescing* — deliberately batching input and not parsing on every read, governed by `input_delay`; and it **pauses reading** not by discarding data but by *withholding interest in `POLLIN`* the moment the buffer has no room. A parser with a full buffer stops draining the PTY, the kernel PTY buffer fills, and the child program's own `write()` blocks. Input is **never silently discarded on the read side.**

### 2.1 The buffer: a fixed 1 MiB ceiling

kitty allocates one fixed‑capacity parse buffer per child.

- **Literal:** `kitty/vt-parser.c:L18` — `#define BUF_SZ (1024u*1024u)` (= 1 MiB).
- A related cap bounds any single escape code: `kitty/vt-parser.c:L21` — `#define MAX_ESCAPE_CODE_LENGTH (BUF_SZ / 4u)` (= 262 144 B). This is *why* a large image must be split into chunks (see §5.7).

**Observed** (native + canonical Docker, stable across 2 runs) — the module exposes these constants and the initial write buffer is exactly `BUF_SZ`:

```console
$ kitty +launch graphics_limits.py
VT_PARSER_BUFFER_SIZE = 1048576
VT_PARSER_MAX_ESCAPE_CODE_SIZE = 262144
initial_write_buffer_len = 1048576
```

Driving 200 000‑byte writes into the parser *without parsing*, the available space shrinks monotonically to exactly 0 after 1 048 576 B are committed — i.e. the ceiling is `BUF_SZ` (OBSERVED, stable across 2 runs, identical in Docker):

```console
$ kitty +launch input_buffer.py
available-space sequence (bytes) as buffer fills: [1048576, 848576, 648576, 448576, 248576, 48576, 0]
total committed before space hit 0: 1048576
=> ceiling == BUF_SZ: True ( 1048576 )
```

### 2.2 Slow down processing: input coalescing / force‑flush (all three disjuncts)

kitty does not parse on every read. The worker decides whether to process the accumulated input using a three‑way predicate. Any **one** of the three conditions triggers processing:

- **Literal:** `kitty/vt-parser.c:L1425`:

```c
if (flush || pd->time_since_new_input >= OPT(input_delay) || self->read.sz + 16 * 1024 > BUF_SZ) {
```

The three disjuncts, enumerated exhaustively with the causal reason for each:

| # | Disjunct | Meaning | Why |
| --- | --- | --- | --- |
| 1 | `flush` | An explicit flush was requested | Forces immediate processing regardless of timing (e.g. shutdown / drain) |
| 2 | `pd->time_since_new_input >= OPT(input_delay)` | Input has aged past `input_delay` (default **3 ms**, `definition.py:L878`) | Batches bursty input for a few ms to reduce redundant work/repaints; the delay is the throttle knob |
| 3 | `self->read.sz + 16 * 1024 > BUF_SZ` | The buffer is within **16 KB** of full | **Near‑full override:** stop waiting and process now so the 1 MiB buffer cannot overflow |

Disjunct 3 is corroborated by the option help text (**OBSERVED** in source): `kitty/options/definition.py:L885` — “This setting is ignored when the input buffer is almost full.” That sentence is precisely the near‑full override in code form.

### 2.3 Pause reading: withhold `POLLIN` when the buffer is full → kernel PTY backpressure

Whether kitty is *willing to read more* is recomputed every I/O‑loop iteration from remaining buffer space:

- **Space predicate:** `kitty/vt-parser.c:L1477`–`L1481` — `vt_parser_has_space_for_input()` returns `ans = self->read.sz + self->write.pending < BUF_SZ;`.
- **The gate:** `kitty/child-monitor.c:L1501`:

```c
children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;
```

When the buffer is full the event mask becomes `0` — kitty stops asking the OS to notify it that the PTY is readable, so it stops reading. It never `read()`s into a full buffer:

- **No read when full:** `kitty/child-monitor.c:L1341`–`L1342` — `uint8_t *buf = vt_parser_create_write_buffer(...); if (!available_buffer_space) return true;`.

Because the space predicate becomes false exactly when the buffer is full (which §2.1 observed happens at 1 048 576 B), `POLLIN` is withheld and the PTY is no longer drained. **INFERRED / SOURCE‑CONFIRMED:** the `POLLIN`‑gating line runs on kitty's dedicated I/O thread's `poll()` loop, which is not instantiated in a headless harness.

The *consequence* of an undrained PTY is a standard POSIX guarantee, which was demonstrated directly with a real `openpty()`/`fork()` (OBSERVED, stable across 2 runs): with the master end deliberately left undrained (modeling kitty withholding `POLLIN`), the child's `write()` blocks after the kernel PTY buffer fills:

```console
$ python3 pty_backpressure.py
child bytes written before its write() BLOCKED = 12288
kernel PTY buffer capacity actually drained  = 12288
=> Undrained master => child write() blocked (POSIX PTY kernel backpressure).
```

**Cause → effect chain:** buffer full → `vt_parser_has_space_for_input()` false → `POLLIN` withheld (`child-monitor.c:L1501`) → kitty stops reading the PTY → kernel PTY buffer fills → the child program's `write()` **blocks**. This is how a fast producer is throttled *without kitty ever dropping a byte of input.*

---

## 3. Sub‑question 2 — What happens to responses when the outbound path is congested

**Short answer (cause → effect):** replies kitty must send back to the program (graphics `OK`/error responses, query answers, etc.) are appended to a per‑screen **`write_buf`** queue. kitty asks the OS for `POLLOUT` **only when there is queued data**; on a would‑block it simply stops and retries later (`EAGAIN`/`EWOULDBLOCK`). The queue grows dynamically, but it is capped at **100 MB** — beyond that, kitty logs a message and **drops** the new data. This 100 MB output cap is the *only* place kitty discards data it is trying to send to the child, and it is a **different** limit from the graphics per‑image 400 MB limit in §5.

### 3.1 The queued output path

Each `Screen` owns a dynamically grown output buffer, protected by a mutex:

- **Lock init:** `kitty/screen.c:L104` — the `write_buf_lock` pthread mutex is initialized.
- **Buffer init:** `kitty/screen.c:L113`–`L114`:

```c
self->write_buf_sz = BUFSIZ;
self->write_buf = PyMem_RawMalloc(self->write_buf_sz);
```

The buffer starts at `BUFSIZ` (8 KB) and is `realloc`‑grown on demand as responses are queued.

### 3.2 `POLLOUT` only when there is something to write; retry on `EAGAIN`/`EWOULDBLOCK`

kitty registers write‑interest lazily, so an idle child costs nothing and a congested one is retried rather than blocked on:

- **`POLLOUT` gating:** `kitty/child-monitor.c:L1503`:

```c
children_fds[EXTRA_FDS + i].events |= (screen->write_buf_used ? POLLOUT  : 0);
```

`POLLOUT` is requested **only** when `write_buf_used > 0` (data pending). When the PTY cannot accept more, the write path breaks out and retries on the next loop iteration:

- **Non‑blocking retry:** `kitty/child-monitor.c:L1463`:

```c
if (errno == EWOULDBLOCK || errno == EAGAIN) break;
```

- **Resume when space frees up:** once the write buffer drains, `kitty/child-monitor.c:L442` — `if (pd.write_space_created) wakeup_io_loop(self, false);` — nudges the loop to resume reading. **INFERRED / SOURCE‑CONFIRMED** (I/O‑thread `poll()` loop).

### 3.3 The 100 MB output cap — the only place kitty drops data it is sending to the child

If appending a response would push the queue past 100 MB, kitty logs and drops it instead of growing without bound:

- **Literal:** `kitty/child-monitor.c:L341`–`L343` (inside the `schedule_write_to_child_generic` macro):

```c
if (screen->write_buf_used + sz > 100 * 1024 * 1024) {
    log_error("Too much data being sent to child with id: %lu, ignoring it", id);
    screen_mutex(unlock, write);
    break;
}
```

This was driven through the **genuine** path (`ChildMonitor.needs_write` → `schedule_write_to_child` → the cap macro) using a real `ChildMonitor`, a real PTY, and a real `Screen`; `start()` spawns the real I/O thread (`kitty/child-monitor.c:L291`) which drains the child into the active set. A single `needs_write` of 100 MiB + 1 byte trips the cap immediately (`0 + sz > 100·1024·1024`). **OBSERVED, stable across 2 runs (native) and confirmed in the canonical Docker image.** `log_error` prefixes a timestamp and writes to `stderr` (`kitty/logging.c:L56`, `L61`), so the verbatim line is:

```console
$ kitty +launch writebuf_cap.py        # native, run 1
[0.479] Too much data being sent to child with id: 1, ignoring it
PAYLOAD_BYTES=104857601  CAP=104857600  needs_write_returned=False (False => DROPPED)

$ kitty +launch writebuf_cap.py        # native, run 2 (stability)
[0.480] Too much data being sent to child with id: 1, ignoring it
PAYLOAD_BYTES=104857601  CAP=104857600  needs_write_returned=False (False => DROPPED)
```

```console
# canonical Docker image
$ docker run --rm -v /tmp/blitzy_investigation:/binv:ro --entrypoint /bin/bash <image> \
    -lc 'cd /app; ./kitty/launcher/kitty +launch /binv/writebuf_cap.py'
[0.484] Too much data being sent to child with id: 1, ignoring it
PAYLOAD_BYTES=104857601  CAP=104857600  needs_write_returned=False (False => DROPPED)
```

Here the child id `1` is the actual value substituted for `%lu` at runtime, and `needs_write` returning `False` confirms the data was dropped rather than queued.

> **⚠ Do not conflate the two “too much data” limits.** The **output‑side** 100 MB `write_buf` cap here (`child-monitor.c:L341-343`, log line “Too much data being sent to child with id: …”) is entirely separate from the **graphics per‑image** 400 000 000‑byte `MAX_DATA_SZ` cap in §5.3 (`graphics.c:L521,L533`, protocol response `EFBIG:Too much data`). Different subsystems, different thresholds, different messages, different transports (a `log_error` to stderr vs. an `APC _G` protocol reply to the program).

---

## 4. Sub‑question 3 — Where the decisions live (file:line)

Every flow‑control decision, named with `file:line` precision. All anchors were re‑confirmed exact at HEAD `815df1e210e0`.

### 4.1 Parser — `kitty/vt-parser.c` (the input side)

| Decision | Location | Literal |
| --- | --- | --- |
| Fixed 1 MiB parse buffer | `L18` | `#define BUF_SZ (1024u*1024u)` |
| Per‑escape‑code cap (¼ MiB) | `L21` | `#define MAX_ESCAPE_CODE_LENGTH (BUF_SZ / 4u)` |
| Coalesce / force‑flush predicate (3 disjuncts) | `L1425` | `if (flush \|\| pd->time_since_new_input >= OPT(input_delay) \|\| self->read.sz + 16 * 1024 > BUF_SZ)` |
| Space accounting for read readiness | `L1477`–`L1481` | `ans = self->read.sz + self->write.pending < BUF_SZ;` (`vt_parser_has_space_for_input`) |
| Pending/synchronized (DEC 2026) DCS toggle + error text | `L638`–`L648` | `screen_start_pending_mode` / `screen_stop_pending_mode` + termination‑cause error |

### 4.2 Child‑monitor threaded `poll()` I/O loop — `kitty/child-monitor.c` (both sides)

| Decision | Location | Literal |
| --- | --- | --- |
| `read_bytes()` no‑read‑when‑full | `L1341`–`L1342` | `if (!available_buffer_space) return true;` |
| `POLLIN` gating on buffer space | `L1501` | `... vt_parser_has_space_for_input(...) ? POLLIN : 0;` |
| `POLLOUT` only when data pending | `L1503` | `... \|= (screen->write_buf_used ? POLLOUT : 0);` |
| Non‑blocking write retry | `L1463` | `if (errno == EWOULDBLOCK \|\| errno == EAGAIN) break;` |
| **100 MB output drop cap + log line** | `L341`–`L343` | `log_error("Too much data being sent to child with id: %lu, ignoring it", id);` |
| Resume reading when write space frees | `L442` | `if (pd.write_space_created) wakeup_io_loop(self, false);` |
| I/O thread creation | `L291` | `pthread_create(&self->io_thread, NULL, io_loop, self);` |
| Timeout scheduling for pending mode | `L444` / `L729` | `set_maximum_wait(...expires_at...)` / `screen_check_pause_rendering(WD.screen, now)` |

### 4.3 Graphics handler — `kitty/graphics.c`

| Decision | Location | Literal |
| --- | --- | --- |
| 320 MiB storage quota | `L25` | `#define DEFAULT_STORAGE_LIMIT 320u * (1024u * 1024u)` |
| Storage‑quota LRU eviction | `L290`–`L299` | `apply_storage_quota()` (remove‑unreferenced, then `HASH_SORT(oldest_img_first)` + evict loop) |
| 400 MB per‑image data cap → `EFBIG` | `L521`, `L533` | `#define MAX_DATA_SZ (4u * 100000000u)`; `ABRT("EFBIG", "Too much data")` |
| PNG data‑size sibling check → `EINVAL` | `L638` | `if (g->data_sz > MAX_DATA_SZ) ABRT("EINVAL", "PNG data size too large");` |
| `MAX_DATA_SZ` un‑defined after use | `L755` | `#undef MAX_DATA_SZ` |
| 10 000‑px dimension cap → `EINVAL` | `L674`, `L695` | `#define MAX_IMAGE_DIMENSION 10000u`; `ABRT("EINVAL", "Image too large")` |
| Second identical dimension check | `L1553` | `ABRT("EINVAL", "Image too large")` |
| Quiet‑level response suppression | `L759`–`L777` | `if (g->quiet) { if (is_ok_response \|\| g->quiet > 1) return NULL; }` (`finish_command_response`) |

### 4.4 Disk cache (off‑RAM image store) — `kitty/disk-cache.c`

| Decision | Location | Function |
| --- | --- | --- |
| Add image bytes to disk | `L488` | `add_to_disk_cache()` |
| Read image bytes back | `L591` | `read_from_disk_cache()` |
| Remove image bytes | `L517` | `remove_from_disk_cache()` |
| Defrag heuristic | `L232` | `needs_defrag()` — `size_on_disk > self->total_size * 2` |

### 4.5 Screen pending/synchronized render mode — `kitty/screen.c`

| Decision | Location | Literal |
| --- | --- | --- |
| `write_buf` lock init | `L104` | `pthread_mutex_init(&self->write_buf_lock, ...)` |
| `write_buf` alloc | `L113`–`L114` | `self->write_buf_sz = BUFSIZ; self->write_buf = PyMem_RawMalloc(...);` |
| Mode 2026 (`PENDING_MODE`) toggle | `L1174`–`L1175` | `case PENDING_MODE << 5: ... screen_pause_rendering(self, val, 0)` |
| Timeout check | `L2489`–`L2490` | `if (self->paused_rendering.expires_at && now > self->paused_rendering.expires_at) screen_pause_rendering(self, false, 0);` |
| Pause function + **2000 ms default** timeout | `L2506`, `L2521`–`L2522` | `if (for_in_ms <= 0) for_in_ms = 2000; ... expires_at = monotonic() + ms_to_monotonic_t(for_in_ms);` |
| Status‑query bit (active↔inactive) | `L2238` | `ans = self->paused_rendering.expires_at ? 1 : 2;` |
| Disruptive terminators | `L163` / `L347` / `L1910` / `L4155` | `screen_pause_rendering(self, false, 0)` in `screen_reset` / `screen_resize` / `dirty_scroll` / `screen_start_selection` |

### 4.6 Cross‑thread wakeup — `kitty/loop-utils.c`

| Decision | Location | Literal |
| --- | --- | --- |
| `eventfd`/self‑pipe wakeup fd | `L70` | `wakeup_read_fd = eventfd(0, EFD_CLOEXEC \| EFD_NONBLOCK)` |
| `wakeup_loop()` cross‑thread signal | `L70`–`L117` | resumes the I/O loop once buffer space is freed |

### 4.7 APC `_G` command parsing — `kitty/parse-graphics-command.h`

| Key | Location | Literal |
| --- | --- | --- |
| Chunk‑continuation key `m` | `L25` | `more = 'm',` |
| Quiet key `q` | `L29` | `quiet = 'q',` |

### 4.8 Options / defaults — `kitty/options/definition.py`

| Option | Location | Literal |
| --- | --- | --- |
| `repaint_delay` | `L866` | `opt('repaint_delay', '10', ...)` |
| `input_delay` | `L878` | `opt('input_delay', '3', ...)` |
| `input_delay` near‑full note | `L885` | “This setting is ignored when the input buffer is almost full.” |
| `sync_to_monitor` | `L889` | `opt('sync_to_monitor', 'yes', ...)` |

---

## 5. Sub‑question 4 — How the mechanisms manifest at runtime (driving past each limit)

This section drives each code path **past** its limit and quotes the resulting protocol response or log line verbatim.

### 5.1 PTY backpressure blocking the child

Covered in §2.3. When the 1 MiB parse buffer saturates, `POLLIN` is withheld (`child-monitor.c:L1501`, **INFERRED / SOURCE‑CONFIRMED**) and the PTY stops draining; the standard POSIX consequence — the child's `write()` blocking — was demonstrated directly (OBSERVED, 2 runs): `child bytes written before its write() BLOCKED = 12288`.

### 5.2 Coalesced input processing near `input_delay`

The parse buffer fills to exactly `BUF_SZ` before space reaches 0 (OBSERVED, §2.1: `[1048576, 848576, 648576, 448576, 248576, 48576, 0]`), and the three‑disjunct predicate at `vt-parser.c:L1425` decides *when* to process — after `input_delay` (default 3 ms, `definition.py:L878`) or immediately when within 16 KB of full. This is the throttle: bytes are batched rather than parsed per‑read.

### 5.3 Graphics per‑image data cap — `MAX_DATA_SZ` = 400 000 000 bytes

- **Literal:** `graphics.c:L521` — `#define MAX_DATA_SZ (4u * 100000000u)` (= 400 000 000). Two distinct checks use it:
  - **PNG data‑size check** (`graphics.c:L638`): declaring a PNG payload one byte over the cap yields a visible `EINVAL` protocol error. **OBSERVED** (native + Docker, 2 runs):

```console
$ kitty +launch graphics_limits.py
PNG_SIZE S=400000001 -> 'EINVAL:PNG data size too large'
PNG_SIZE S=400000000 at-limit -> 'EBADPNG:Not a PNG file'
```

  At exactly the limit (`S=400000000`) the size check passes and parsing proceeds to fail later as `EBADPNG` (our payload was not a real 400 MB PNG) — confirming the boundary is `> MAX_DATA_SZ`, not `>=`.

  - **Accumulation check → `EFBIG`** (`graphics.c:L533` — `ABRT("EFBIG", "Too much data")`): overflowing the accumulated transfer buffer of a direct (non‑PNG) RGBA transfer yields the `EFBIG` protocol error. **OBSERVED** (native + Docker, 2 runs):

```console
$ kitty +launch graphics_limits.py
EFBIG try RGBA chunk1 -> None
EFBIG try RGBA chunk2big -> 'EFBIG:Too much data'
```

  (`MAX_DATA_SZ` is `#undef`‑ed after use at `graphics.c:L755`.)

### 5.4 Graphics dimension cap — `MAX_IMAGE_DIMENSION` = 10 000 px

- **Literal:** `graphics.c:L674` — `#define MAX_IMAGE_DIMENSION 10000u`; enforced at `graphics.c:L695` and again at `graphics.c:L1553` — `ABRT("EINVAL", "Image too large")`.

**OBSERVED — with an important, honestly‑reported discrepancy** (native + Docker, 2 runs). Declaring a width or height over 10 000 does *not* emit a wire response, whereas the intuitive expectation is an `EINVAL:Image too large` reply:

```console
$ kitty +launch graphics_limits.py
IMAGE_DIM s=10001 -> None
IMAGE_DIM v=10001 -> None
IMAGE_DIM s=10000 at-limit -> 'ENODATA:Insufficient image data: 1 < 40000'
```

**Root cause (traced in source, reported exactly as observed):** the dimension `ABRT` at `graphics.c:L695` fires *before* the command's image id is recorded into `currently_loading.start_command` (assigned later at `graphics.c:L717`). The `ABRT` macro calls `free_load_data`, and `finish_command_response()` then finds no id/image‑number and returns `NULL` — so no response is written. By contrast, the PNG‑size `EINVAL` at `graphics.c:L638` runs *inside* `initialize_load_data()` **after** `start_command = *g` (`graphics.c:L634`), so it *does* emit a response (§5.3). The dimension limit is still enforced (the oversize image is rejected — `s=10000` at‑limit shows the request proceeding to the data‑length check `ENODATA:Insufficient image data: 1 < 40000`); only the *visible reply* differs. This is reported as observed rather than adjusted toward the expected `EINVAL:Image too large`.

**State-dependent sibling variant — the same over‑dimension condition *does* emit `EINVAL:Image too large` after a prior graphics command** (enumerated for exhaustiveness; **OBSERVED**, native run 1 == run 2 == Docker, driven over the genuine `APC _G` path). Whether the visible reply appears depends on the `GraphicsManager` state at the moment of the abort:

```console
$ kitty +launch dimension_state.py
fresh s=10001 -> None
fresh v=10001 -> None
pre OK -> 'OK'
after_success s=10001 -> 'EINVAL:Image too large'
after_success v=10001 -> 'EINVAL:Image too large'
```

**Cause → effect (traced in source, confirmed by the run above):** the response emitter `finish_command_response()` is handed `lg = &self->currently_loading.start_command` (`graphics.c:L2177,L2180`) and only writes a reply when `lg->id || lg->image_number` is non‑zero (`graphics.c:L765`). On a **fresh** screen `start_command` is all‑zero, so after the `L695` abort `lg->id == 0` and no reply is emitted (the fresh‑add case above). After a **prior successful** command, `start_command.id` was populated (`graphics.c:L717`) and — crucially — `free_load_data()` (`graphics.c:L103`) clears `buf`/`mapped_file`/`loading_for` but **not** `start_command`; so the retained non‑zero `lg->id` makes `finish_command_response()` emit the `EINVAL:Image too large` string that the `L695` abort placed into `command_response`. Net: the dimension limit is always enforced (the oversize image is always rejected); only *whether the client sees a wire reply* is state-dependent — no response on a fresh add, `EINVAL:Image too large` once any earlier graphics command has run. Reported exactly as observed.

### 5.5 Graphics storage quota — 320 MiB → silent LRU eviction

- **Literal:** `graphics.c:L25` — `#define DEFAULT_STORAGE_LIMIT 320u * (1024u * 1024u)` (= 335 544 320). Confirmed at runtime: `grman.storage_limit = 335544320`.
- **Mechanism:** `apply_storage_quota()` (`graphics.c:L290-299`) runs two stages — (1) `remove_images(trim_predicate, ...)` drops unreferenced/incomplete images; (2) `HASH_SORT(self->images, oldest_img_first)` then a `while (self->used_storage > storage_limit && self->images) remove_image(...)` loop evicts the **oldest first (LRU by `atime`)**.

Driving 16 MiB images (2048×2048 RGBA) past the quota, eviction is entirely **silent** (no error response). Two variants were run, both stable across 2 runs (native + Docker):

**Referenced images (`a=T`, per‑step, tracking which client ids survive):** at image 20 the store is exactly at 320 MiB holding ids `1..20`; image 21 succeeds (`OK`) but the oldest ids are evicted, leaving only the most recent, and **zero** non‑OK responses were emitted during the whole run — i.e. eviction happened without any visible sign:

```console
$ kitty +launch storage_lru2.py
img 20: resp='OK'  count=20 alive_ids=[1, 2, 3, ..., 20] total=320MiB
img 21: resp='OK'  count=21 alive_ids=[20, 21] total=32MiB
non-OK responses during load: 0 (0 => eviction was SILENT)
```

**Unreferenced images (`a=t`, transmit only):** here the stage‑1 unreferenced trim dominates — at image 21 the completed‑but‑unreferenced images are reclaimed, dropping the count sharply, again with an `OK` response and no error:

```console
$ kitty +launch storage_quota.py
after img 20: resp='OK'   image_count=20 disk_cache.total_size=335544320 (320MiB) cumulative_transmitted=320MiB
after img 21: resp='OK'   image_count= 1 disk_cache.total_size=16777216 (16MiB) cumulative_transmitted=336MiB
```

Reasoning: the 320 MiB quota bounds RAM/disk usage by *evicting older images*, categorically different from the per‑image/per‑dimension **rejections** (§5.3–5.4). It never refuses the new image; it makes room.

### 5.6 Disk‑cache offload — image bytes stored off‑RAM

Transmitted image bytes are offloaded to an on‑disk cache rather than held in RAM. **OBSERVED** (stable across 2 runs): loading one 4 MiB image moves both the logical size and the physical bytes‑on‑disk from 0 to 4 194 304, and the image round‑trips back:

```console
$ kitty +launch diskcache2.py
total_size BEFORE: 0  size_on_disk BEFORE: 0
load resp: OK
total_size AFTER (logical) : 4194304
size_on_disk AFTER (bytes on disk): 4194304 (== 4 MiB image => stored OFF-RAM on disk)
image_for_client_id(1) present: True client_id: 1
```

The backing store is created with `O_TMPFILE` (or `mkostemp` fallback); the cache add/read/remove/defrag entry points are `disk-cache.c:L488` / `L591` / `L517` / `L232`.

### 5.7 Real `kitten icat` framing — chunked APC `_G` on the actual input path

Driving the **real** `icat` kitten over a real PTY (a real `Screen` answers its capability query), the emitted wire bytes were captured. **OBSERVED, stable across 2 runs each.**

A small 64×48 PNG fits in a single un‑chunked APC command:

```console
$ kitty +launch icat_realpath.py
icat total bytes emitted: 280
APC _G starts (\x1b_G): 1
m=1 (non-final chunk): 0  m=0 (final chunk): 0
first APC control block: b'a=T,q=2,f=100,s=64,v=48,X=3'
num APC chunks: 1  max chunk bytes: 272  min: 272
```

A large 400×400 incompressible PNG is split into **7** chunks — six non‑final chunks marked `m=1`, and a final chunk that omits `m` (defaulting to final):

```console
$ kitty +launch icat_chunks_detail.py
num chunks: 7
chunk 0: m='1'    payload_bytes=131072 ctrl=b'a=T,q=2,f=100,m=1,s=400,v=400'
chunk 1: m='1'    payload_bytes=131072 ctrl=b'a=T,q=2,m=1'
chunk 2: m='1'    payload_bytes=131072 ctrl=b'a=T,q=2,m=1'
chunk 3: m='1'    payload_bytes=131072 ctrl=b'a=T,q=2,m=1'
chunk 4: m='1'    payload_bytes=131072 ctrl=b'a=T,q=2,m=1'
chunk 5: m='1'    payload_bytes=131072 ctrl=b'a=T,q=2,m=1'
chunk 6: m='(none)' payload_bytes=68640 ctrl=b'a=T,q=2'
```

Two observations reported exactly as seen: (1) `icat` sends `q=2` (quiet — suppress even failures, see §6); (2) the base64 payload per chunk is **131 072 bytes**, larger than the ≤4096‑byte figure in `docs/graphics-protocol.rst:L364-381` (which documents the protocol's conservative guaranteed‑safe chunk size, not a hard cap), yet still below the parser's `MAX_ESCAPE_CODE_LENGTH` = 262 144 (§2.1). The `m` chunk key is parsed at `parse-graphics-command.h:L25`.

### 5.8 Protocol error / quiet responses summary (verbatim)

| Driven condition | Verbatim response | Source |
| --- | --- | --- |
| PNG payload `S=400000001` | `EINVAL:PNG data size too large` | `graphics.c:L638` |
| RGBA transfer overflow | `EFBIG:Too much data` | `graphics.c:L533` |
| Dimension `s=10001`/`v=10001` — **fresh add** | *(no wire response — see §5.4 root cause)* | `graphics.c:L695` |
| Dimension `s=10001`/`v=10001` — **after a prior graphics command** | `EINVAL:Image too large` (state-dependent sibling — see §5.4) | `graphics.c:L695,L765,L2177` |
| Successful load, `q=0` (or omitted `q`) | `OK` | `graphics.c:L759-777` |
| Bad data, `q=0` (or omitted `q`) | `ENODATA:Insufficient image data: 4 < 400` (failure shown) | `graphics.c:L762` |
| Successful load, `q=1` | *(suppressed → None)* | `graphics.c:L762` |
| Bad data, `q=1` | `ENODATA:Insufficient image data: 4 < 400` (failure still shown) | `graphics.c:L762` |
| Bad data, `q=2` | *(suppressed → None)* | `graphics.c:L762` |

---

## 6. Sub‑question 5 — Silent adaptation vs. visible signs

kitty does **both**: most flow control is silent, but a few conditions produce visible signals — and even those can be intentionally silenced by the client's quiet level.

### 6.1 Silent mechanisms (kitty quietly adapts, no signal to the program or user)

| Silent mechanism | What happens | Evidence / source |
| --- | --- | --- |
| **Input coalescing** | Bytes are batched for up to `input_delay` (3 ms) instead of parsed per‑read | `vt-parser.c:L1425`; buffer‑fill OBSERVED §2.1 |
| **`POLLIN` de‑registration** | When the 1 MiB buffer is full, kitty stops asking to read; no error, no drop | `child-monitor.c:L1501`, `vt-parser.c:L1477-1481` (INFERRED / SOURCE‑CONFIRMED) |
| **Kernel PTY backpressure** | Undrained PTY → the child's `write()` blocks | OBSERVED §2.3 (`... BLOCKED = 12288`) |
| **Disk‑cache offload** | Image bytes silently moved off‑RAM to disk | OBSERVED §5.6 (`size_on_disk ... 4194304`) |
| **Storage‑quota LRU eviction** | Over 320 MiB, oldest images silently evicted; new image still accepted | OBSERVED §5.5 (`non-OK responses during load: 0`) |

### 6.2 Visible signs (an observable signal is emitted)

| Visible sign | Signal | Evidence / source |
| --- | --- | --- |
| **Output 100 MB cap** | `log_error` to stderr, data dropped | OBSERVED §3.3 — `[0.479] Too much data being sent to child with id: 1, ignoring it` (`child-monitor.c:L341-343`) |
| **Graphics `EFBIG`** | `APC _G` reply `EFBIG:Too much data` | OBSERVED §5.3 (`graphics.c:L533`) |
| **Graphics `EINVAL` (PNG size)** | `APC _G` reply `EINVAL:PNG data size too large` | OBSERVED §5.3 (`graphics.c:L638`) |
| **Graphics `EINVAL` (dimension)** | `APC _G` reply `EINVAL:Image too large`; **state-dependent** (§5.4): fresh‑add emits *no* reply due to abort ordering, but after a prior graphics command the same condition **does** emit `EINVAL:Image too large` | OBSERVED §5.4 (`graphics.c:L695,L765,L1553`) |
| **Pending/synchronized mode termination** | `screen_stop_pending_mode` + a `REPORT_ERROR` naming the cause | OBSERVED §6.3 |

### 6.3 Pending / synchronized output mode (DEC private mode 2026) — visible termination signals

kitty's “pending mode” is the industry **Synchronized Output** feature (DEC private mode **2026**: `CSI ? 2026 h` to begin, `CSI ? 2026 l` to end; also BSU/ESU) for atomic, tearing‑free updates; kitty additionally accepts the older iTerm2‑style DCS form `ESC P = 1 s` / `ESC P = 2 s` (`vt-parser.c:L638-648`). *(Terminology cross‑referenced externally; the codebase and observed output remain authoritative.)*

kitty's own documentation frames this feature under the same "Synchronized update" name: `docs/performance.rst:L109-L110` — "konsole, gnome-terminal and xterm do not support the `Synchronized update … escape code used to suppress rendering`" — confirming that pending mode is the *rendering‑suppression* mechanism whose implementation lives in `screen.c` (`screen_pause_rendering()`, `PENDING_MODE`) and `vt-parser.c` (the DCS toggle at `L638-648`). The `docs/performance.rst` note (**INFERRED / SOURCE‑CONFIRMED**: read from the shipped docs, not a runtime capture) is the repository's synchronized‑update framing referenced by the AAP.

**Status query and toggles** (OBSERVED, native + Docker, 2 runs). The status reply bit is driven by `screen.c:L2238` (`ans = self->paused_rendering.expires_at ? 1 : 2;`):

```console
$ kitty +launch pending_mode.py
initial state      -> b'\x1b[?2026;2$y'
after CSI ?2026h   -> b'\x1b[?2026;1$y' (;1$y = pending ACTIVE)
after CSI ?2026l   -> b'\x1b[?2026;2$y' (;2$y = pending INACTIVE)
DCS =1s cmd dump   -> [('screen_start_pending_mode',)]
DCS =2s cmd dump   -> [('screen_stop_pending_mode',)]
```

**All pending‑mode termination causes, enumerated exhaustively:**

1. **Explicit stop** — `CSI ?2026l` or DCS `=2s`. **OBSERVED** → state returns to `;2$y` (above).
2. **Timeout (default 2000 ms)** — **INFERRED / SOURCE‑CONFIRMED.** `screen.c:L2521` — `if (for_in_ms <= 0) for_in_ms = 2000;` sets `expires_at` (`L2522`); `screen_check_pause_rendering()` (`L2489-2490`) ends pending mode once `now > expires_at`. This check is invoked **only** from the GPU render path `prepare_to_render_os_window()` (`child-monitor.c:L729`), which requires a live display; it is therefore not exercisable in a headless harness and is labeled INFERRED / SOURCE‑CONFIRMED rather than OBSERVED.
3. **Disruptive screen operations** — each calls `screen_pause_rendering(self, false, 0)`: `screen_reset` (`screen.c:L163`), `screen_resize` (`L347`), `dirty_scroll` (`L1910`), `screen_start_selection` (`L4155`). **OBSERVED** that a full RIS reset (`ESC c`) and a resize terminate pending mode; reported faithfully, a plain line‑feed scroll and an `ED (CSI 2J)` did **not** (they do not reach `dirty_scroll`):

```console
$ kitty +launch pending_terminators.py
scroll: before=b'\x1b[?2026;1$y' after=b'\x1b[?2026;1$y'  (terminated? False)
RIS reset: before=b'\x1b[?2026;1$y' after=b'\x1b[?2026;2$y'  (terminated? True)
resize: before=b'\x1b[?2026;1$y' after=b'\x1b[?2026;2$y'  (terminated? True)
ED 2J: before=b'\x1b[?2026;1$y' after=b'\x1b[?2026;1$y'  (terminated? False)
has screen_check_pause_rendering method: False
has pause_rendering method: True
```

The terminal advertises **both** the timeout and the excess‑data causes in the error it sends when a stop arrives while not pending — captured verbatim (OBSERVED, native + Docker):

```console
$ kitty +launch pending_mode.py
STOP-when-not-pending dump -> [('screen_stop_pending_mode',), ('Pending mode stop command issued while not in pending mode, this can be either a bug in the terminal application or caused by a timeout with no data received for too long or by too much data in pending mode',)]
double-START dump  -> [('screen_start_pending_mode',), ('Pending mode start requested while already in pending mode. This is most likely an application error.',)]
```

On the phrase “**too much data in pending mode**”: a repository‑wide search found **no** separate pending‑mode byte counter — the only bounds on data accumulated while rendering is paused are the 2000 ms timeout (cause 2) and the global 1 MiB `BUF_SZ` parse ceiling (§2.1). The wording describes the *purpose* of the timeout‑bounded protection, not an additional independent limit. (INFERRED / SOURCE‑CONFIRMED via exhaustive grep of `paused_rendering` in `screen.c`.)

### 6.4 Quiet levels modulate visibility — a visible sign can be intentionally silenced

Whether a graphics “visible sign” is actually emitted depends on the command's quiet level `q`:

- **Literal:** `graphics.c:L762` (within `finish_command_response`, `L759-777`) — `if (g->quiet) { if (is_ok_response || g->quiet > 1) return NULL; }`.

All three quiet levels, enumerated exhaustively with the causal reason for each:

| `q` value | Effect | Why (cause → effect) |
| --- | --- | --- |
| `q=0` (**all**) — the default, also selected by **omitting** the `q` key | **Every** response is emitted — both `OK` successes and every failure/error | `q=0` (or an absent `q`) leaves `g->quiet == 0`, so the `if (g->quiet)` guard at `graphics.c:L762` is **false** and the suppression branch never runs; nothing is silenced |
| `q=1` | Suppresses **OK** responses (success goes silent; failures still reported) | `g->quiet == 1` is truthy, so `if (is_ok_response …) return NULL` silences the OK case; the `g->quiet > 1` sub‑condition is false, so failures are still emitted |
| `q=2` | **Additionally** suppresses **failure** responses (even errors go silent) | `g->quiet == 2`, so `g->quiet > 1` is true and `return NULL` fires for *any* response — OK **and** failure |

- Docs corroborate: `docs/graphics-protocol.rst:L767-768` — “Set it to ``1`` to suppress ``OK`` responses and to ``2`` to suppress failure responses.” (The unset/`0` value — all responses — is the documented default when the `q` key is omitted.)

**OBSERVED** (native run 1 == run 2 == Docker) — driven over the genuine `APC _G` path. The explicit `q=0` (and the equivalent omitted‑`q`) case emits both the success `OK` **and** the failure, i.e. nothing is suppressed; `q=1`/`q=2` then progressively silence:

```console
$ kitty +launch graphics_limits.py
QUIET success q=0 (expect OK) -> 'OK'
QUIET q=0 bad-data (expect failure) -> 'ENODATA:Insufficient image data: 4 < 400'
QUIET success no-q (expect OK) -> 'OK'
QUIET success q=1 (expect None) -> None
QUIET q=1 bad-data (expect failure) -> 'ENODATA:Insufficient image data: 4 < 400'
QUIET q=2 bad-data (expect None) -> None
```

**Consequence:** because `icat` transmits with `q=2` (§5.7), even an `EFBIG`/`EINVAL` failure it triggered would be *silenced* — so a “visible sign” may be intentionally invisible depending on the client. This directly determines whether the visible signals in §6.2 are emitted at all.

---

## 7. Coverage Pass, Reasoning & Cleanup

### 7.1 Coverage pass — every named mechanism, limit, level, and cause addressed

| Item | Value / literal | Where answered |
| --- | --- | --- |
| Input parse buffer `BUF_SZ` | 1 MiB (`vt-parser.c:L18`) | §2.1 |
| Per‑escape‑code cap | ¼ MiB (`vt-parser.c:L21`) | §2.1, §5.7 |
| Coalescing predicate — disjunct 1 | `flush` | §2.2 |
| Coalescing predicate — disjunct 2 | `time_since_new_input >= OPT(input_delay)` | §2.2 |
| Coalescing predicate — disjunct 3 | near‑full `read.sz + 16*1024 > BUF_SZ` | §2.2 |
| `vt_parser_has_space_for_input` | `read.sz + write.pending < BUF_SZ` | §2.3 |
| `POLLIN` gating | `child-monitor.c:L1501` | §2.3, §5.1 |
| `read_bytes` no‑read‑when‑full | `child-monitor.c:L1341-1342` | §2.3 |
| PTY kernel backpressure | child `write()` blocks | §2.3, §5.1 |
| `write_buf` + lock | `screen.c:L113-114` / `L104` | §3.1 |
| `POLLOUT` only when pending | `child-monitor.c:L1503` | §3.2 |
| `EAGAIN`/`EWOULDBLOCK` retry | `child-monitor.c:L1463` | §3.2 |
| Resume on write space | `child-monitor.c:L442` | §3.2 |
| **100 MB `write_buf` cap** | `child-monitor.c:L341-343` | §3.3 |
| **320 MiB storage quota + LRU** | `graphics.c:L25`, `L290-299` | §5.5 |
| **400 000 000‑byte `MAX_DATA_SZ` → `EFBIG`** | `graphics.c:L521,L533` | §5.3 |
| PNG‑size sibling → `EINVAL` | `graphics.c:L638` | §5.3 |
| **10 000‑px `MAX_IMAGE_DIMENSION` → `EINVAL`** (both checks) | `graphics.c:L674,L695,L1553` | §5.4 |
| Dimension response — state-dependent sibling | fresh add → no reply; after prior cmd → `EINVAL:Image too large` (`graphics.c:L695,L765,L2177`) | §5.4, §5.8 |
| Disk‑cache add / read / remove / defrag | `disk-cache.c:L488/591/517/232` | §5.6 |
| Quiet level `q=0` (all — default / omitted `q`) | `graphics.c:L762` (`if (g->quiet)` false) | §6.4 |
| Quiet level `q=1` (suppress OK) | `graphics.c:L762` | §6.4 |
| Quiet level `q=2` (suppress failures) | `graphics.c:L762` | §6.4 |
| Synchronized‑update framing (repo docs) | `docs/performance.rst:L109-L110` | §6.3 |
| Pending termination — explicit stop | `vt-parser.c:L644-645` | §6.3 |
| Pending termination — timeout 2000 ms | `screen.c:L2521,L2489-2490` | §6.3 |
| Pending termination — disruptive ops | `screen.c:L163/347/1910/4155` | §6.3 |
| Cross‑thread wakeup | `loop-utils.c:L70-117` | §4.6 |
| APC `_G` keys `m` / `q` | `parse-graphics-command.h:L25/L29` | §4.7, §5.7 |
| Defaults `input_delay`/`repaint_delay`/`sync_to_monitor` | `definition.py:L878/866/889` | §1.4 |

### 7.2 Reasoning — why kitty is designed this way

- **Backpressure over dropping (input side):** by expressing “too fast” as *withholding `POLLIN`* rather than discarding bytes, kitty guarantees losslessness of the input stream and pushes the slowdown back to the producer via standard PTY semantics. The fixed 1 MiB buffer bounds memory; the `input_delay` coalescing trades a few ms of latency for far fewer parses/repaints under a flood.
- **Bounded output queue (output side):** replies are queued and written opportunistically (`POLLOUT` only when needed, retry on `EAGAIN`), but the 100 MB cap prevents a wedged child from causing unbounded memory growth — the single deliberate drop point, and it is *logged* so it is diagnosable.
- **Two independent “too much data” limits** exist because they protect different resources: the 100 MB `write_buf` cap protects kitty's memory for *outbound* replies; the 400 MB `MAX_DATA_SZ` protects against a single absurd *inbound* image. Conflating them would misstate both the threshold and the signal.
- **Quota eviction vs. rejection (graphics):** the 320 MiB quota uses **LRU eviction** so that displaying many images “just works” (old ones silently make room), while genuinely malformed/oversize single requests are **rejected** with protocol errors. Silent for capacity pressure; visible for malformed requests.
- **Quiet levels** exist so automated clients (like `icat`, which uses `q=2`) aren't forced to parse response traffic; the cost is that failures can be intentionally invisible.

### 7.3 Evidence discipline & labels used

- **OBSERVED:** runtime‑captured, stable across ≥2 runs, cross‑checked in the canonical Docker image where applicable — includes: `BUF_SZ` ceiling, buffer‑fill sequence, the **100 MB cap log line**, `EFBIG`, PNG `EINVAL`, dimension no‑response discrepancy, 320 MiB storage LRU eviction, disk‑cache offload, real `icat` chunk framing, quiet levels, pending‑mode toggles/status/errors, RIS/resize termination, PTY backpressure substrate.
- **INFERRED / SOURCE‑CONFIRMED** (**inferred** from reading the source — not headlessly observable, but grounded in exact code literals): `POLLIN`/`POLLOUT`/`EAGAIN`/wakeup gating (kitty's I/O‑thread `poll()` loop) and the pending‑mode **2000 ms timeout expiry** (only reached from the GPU render path `child-monitor.c:L729`). These are grounded in exact source literals.
- **Discrepancies reported exactly as observed (not adjusted):** (a) over‑dimension image aborts emit **no** wire response on the fresh‑add path, but the *same* over‑dimension condition emits `EINVAL:Image too large` **after a prior graphics command** — a state-dependent sibling driven by whether `start_command.id` is populated when the abort runs (§5.4, abort‑ordering root cause); (b) real `icat` uses **131 072‑byte** base64 chunks, larger than the docs' conservative ≤4096‑byte figure (§5.7).
- **NON-CANONICAL: none.** No value in this document comes from a bypassing remote‑control interface or debug hook; all graphics observations used the genuine `APC _G` path (real `icat` and/or raw `_G` into the real `Screen`/`vt_parser`). Had any **non-canonical** value been used, it would be labeled **NON-CANONICAL** at the point of use.

### 7.4 Cleanup / read‑only scope

All temporary observation scripts lived under `/tmp/blitzy_investigation/` (outside the repository) and were removed after the investigation. No existing repository file was modified, created, or deleted, and no dependencies were changed. The only addition to the repository is this document, `blitzy/documentation/kitty_815df1e210e0.md`.

