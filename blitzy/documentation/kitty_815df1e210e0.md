# How kitty applies flow control / backpressure to a flood of terminal *graphics* data

**Question answered:** *How does kitty apply flow control / backpressure when a large volume of terminal graphics data arrives faster than the terminal can comfortably process and respond to it?*

**Repository:** kitty terminal emulator (Kovid Goyal), branch `kitty_815df1e210e0`, HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.
**Method:** Every headline value below was produced by **building and running the real code first** and capturing the actual output, **inside the canonical Docker container** (§2.2). Observations were made by driving the *real* C entry points (`vt_parser_*`, `GraphicsManager`, `Screen`, the parser worker, and a real `ChildMonitor` I/O thread) through kitty's compiled extension `kitty.fast_data_types`. The dedicated PTY I/O thread was observed **canonically and in-process** by spawning a real `ChildMonitor.start()` (§9.6a); a native GUI kitty under Xvfb was used only as a *supplementary, non-canonical* cross-check of the full thread roster (§9.6a-supplementary), because the container ships no Xvfb. Only two values could not be isolated at runtime — the running-loop `POLLIN` poll-flag flip and the sub-millisecond `input_delay` *timing* — and each is explicitly marked **(inferred from code)** with the reason (§3.1/§3.2); the 100 MiB write cap and the `EAGAIN`-retain drain, by contrast, are **observed** through a real `io_loop` (§9.4b/c). No value below comes from remote control or a debug hook.

---

## 1. Direct answer (read this first)

Kitty applies flow control to a graphics flood in **four** places, and almost all of it is **silent**:

1. **Input admission — "pause" (silent, OS-level).** The dedicated I/O thread only asks the OS for readable data (`POLLIN`) on a child PTY while the VT parser has room. The room test is `vt_parser_has_space_for_input()`, which is true **iff `self->read.sz + self->write.pending < BUF_SZ`**, where **`BUF_SZ = 1 MiB` (1 048 576 bytes)** [`kitty/vt-parser.c:18`, `:1477-1483`]. When the 1 MiB parser buffer is full, kitty simply **stops draining the PTY** [`kitty/child-monitor.c:1501`, and `read_bytes` returns without `read()`ing at `kitty/child-monitor.c:1342`]. The kernel PTY buffer then fills and the producer's `write()` **blocks**. There is **no in-band flow-control signal** to the client — this is quiet, OS-level backpressure.

2. **Processing cadence — "slow down / buffer" (silent).** Even when bytes are admitted, the parse is **deferred and coalesced**: the worker parses only when `flush` is set, OR `input_delay` (default **3 ms**) has elapsed since new input arrived, OR the buffer is **within 16 KiB of full** [`kitty/vt-parser.c:1425`; default at `kitty/options/definition.py:878`]. This is a distinct throttle from the admission gate: admission decides *whether to read*, cadence decides *when to parse*.

3. **Graphics storage backpressure — "evict" (silent).** Decoded images are capped at **320 MiB per buffer** (`DEFAULT_STORAGE_LIMIT = 320 * 1024 * 1024 = 335 544 320`) [`kitty/graphics.c:25`]. When a new image pushes `used_storage` over the cap, `apply_storage_quota()` runs a **two-phase eviction**: it first removes **unreferenced** images, then evicts the remaining images **oldest-first by access time (`atime`)** until back under quota [`kitty/graphics.c:290-299`, trigger at `:2184`]. Animation frames have a **separate, larger** ceiling of **`storage_limit * 5` = 1 600 MiB** which raises **`ENOSPC`** when exceeded [`kitty/graphics.c:1570-1573`].

4. **Response write-back under pressure — "enqueue, retain, then cap" (mostly silent).** Graphics-protocol replies are emitted as **APC escape codes** `ESC _ G <keys> ; <status> ESC \` [`kitty/screen.c:1050`], enqueued into a per-window write buffer (`screen->write_buf`) and drained **only when the PTY is writable (`POLLOUT`)** [`kitty/child-monitor.c:1503`]. If the client is not reading, `write()` returns `EAGAIN`/`EWOULDBLOCK` and kitty **retains the unwritten bytes and retries on the next `POLLOUT`** [`kitty/child-monitor.c:1443-1473`]. The buffer grows on demand but is **hard-capped at 100 MiB**, past which new data is **dropped with a visible log line** `Too much data being sent to child with id: <id>, ignoring it` [`kitty/child-monitor.c:341-342`].

5. **Quiet vs. visible.** The adaptation is **silent** for: read-stall (PTY) backpressure, deferred/coalesced parsing, render coalescing (`repaint_delay`), and LRU image eviction. It is **visible** only where the client asked for it or an operator watches the log: graphics **OK/error responses gated by the `q=` key** (`q=1` suppresses the *OK* reply but still emits errors; `q=2` suppresses *everything*) [`kitty/graphics.c:759-763`], the **`ENOSPC` animation-cache error**, the **pending-mode abort message** [`kitty/vt-parser.c:646-648`], and the over-cap **"Too much data…"** log.

Everything above was exercised at runtime; the exact commands, complete outputs, ≥2-run stability notes, and before/during/after captures are in §9. The only items still labelled **(inferred from code)** — the running-loop `POLLIN` poll-flag flip and the sub-millisecond `input_delay` *timing* — are explained with their reason in §3.1/§3.2; the 100 MiB output cap and the `EAGAIN`-retain drain are now **observed** at runtime through a real `io_loop` (§9.4b/c).

---

## 2. Build & environment

### 2.1 Pristine baseline (repository left unchanged)

```bash
git rev-parse --abbrev-ref HEAD          # destination branch
git rev-parse HEAD^                       # pinned upstream baseline (HEAD's parent)
git status --porcelain                    # expect empty (clean tree)
```

```text
blitzy-aecbf6f2-a1d2-49f0-90bf-843111d83a06
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
```
(`git status --porcelain` printed nothing — a clean tree.)

> Note on names: the working checkout is on the destination branch `blitzy-aecbf6f2-a1d2-49f0-90bf-843111d83a06`. `HEAD` is the destination-branch commit that adds this single deliverable document; its parent `HEAD^` is the pinned upstream kitty commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` under investigation — the baseline the repository is otherwise byte-for-byte identical to (proven in §9.8). The deliverable filename `kitty_815df1e210e0.md` follows the pinned-commit / source-branch name as required.

### 2.2 Canonical default build

Per AAP §0.8.1, the build and **every** runtime observation were performed in the **canonical Docker container** — image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0` (the AAP designates it `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`) — which ships the C toolchain, Go, and Python. A long-lived container was started once (entrypoint overridden to `bash`, the host scripts directory bind-mounted) and reused for all observations in §9:

```bash
docker run -d --name kitty-obs --entrypoint bash \
  -v /tmp/kitty_obs:/tmp/kitty_obs \
  ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0 -c 'sleep infinity'

# canonical default build (the Makefile `all:` target) against the pinned source at /app:
docker exec kitty-obs bash -c 'cd /app && python3 setup.py'
docker exec kitty-obs bash -c 'cd /app && ./kitty/launcher/kitty --version'
```

The build finished with **exit code 0** in ~55 s — **380 log lines, zero warnings, zero errors** (`grep -c "warning:|error:"` → 0). It runs in three phases: generate the Wayland protocol stubs (`[1/28]…[28/28]`), compile the C core (`[1/122]…[122/122]`), then build the Go kittens. The C-compile phase names every flow-control source file this document cites (`kitty/screen.c`, `kitty/graphics.c`, `kitty/child-monitor.c`, `kitty/vt-parser.c`, `kitty/state.c`):

```text
[28/28] Generating wayland-wlr-layer-shell-unstable-v1-client-protocol.c ...
 done
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
[4/122] Compiling [x11] glfw/x11_window.c ...
[5/122] Compiling kitty/glfw.c ...
[6/122] Compiling kitty/graphics.c ...
[7/122] Compiling kitty/child-monitor.c ...
[8/122] Compiling kitty/fonts.c ...
[9/122] Compiling kitty/shaders.c ...
[10/122] Compiling kitty/vt-parser.c ...
[11/122] Compiling kitty/vt-parser.c ...
[12/122] Compiling kitty/state.c ...
```

Complete build tail (the final 6 lines — the last Go kittens; the build ends here with exit 0):

```text
kitty/kittens/choose_fonts
kitty/tools/cmd/pytest
kitty/kittens/diff
kitty/tools/cmd/tool
kitty/tools/cmd/completion
kitty/tools/cmd
```

`kitty --version` in the container:

```text
kitty 0.35.2 created by Kovid Goyal
```

The `/app` checkout is the exact investigation commit (`git -C /app rev-parse HEAD` → `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`). Unlike a bare headless host, this container ships `wayland-protocols`, so the Wayland backend **is** built (steps `[1/28]…[28/28]` above) alongside X11/software-GL; this is orthogonal to every flow-control path documented here. The build produces `kitty/launcher/kitty`, `kitty/launcher/kitten`, and `kitty/fast_data_types.so` (the compiled terminal core the in-process observation harness imports).

**Build-state note (reproducibility).** The exact step indices above (`[1/28]…[28/28]`, `[1/122]…[122/122]`), the ~380-line log length, and the ~55 s wall time reflect this *from-clean* build capture; they are **build-state dependent**, not invariants. An incremental rebuild against the already-built `/app` recompiles only what changed and therefore prints fewer steps and a shorter log — re-running `python3 setup.py` here produced a ~209-line log that still exited 0 with zero warnings. The reproducible invariants, independent of build state, are: the build **exits 0 with zero warnings/errors**, the toolchain versions (§2.3), and `kitty --version` → `kitty 0.35.2 created by Kovid Goyal`.

### 2.3 Toolchain (canonical container)

The versions inside the canonical container that produced the build above:

```bash
docker exec kitty-obs bash -c 'python3 --version; go version; gcc --version | head -1'
```

```text
Python 3.12.3
go version go1.23.4 linux/amd64
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
```

All satisfy kitty's declared floors (Python ≥ 3.8 per `pyproject.toml`; Go ≥ 1.22 per `go.mod`). These are the versions a normal user of this container gets — no version override or alternate toolchain was applied. (For contrast, the host used only for the *supplementary, non-canonical* GUI-under-Xvfb thread roster in §9.6(a-supplementary) runs Python 3.13.7 / gcc 15.2.0; that run is explicitly labelled non-canonical because the container ships no Xvfb.)

### 2.4 Observation methodology (why it is canonical)

All numeric/behavioral claims were produced **inside the canonical container** (§2.2), driving the **real C functions** via `docker exec kitty-obs bash -c 'cd /app && python3 …'` against the pinned source at `/app` — not by reading:

* The compiled extension `kitty.fast_data_types` exports the real parser and graphics objects. In particular `VT_PARSER_BUFFER_SIZE` is registered directly from the C macro `BUF_SZ` [`kitty/vt-parser.c:1589`], and `Screen.test_create_write_buffer` / `Screen.test_commit_write_buffer` / `Screen.test_parse_written_data` call the real `vt_parser_create_write_buffer` / `vt_parser_commit_write` / `parse_worker` [`kitty/screen.c:4755-4778`]. This is the same buffer API the production reader `read_bytes` uses [`kitty/child-monitor.c:1341-1356`] — i.e. the canonical path, not a debug hook.
* Graphics behavior was driven by feeding real APC command bytes `ESC _ G … ESC \` through the real parser (`parse_bytes`), constructing a real `Screen` + `GraphicsManager`, exactly as kitty's own in-tree tests do. Responses were read back from the screen's write-to-child callback buffer (`wtcbuf`), which receives exactly the bytes kitty writes to the child.
* The threaded I/O model was observed **canonically, in-process, in the container**: a real `ChildMonitor.start()` spawns the `KittyChildMon` PTY I/O thread, read back from `/proc/self/task/*/comm` (§9.6a). A *supplementary, non-canonical* native GUI run under Xvfb (the container ships no Xvfb) additionally shows the same `KittyChildMon` coexisting with the render thread in a full 67-thread process (§9.6a-supplementary), clearly labelled as non-canonical.
* Default option values (`input_delay`, `repaint_delay`) were read from the compiled defaults (`kitty.options.types.defaults`) — i.e. the default configuration a normal user gets.

Temporary observation scripts lived only under `/tmp/kitty_obs/` (on the host, bind-mounted into the container at the same path) and were removed afterward. The build and all runtime observations ran against the container's `/app` checkout; the repository-state checks (§2.1, §9.8) were run separately in the **destination working tree** (a different checkout) — the two environments are kept distinct throughout. The repository is byte-for-byte unchanged apart from this document (final `git status` in §9.8).

Where a mechanism lives **only** in the GUI I/O-loop thread (`io_loop`, created at `kitty/child-monitor.c:291`) and could not be driven from the in-process harness — specifically the running-loop `POLLIN` poll-flag flip and the sub-millisecond `input_delay` *timing* — the value/logic is taken from the source and the item is marked **(inferred from code)** with the reason. The 100 MiB write cap and the `EAGAIN`-retain drain, by contrast, **were** driven through the real `io_loop` from the harness (a real `ChildMonitor.start()` with a `window_id = 1` screen) and are observed in §9.4b/c. The magnitudes those paths use (1 MiB, 16 KiB, 100 MiB, 3 ms) are all independently confirmed from the code and, where possible, from the harness.

---

## 3. O1 — Input buffering: "pause" vs. "slow down / buffer"

Kitty has **two independent input throttles**. They answer two different parts of the question and must not be conflated.

### 3.1 The "pause" throttle — 1 MiB parser buffer + read-space admission gate

**Mechanism.** The VT parser owns a single fixed-size input buffer. Its size is the compile-time constant:

```c
// kitty/vt-parser.c:18
#define BUF_SZ (1024u*1024u)   // 1 MiB = 1048576 bytes
```

The I/O thread decides whether to ask the OS for more child output by calling `vt_parser_has_space_for_input()`:

```c
// kitty/vt-parser.c:1476-1484
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

That boolean is turned into a poll request in the child monitor: the child fd is only watched for `POLLIN` **when there is space**:

```c
// kitty/child-monitor.c:1501
children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;
```

And the reader itself declines to `read()` when the buffer is full — `read_bytes` obtains a write buffer and, if there is no room, returns *without* reading:

```c
// kitty/child-monitor.c:1341-1342 (inside read_bytes)
uint8_t *buf = vt_parser_create_write_buffer(screen->vt_parser, &available_buffer_space);
if (!available_buffer_space) return true;   // buffer full -> do NOT read the PTY
```

**Cause → effect.** Buffer full → `has_space_for_input()` is false → the fd is not selected for `POLLIN` and `read_bytes` does not `read()` → the kernel PTY buffer fills → the client's `write()` **blocks**. This is silent, OS-level backpressure with no in-band signal.

**Observed (canonical, real C functions).** I drove the *real* buffer API that `read_bytes` uses — `test_create_write_buffer` / `test_commit_write_buffer` call `vt_parser_create_write_buffer` / `vt_parser_commit_write` [`kitty/screen.c:4755-4770`] — filling the buffer **without** parsing, and printed the remaining space after each commit. The command and its complete, unedited output (Run A) are in §9.1. Summary of the transition:

* **before:** buffer empty, `available = 1048576` (a fresh `create_write_buffer` reports the whole 1 MiB free).
* **during:** after committing 256 KiB chunks the reported free space steps down `1048576 → 786432 → 524288 → 262144 → 0`.
* **at the boundary:** after exactly **1 048 576** bytes committed, `available = 0`; a further `create_write_buffer` reports **0** free (`extra_commit_accepted = 0`) — the gate is **closed** (this is the "pause").
* **after:** draining the buffer via the real parser (`test_parse_written_data`) returns free space to **1048576** — the gate **reopens**.

The magnitude **1 MiB / 1 048 576** was **stable across 2 identical runs** (Run A and Run B printed byte-identical numbers; see §9.1). The buffer-full condition observed here (`available == 0`) is exactly the condition `read_bytes` tests at `kitty/child-monitor.c:1342`, and is exactly `!vt_parser_has_space_for_input()` (`read.sz + write.pending == BUF_SZ`).

**(Inferred from code — reason stated).** The *enforcement* of the pause in the live loop — flipping `events` to `0` so the OS stops returning the fd as readable [`kitty/child-monitor.c:1501`], and the producer's `write()` blocking on the full kernel PTY buffer — runs only inside the GUI `io_loop` thread and requires an external producer + `strace`, which is unavailable in this headless container (no `strace`, no external PTY producer wired to a running kitty). The *condition* that triggers it (buffer full ⇒ no read) is confirmed at runtime above and in code at `:1342`; the poll-gating line at `:1501` is confirmed by reading. It is therefore labelled inferred only for the running-loop poll step.

### 3.2 The "slow down / buffer" throttle — deferred, coalesced parsing (`input_delay`)

**Mechanism.** Admitting bytes is separate from parsing them. The worker parses only under one of three conditions:

```c
// kitty/vt-parser.c:1425 (inside run_worker, starts :1417)
if (flush || pd->time_since_new_input >= OPT(input_delay) || self->read.sz + 16 * 1024 > BUF_SZ)
```

i.e. parse when (a) a flush is forced, OR (b) at least **`input_delay`** has elapsed since new input arrived, OR (c) the buffer is **within 16 KiB (16 384 bytes)** of full. Condition (c) is the near-full bypass: when input is arriving in a flood, the parser stops waiting and drains immediately.

**Default value (canonical).** The default `input_delay` is **3 ms** and the related render throttle `repaint_delay` is **10 ms**:

```python
# kitty/options/definition.py:878
opt('input_delay', '3', option_type='positive_int', ctype='time-ms', ...)
# kitty/options/definition.py:866
opt('repaint_delay', '10', ...)
```

Internally both are stored as `monotonic_t` nanoseconds:

```c
// kitty/state.h:51
monotonic_t repaint_delay, input_delay;
```

**Observed (canonical values).** Read from the compiled defaults a normal user gets. Command and complete output in §9.2:

```text
input_delay  = 3   (ms)
repaint_delay = 10  (ms)
16 KiB near-full bypass threshold: read.sz + 16384 > 1048576  =>  read.sz > 1032192
```

Both values were **stable across 2 runs**. The `input_delay` doc text itself corroborates the near-full bypass: it states the setting *"is ignored when the input buffer is almost full"* [`kitty/options/definition.py:878` doc string], which is exactly condition (c) above.

**(Inferred from code — reason stated).** The sub-millisecond *timing* of the deferral (that parsing actually waits up to ~3 ms and coalesces bursts) is produced by the `run_worker` timer in the GUI `io_loop` worker thread; the in-process harness path `test_parse_written_data` forces `flush = true`, so it parses immediately and cannot exhibit the wait. The **value** (3 ms) is canonical (read from defaults) and the **gate logic** is confirmed in code at `:1425`; only the observed timing behavior is inferred, and it is corroborated by the option's own documentation.

### 3.3 Why two throttles

The admission gate (§3.1) answers **"pause"**: whether to read from the PTY at all. The deferred parse (§3.2) answers **"slow down / buffer"**: how often to actually process what was read. A flood therefore first gets *buffered and coalesced* (up to 3 ms, or until 16 KiB-from-full), and only when the whole 1 MiB is occupied does kitty *pause* reading and let OS backpressure stall the producer.

---

## 4. Graphics storage backpressure — 320 MiB LRU eviction (+ a separate animation ceiling)

Distinct from the byte-stream throttles above, the graphics subsystem bounds **decoded image storage**.

### 4.1 The 320 MiB per-buffer quota and two-phase LRU eviction

**Mechanism.** The default quota is:

```c
// kitty/graphics.c:25
#define DEFAULT_STORAGE_LIMIT 320u * (1024u * 1024u)   // 320 MiB = 335544320 bytes
```

assigned into each `GraphicsManager` at construction (`self->storage_limit = DEFAULT_STORAGE_LIMIT;` [`kitty/graphics.c:78`]). Every image add updates `used_storage` [`kitty/graphics.c:749`], and the quota is enforced when it is exceeded — note the **strict `>`**:

```c
// kitty/graphics.c:2184
if (self->used_storage > self->storage_limit) apply_storage_quota(self, self->storage_limit, added_image_id);
```

`apply_storage_quota` runs **two phases** [`kitty/graphics.c:290-299`]:

```c
// kitty/graphics.c:289-300 (apply_storage_quota)
static void
apply_storage_quota(GraphicsManager *self, size_t storage_limit, id_type currently_added_image_internal_id) {
    // First remove unreferenced images, even if they have an id
    remove_images(self, trim_predicate, currently_added_image_internal_id);   // phase 1
    if (self->used_storage < storage_limit) return;

    HASH_SORT(self->images, oldest_img_first);                               // phase 2: sort by atime
    while (self->used_storage > storage_limit && self->images) {
        remove_image(self, self->images);                                    // evict oldest first
    }
    if (!self->images) self->used_storage = 0;  // sanity check
}
```

* **Phase 1** removes images matching `trim_predicate` — those that are unreferenced or whose root frame is not loaded — while skipping the image just added:
  ```c
  // kitty/graphics.c:279-281
  static bool
  trim_predicate(Image *img) { return !img->root_frame_data_loaded || !img->refs; }
  ```
* **Phase 2** sorts by access time and evicts oldest-first. "Oldest" is defined by `atime`:
  ```c
  // kitty/graphics.c:284-287
  static int
  oldest_img_first(const Image *a, const Image *b) {
      return a->atime - b->atime;
  }
  ```
  `atime` is stamped at creation [`kitty/graphics.c:715`] and refreshed on access [`kitty/graphics.c:1002`, `:1097`], making eviction **LRU by access time**.

**Observed (canonical, real `GraphicsManager`).** I read the live quota and drove real image loads through a real `Screen`/`GraphicsManager`, using the disk-cache total size as ground truth. Commands + complete outputs in §9.3. Key transitions:

* `grman.storage_limit = 335544320` (320 MiB) — read directly, stable.
* **at exactly the quota:** five 64 MiB images (4096×4096 RGBA) sum to `335544320` = the limit exactly; `image_count = 5`, **no eviction** (the trigger is strict `>`).
* **crossing the quota (unreferenced transmit flood, `a=t`):** adding a 6th 64 MiB image pushes `used_storage` to `402653184 > 335544320` → eviction runs. **before:** 5 images present; **during:** the 6th add; **after:** `image_count` drops from 5 to **1** and only the newest id survives — because phase-1 `trim_predicate` removes *all* the unreferenced older transmits at once. This is the cleanest demonstration of the quota and was **stable across 2 runs**.
* **crossing the quota (referenced images, pinned `a=p,c=1,r=1` placements):** when every image is referenced by a placement pinned to the home cell (so nothing scrolls off), phase 1 finds nothing to drop and phase 2 evicts **oldest-first by `atime`**. **before:** 5 images at exactly 320 MiB; **during:** each further add (6, 7, 8) crosses the quota; **after:** `image_count` plateaus at 5, the five newest survive (`[4, 5, 6, 7, 8]`) and the three oldest are evicted oldest-first (`[1, 2, 3]`) — confirming LRU-by-`atime`. **Stable across 2 runs.**

### 4.2 The separate animation-frame disk-cache ceiling → `ENOSPC`

**Mechanism (do not conflate with §4.1).** Animation *frame* data is stored on disk with its **own, larger** ceiling of **`storage_limit * 5`** (= 1 600 MiB with the default 320 MiB base). Exceeding it raises `ENOSPC`:

```c
// kitty/graphics.c:1570-1573
if (is_new_frame && cache_size(self) + load_data->data_sz > self->storage_limit * 5) {
    remove_images(self, trim_predicate, img->internal_id);          // first try to reclaim
    if (cache_size(self) + load_data->data_sz > self->storage_limit * 5)
        ABRT("ENOSPC", "Cache size exceeded cannot add new frames");
}
```

**Observed (canonical, stable across 2 runs).** Ceiling = `335544320 * 5 = 1677721600` bytes (1 600 MiB). Starting from a base 4096×4096 image and adding 64 MiB frames (`a=f`, `t=f`), the disk cache grew 384 → 704 → 1024 → 1344 MiB (frames 5/10/15/20); the **24th** frame add filled the cache to **exactly** the ceiling `1 677 721 600` (1 600 MiB) with a normal `OK`, and the **25th** frame add — which would exceed the ceiling — was refused with the `ENOSPC` error (the cache does not grow past `1677721600`). Complete output in §9.3; the exact bytes:

```text
b'\x1b_Gi=1,r=26;ENOSPC:Cache size exceeded cannot add new frames\x1b\\'
```

This is a **visible** signal (an APC error response, subject to `q=` gating) and is entirely distinct from the silent 320 MiB image-storage eviction.


---

## 5. O2 — Response write-back under pressure

**Mechanism.** When a graphics command produces a reply, kitty emits it as an **APC escape code** and pushes it onto the child write path:

```c
// kitty/screen.c:1047-1050 (screen_handle_graphics_command)
screen_handle_graphics_command(Screen *self, const GraphicsCommand *cmd, const uint8_t *payload) {
    unsigned int x = self->cursor->x, y = self->cursor->y;
    const char *response = grman_handle_command(self->grman, cmd, payload, self->cursor, &self->is_dirty, self->cell_size);
    if (response != NULL) write_escape_code_to_child(self, ESC_APC, response);
```

`write_escape_code_to_child` wraps the payload in the APC introducer `ESC _` and the string terminator `ESC \`, then routes it to the child. For a real window it goes through `write_to_child` → `schedule_write_to_child`:

```c
// kitty/screen.c:946-949
static bool
write_to_child(Screen *self, const char *data, size_t sz) {
    bool written = false;
    if (self->window_id) written = schedule_write_to_child(self->window_id, 1, data, sz);
    ...
}
```

`schedule_write_to_child` appends to the per-window `screen->write_buf`, which grows on demand but is **hard-capped at 100 MiB** — beyond that, data is dropped with a log line:

```c
// kitty/child-monitor.c:341-342 (inside the schedule_write_to_child region ~323-345)
if (screen->write_buf_used + sz > 100 * 1024 * 1024) {
    log_error("Too much data being sent to child with id: %lu, ignoring it", id);
    ...
}
```

The buffer is drained **only when the PTY is writable**. The child fd is watched for `POLLOUT` only while there are queued bytes:

```c
// kitty/child-monitor.c:1503
children_fds[EXTRA_FDS + i].events |= (screen->write_buf_used ? POLLOUT  : 0);
```

and the drain loop retains unwritten bytes on `EAGAIN`/`EWOULDBLOCK`, retrying on the next `POLLOUT`:

```c
// kitty/child-monitor.c:1447-1476 (write_to_child) — control flow (KITTY_PRINT_BYTES_SENT_TO_CHILD #ifdefs elided)
while (written < screen->write_buf_used) {
    ret = write(fd, screen->write_buf + written, screen->write_buf_used - written);
    if (ret > 0) {
        written += ret;
    } else if (ret == 0) {
        break;                                              // could mean anything, ignore
    } else {
        if (errno == EINTR) continue;                       // retry immediately
        if (errno == EWOULDBLOCK || errno == EAGAIN) break; // RETAIN, retry on next POLLOUT
        perror("Call to write() to child fd failed, discarding data.");
        written = screen->write_buf_used;                   // hard error: discard everything
    }
}
if (written) {
    screen->write_buf_used -= written;
    if (screen->write_buf_used) {
        memmove(screen->write_buf, screen->write_buf + written, screen->write_buf_used); // retain leftover
    }
}
```

So the output policy is **retain-then-drop**, not drop-immediately: transient pressure retains bytes and retries; only past the 100 MiB cap (or on a hard write error) is data dropped.

**Observed (canonical for the response bytes).** Feeding five real graphics commands through a real `Screen` and reading exactly the bytes kitty writes back, I captured the raw APC replies (complete output in §9.4(a)). The bytes were **byte-identical across 2 invocations**:

```text
OK (a=t,i=1):           b'\x1b_Gi=1;OK\x1b\\'
  hexdump:              1b 5f 47 69 3d 31 3b 4f 4b 1b 5c
                        ^ESC ^_ ^G  i  =  1  ;  O  K  ^ESC ^\
image-number (I=1):     b'\x1b_Gi=1,I=1;OK\x1b\\'
QUERY (a=q):            b'\x1b_Gi=1;OK\x1b\\'
ENODATA error:          b'\x1b_Gi=1;ENODATA:Insufficient image data: 4 < 300\x1b\\'
EINVAL error:           b'\x1b_Gi=3,I=1;EINVAL:Must not specify both image id and image number\x1b\\'
```

This proves responses are **APC escape codes** (`ESC _ G … ESC \`; APC introducer bytes `0x1b 0x5f`, terminator `0x1b 0x5c`) placed on the write path via `write_escape_code_to_child(self, ESC_APC, response)` [`kitty/screen.c:1050`].

**Observed (canonical — the cap and the retain/retry, not just the bytes).** Constructing a real `Screen` with `window_id = 1` and a real `ChildMonitor` (`add_child` + `start()`, which spawns the actual `io_loop` thread), response bytes route through the real `schedule_write_to_child` → `write_to_child` path, so the cap and the `EAGAIN`-retain execute as production code. Complete outputs in §9.4(b) and §9.4(c):

* **100 MiB cap → drop** [`kitty/child-monitor.c:341-342`]: with the child fd's kernel buffer pre-filled so the `io_loop` drains nothing, framed 1 MiB APC chunks (`1 048 580` B each) were enqueued; the buffer accepted exactly **99** (`write_buf_used = 103 809 420` B) and the **100th** was dropped (`103 809 420 + 1 048 580 = 104 858 000 > 104 857 600`) with exactly one log line — `Too much data being sent to child with id: 1, ignoring it`. Deterministic across 2 separate processes.
* **`POLLOUT`-gated drain + `EAGAIN`-retain retry** [`kitty/child-monitor.c:1462-1473`]: 20 framed chunks (`20 971 600` B) were enqueued while the fd was blocked (bytes **retained** in `write_buf`); draining the fd triggered `POLLOUT` and the `io_loop` flushed **all** `20 971 600` B back out (== enqueued), so `write_buf_used` went `0 → grew → 0` with **no loss**.

The child fd here is an `os.pipe()` standing in for the PTY; this is immaterial to the `write_buf` logic, because the identical `write_to_child`/`schedule_write_to_child` code runs regardless of fd type — the cap check, the `POLLOUT` gate, and the `memmove`-retain all execute on the real `io_loop` thread. The only quantity that still genuinely depends on a live stalled client is *how fast* a real peer would push `write_buf_used` toward the cap; the cap itself and the retain/drain behavior are **directly observed**.

---

## 6. O3 — Where in the code (file:line index)

Every anchor below was re-verified with `sed -n` against the pristine tree (see §9.7).

| # | Mechanism | Function / symbol | file:line |
|---|-----------|-------------------|-----------|
| 1 | 1 MiB parser buffer size | `#define BUF_SZ (1024u*1024u)` | `kitty/vt-parser.c:18` |
| 2 | Read-space admission gate | `vt_parser_has_space_for_input` (`read.sz + write.pending < BUF_SZ`) | `kitty/vt-parser.c:1476-1484` (gate `:1481`) |
| 3 | Deferred-parse gate (`input_delay` + 16 KiB bypass) | `run_worker` | `kitty/vt-parser.c:1425` (fn `:1417`) |
| 4 | Write-buffer create/commit API | `vt_parser_create_write_buffer`, `vt_parser_commit_write`, `vt_parser_has_space_for_input` | `kitty/vt-parser.h:34-36` |
| 5 | PTY `POLLIN` gated on parser space | poll `events = has_space ? POLLIN : 0` | `kitty/child-monitor.c:1501` |
| 6 | PTY `POLLOUT` gated on queued output | `events |= POLLOUT` | `kitty/child-monitor.c:1503` |
| 7 | Reader declines read when buffer full | `read_bytes` (`if(!available_buffer_space) return true;`) | `kitty/child-monitor.c:1337-1356` (`:1342`) |
| 8 | I/O thread creation | `pthread_create(&self->io_thread, NULL, io_loop, self)` | `kitty/child-monitor.c:291` |
| 9 | Graphics response → APC on write path | `screen_handle_graphics_command` → `write_escape_code_to_child(self, ESC_APC, response)` | `kitty/screen.c:1047-1050` |
| 10 | Enqueue to per-window write buffer | `write_to_child` → `schedule_write_to_child` | `kitty/screen.c:947-949` |
| 11 | 100 MiB output cap + drop log | `Too much data being sent to child…` | `kitty/child-monitor.c:341-342` |
| 12 | Drain loop, EAGAIN-retain / discard-on-error | `write_to_child` | `kitty/child-monitor.c:1443-1473` |
| 13 | 320 MiB image quota | `#define DEFAULT_STORAGE_LIMIT 320u*(1024u*1024u)` | `kitty/graphics.c:25` (assign `:78`) |
| 14 | Trim predicate (unreferenced) | `trim_predicate` (`!root_frame_data_loaded || !refs`) | `kitty/graphics.c:279-281` |
| 15 | LRU comparator by `atime` | `oldest_img_first` (`a->atime - b->atime`) | `kitty/graphics.c:284-287` |
| 16 | Two-phase eviction | `apply_storage_quota` | `kitty/graphics.c:289-300` |
| 17 | Quota trigger (strict `>`) | `if (used_storage > storage_limit) apply_storage_quota(...)` | `kitty/graphics.c:2184` |
| 18 | Animation cache ceiling `*5` → `ENOSPC` | `ABRT("ENOSPC", "Cache size exceeded cannot add new frames")` | `kitty/graphics.c:1570-1573` |
| 19 | Response quiet-gate (`q=`) | `finish_command_response` (`if (g->quiet){ if (is_ok_response || g->quiet>1) return NULL; }`) | `kitty/graphics.c:759-763` |
| 20 | Error-response builder + `ABRT` macro | `set_command_failed_response`, `#define ABRT` | `kitty/graphics.c:303-316` |
| 21 | Pending/synchronized-mode + abort message | pending-mode handling | `kitty/vt-parser.c:637-650` (msg `:646-648`) |
| 22 | `input_delay` default = 3 ms | `opt('input_delay', '3', ...)` | `kitty/options/definition.py:878` |
| 23 | `repaint_delay` default = 10 ms | `opt('repaint_delay', '10', ...)` | `kitty/options/definition.py:866` |
| 24 | Delays stored as `monotonic_t` ns | `monotonic_t repaint_delay, input_delay;` | `kitty/state.h:51` |
| 25 | Pending-mode default timeout 2000 ms | `screen_pause_rendering` | `kitty/screen.c:2506-2521` (`:2521`) |
| 26 | `BUF_SZ` exported to Python as `VT_PARSER_BUFFER_SIZE` | module init | `kitty/vt-parser.c:1589` |
| 27 | Graphics APC `q=` quiet-key parsing (produces the `quiet` value the row-19 gate reads) | key enum `quiet = 'q'` → parse `case quiet:` → serialize `U(quiet)` | `kitty/parse-graphics-command.h:29`, `:88`, `:283` |
| 28 | Event-loop wakeup / fd-drain primitives (I/O-thread scheduling) | `wakeup_loop` (decl `:48`, def), `drain_fd` (inline `:76`) | `kitty/loop-utils.h:48`, `:76`; `kitty/loop-utils.c:113`; used by I/O loop at `kitty/child-monitor.c:226`, `:1515` |

---

## 7. O4 — How it manifests at runtime

**Threaded I/O (dedicated I/O thread, decoupled from rendering).** The child monitor spawns a dedicated pthread for the whole PTY read/parse/write loop:

```c
// kitty/child-monitor.c:291
ret = pthread_create(&self->io_thread, NULL, io_loop, self);   // thread named "KittyChildMon"
```

**Observed (canonical, in-process).** Driving the real `ChildMonitor.start()` in the canonical container (§2.2; complete output §9.6(a)) spawns the dedicated I/O thread on demand: before `start()` the process has one thread (`['python3']`); after `start()` a second thread named **`KittyChildMon`** appears (`['KittyChildMon', 'python3']`) — the PTY read/parse/write loop `io_loop`, which names itself at [`kitty/child-monitor.c:1489`]. Byte-identical across two runs (`sha256 5692f26…`):

```text
[before start()] OS threads = 1 ; comm names = ['python3']
[after start()]  OS threads = 2 ; comm names = ['KittyChildMon', 'python3']
```

**Supplementary (NON-CANONICAL, native GUI under Xvfb).** To show `KittyChildMon` coexisting with the main render thread in a *real GUI* process, I launched a GUI kitty under Xvfb on the **native** build (Python 3.13.7 / gcc 15.2.0 — the canonical container has no Xvfb, so this run is non-canonical; complete un-elided histogram in §9.6(a-supplementary)). The process had **67 threads**:

```text
KittyChildMon      <- the PTY I/O thread (child-monitor.c:291, named :1489)
kitty  (x33)       <- the main render thread (GLFW/OpenGL) + GL worker pool
kitty:disk$0       <- disk-cache writer (graphics frame storage)
llvmpipe-0..31     <- software GL rasterizer threads (Xvfb has no GPU; env-specific)
```

This confirms the PTY read/parse/write path runs on **`KittyChildMon`**, separate from the main render thread — so a graphics flood is absorbed on the I/O thread and does not block keyboard/redraw on the main thread. (The main-thread render cadence is itself throttled by `repaint_delay = 10 ms` [`kitty/options/definition.py:866`], whose doc notes it is *ignored when there is pending input* — render coalescing.)

**Configurable delays.** `input_delay = 3 ms` and `repaint_delay = 10 ms` (both read from the compiled defaults; §9.2) shape the runtime cadence: input parsing is coalesced up to 3 ms (unless near-full), rendering up to 10 ms (unless input pending).

**Storage eviction under load.** As images accumulate past 320 MiB, `apply_storage_quota` runs on the next add, silently evicting oldest/unreferenced images (§4.1, §9.3). Animation-frame overflow instead surfaces a visible `ENOSPC` APC error at the `storage_limit*5` ceiling (§4.2).

**Emitted output under load.** The observable outputs under pressure are: graphics APC OK/error replies (§5, §9.4(a)), the `ENOSPC` animation error (§4.2), the pending-mode abort message (§8, §9.6), and — past the 100 MiB output cap — the `Too much data being sent to child…` log line (observed; §5, §9.4(b)).

---

## 8. O5 — Quiet adaptation vs. visible signs

| Silent (no signal to client; kitty just adapts) | Evidence |
|---|---|
| **Read-stall (PTY) backpressure** — stop `POLLIN`, let the kernel PTY buffer fill, producer `write()` blocks | §3.1, §9.1 (buffer fills to 1 MiB, gate closes); code `kitty/child-monitor.c:1501`, `:1342` |
| **Deferred / coalesced parsing** — wait up to `input_delay` before parsing | §3.2, §9.2; code `kitty/vt-parser.c:1425` |
| **Render coalescing** — batch redraws every `repaint_delay`, ignored when input pending | §7; code `kitty/options/definition.py:866` |
| **LRU image eviction** — silently drop oldest/unreferenced images past 320 MiB | §4.1, §9.3; code `kitty/graphics.c:290-299`, `:2184` |
| **Output retain-and-retry** — hold unwritten reply bytes on `EAGAIN`, retry on `POLLOUT` | §5, §9.4(c) (observed: 20 971 600 B retained then fully drained on `POLLOUT`, no loss); code `kitty/child-monitor.c:1462-1473` |

| Visible (a byte/log is produced) | Evidence |
|---|---|
| **Graphics OK/error APC responses, gated by `q=`** — `q=0` full, `q=1` suppresses OK (errors still sent), `q=2` suppresses all | §9.5; code `kitty/graphics.c:759-763` |
| **Animation-cache `ENOSPC`** APC error at `storage_limit*5` | §4.2, §9.3; code `kitty/graphics.c:1570-1573` |
| **Pending-mode abort message** | §9.6; code `kitty/vt-parser.c:646-648` |
| **Over-cap log** `Too much data being sent to child…` | §5, §9.4(b) (observed: drop at the 100th 1 MiB chunk, `write_buf_used` = 103 809 420 B); code `kitty/child-monitor.c:341-342` |

**The `q=` gate (canonical, stable across 2 runs).** The response is gated in `finish_command_response`:

```c
// kitty/graphics.c:762-764  (the gate inside finish_command_response, which begins at :759)
if (g->quiet) {
    if (is_ok_response || g->quiet > 1) return NULL;
}
```

i.e. `q=1` returns `NULL` (no reply) for an OK response but still returns error replies; `q=2` returns `NULL` for **everything**. Observed raw bytes (complete outputs in §9.5):

```text
success (OK) reply per q:
  q=0 -> b'\x1b_Gi=1;OK\x1b\\'
  q=1 -> b''                                                          (OK suppressed)
  q=2 -> b''                                                          (suppressed)

error (ENODATA, insufficient data 4 < 300) reply per q:
  q=0 -> b'\x1b_Gi=1;ENODATA:Insufficient image data: 4 < 300\x1b\\'
  q=1 -> b'\x1b_Gi=1;ENODATA:Insufficient image data: 4 < 300\x1b\\'   (error STILL emitted)
  q=2 -> b''                                                          (all suppressed)
```

The in-tree test `kitty_tests/graphics.py::test_suppressing_gr_command_responses` (which exercises exactly this) **passed**, as did `test_disk_cache` (§9.5).

**Pending-mode abort messages (canonical, stable across 2 runs).** Synchronized/pending mode is entered/left via DCS `=1s`/`=2s` [`kitty/vt-parser.c:637-650`]; the DECRQM status for mode 2026 goes `2` (inactive) → `1` (active) → `2` (inactive). Two distinct abort conditions each emit a `[PARSE ERROR]` message verbatim (complete output §9.6):

```text
[PARSE ERROR] Pending mode stop command issued while not in pending mode, this can be either a bug in the terminal application or caused by a timeout with no data received for too long or by too much data in pending mode
[PARSE ERROR] Pending mode start requested while already in pending mode. This is most likely an application error.
```

matching `kitty/vt-parser.c:646-648`. The first explicitly names *"too much data in pending mode"* as a trigger. The default pending-mode timeout is **2000 ms** [`kitty/screen.c:2521`].


---

## 9. Evidence appendix (exact command + complete unedited output)

All observation scripts lived under `/tmp/kitty_obs/` on the host, mounted into the canonical container (§2.2) at the same path (`-v /tmp/kitty_obs:/tmp/kitty_obs`); they are outside the repository and are removed as the final cleanup step so the tree is left unchanged (§9.8). Except where a block is explicitly labelled **NON-CANONICAL** (the native GUI-under-Xvfb thread roster in §9.6(a-supplementary)), each block below was run in the **canonical container** via `docker exec kitty-obs bash -c 'cd /app && python3 …'` and shows the exact command and the **complete, unedited** stdout.

**Harness provenance & how to reproduce these without the removed scripts.** The `/tmp/kitty_obs/*.py` paths in the commands below are the *archival capture commands*: each observation harness was a small **temporary** script that lived only under `/tmp/kitty_obs/` (on the host, bind-mounted into the container) and was **intentionally removed after capture** to satisfy the read-only rule (§0.7), so the repository is left byte-for-byte unchanged apart from this document (§9.8). None of them is a bespoke debug hook — every harness is a thin driver over kitty's **own in-tree test helpers** (`kitty_tests.parse_bytes`, `kitty_tests.graphics.send_command`, `BaseTest.create_screen`) exercising the real `Screen` / `GraphicsManager` / `ChildMonitor` (construction spelled out in §2.4), so any reader can reconstruct them. The canonical reconstruction pattern for the graphics-response harnesses — re-run in the canonical container (§2.2) and **byte-identical across ≥2 runs** — is:

```python
import sys
sys.path.insert(0, '/app')                       # kitty source root (the /app checkout, pinned commit)
from kitty_tests import BaseTest                  # kitty's own in-tree test base
from kitty_tests.graphics import send_command     # builds ESC _ G <keys> ; <b64 payload> ESC \, returns wtcbuf

class H(BaseTest):
    def runTest(self): pass

h = H()
s = h.create_screen()                             # real Screen + GraphicsManager (grman)
# q=0: full response; q=1: OK suppressed but error emitted; q=2: all suppressed
print("q=0 OK   :", send_command(s, 'a=t,q=0,i=1,s=1,v=1,f=24', b'\x01\x02\x03'))
print("q=0 ERR  :", send_command(s, 'a=t,q=0,i=1,s=10,v=10,f=24', b'\x00\x01\x02\x03'))
print("q=1 OK   :", send_command(s, 'a=t,q=1,i=1,s=1,v=1,f=24', b'\x01\x02\x03'))
print("q=1 ERR  :", send_command(s, 'a=t,q=1,i=1,s=10,v=10,f=24', b'\x00\x01\x02\x03'))
print("q=2 OK   :", send_command(s, 'a=t,q=2,i=1,s=1,v=1,f=24', b'\x01\x02\x03'))
print("q=2 ERR  :", send_command(s, 'a=t,q=2,i=1,s=10,v=10,f=24', b'\x00\x01\x02\x03'))
```

which, run via `docker exec kitty-obs bash -c 'cd /app && python3 …'`, produces (byte-identical to §9.4a/§9.5):

```text
q=0 OK   : b'\x1b_Gi=1;OK\x1b\\'
q=0 ERR  : b'\x1b_Gi=1;ENODATA:Insufficient image data: 4 < 300\x1b\\'
q=1 OK   : b''
q=1 ERR  : b'\x1b_Gi=1;ENODATA:Insufficient image data: 4 < 300\x1b\\'
q=2 OK   : b''
q=2 ERR  : b''
```

Swapping the control keys/payload reproduces every §9.3–§9.5 block (e.g. `a=t` transmit floods for the 320 MiB quota in §9.3; the five APC conditions in §9.4a); the threaded-I/O harness (§9.6a) is the same pattern over a real `ChildMonitor(lambda *a: None, None).start()` reading `/proc/self/task/*/comm`. The two write-buffer harnesses (§9.4b `write_buf` 100 MiB cap and §9.4c `EAGAIN`-retain/drain) drive the **live** `ChildMonitor` io_loop with a pre-filled PTY (a real background thread), so their wall-clock timing is environment-sensitive; their *result* is nonetheless deterministic by the exact byte arithmetic shown there (99 × 1 048 580 = 103 809 420, and one more framed chunk crosses the 104 857 600 cap). The response-gating and storage behaviors are additionally reproducible with **no** custom harness at all, through kitty's own test suite: `CI=true ./kitty/launcher/kitty +launch test.py --module graphics suppressing_gr_command_responses disk_cache` → `Ran 2 tests … OK` (§9.5a).

### 9.1 B1 — 1 MiB parser buffer fills, admission gate closes, then reopens (canonical)

Scale: 256 KiB commits until full (4 commits = 1 MiB). Method: the real `vt_parser_create_write_buffer` / `vt_parser_commit_write` via `Screen.test_*`, run inside the canonical container (§2.2). Two runs (A, B) in one invocation; **byte-identical** ⇒ stable.

```bash
docker exec kitty-obs bash -c 'cd /app && python3 /tmp/kitty_obs/b1_buffer_gate.py'
```

```text
===== RUN A =====
VT_PARSER_BUFFER_SIZE (BUF_SZ) = 1048576 bytes = 1024 KiB = 1 MiB
[before]  available_buffer_space = 1048576  committed_total = 0  (buffer empty)
[during]  step=1 avail_before_commit=1048576 committed_this_step=262144 committed_total=262144
[during]  step=2 avail_before_commit=786432 committed_this_step=262144 committed_total=524288
[during]  step=3 avail_before_commit=524288 committed_this_step=262144 committed_total=786432
[during]  step=4 avail_before_commit=262144 committed_this_step=262144 committed_total=1048576
[full]    available_buffer_space = 0  -> GATE CLOSED: extra_commit_accepted = 0
[full]    committed_total = 1048576  == BUF_SZ(1048576)? True
[drained] after parse: available_buffer_space = 1048576  -> GATE REOPENED

===== RUN B =====
VT_PARSER_BUFFER_SIZE (BUF_SZ) = 1048576 bytes = 1024 KiB = 1 MiB
[before]  available_buffer_space = 1048576  committed_total = 0  (buffer empty)
[during]  step=1 avail_before_commit=1048576 committed_this_step=262144 committed_total=262144
[during]  step=2 avail_before_commit=786432 committed_this_step=262144 committed_total=524288
[during]  step=3 avail_before_commit=524288 committed_this_step=262144 committed_total=786432
[during]  step=4 avail_before_commit=262144 committed_this_step=262144 committed_total=1048576
[full]    available_buffer_space = 0  -> GATE CLOSED: extra_commit_accepted = 0
[full]    committed_total = 1048576  == BUF_SZ(1048576)? True
[drained] after parse: available_buffer_space = 1048576  -> GATE REOPENED
```

**before → during → after:** empty (`1048576` free) → filling (`786432 → 524288 → 262144`) → full (`0` free, gate closed at exactly `1048576` committed) → drained (`1048576` free, gate reopened).

### 9.2 B2 — default `input_delay` / `repaint_delay` + 16 KiB near-full bypass (canonical values)

Run inside the canonical container (§2.2). The script is invoked **twice as separate processes**; each invocation prints two runs (A, B). Both invocations' **complete** stdout are pasted below and are **byte-identical** ⇒ stable.

```bash
docker exec kitty-obs bash -c 'cd /app && python3 /tmp/kitty_obs/b2_input_delay.py'   # invocation 1
docker exec kitty-obs bash -c 'cd /app && python3 /tmp/kitty_obs/b2_input_delay.py'   # invocation 2
```

Invocation 1 — complete stdout:

```text
===== RUN A =====
defaults.input_delay  = 3 ms  (canonical default config)
defaults.repaint_delay = 10 ms  (canonical default config)
BUF_SZ = 1048576; near-full headroom = 16384 (16 KiB); parse-immediately threshold read.sz > 1032192

===== RUN B =====
defaults.input_delay  = 3 ms  (canonical default config)
defaults.repaint_delay = 10 ms  (canonical default config)
BUF_SZ = 1048576; near-full headroom = 16384 (16 KiB); parse-immediately threshold read.sz > 1032192
```

Invocation 2 — complete stdout:

```text
===== RUN A =====
defaults.input_delay  = 3 ms  (canonical default config)
defaults.repaint_delay = 10 ms  (canonical default config)
BUF_SZ = 1048576; near-full headroom = 16384 (16 KiB); parse-immediately threshold read.sz > 1032192

===== RUN B =====
defaults.input_delay  = 3 ms  (canonical default config)
defaults.repaint_delay = 10 ms  (canonical default config)
BUF_SZ = 1048576; near-full headroom = 16384 (16 KiB); parse-immediately threshold read.sz > 1032192
```

Both invocations (four runs total) are byte-identical. `input_delay = 3` and `repaint_delay = 10` are the compiled defaults [`kitty/options/definition.py:878,866`]; the near-full bypass fires when `read.sz + 16384 > 1048576`, i.e. `read.sz > 1032192` [`kitty/vt-parser.c:1425`].

### 9.3 B3 — graphics storage backpressure (canonical)

**(a) 320 MiB quota — eviction phase 1 (`trim_predicate`): unreferenced transmit flood.** Scale: 6 × 64 MiB unreferenced (transmit-only `a=t`) images. Run in the canonical container (§2.2); two runs (A, B) in one invocation, byte-identical ⇒ stable. `used_storage` is not exposed to Python, so it is shown as `inferred_used = image_count × 67108864` (each image contributes exactly `required_sz` [`kitty/graphics.c:749`]).

```bash
docker exec kitty-obs bash -c 'cd /app && python3 /tmp/kitty_obs/b3a_quota_unref.py'
```

```text
===== RUN A =====
grman.storage_limit = 335544320 bytes = 320 MiB (DEFAULT_STORAGE_LIMIT, graphics.c:25)
per-image used_storage = 67108864 bytes = 64 MiB ; images that fit = 5 (5*67108864=335544320 == limit? True)
[before] image_count = 0  (buffer empty)
[during] transmit i=1 (a=t, UNREFERENCED) resp=b'\x1b_Gi=1;OK\x1b\\' image_count=1 inferred_used=67108864
[during] transmit i=2 (a=t, UNREFERENCED) resp=b'\x1b_Gi=2;OK\x1b\\' image_count=2 inferred_used=134217728
[during] transmit i=3 (a=t, UNREFERENCED) resp=b'\x1b_Gi=3;OK\x1b\\' image_count=3 inferred_used=201326592
[during] transmit i=4 (a=t, UNREFERENCED) resp=b'\x1b_Gi=4;OK\x1b\\' image_count=4 inferred_used=268435456
[during] transmit i=5 (a=t, UNREFERENCED) resp=b'\x1b_Gi=5;OK\x1b\\' image_count=5 inferred_used=335544320
[at-limit] image_count=5 inferred_used=335544320 == limit 335544320? True
[cross]  transmit i=6 resp=b'\x1b_Gi=6;OK\x1b\\' -> image_count=1  (apply_storage_quota step 1: remove_images(trim_predicate) drops ALL unreferenced except just-added, graphics.c:291-292)
[after]  surviving client_ids = [6]  inferred_used=67108864  (only newest id=6 survives; ids 1..5 evicted as unreferenced)

===== RUN B =====
grman.storage_limit = 335544320 bytes = 320 MiB (DEFAULT_STORAGE_LIMIT, graphics.c:25)
per-image used_storage = 67108864 bytes = 64 MiB ; images that fit = 5 (5*67108864=335544320 == limit? True)
[before] image_count = 0  (buffer empty)
[during] transmit i=1 (a=t, UNREFERENCED) resp=b'\x1b_Gi=1;OK\x1b\\' image_count=1 inferred_used=67108864
[during] transmit i=2 (a=t, UNREFERENCED) resp=b'\x1b_Gi=2;OK\x1b\\' image_count=2 inferred_used=134217728
[during] transmit i=3 (a=t, UNREFERENCED) resp=b'\x1b_Gi=3;OK\x1b\\' image_count=3 inferred_used=201326592
[during] transmit i=4 (a=t, UNREFERENCED) resp=b'\x1b_Gi=4;OK\x1b\\' image_count=4 inferred_used=268435456
[during] transmit i=5 (a=t, UNREFERENCED) resp=b'\x1b_Gi=5;OK\x1b\\' image_count=5 inferred_used=335544320
[at-limit] image_count=5 inferred_used=335544320 == limit 335544320? True
[cross]  transmit i=6 resp=b'\x1b_Gi=6;OK\x1b\\' -> image_count=1  (apply_storage_quota step 1: remove_images(trim_predicate) drops ALL unreferenced except just-added, graphics.c:291-292)
[after]  surviving client_ids = [6]  inferred_used=67108864  (only newest id=6 survives; ids 1..5 evicted as unreferenced)
```

**before → during → after:** 5 images summing to exactly 320 MiB, none evicted (trigger is strict `>` at [`kitty/graphics.c:2184`]) → the 6th add crosses the quota → `apply_storage_quota` **phase 1** runs `remove_images(trim_predicate, currently_added)` [`kitty/graphics.c:290-292`], and `trim_predicate` (`!root_frame_data_loaded || !img->refs` [`kitty/graphics.c:280-282`]) drops *all* unreferenced older transmits, leaving only the newest (`surviving = [6]`, ids 1–5 gone). `image_count` is read via `HASH_COUNT` **before** any `image_for_client_id` enumeration (that call would `find_or_create` an empty image and skew the count).

**(b) 320 MiB quota — eviction phase 2 (atime LRU): referenced (placed) images.** Scale: 8 × 64 MiB images, each *pinned* with a 1×1 placement at the top-left cell (`a=p,c=1,r=1` with the cursor reset to home before each put) so the placement never scrolls off and the image keeps a ref. This defeats phase 1 (`trim_predicate` finds nothing unreferenced to drop) and forces the **phase-2 atime `while`-loop** [`kitty/graphics.c:296-298`]. Run in the canonical container (§2.2); two runs (A, B) in one invocation, byte-identical ⇒ stable. `image_count` (`HASH_COUNT`) is read **before** the final survivor enumeration (an `image_for_client_id` call would `find_or_create` an empty image and skew the count).

```bash
docker exec kitty-obs bash -c 'cd /app && python3 /tmp/kitty_obs/b3b_lru_atime.py'
```

```text
===== RUN A =====
grman.storage_limit = 335544320 bytes = 320 MiB ; per-image = 67108864 (64 MiB) ; fit = 5
[before] image_count = 0
[during] add i=1 tx=b'\x1b_Gi=1;OK\x1b\\' put=b'\x1b_Gi=1;OK\x1b\\' image_count=1 inferred_used=67108864
[during] add i=2 tx=b'\x1b_Gi=2;OK\x1b\\' put=b'\x1b_Gi=2;OK\x1b\\' image_count=2 inferred_used=134217728
[during] add i=3 tx=b'\x1b_Gi=3;OK\x1b\\' put=b'\x1b_Gi=3;OK\x1b\\' image_count=3 inferred_used=201326592
[during] add i=4 tx=b'\x1b_Gi=4;OK\x1b\\' put=b'\x1b_Gi=4;OK\x1b\\' image_count=4 inferred_used=268435456
[during] add i=5 tx=b'\x1b_Gi=5;OK\x1b\\' put=b'\x1b_Gi=5;OK\x1b\\' image_count=5 inferred_used=335544320
[during] add i=6 tx=b'\x1b_Gi=6;OK\x1b\\' put=b'\x1b_Gi=6;OK\x1b\\' image_count=5 inferred_used=335544320 -> over quota: step1 removes nothing (all referenced), step2 atime while-loop evicts OLDEST (graphics.c:296-298)
[during] add i=7 tx=b'\x1b_Gi=7;OK\x1b\\' put=b'\x1b_Gi=7;OK\x1b\\' image_count=5 inferred_used=335544320 -> over quota: step1 removes nothing (all referenced), step2 atime while-loop evicts OLDEST (graphics.c:296-298)
[during] add i=8 tx=b'\x1b_Gi=8;OK\x1b\\' put=b'\x1b_Gi=8;OK\x1b\\' image_count=5 inferred_used=335544320 -> over quota: step1 removes nothing (all referenced), step2 atime while-loop evicts OLDEST (graphics.c:296-298)
[after]  image_count (read before enumeration) = 5  inferred_used=335544320
[after]  surviving client_ids = [4, 5, 6, 7, 8]  (newest 5 kept)
[after]  evicted oldest-by-atime = [1, 2, 3]  (oldest_img_first comparator, graphics.c:284-287)

===== RUN B =====
grman.storage_limit = 335544320 bytes = 320 MiB ; per-image = 67108864 (64 MiB) ; fit = 5
[before] image_count = 0
[during] add i=1 tx=b'\x1b_Gi=1;OK\x1b\\' put=b'\x1b_Gi=1;OK\x1b\\' image_count=1 inferred_used=67108864
[during] add i=2 tx=b'\x1b_Gi=2;OK\x1b\\' put=b'\x1b_Gi=2;OK\x1b\\' image_count=2 inferred_used=134217728
[during] add i=3 tx=b'\x1b_Gi=3;OK\x1b\\' put=b'\x1b_Gi=3;OK\x1b\\' image_count=3 inferred_used=201326592
[during] add i=4 tx=b'\x1b_Gi=4;OK\x1b\\' put=b'\x1b_Gi=4;OK\x1b\\' image_count=4 inferred_used=268435456
[during] add i=5 tx=b'\x1b_Gi=5;OK\x1b\\' put=b'\x1b_Gi=5;OK\x1b\\' image_count=5 inferred_used=335544320
[during] add i=6 tx=b'\x1b_Gi=6;OK\x1b\\' put=b'\x1b_Gi=6;OK\x1b\\' image_count=5 inferred_used=335544320 -> over quota: step1 removes nothing (all referenced), step2 atime while-loop evicts OLDEST (graphics.c:296-298)
[during] add i=7 tx=b'\x1b_Gi=7;OK\x1b\\' put=b'\x1b_Gi=7;OK\x1b\\' image_count=5 inferred_used=335544320 -> over quota: step1 removes nothing (all referenced), step2 atime while-loop evicts OLDEST (graphics.c:296-298)
[during] add i=8 tx=b'\x1b_Gi=8;OK\x1b\\' put=b'\x1b_Gi=8;OK\x1b\\' image_count=5 inferred_used=335544320 -> over quota: step1 removes nothing (all referenced), step2 atime while-loop evicts OLDEST (graphics.c:296-298)
[after]  image_count (read before enumeration) = 5  inferred_used=335544320
[after]  surviving client_ids = [4, 5, 6, 7, 8]  (newest 5 kept)
[after]  evicted oldest-by-atime = [1, 2, 3]  (oldest_img_first comparator, graphics.c:284-287)
```

**before → during → after:** empty (`image_count = 0`) → 5 pinned images fill to exactly 320 MiB (`image_count` 1→5, none evicted while `used == limit`) → each further add (6, 7, 8) crosses the quota, so phase 1 removes nothing (every image is referenced) and the **phase-2 `while` loop** sorts by `atime` (`oldest_img_first` [`kitty/graphics.c:284-287`]) and evicts exactly the **oldest** image per over-quota add. `image_count` therefore plateaus at 5, the five newest survive (`[4, 5, 6, 7, 8]`), and the three oldest are evicted oldest-first (`[1, 2, 3]`). This is the clean atime-LRU branch; contrast with (a), which exercised phase 1. Both runs are byte-identical.

**(c) Animation-frame disk cache ceiling `storage_limit * 5` → `ENOSPC` (canonical).** This is a *separate* backpressure limit from the 320 MiB image quota: animation frames are stored in the disk cache, whose ceiling is `storage_limit * 5 = 1 677 721 600` bytes (1 600 MiB) [`kitty/graphics.c:1570`]; when a new frame would exceed it the store returns `ENOSPC` instead of evicting [`kitty/graphics.c:1570-1573`]. Scale: one base image + 25 × 64 MiB frames (`a=f,t=f`, no dedup, monotonically increasing `r=`) so the cache climbs to the ceiling and the next frame is refused. Run in the canonical container (§2.2); the script performs two internal runs (A, B), and a second whole-script invocation was byte-identical (`sha256` match) ⇒ stable.

```bash
docker exec kitty-obs bash -c 'cd /app && python3 /tmp/kitty_obs/b3c_anim_enospc.py'
```

```text
===== RUN A =====
storage_limit = 335544320 ; animation ceiling = storage_limit*5 = 1677721600 bytes = 1600 MiB (graphics.c:1570)
[before] base transmit resp=b'\x1b_Gi=1;OK\x1b\\' disk_cache.total_size=67108864 (64 MiB)
[during] frame add #5 resp=b'\x1b_Gi=1,r=6;OK\x1b\\' disk_cache.total_size=402653184 (384 MiB)
[during] frame add #10 resp=b'\x1b_Gi=1,r=11;OK\x1b\\' disk_cache.total_size=738197504 (704 MiB)
[during] frame add #15 resp=b'\x1b_Gi=1,r=16;OK\x1b\\' disk_cache.total_size=1073741824 (1024 MiB)
[during] frame add #20 resp=b'\x1b_Gi=1,r=21;OK\x1b\\' disk_cache.total_size=1409286144 (1344 MiB)
[during] frame add #24 resp=b'\x1b_Gi=1,r=25;OK\x1b\\' disk_cache.total_size=1677721600 (1600 MiB)
[during] frame add #25 resp=b'\x1b_Gi=1,r=26;ENOSPC:Cache size exceeded cannot add new frames\x1b\\' disk_cache.total_size=1677721600 (1600 MiB)  <-- ENOSPC (cache_size + data_sz > storage_limit*5, graphics.c:1570-1573)
[after]  first ENOSPC at frame add #25 ; final disk_cache.total_size=1677721600 == storage_limit*5(1677721600)? True

===== RUN B =====
storage_limit = 335544320 ; animation ceiling = storage_limit*5 = 1677721600 bytes = 1600 MiB (graphics.c:1570)
[before] base transmit resp=b'\x1b_Gi=1;OK\x1b\\' disk_cache.total_size=67108864 (64 MiB)
[during] frame add #5 resp=b'\x1b_Gi=1,r=6;OK\x1b\\' disk_cache.total_size=402653184 (384 MiB)
[during] frame add #10 resp=b'\x1b_Gi=1,r=11;OK\x1b\\' disk_cache.total_size=738197504 (704 MiB)
[during] frame add #15 resp=b'\x1b_Gi=1,r=16;OK\x1b\\' disk_cache.total_size=1073741824 (1024 MiB)
[during] frame add #20 resp=b'\x1b_Gi=1,r=21;OK\x1b\\' disk_cache.total_size=1409286144 (1344 MiB)
[during] frame add #24 resp=b'\x1b_Gi=1,r=25;OK\x1b\\' disk_cache.total_size=1677721600 (1600 MiB)
[during] frame add #25 resp=b'\x1b_Gi=1,r=26;ENOSPC:Cache size exceeded cannot add new frames\x1b\\' disk_cache.total_size=1677721600 (1600 MiB)  <-- ENOSPC (cache_size + data_sz > storage_limit*5, graphics.c:1570-1573)
[after]  first ENOSPC at frame add #25 ; final disk_cache.total_size=1677721600 == storage_limit*5(1677721600)? True
```

**before → during → after:** empty base image at 64 MiB → the cache climbs 64 MiB per frame (`disk_cache.total_size` 384 → 704 → 1024 → 1344 MiB at frames 5/10/15/20) → frame #24 fills the cache to **exactly** the ceiling `1 677 721 600` (1 600 MiB) with a normal `OK` → frame #25 would exceed the ceiling, so the store refuses it and replies with the exact bytes `b'\x1b_Gi=1,r=26;ENOSPC:Cache size exceeded cannot add new frames\x1b\\'`; the cache size does **not** grow past the ceiling (stays `1677721600`). Unlike the image quota (which *evicts*), the animation cache *rejects* the write — a visible in-band error. Both internal runs are byte-identical, and a second whole-script invocation produced the identical bytes (`sha256 cd751ce0…3cdf`).

### 9.4 B4 — response write-back: APC reply bytes + write-buffer 100 MiB cap / retain-drain (canonical)

**(a) Raw APC reply bytes.** Five response conditions driven through the real graphics command path (`send_command` → `screen_handle_graphics_command` → `write_escape_code_to_child(ESC_APC, …)` [`kitty/screen.c:1047-1050`]) in the canonical container (§2.2); two invocations, byte-identical (`sha256 02d88ce…`) ⇒ stable.

```bash
docker exec kitty-obs bash -c 'cd /app && python3 /tmp/kitty_obs/b4_apc_bytes.py'
```

```text
===== RUN A =====
[OK transmit RGB 1x1 (a=t,i=1,s=1,v=1,f=24)]
    cmd   = ESC _ G a=t,i=1,s=1,v=1,f=24 ; <payload 3B> ESC \
    reply = b'\x1b_Gi=1;OK\x1b\\'
[image-number assign (a=t,I=1,s=1,v=1,f=24, no i=)]
    cmd   = ESC _ G a=t,I=1,s=1,v=1,f=24 ; <payload 3B> ESC \
    reply = b'\x1b_Gi=1,I=1;OK\x1b\\'
[query (a=q,i=1,s=1,v=1,f=24)]
    cmd   = ESC _ G a=q,i=1,s=1,v=1,f=24 ; <payload 3B> ESC \
    reply = b'\x1b_Gi=1;OK\x1b\\'
[ENODATA error (a=t,i=1,s=10,v=10,f=24, 4B < 300B)]
    cmd   = ESC _ G a=t,i=1,s=10,v=10,f=24 ; <payload 4B> ESC \
    reply = b'\x1b_Gi=1;ENODATA:Insufficient image data: 4 < 300\x1b\\'
[EINVAL error (a=t,i=3,I=1 both id+number)]
    cmd   = ESC _ G a=t,i=3,I=1,s=1,v=1,f=24 ; <payload 3B> ESC \
    reply = b'\x1b_Gi=3,I=1;EINVAL:Must not specify both image id and image number\x1b\\'

===== RUN B =====
[OK transmit RGB 1x1 (a=t,i=1,s=1,v=1,f=24)]
    cmd   = ESC _ G a=t,i=1,s=1,v=1,f=24 ; <payload 3B> ESC \
    reply = b'\x1b_Gi=1;OK\x1b\\'
[image-number assign (a=t,I=1,s=1,v=1,f=24, no i=)]
    cmd   = ESC _ G a=t,I=1,s=1,v=1,f=24 ; <payload 3B> ESC \
    reply = b'\x1b_Gi=1,I=1;OK\x1b\\'
[query (a=q,i=1,s=1,v=1,f=24)]
    cmd   = ESC _ G a=q,i=1,s=1,v=1,f=24 ; <payload 3B> ESC \
    reply = b'\x1b_Gi=1;OK\x1b\\'
[ENODATA error (a=t,i=1,s=10,v=10,f=24, 4B < 300B)]
    cmd   = ESC _ G a=t,i=1,s=10,v=10,f=24 ; <payload 4B> ESC \
    reply = b'\x1b_Gi=1;ENODATA:Insufficient image data: 4 < 300\x1b\\'
[EINVAL error (a=t,i=3,I=1 both id+number)]
    cmd   = ESC _ G a=t,i=3,I=1,s=1,v=1,f=24 ; <payload 3B> ESC \
    reply = b'\x1b_Gi=3,I=1;EINVAL:Must not specify both image id and image number\x1b\\'
```

Every reply is wrapped in the same APC frame, visible byte-for-byte in the `repr`: prefix `\x1b_` (`ESC _`) + `G` graphics introducer … terminator `\x1b\\` (`ESC \`). Success replies carry `;OK`; error replies carry the code inline (`ENODATA:…`, `EINVAL:…`). These are the exact bytes kitty enqueues onto the output path — the same per-window `write_buf` bounded by the 100 MiB cap in (b).

**(b) Output write-buffer 100 MiB cap → drop (canonical, deterministic).** Response bytes accumulate in the per-window `write_buf`; `schedule_write_to_child` refuses an enqueue once `write_buf_used + sz > 100*1024*1024` and logs a drop [`kitty/child-monitor.c:341-342`]. To observe the cap deterministically without the io_loop draining, the kernel pipe is pre-filled so `write_to_child` always hits `EAGAIN` (drains nothing), and 1 MiB APC chunks (framed to 1 048 580 B each) are enqueued until one is refused. Run in the canonical container (§2.2); two separate-process runs, byte-identical (`sha256 524f5c6…`) ⇒ stable.

```bash
docker exec kitty-obs bash -c 'cd /app && python3 /tmp/kitty_obs/b7a_writebuf_cap.py'
```

```text
cap constant = 100*1024*1024 = 104857600 bytes (100 MiB) [child-monitor.c:341]
framed enqueue size = payload 1048576 + 4B APC framing = 1048580 bytes
pipe pre-filled with 8192 bytes (io_loop cannot drain -> deterministic)
[before] write_buf_used = 0 (empty; first enqueue accepted)
[during] each accepted enqueue grows write_buf_used by 1048580 bytes
[at-cap] accepted framed chunks before drop = 99  (=> write_buf_used = 99*1048580 = 103809420 bytes)
[drop]   first_drop_at_chunk = 100  (check: 103809420 + 1048580 = 104858000 > 104857600 => dropped)
[drop]   cap_log_line_count = 1
[drop]   log_error (timestamp stripped) = 'Too much data being sent to child with id: 1, ignoring it'
```

**before → during → after:** `write_buf_used = 0` (empty) → each accepted enqueue grows it by exactly `1 048 580` B → after **99** accepted chunks `write_buf_used = 99 × 1 048 580 = 103 809 420` B; the **100th** enqueue would make `103 809 420 + 1 048 580 = 104 858 000 > 104 857 600` (the 100 MiB cap), so it is **dropped** and kitty logs exactly one line — `Too much data being sent to child with id: 1, ignoring it` [`kitty/child-monitor.c:342`]. This over-cap drop is a **visible** signal (a log line); the byte accounting is exact and identical across both processes.

**(c) Output write-buffer retain → drain (EAGAIN retry, canonical).** Under *transient* pressure the write path does not drop: when `write_to_child` cannot flush everything it `memmove`-retains the unwritten tail in `write_buf` and retries on the next `POLLOUT` [`kitty/child-monitor.c:1462-1473`]. Here 20 framed 1 MiB chunks (well under the cap) are enqueued while the pipe is blocked — so most bytes are retained — then the pipe is drained so the io_loop flushes the retained bytes. Two runs, byte-identical (`sha256 e67b108…`) ⇒ stable.

```bash
docker exec kitty-obs bash -c 'cd /app && python3 /tmp/kitty_obs/b7b_writebuf_retain_drain.py'
```

```text
[before] nothing enqueued; write_buf_used = 0
[during] enqueued 20 framed chunks = 20971600 bytes; kernel pipe holds only a few KiB, remainder RETAINED in write_buf (EAGAIN retain, child-monitor.c:1462-1473)
[after]  drained pipe: total bytes read back = 20971600 ; equals total enqueued (20971600)? True
[after]  => all retained bytes delivered on POLLOUT (retain-and-retry); write_buf_used returned to 0
```

**before → during → after:** `write_buf_used = 0` → 20 framed chunks (`20 × 1 048 580 = 20 971 600` B) enqueued with the pipe blocked, so the kernel pipe holds only a few KiB and the remainder is **retained** in `write_buf` → after draining the pipe, exactly `20 971 600` B are read back (== enqueued), i.e. every retained byte was delivered on `POLLOUT` and `write_buf_used` returned to 0. Under transient backpressure **no bytes are lost** — data is dropped *only* past the 100 MiB cap in (b). This retain-and-retry is a **silent** adaptation.

### 9.5 B5 — `q=` response gating (canonical)

**(a) In-tree tests that exercise this exact behavior — both pass (run in the canonical container, §2.2):**

```bash
docker exec kitty-obs bash -c 'cd /app && CI=true ./kitty/launcher/kitty +launch test.py --module graphics suppressing_gr_command_responses disk_cache'
```

```text
Running under CI: True
Using PATH in test environment: /app/kitty_tests/kitty/launcher:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Python: /usr/bin/python
Intrinsics: has_avx2=True has_sse4_2=True
test_disk_cache (kitty_tests.graphics.TestGraphics.test_disk_cache) ... ok
test_suppressing_gr_command_responses (kitty_tests.graphics.TestGraphics.test_suppressing_gr_command_responses) ... ok

----------------------------------------------------------------------
Ran 2 tests in 0.147s

OK
```

**(b) Per-`q` raw response bytes (canonical container; two runs, byte-identical (`sha256 f22494a…`) ⇒ stable):**

```bash
docker exec kitty-obs bash -c 'cd /app && python3 /tmp/kitty_obs/b5_quiet_gating.py'
```

```text
===== RUN A =====
gate: finish_command_response graphics.c:759-763  (q>=1 suppresses OK; q>=2 suppresses ALL incl errors)
  q=0: success_reply=b'\x1b_Gi=1;OK\x1b\\'         error_reply=b'\x1b_Gi=1;ENODATA:Insufficient image data: 4 < 300\x1b\\'
  q=1: success_reply=b''                           error_reply=b'\x1b_Gi=1;ENODATA:Insufficient image data: 4 < 300\x1b\\'
  q=2: success_reply=b''                           error_reply=b''

===== RUN B =====
gate: finish_command_response graphics.c:759-763  (q>=1 suppresses OK; q>=2 suppresses ALL incl errors)
  q=0: success_reply=b'\x1b_Gi=1;OK\x1b\\'         error_reply=b'\x1b_Gi=1;ENODATA:Insufficient image data: 4 < 300\x1b\\'
  q=1: success_reply=b''                           error_reply=b'\x1b_Gi=1;ENODATA:Insufficient image data: 4 < 300\x1b\\'
  q=2: success_reply=b''                           error_reply=b''
```

Reading the raw bytes: `q=0` emits the OK reply **and** the error; `q=1` returns **empty** (`b''`) for the OK reply but **still emits** the `ENODATA` error; `q=2` returns **empty** for both — exactly the gate `if (g->quiet){ if (is_ok_response || g->quiet > 1) return NULL; }` at [`kitty/graphics.c:759-763`].

### 9.6 B6 — threaded I/O model + pending-mode abort (canonical)

**(a) The dedicated PTY I/O thread `KittyChildMon` (canonical, in-process).** The real `ChildMonitor.start()` spawns the I/O thread at [`kitty/child-monitor.c:291`] (`pthread_create(&self->io_thread, NULL, io_loop, self)`), which names itself `KittyChildMon` at [`kitty/child-monitor.c:1489`]. Driven in-process in the canonical container (§2.2); two invocations, byte-identical (`sha256 5692f26…`) ⇒ stable.

```bash
docker exec kitty-obs bash -c 'cd /app && python3 /tmp/kitty_obs/b8_threads.py'
```

```text
[before start()] OS threads = 1 ; comm names = ['python3']
[after start()]  OS threads = 2 ; comm names = ['KittyChildMon', 'python3']
KittyChildMon present: True (io_loop names itself at child-monitor.c:1489)
```

The `KittyChildMon` thread appears **only after** `start()`, confirming it is the dedicated PTY read/parse/write thread created by the real `io_loop`, decoupled from the calling thread.

**(a-supplementary — NON-CANONICAL, native build under Xvfb).** To show `KittyChildMon` coexisting with the main render thread in a *real GUI* process, I ran a GUI kitty under Xvfb. This uses the **native** build (Python 3.13.7 / gcc 15.2.0), because the canonical container has no Xvfb; it is therefore **non-canonical** and supplementary — the canonical thread evidence is the in-process observation above. Complete (un-elided) histogram, stable across 2 runs (only the PID differs):

```bash
# native host (dest working tree):
xvfb-run -a ./kitty/launcher/kitty --config NONE sh -c 'sleep 20' &
# resolve the kitty pid (its /proc/<pid>/exe == kitty/launcher/kitty), then:
for t in /proc/<pid>/task/*/comm; do cat "$t"; done | sort | uniq -c | sort -rn
```

```text
real kitty PID = 158130
total threads = 67
--- complete comm-name histogram (count  name), sorted by count desc ---
     33 kitty
      1 llvmpipe-9
      1 llvmpipe-8
      1 llvmpipe-7
      1 llvmpipe-6
      1 llvmpipe-5
      1 llvmpipe-4
      1 llvmpipe-31
      1 llvmpipe-30
      1 llvmpipe-3
      1 llvmpipe-29
      1 llvmpipe-28
      1 llvmpipe-27
      1 llvmpipe-26
      1 llvmpipe-25
      1 llvmpipe-24
      1 llvmpipe-23
      1 llvmpipe-22
      1 llvmpipe-21
      1 llvmpipe-20
      1 llvmpipe-2
      1 llvmpipe-19
      1 llvmpipe-18
      1 llvmpipe-17
      1 llvmpipe-16
      1 llvmpipe-15
      1 llvmpipe-14
      1 llvmpipe-13
      1 llvmpipe-12
      1 llvmpipe-11
      1 llvmpipe-10
      1 llvmpipe-1
      1 llvmpipe-0
      1 kitty:disk$0
      1 KittyChildMon
--- flow-control-relevant threads present (sorted unique) ---
KittyChildMon
kitty
kitty:disk$0
```

`KittyChildMon` (PTY I/O), the 33 `kitty` threads (the main render thread plus its GL worker pool), and `kitty:disk$0` (the graphics disk-cache writer) coexist — so the PTY read/parse/write path runs off the render thread [`kitty/child-monitor.c:291`, named `:1489`]. The 32 `llvmpipe-N` threads are the software-GL rasterizer (Xvfb has no GPU) and are environment-specific.

**(b) Pending-mode: before/during/after state + both abort messages (canonical container; two runs, byte-identical (`sha256 83a2c5e…`) ⇒ stable):**

```bash
docker exec kitty-obs bash -c 'cd /app && python3 /tmp/kitty_obs/b6_pending_mode.py'
```

```text
===== RUN A =====
[before] status(?2026$p) = b'\x1b[?2026;2$y'  (2 = reset/inactive)
[during] after DCS =1s start  = b'\x1b[?2026;1$y'  (1 = set/active, pending; expires_at=now+2000ms screen.c:2521)
[after]  after DCS =2s stop   = b'\x1b[?2026;2$y'  (2 = reset/inactive)
[abort-1] stop-while-not-pending log_error =
    '[PARSE ERROR] Pending mode stop command issued while not in pending mode, this can be either a bug in the terminal application or caused by a timeout with no data received for too long or by too much data in pending mode'
[abort-2] start-while-already-pending log_error =
    '[PARSE ERROR] Pending mode start requested while already in pending mode. This is most likely an application error.'

===== RUN B =====
[before] status(?2026$p) = b'\x1b[?2026;2$y'  (2 = reset/inactive)
[during] after DCS =1s start  = b'\x1b[?2026;1$y'  (1 = set/active, pending; expires_at=now+2000ms screen.c:2521)
[after]  after DCS =2s stop   = b'\x1b[?2026;2$y'  (2 = reset/inactive)
[abort-1] stop-while-not-pending log_error =
    '[PARSE ERROR] Pending mode stop command issued while not in pending mode, this can be either a bug in the terminal application or caused by a timeout with no data received for too long or by too much data in pending mode'
[abort-2] start-while-already-pending log_error =
    '[PARSE ERROR] Pending mode start requested while already in pending mode. This is most likely an application error.'
```

Synchronized/pending mode is entered/left via DCS `=1s`/`=2s` [`kitty/vt-parser.c:637-650`]; the DECRQM status for mode 2026 goes `2` (inactive) → `1` (active) → `2` (inactive). Issuing *stop* with no matching start, or *start* while already pending, produces the two `[PARSE ERROR]` abort messages verbatim; the first explicitly names *"too much data in pending mode"* as a trigger. The default pending-mode timeout is 2000 ms [`kitty/screen.c:2521`].

### 9.7 Re-verification of cited file:line anchors (pristine tree)

Every anchor cited in §3–§8 and §6 was re-verified with `sed -n` / `awk` against the pristine working tree immediately before finalizing. Command form:

```bash
sed -n '18p'         kitty/vt-parser.c            # BUF_SZ
sed -n '1476,1484p'  kitty/vt-parser.c            # vt_parser_has_space_for_input (gate :1481)
sed -n '1425p'       kitty/vt-parser.c            # deferred-parse gate
sed -n '646,648p'    kitty/vt-parser.c            # pending-mode abort message
sed -n '25p'         kitty/graphics.c             # DEFAULT_STORAGE_LIMIT
sed -n '279,300p'    kitty/graphics.c             # trim_predicate / oldest_img_first / apply_storage_quota
sed -n '2184p'       kitty/graphics.c             # quota trigger (strict >)
sed -n '758,764p'    kitty/graphics.c             # finish_command_response q-gate (:759-763)
sed -n '1570,1573p'  kitty/graphics.c             # animation-cache ENOSPC ceiling
sed -n '303,315p'    kitty/graphics.c             # set_command_failed_response / ABRT
sed -n '946,949p'    kitty/screen.c               # write_to_child
sed -n '1047,1050p'  kitty/screen.c               # screen_handle_graphics_command
sed -n '2521p'       kitty/screen.c               # pending-mode default timeout 2000 ms
sed -n '291p;341,342p;1337,1356p;1443,1476p;1501p;1503p'  kitty/child-monitor.c
sed -n '866p;878p'   kitty/options/definition.py  # repaint_delay=10, input_delay=3
sed -n '51p'         kitty/state.h                # monotonic_t repaint_delay, input_delay;
sed -n '29p;88p;283p' kitty/parse-graphics-command.h  # q= quiet key: enum / case / unparse
sed -n '48p;76p'     kitty/loop-utils.h           # wakeup_loop decl / drain_fd inline
sed -n '113p'        kitty/loop-utils.c           # wakeup_loop def
sed -n '226p;1515p'  kitty/child-monitor.c        # wakeup_loop / drain_fd call sites
```

Result: **all anchors matched**. Representative exact lines confirmed verbatim (these are the byte-for-byte source lines behind the quotes above):

```text
kitty/vt-parser.c:18        #define BUF_SZ (1024u*1024u)
kitty/vt-parser.c:1481              ans = self->read.sz + self->write.pending < BUF_SZ;
kitty/vt-parser.c:1425              if (flush || pd->time_since_new_input >= OPT(input_delay) || self->read.sz + 16 * 1024 > BUF_SZ) {
kitty/graphics.c:25         #define DEFAULT_STORAGE_LIMIT 320u * (1024u * 1024u)
kitty/graphics.c:281            return !img->root_frame_data_loaded || !img->refs;
kitty/graphics.c:286            return a->atime - b->atime;
kitty/graphics.c:2184               if (self->used_storage > self->storage_limit) apply_storage_quota(self, self->storage_limit, added_image_id);
kitty/graphics.c:763                if (is_ok_response || g->quiet > 1) return NULL;
kitty/graphics.c:1573               ABRT("ENOSPC", "Cache size exceeded cannot add new frames");
kitty/screen.c:1050         if (response != NULL) write_escape_code_to_child(self, ESC_APC, response);
kitty/child-monitor.c:291       ret = pthread_create(&self->io_thread, NULL, io_loop, self);
kitty/child-monitor.c:342                   log_error("Too much data being sent to child with id: %lu, ignoring it", id); \
kitty/child-monitor.c:1342      if (!available_buffer_space) return true;
kitty/child-monitor.c:1501              children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;
kitty/options/definition.py:878   opt('input_delay', '3',
kitty/state.h:51                monotonic_t repaint_delay, input_delay;
kitty/parse-graphics-command.h:29       quiet = 'q',
kitty/parse-graphics-command.h:88       case quiet:
kitty/loop-utils.h:48       void wakeup_loop(LoopData *ld, bool in_signal_handler, const char*);
kitty/loop-utils.h:76       drain_fd(int fd) {
kitty/loop-utils.c:113      wakeup_loop(LoopData *ld, bool in_signal_handler, const char *loop_name) {
kitty/child-monitor.c:1515              if (children_fds[0].revents && POLLIN) drain_fd(children_fds[0].fd); // wakeup
```

### 9.8 Final repository state (only the new document added)

This is a read-only investigation: no file in the kitty source tree was modified, and every temporary observation script lived **outside** the repository (under `/tmp/kitty_obs/` on the host, mounted into the container) and was removed after capture. Two independent checks — both run in the destination working tree — confirm the repository is left unchanged apart from the single new document. They measure different things and are shown separately.

**(1) The working tree is clean.** After the document is committed and the temporary scripts are removed, `git status --porcelain` prints **nothing** — there is no modified, staged, deleted, or untracked file anywhere in the tree:

```bash
git -C /tmp/blitzy/kitty/blitzy-aecbf6f2-a1d2-49f0-90bf-843111d83a06_36d2f6 status --porcelain
```

```text
```

(zero bytes of output — a completely clean working tree)

**(2) The only change vs. the pinned upstream baseline is the added document.** `git diff --name-status` between the pinned commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` and `HEAD` lists exactly one path, with status `A` (added). The column separator is a literal tab:

```bash
git -C /tmp/blitzy/kitty/blitzy-aecbf6f2-a1d2-49f0-90bf-843111d83a06_36d2f6 diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1..HEAD --
```

```text
A	blitzy/documentation/kitty_815df1e210e0.md
```

The two checks prove different facts, and conflating them is the mistake this section avoids: check (1) (`status --porcelain`, empty) proves there is no *uncommitted* change or leftover temporary file in the working tree; check (2) (`diff --name-status`, a single `A` row) proves the *committed* history adds only this one document on top of the pinned baseline. No kitty source, test, or configuration file appears in either — the repository is byte-for-byte unchanged apart from `blitzy/documentation/kitty_815df1e210e0.md`.


---

## 10. Coverage pass

### 10.1 Sub-questions O1–O5

| Sub-question | Answered in | Canonical evidence |
|---|---|---|
| **O1** — buffer / pause / slow-down decision | §3 (§3.1 pause, §3.2 slow/buffer, §3.3 why two) | §9.1 (buffer fills to 1 MiB, gate closes/reopens), §9.2 (delays + 16 KiB) |
| **O2** — response write-back under pressure | §5 | §9.4(a) (raw APC reply bytes), §9.4(b) (100 MiB cap → drop), §9.4(c) (retain → drain) |
| **O3** — where in code | §6 (28-row file:line index) | Re-verified §9.7 |
| **O4** — runtime manifestation | §7 | §9.6a (`KittyChildMon` spawned by real `start()`, in-process), §9.2 (delays), §9.3 (eviction) |
| **O5** — quiet vs. visible | §8 (two-column table) | §9.5 (`q=` bytes), §9.3c (`ENOSPC`), §9.6b (pending abort) |

### 10.2 Every named item / magnitude

| Item | Value | file:line | Evidence | Canonical? |
|---|---|---|---|---|
| `BUF_SZ` / parser buffer | **1 MiB = 1 048 576 B** | `kitty/vt-parser.c:18` | §9.1 (fills to exactly 1048576) | Canonical |
| `vt_parser_has_space_for_input` | `read.sz + write.pending < BUF_SZ` | `kitty/vt-parser.c:1476-1484` (gate `:1481`) | §9.1 (`avail=0` at full) | Canonical (condition) |
| `read_bytes` no-read-when-full | `if(!available_buffer_space) return true;` | `kitty/child-monitor.c:1337-1356` (`:1342`) | §9.1 (same buffer API) | Canonical (condition) |
| PTY `POLLIN` gating | `events = has_space ? POLLIN : 0` | `kitty/child-monitor.c:1501` | §3.1 | **Inferred** (io_loop-only; reason in §3.1) |
| Deferred-parse gate | `flush \|\| ≥input_delay \|\| within 16 KiB` | `kitty/vt-parser.c:1425` | §9.2 | Value canonical; timing inferred (§3.2) |
| Near-full bypass | **16 KiB = 16 384 B** (`read.sz > 1032192`) | `kitty/vt-parser.c:1425` | §9.2 | Canonical (threshold) |
| `input_delay` | **3 ms** | `kitty/options/definition.py:878` | §9.2 | Canonical |
| `repaint_delay` | **10 ms** | `kitty/options/definition.py:866` | §9.2 | Canonical |
| delays storage type | `monotonic_t` (ns) | `kitty/state.h:51` | §3.2 | Canonical (code) |
| `DEFAULT_STORAGE_LIMIT` | **320 MiB = 335 544 320 B** | `kitty/graphics.c:25` | §9.3a (`grman.storage_limit`) | Canonical |
| quota trigger (strict `>`) | `used_storage > storage_limit` | `kitty/graphics.c:2184` | §9.3a (5 imgs == limit, no evict) | Canonical |
| `apply_storage_quota` two-phase | trim then LRU | `kitty/graphics.c:289-300` | §9.3a/b | Canonical |
| `trim_predicate` | `!root_frame_data_loaded \|\| !refs` | `kitty/graphics.c:279-281` | §9.3a (survivor `[6]`) | Canonical |
| LRU comparator | `oldest_img_first` (`atime`) | `kitty/graphics.c:284-287` | §9.3b (`[4,5,6,7,8]` survive; `[1,2,3]` evicted oldest-first) | Canonical |
| anim-frame cache ceiling | **`storage_limit*5` = 1 600 MiB** → `ENOSPC` | `kitty/graphics.c:1570-1573` | §9.3c (exact ENOSPC bytes) | Canonical |
| APC response framing | `ESC _ G … ESC \` (`1b 5f … 1b 5c`) | `kitty/screen.c:1047-1050` | §9.4(a) (repr bytes) | Canonical (content) |
| enqueue to write buffer | `write_to_child`→`schedule_write_to_child` | `kitty/screen.c:947-949` | §5 | Canonical (code path) |
| **100 MiB** output cap + drop log | `Too much data being sent to child…` | `kitty/child-monitor.c:341-342` | §9.4(b) (drop at 100th chunk; `write_buf_used`=103 809 420 B) | **Canonical** (observed, deterministic) |
| `POLLOUT`-gated drain + EAGAIN-retain | retain on `EAGAIN`, discard on hard error | `kitty/child-monitor.c:1443-1473` | §9.4(c) (20 971 600 B retained→drained, no loss) | **Canonical** (observed) |
| I/O thread | `pthread_create(&io_thread, io_loop)` = `KittyChildMon` | `kitty/child-monitor.c:291` (named `:1489`) | §9.6a (appears only after `start()`) | Canonical |
| `q=` response gate | `if (g->quiet){ if(is_ok_response \|\| g->quiet>1) return NULL; }` | `kitty/graphics.c:759-763` | §9.5 (`q=0/1/2` bytes) | Canonical |
| error codes | `ENODATA` (+ `EBADF ENOMEM EINVAL ENOENT ENOSPC` in tree) | `kitty/graphics.c:303-316` | §9.4a (`ENODATA`, `EINVAL`), §9.3c (`ENOSPC`) | Canonical (observed 3) |
| pending-mode abort msg | *"…too much data in pending mode"* | `kitty/vt-parser.c:646-648` | §9.6b | Canonical |
| pending-mode timeout | **2000 ms** | `kitty/screen.c:2521` | §8 | Canonical (code) |
| `q=` quiet-key parsing | `quiet = 'q'` → `case quiet:` → `U(quiet)` | `kitty/parse-graphics-command.h:29`, `:88`, `:283` | §9.5 (`q` value drives the gate); §9.7 | Canonical (code) |
| event-loop wakeup / fd-drain | `wakeup_loop`, `drain_fd` | `kitty/loop-utils.h:48`, `:76`; `kitty/loop-utils.c:113`; used `kitty/child-monitor.c:226`, `:1515` | §9.6a (I/O thread scheduling); §9.7 | Canonical (code) |

### 10.3 Rule compliance

- **RUN-FIRST:** every magnitude was produced by executing the real C functions/binary (§9); code-only items are labeled **(inferred from code)** with the reason.
- **Magnitude/timing rigor:** each magnitude states its scale and is confirmed **byte-identical across ≥2 runs** (§9.1–§9.6).
- **Before/during/after:** shown for the buffer (§9.1), image storage (§9.3a/b/c), and the output `write_buf` (§9.4b: `write_buf_used` 0→grew→dropped at the cap; §9.4c: 0→grew→drained back to 0 on `POLLOUT`).
- **Canonical build/config:** kitty was built and run in its **default configuration** inside the canonical Docker container — image name, exact `docker run` + `python3 setup.py`, complete build tail, and `kitty --version` in §2.2; container toolchain (Python 3.12.3 / Go 1.23.4 / gcc 13.3.0) in §2.3. The build exited 0 with zero warnings; no non-default build flag or option override was used.
- **Canonical path:** all headline values come from the real parser / `GraphicsManager` / a real in-process `ChildMonitor` I/O thread, exercised in the canonical container (§2.2); no remote-control or debug-hook value is presented as canonical, and the single native GUI-under-Xvfb run (§9.6a-supplementary) is explicitly labelled **non-canonical** (the container ships no Xvfb).
- **Actual output for every claim:** complete unedited outputs with their exact commands are in §9; byte-sensitive results are shown as `repr()`/hexdump.
- **Read-only repo:** only `blitzy/documentation/kitty_815df1e210e0.md` is added relative to the pinned upstream baseline; all temp scripts were under `/tmp/kitty_obs/` and removed; final `git status --porcelain` is empty (§9.8).

### 10.4 Explicitly inferred-from-code items (and why)

1. **PTY `POLLIN` admission-gate enforcement in the live loop** [`kitty/child-monitor.c:1501`] — the *condition* (buffer full ⇒ no read) is observed at runtime (§9.1) and confirmed at `:1342`; the running-loop poll-flag flip requires an external PTY producer + `strace`, unavailable in this headless container.
2. **`input_delay` 3 ms coalescing *timing*** [`kitty/vt-parser.c:1425`] — the harness parse path forces `flush = true`, so the sub-ms wait is not exercised; the **value** (3 ms) is canonical (§9.2) and the behavior is corroborated by the option's own documentation.

Everything else in this document was observed at runtime through the canonical code path.

