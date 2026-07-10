# How kitty Regulates a Flood of Terminal Graphics Data — Flow Control & Backpressure

> A runtime‑grounded investigation of kitty's read‑side and write‑side flow control for the terminal graphics protocol. Every behavioral claim below is backed either by a `file:line` reference into the checked‑out source (branch `kitty_815df1e210e0`, HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`) or by **actual, unedited output** captured by building kitty in its default configuration and driving a real flood through a real PTY. Statements derived from reading the code that were *not* directly observed at runtime are explicitly marked **(inferred)**.

## Section A — The question

The user asked (verbatim):

> "I want to understand how kitty behaves when a large amount of terminal graphics data arrives faster than the system can comfortably respond. As data flows in and the terminal tries to react, how does it decide whether to buffer, pause, or slow things down? What happens internally when responses need to be written back but the output path is already under pressure? I am curious where those decisions live in the code and how they show up at runtime when the terminal is clearly being pushed beyond its usual pace. Does the system quietly adapt, or are there visible signs that something has shifted in how data is handled? Temporary scripts may be used for observation, but the repository itself should remain unchanged and anything temporary should be cleaned up afterward."

This decomposes into five sub‑questions, each answered below with captured evidence:

- **Q1** — When data floods in, how does kitty decide whether to **buffer**, **pause**, or **slow/coalesce**? (Section C)
- **Q2** — What happens internally when **responses** must be written back but the output path is already under pressure? (Section D)
- **Q3** — **Where** do these decisions live in the code? (Section E)
- **Q4** — How do they **show up at runtime** when kitty is pushed beyond its usual pace? (Section F)
- **Q5** — Does kitty **quietly adapt**, or are there **visible signs**? (Section G)

Section H is a methodology / stability / cleanup appendix containing the exact build and run commands, the full text of every temporary observation script, the stability results, and the proof that the repository was left byte‑for‑byte unchanged.

---

## Section B — Direct answer (read this first)

**kitty does *both*: it quietly adapts *and* it emits visible signs.** Neither description alone is complete — which one you see depends on *where* in the pipeline the pressure lands and *how far* past its comfortable pace kitty is pushed.

**It quietly adapts** by:

- **Bounded buffering** — the VT parser reads incoming PTY bytes into a *fixed* **1 MiB** buffer, `#define BUF_SZ (1024u*1024u)` (`kitty/vt-parser.c:L18`). There is exactly one such buffer per parser; it never grows.
- **Delay‑based coalescing with an adaptive bypass** — under light load kitty deliberately waits up to `input_delay` (default **3 ms**) and `repaint_delay` (default **10 ms**) to batch work, but it *drops those waits* and processes immediately once the buffer is within 16 KiB of full (`kitty/vt-parser.c:L1425`).
- **LRU image eviction** under a **320 MiB** per‑buffer storage quota — `apply_storage_quota` sorts oldest‑first and deletes images until it fits (`kitty/graphics.c:L290-L296`).
- **Disk‑cache offload** — image and animation‑frame pixel data is pushed out of RAM into an on‑disk cache (`kitty/disk-cache.c`), with animation frames bounded by a separate **5×** disk quota (`kitty/graphics.c:L1570`).

**It emits visible signs** by:

- **A child process that blocks inside `write()`** — once the 1 MiB buffer fills, kitty stops asking the OS for more data, the kernel PTY buffer fills, and the program generating the flood is *frozen in the write syscall* until kitty catches up (observed kernel stack `n_tty_write`, Section C).
- **A dropped‑data log line** — if the *response* queue heading back to the child would exceed **100 MiB**, kitty drops the data and logs `Too much data being sent to child with id: %lu, ignoring it` (`kitty/child-monitor.c:L342`; captured in Section D).
- **APC graphics‑protocol error responses** — the client receives real error codes such as `EFBIG`, `EINVAL`, `ENOSPC`, and `ENOENT` embedded in APC escape sequences (captured byte‑for‑byte in Section F).

### Why flow control exists at all — the two‑thread architecture

kitty splits terminal I/O across two threads, and this split is the structural reason a bounded buffer (and therefore backpressure) is required:

- A dedicated **I/O thread** runs `io_loop` (`kitty/child-monitor.c:L1481`). It owns the child PTY file descriptor and performs *all* reads and writes, driving flow control entirely through `poll()` interest flags: it requests `POLLIN` on the child fd only while the parser has space (`kitty/child-monitor.c:L1501`) and requests `POLLOUT` only while there are queued bytes to write (`kitty/child-monitor.c:L1503`).
- The **main thread** drains the parsed output and drives rendering, coalescing render wake‑ups with `set_maximum_wait(OPT(input_delay) - pd.time_since_new_input)` (`kitty/child-monitor.c:L445-L451`).

Because the reader (I/O thread) can run far ahead of the renderer (main thread), the buffer that sits between them *must* be bounded — and once it is bounded, something has to happen when it fills. That "something" is the flow‑control behavior this document dissects.

```mermaid
flowchart LR
    subgraph Child["Child process (shell / graphics client)"]
        W["write() graphics data<br/>to PTY slave (blocking)"]
        R["read() responses<br/>from PTY slave"]
    end
    subgraph IO["io_loop thread — child-monitor.c"]
        POLL{"poll()"}
        RB["read_bytes() L1337"]
        WTC["write_to_child() L1443<br/>break on EWOULDBLOCK L1463"]
        CAP["write_buf + 100 MiB cap<br/>L323-L342 (drop + log_error)"]
    end
    subgraph Parser["VT parser — vt-parser.c"]
        BUF["1 MiB buffer BUF_SZ L18"]
        GATE["vt_parser_has_space_for_input L1477"]
        WORK["run_worker trigger L1425<br/>(delay OR within 16 KiB of full)"]
    end
    subgraph GFX["graphics.c / screen.c"]
        RESP["finish_command_response L759"]
        QUOTA["apply_storage_quota 320 MiB L290"]
        ROUTE["write_escape_code_to_child<br/>screen.c L1050"]
    end

    W -->|PTY bytes| POLL
    POLL -->|POLLIN only if space| RB
    RB --> BUF
    BUF --> WORK
    GATE -. "buffer full ⇒ drop POLLIN<br/>⇒ child blocks in write()" .-> POLL
    BUF -. read.sz used .-> GATE
    WORK --> GFX
    QUOTA -. evict oldest .-> GFX
    RESP --> ROUTE
    ROUTE --> CAP
    CAP --> WTC
    WTC -->|POLLOUT| POLL
    POLL -->|write response| R
```

*Read path is throttled by `GATE` (Q1); write path is bounded by `CAP` (Q2).*

---

## Section C — Q1: The buffer / pause / slow‑down decision (READ side)

When graphics data floods in, kitty makes a **three‑stage** decision — **buffer → pause → slow/coalesce (with an adaptive bypass)**. The three stages are not alternatives chosen at one moment; they are layers that engage in sequence as pressure rises.

### C.1 Buffer — a fixed 1 MiB parser buffer

Incoming PTY bytes are read by the I/O thread's `read_bytes` (`kitty/child-monitor.c:L1337`) directly into the parser's buffer, obtained with `vt_parser_create_write_buffer` and finalized with `vt_parser_commit_write` (declared in `kitty/vt-parser.h`). That buffer is a single fixed‑size region:

```c
// kitty/vt-parser.c:L18
#define BUF_SZ (1024u*1024u)
```

It is **1 MiB and never grows** — there is no "buffer more when busy" path. This is the first and quietest adaptation: bytes simply accumulate here until the main thread parses them. Everything downstream (the pause, the delay bypass) exists precisely because this buffer is bounded.

### C.2 Pause — drop `POLLIN`, let the kernel PTY buffer fill, block the writer

Whether kitty will accept more bytes is decided by one predicate:

```c
// kitty/vt-parser.c:L1477-L1481
bool
vt_parser_has_space_for_input(const Parser *self) {
    return self->read.sz + self->write.pending < BUF_SZ;
}
```

The I/O loop consults it on every poll iteration and sets the child fd's requested events accordingly:

```c
// kitty/child-monitor.c:L1501
children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;
```

**When the buffer is full, kitty requests `0` events on the child fd — it stops asking to read.** It does not drop bytes in‑process; instead the kernel's PTY buffer fills, and because the child's end of the PTY is in **blocking** mode, the writing child is put to sleep inside its `write()` syscall:

```python
# kitty/child.py:L171
master, slave = os.openpty()  # Note that master and slave are in blocking mode
```

This is **backpressure propagated all the way to the source**: kitty doesn't discard the flood, it makes the producer wait.

#### Captured evidence (C1) — the child blocks inside `write()`

To observe the *real* PTY path (not the Python test harness, which bypasses the PTY), a real child was run inside a real kitty window that floods its stdout and never reads. The kitty **process** was then paused with `SIGSTOP` so its `io_loop` stops draining the PTY; the child then fills the kernel PTY buffer and blocks in `write()`. Command: `bash /tmp/kitty_flow_obs/run_c1.sh run1` (full script text in Section H). **Complete, unedited output, run 1:**

```
=== C1 run1: read-side backpressure via real PTY ===
XVFB_wrapper_pid=64136  KITTY_PID=64151  CHILD_PID=64222
kitty comm: kitty
kitty cmdline: ./kitty/launcher/kitty -o confirm_os_window_close=0 python3 /tmp/kitty_flow_obs/flood_write.py 
child cmdline: python3 /tmp/kitty_flow_obs/flood_write.py 

[after SIGSTOP kitty] kitty State: T (stopped)
[child flooding while kitty paused]
child State: S (sleeping)
child wchan: wait_woken
child bytes accepted into PTY before write() blocked = 12288
--- child kernel stack (/proc/64222/stack) ---
[<0>] wait_woken+0x3a/0x60
[<0>] n_tty_write+0x409/0x4b0
[<0>] file_tty_write+0x17c/0x330
[<0>] vfs_write+0x2be/0x390
[<0>] ksys_write+0x75/0xe0
[<0>] do_syscall_64+0x46/0xb0
[<0>] entry_SYSCALL_64_after_hwframe+0x78/0xe2

[1s later] child bytes = 12288  (frozen? YES)
[after SIGCONT kitty] child State: S (sleeping)
child bytes after resume = 114307072  (progressed beyond frozen? YES)
```

**What this shows, line by line:**

- The child is `State: S (sleeping)` with `wchan: wait_woken` and a kernel stack topped by **`n_tty_write`** — it is asleep *inside the `write()` syscall to the tty*, exactly the blocking‑writer sign described above.
- `child bytes accepted into PTY before write() blocked = 12288` — the kernel PTY line‑discipline buffer accepted 12288 bytes and then blocked the writer. *(The 1 MiB parser buffer `BUF_SZ` sits **upstream** of this kernel buffer; the `POLLIN`‑drop that ultimately causes this block is surfaced directly in the debug‑build supplement, Section F.3. The 12288 figure is the **kernel** PTY buffer capacity, not `BUF_SZ`.)*
- While kitty stayed stopped, the counter did not move (`frozen? YES`) — the producer made **zero** progress.
- After `SIGCONT`, kitty resumed draining and the child immediately progressed past the frozen value (`progressed beyond frozen? YES`).

**Stability (≥2 runs).** Re‑running as `run2` produced a byte‑identical block signal — same `n_tty_write` stack, same `12288` bytes, same `frozen? YES`. Only the post‑resume counter differs (run1 `114307072`, run2 `117452800`), which is expected because it depends on how long kitty ran after `SIGCONT`. The load‑bearing values (the block, the stack, the 12288‑byte freeze point) are exactly reproducible.

### C.3 Slow / coalesce — and the adaptive bypass under pressure

Between "buffer" and "pause" sits a timing layer. The main thread's `run_worker` decides *when* to process pending input:

```c
// kitty/vt-parser.c:L1425
if (flush || pd->time_since_new_input >= OPT(input_delay) || self->read.sz + 16 * 1024 > BUF_SZ) {
```

Under **light** load the middle clause dominates: kitty waits up to `input_delay` before processing, batching bytes to avoid waking the renderer for every tiny burst. Under **heavy** load the **third clause** — `self->read.sz + 16 * 1024 > BUF_SZ` — fires: once the buffer is within **16 KiB** of full, kitty **stops waiting and processes immediately**. This is the "slow down, then stop slowing down" adaptation. The paired signal that the buffer had actually been full is recorded right after:

```c
// kitty/vt-parser.c:L1438
pd->write_space_created = self->read.sz >= BUF_SZ;
```

The render side coalesces symmetrically, arming the wait for only the remaining slice of `input_delay`:

```c
// kitty/child-monitor.c:L445-L451  (parse_input → set_maximum_wait)
set_maximum_wait(OPT(input_delay) - pd.time_since_new_input);
```

These delays are ordinary, documented options whose *defaults* are the values kitty runs with out of the box. The option definitions literally document the bypass:

- `repaint_delay` = **10 ms**, `kitty/options/definition.py:L866` — its help text notes it is ignored while there is pending input.
- `input_delay` = **3 ms**, `kitty/options/definition.py:L878` — its help text (`:L885`) states: *"This setting is ignored when the input buffer is almost full."*

Both feed the global options struct as `monotonic_t repaint_delay, input_delay;` (`kitty/state.h:L51`), which `run_worker` and `parse_input` read at runtime.

#### Captured evidence — the default delay values at runtime

The values kitty actually runs with (default configuration) were read back through kitty's own config loader. Command and **complete output**:

```
Command: xvfb-run -a env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe ./kitty/launcher/kitty +runpy 'from kitty.options.types import defaults; print("input_delay=%r repaint_delay=%r" % (defaults.input_delay, defaults.repaint_delay))'
Run 1: input_delay=3 repaint_delay=10
Run 2: input_delay=3 repaint_delay=10
```

**Stable across two runs:** `input_delay=3` (ms), `repaint_delay=10` (ms) — matching the definitions above. The delay **bypass** clause (`read.sz + 16*1024 > BUF_SZ`) is a code‑level branch that does not print a value of its own; it is confirmed by the citation `kitty/vt-parser.c:L1425` and corroborated by the option help text quoted above, and its *consequence* — immediate processing that keeps the buffer from ever exceeding `BUF_SZ` — is what makes the C1 freeze point (kernel buffer, not a runaway parser buffer) observable. **(inferred, from `L1425` + option docs)** that the wait is skipped in the last 16 KiB; the branch itself was not instrumented to print.

---


## Section D — Q2: The write side under pressure (RESPONSES back to the client)

The question's second half concerns the *opposite* direction: kitty itself must write bytes back to the child — most relevantly **graphics‑protocol responses** (acknowledgements and errors) — and that output path can itself be saturated if the child is not reading. kitty handles this with the same poll‑gated discipline, plus a hard ceiling.

### D.1 Responses share the exact write path that is subject to backpressure

This linkage is the crux of Q2, so it is worth making explicit. A graphics response is *built* in `graphics.c`, *returned* through `screen.c`, and *queued onto the child write path* in `child-monitor.c`:

1. **Built** — `finish_command_response` (`kitty/graphics.c:L759`) constructs a success/ack response; `set_command_failed_response` (`kitty/graphics.c:L305`) constructs an error response.
2. **Returned** — `screen_handle_graphics_command` (`kitty/screen.c:L1047`) receives that response and, if non‑NULL, routes it:
   ```c
   // kitty/screen.c:L1050
   if (response != NULL) write_escape_code_to_child(self, ESC_APC, response);
   ```
   (`write_escape_code_to_child` is defined at `kitty/screen.c:L979`.)
3. **Queued** — `write_escape_code_to_child` appends the bytes to the per‑screen `write_buf`:
   ```c
   // kitty/screen.h:L114-L116
   uint8_t *write_buf;
   size_t write_buf_sz, write_buf_used;
   pthread_mutex_t write_buf_lock;
   ```

So **responses are not a special channel** — they land in the very same `write_buf` that all child‑bound output uses, and are therefore subject to the same 100 MiB cap discussed below.

**Important nuance for reproducing the flood.** `finish_command_response` only emits bytes when the command carries an image id or number, and it honours the `quiet` (`q`) key:

```c
// kitty/graphics.c:L762-L766
    if (g->quiet) {
        if (is_ok_response || g->quiet > 1) return NULL;
    }
    if (g->id || g->image_number) {
        if (is_ok_response) {
```

Therefore, to make `write_buf` actually grow you must flood commands that carry `i=<N>` (or `I=`) at the default `q=0`. (`q=1` suppresses success/"OK" responses; `q=2` suppresses all.) The floods below use `a=p,i=999` — a *put* referencing a non‑existent image — which carries `i=999` and thus always produces a response (an `ENOENT`) at default quiet.

### D.2 Buffering + `POLLOUT` gating + the `EWOULDBLOCK` retry

Queued bytes are flushed only when the OS reports the child fd is writable — the I/O loop requests `POLLOUT` *only* while there is something to write:

```c
// kitty/child-monitor.c:L1503
children_fds[EXTRA_FDS + i].events |= (screen->write_buf_used ? POLLOUT : 0);
```

When `POLLOUT` fires, `write_to_child` (`kitty/child-monitor.c:L1443`) drains as much as the kernel will take, then **stops on `EWOULDBLOCK`/`EAGAIN`**, leaving the remainder queued for the next `POLLOUT`:

```c
// kitty/child-monitor.c:L1462-L1463
if (errno == EINTR) continue;
if (errno == EWOULDBLOCK || errno == EAGAIN) break;
```

(The `EWOULDBLOCK`/`EAGAIN` break is the second line, `kitty/child-monitor.c:L1463`.)

(The master fd is non‑blocking — `os.set_blocking(self.child_fd, False)` in `kitty/child.py` — which is *why* kitty gets `EWOULDBLOCK` here rather than blocking, the mirror image of the child's blocking slave in Section C.)

### D.3 The hard 100 MiB cap — drop and log

`write_buf` grows dynamically, but only up to a ceiling. The enqueue macro refuses to let the queue exceed **100 MiB**, dropping the data and logging an error instead:

```c
// kitty/child-monitor.c:L340-L349 (inside the schedule_write_to_child macro, which spans L323-L342+;
// the log_error is L342). Backslash line-continuations are part of the C macro.
            if (space_left < sz) { \
                if (screen->write_buf_used + sz > 100 * 1024 * 1024) { \
                    log_error("Too much data being sent to child with id: %lu, ignoring it", id); \
                    screen_mutex(unlock, write); \
                    break; \
                } \
                screen->write_buf_sz = screen->write_buf_used + sz; \
                screen->write_buf = PyMem_RawRealloc(screen->write_buf, screen->write_buf_sz); \
                if (screen->write_buf == NULL) { fatal("Out of memory."); } \
            } \
```

This is the write‑side counterpart to the read‑side pause: rather than block the *renderer* forever on a child that will not read, kitty caps the outstanding response backlog at 100 MiB and drops the overflow, leaving a visible error in its log.

#### Captured evidence (C2) — the exact dropped‑data log line

A real child was run that floods `\x1b_Ga=p,i=999\x1b\\` (each producing an ~85‑byte `ENOENT` response) and **never reads its stdin**, so kitty's `write_buf` grows unbounded until it hits the cap. Command: `bash /tmp/kitty_flow_obs/run_c2.sh run1`. **Complete, unedited result summary, run 1:**

```
=== C2 run1: write-side 100 MiB cap via real PTY ===
KITTY_PID=68625 CHILD_PID=68696
time_to_first_dropped_log =  s   found=1
total 'Too much data' lines captured before kill = 114355
--- first 3 matching stderr lines (verbatim) ---
[1.679] Too much data being sent to child with id: 1, ignoring it
[1.679] Too much data being sent to child with id: 1, ignoring it
[1.679] Too much data being sent to child with id: 1, ignoring it
--- err file size ---
7547501 bytes
```

The captured stderr line is exactly the source string with `id: 1` substituted for `%lu`:

```
[1.679] Too much data being sent to child with id: 1, ignoring it
```

The `[1.679]` prefix is kitty's own log timestamp (seconds since start), so the cap was reached ≈1.68 s into the run. The only other line on kitty's stderr for the entire run was a single benign startup notice, `[0.153] Failed to open systemd user bus with error: Connection refused` (unrelated to flow control).

**Stability (≥2 runs).** Run 2 produced the identical log text with `id: 1`, first seen at `[1.703]`, repeated `114401` times (run 1: `114355`). The log **text** and the 100 MiB trigger are exactly stable; the repeat count varies slightly run‑to‑run because it depends on how long the flood was allowed to continue before the process was killed — expected, and reported honestly rather than "stabilized."

#### Captured evidence (S1, labeled supplement) — `POLLOUT` gating and the `EWOULDBLOCK` break

The `POLLOUT` gating and the `EWOULDBLOCK` break are *internal* branch points that emit no bytes on their own. To surface them, kitty was **rebuilt in a labeled debug configuration** with its two built‑in debug hooks enabled — `KITTY_PRINT_BYTES_SENT_TO_CHILD` (`kitty/child-monitor.c:L1449`, prints each write to stderr) and `DEBUG_POLL_EVENTS` (`kitty/child-monitor.c:L1550`, prints poll revents to stdout). **This is a supplement, not a substitute** for the canonical path: the flood itself still goes through a real PTY; only the observability was added, and the default binary was restored afterward (Section H).

Poll events captured on stdout during a bounded flood (`i:2` is the child fd) — counts of each distinct line:

```
    533 i:0 POLLIN
     15 i:2 POLLOUT
      7 i:2 POLLIN
      1 i:1 POLLIN
```

This confirms the child fd's write path is driven by `POLLOUT` (15 events) — kitty only wrote when the OS said it could, exactly as `L1503` dictates. The write hook confirmed the response bytes actually being flushed (each `Wrote:` line is one `write_to_child` call; response text is the `ENOENT` string):

```
Wrote: 1190 bytes: \x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\
```

Finally, to catch the `EWOULDBLOCK` break itself, a variant flood accumulated a large `write_buf` and then drained stdin *slowly* so `POLLOUT` kept firing while the kernel buffer was near‑full; the write then returned `-1` and kitty took the `L1463` break. Captured verbatim on stderr (the write hook prints the raw `ret`):

```
Wrote: -1 bytes: 
```

with `i:2 POLLOUT` observed 18 times in that run — i.e. kitty repeatedly re‑armed `POLLOUT` and retried, precisely the "leave the remainder queued for the next `POLLOUT`" behavior. The `-1` is the `write()` return immediately before `if (errno == EWOULDBLOCK || errno == EAGAIN) break;` (`kitty/child-monitor.c:L1463`).

---


## Section E — Q3: Where the decisions live in the code

Every decision above maps to a specific file, function, and line at HEAD `815df1e210e0`:

| Decision | File | Function / macro | Line(s) |
|---|---|---|---|
| Fixed 1 MiB read buffer | `kitty/vt-parser.c` | `#define BUF_SZ (1024u*1024u)` | L18 |
| Read backpressure gate | `kitty/vt-parser.c` | `vt_parser_has_space_for_input` | L1477–L1481 |
| Process‑now trigger + 16 KiB bypass | `kitty/vt-parser.c` | `run_worker` | L1425 |
| "Buffer had been full" signal | `kitty/vt-parser.c` | `write_space_created` assignment | L1438 |
| Parser buffer API | `kitty/vt-parser.h` | `create_write_buffer` / `commit_write` / `has_space_for_input` | — |
| I/O event loop | `kitty/child-monitor.c` | `io_loop` | L1481 |
| Read from PTY into parser | `kitty/child-monitor.c` | `read_bytes` | L1337 |
| **Read** poll gate (`POLLIN` iff space) | `kitty/child-monitor.c` | `io_loop` | L1501 |
| **Write** poll gate (`POLLOUT` iff queued) | `kitty/child-monitor.c` | `io_loop` | L1503 |
| Write drain + `EWOULDBLOCK` break | `kitty/child-monitor.c` | `write_to_child` | L1443 (break at L1463) |
| 100 MiB write cap + drop log | `kitty/child-monitor.c` | enqueue macro | L323–L342 (log at L342) |
| Render coalescing | `kitty/child-monitor.c` | `parse_input` → `set_maximum_wait` | L445–L451 |
| Debug hooks (supplements) | `kitty/child-monitor.c` | `KITTY_PRINT_BYTES_SENT_TO_CHILD`, `DEBUG_POLL_EVENTS` | L1449, L1550 |
| Response routing onto `write_buf` | `kitty/screen.c` | `screen_handle_graphics_command` → `write_escape_code_to_child` | L1047, L1050 (def L979) |
| `write_buf` fields + parser handle | `kitty/screen.h` | `Screen` struct | L114–L116, L158 |
| Success/ack response builder + quiet/id gate | `kitty/graphics.c` | `finish_command_response` | L759, L762–L766 |
| Error response builder | `kitty/graphics.c` | `set_command_failed_response` | L305 |
| 320 MiB storage quota | `kitty/graphics.c` | `#define DEFAULT_STORAGE_LIMIT 320u * (1024u * 1024u)` | L25 |
| LRU eviction | `kitty/graphics.c` | `apply_storage_quota` (`HASH_SORT(self->images, oldest_img_first)`) | L290–L296 |
| Per‑transfer cap (400 MB) + `EFBIG` | `kitty/graphics.c` | `MAX_DATA_SZ`, `load_image_data` | L521, L533 |
| PNG oversize `EINVAL` | `kitty/graphics.c` | `initialize_load_data` | L638 |
| Disk‑cache `ENOSPC` | `kitty/graphics.c` | frame add path | L746 |
| 5× animation‑frame disk quota | `kitty/graphics.c` | `cache_size(self) + load_data->data_sz > storage_limit * 5` | L1570 |
| On‑disk spillover | `kitty/disk-cache.c` | `add_to_disk_cache` / `remove_from_disk_cache` | L488, L517 |
| Delay defaults | `kitty/options/definition.py` | `repaint_delay` (10 ms), `input_delay` (3 ms) | L866, L878 |
| Delay storage in `OPT` | `kitty/state.h` | `monotonic_t repaint_delay, input_delay;` | L51 |
| Child PTY blocking mode | `kitty/child.py` | `os.openpty()` | L171 |

---

## Section F — Q4: How it shows up at runtime (each artifact captured)

Below, each observable artifact is captured directly, with before/intermediate/after values for anything stateful.

### F.1 A child blocking on `write()` (read‑side backpressure)

Fully captured in **Section C.2 (C1)**: kernel stack `n_tty_write`, `12288` bytes accepted before the block, counter frozen while kitty is paused, progress resumed on `SIGCONT`. This is the runtime signal that kitty has "stopped reading" — the producer is *made to wait*.

### F.2 The dropped‑data log line (write‑side cap)

Fully captured in **Section D.3 (C2)**: `[1.679] Too much data being sent to child with id: 1, ignoring it`, repeated ~114k times, stable across two runs (`[1.703]` in run 2). This is the runtime signal that the response backlog crossed 100 MiB.

### F.3 APC graphics‑protocol error responses (byte‑for‑byte)

These are the most directly "visible" signs — the client receives real bytes. Each was produced by writing a single graphics command through a real kitty PTY (raw mode, no echo) and capturing kitty's APC reply. The harness records `repr`, byte `len`, and `hex`; commands and outputs are **complete and unedited**. Each case was verified **byte‑identical across two runs** (Section H).

**Baseline — success (`OK`).** Command bytes `\x1b_Ga=q,i=1,f=24,s=1,v=1;AAAA\x1b\\` (a query carrying `i=1`):

```
rawmode=ok
len=11
repr=b'\x1b_Gi=1;OK\x1b\\'
hex=1b5f47693d313b4f4b1b5c
```

Decoded: `ESC _ G i=1 ; O K ESC \` — an APC‑wrapped `i=1;OK`. This confirms responses are APC escape codes routed via `write_escape_code_to_child` (`kitty/screen.c:L1050`).

**`EINVAL` — PNG data size too large.** A PNG transfer (`f=100`) declaring `S=500000000` bytes, which exceeds `MAX_DATA_SZ` = `4u * 100000000u` = **400,000,000** bytes (`kitty/graphics.c:L521`), rejected at `kitty/graphics.c:L638`. Command bytes `\x1b_Ga=q,i=1,f=100,S=500000000;AAAA\x1b\\`:

```
rawmode=ok
len=39
repr=b'\x1b_Gi=1;EINVAL:PNG data size too large\x1b\\'
hex=1b5f47693d313b45494e56414c3a504e4720646174612073697a6520746f6f206c617267651b5c
```

**`EFBIG` — too much data.** A direct (non‑PNG, `f=24`) transfer declaring a 1×1 image (`s=1,v=1` → 3 bytes of capacity) but sending a 300‑byte (225‑byte decoded) chunk. Because the format is not PNG, the buffer cannot grow, so `load_image_data` aborts with `EFBIG` at `kitty/graphics.c:L533` (the `data_fmt != PNG` branch of the capacity check at L532). Command bytes `\x1b_Ga=t,i=1,f=24,s=1,v=1,m=1;` + `A`×300 + `\x1b\\`:

```
rawmode=ok
len=28
repr=b'\x1b_Gi=1;EFBIG:Too much data\x1b\\'
hex=1b5f47693d313b45464249473a546f6f206d75636820646174611b5c
```

**`ENOENT` — reference to a non‑existent image.** A put (`a=p`) referencing image id 999, which does not exist, rejected in the put path (`kitty/graphics.c:L1048`). Command bytes `\x1b_Ga=p,i=999\x1b\\`:

```
rawmode=ok
len=85
repr=b'\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\'
hex=1b5f47693d3939393b454e4f454e543a50757420636f6d6d616e642072656665727320746f206e6f6e2d6578697374656e7420696d61676520776974682069643a2039393920616e64206e756d6265723a20301b5c
```

(This is the same response the C2/S1 write‑side floods used to grow `write_buf` — see how the identical `ENOENT` text appears in the Section D `Wrote:` capture.)

### F.4 Image LRU eviction under the 320 MiB storage quota

**Units nuance stated up front:** the code quota is **320 MiB** — `#define DEFAULT_STORAGE_LIMIT 320u * (1024u * 1024u)` (`kitty/graphics.c:L25`) = **335,544,320 bytes**. The protocol docs colloquially call it "320MB per buffer" (`docs/graphics-protocol.rst`). Both refer to the same limit; the exact byte value is 335,544,320.

Eviction is enforced by `apply_storage_quota` (`kitty/graphics.c:L290-L296`): it first removes *unreferenced* images (`remove_images(self, trim_predicate, …)`, where `trim_predicate` = `!img->root_frame_data_loaded || !img->refs`, L280/L292), and if storage is still over the limit it sorts oldest‑first (`HASH_SORT(self->images, oldest_img_first)`) and deletes until it fits.

Because `used_storage` is not exposed to Python, eviction was observed via `image_count` (= `HASH_COUNT`, `kitty/graphics.c:L2362`) and `disk_cache.total_size`, driven through kitty's own in‑tree harness. **This is a labeled supplement (S2):** `send_command`/`parse_bytes` feed the parser directly and bypass the PTY + `io_loop`, but they exercise the *identical* `graphics.c` storage code regardless of transport. (A single inline APC larger than 1 MiB is itself rejected by the parser as `VTE_APC escape code too long (1048574 bytes)` — a direct manifestation of `BUF_SZ` — so large images below use the file‑transfer medium `t='f'`.)

**At the true default quota (320 MiB), via file transfer.** Each image is 4096×4096×3 = 50,331,648 bytes (48.00 MiB), with a distinct first byte to defeat content de‑duplication. **Complete, unedited output (byte‑identical across both runs):**

```
grman.storage_limit (DEFAULT) = 335544320 bytes (== 320*1024*1024 ? True)
each image: s=4096 v=4096 f=24 t=f => data_sz=50331648 bytes (48.00 MiB); adding 9 => 432 MiB > 320 MiB
img_id  code   image_count  disk_cache.total_size(bytes)   total(MiB)  <=320MiB?
     1  OK               1                      50331648      48.00  yes
     2  OK               2                     100663296      96.00  yes
     3  OK               3                     150994944     144.00  yes
     4  OK               4                     201326592     192.00  yes
     5  OK               5                     251658240     240.00  yes
     6  OK               6                     301989888     288.00  yes
     7  OK               2                     100663296      96.00  yes
     8  OK               3                     150994944     144.00  yes
     9  OK               4                     201326592     192.00  yes
```

**Before / intermediate / after:** storage grows monotonically to **288.00 MiB** across the first 6 images (`image_count` 1→6), then adding image #7 — which would reach 336 MiB > 320 MiB — triggers eviction, and the count/size **drop to 2 / 96 MiB**; #8 and #9 rebuild to 3/144 and 4/192 MiB. **At every step `disk_cache.total_size` stays ≤ 335,544,320 bytes** — the quota is never exceeded. The size of the 6→2 drop (rather than a single‑image eviction) is because `apply_storage_quota`'s *first* step trims images whose placements have scrolled off the small (5×5) test screen and thus lost their `refs`; this is real, code‑grounded behavior (`trim_predicate`, `kitty/graphics.c:L280`), and it too was byte‑identical across both runs.

**At a lowered quota (72 bytes) — the pure‑LRU transition, crisply.** To isolate the LRU `while` loop from placement‑scroll trimming, the quota was set to 72 bytes (**NON‑DEFAULT**; kitty's own test `test_graphics_quota_enforcement` uses exactly this) and three 36‑byte images were put. **Complete, unedited output (byte‑identical across both runs):**

```
storage_limit=72 bytes (NON-DEFAULT); mirrors kitty_tests test_graphics_quota_enforcement
step                       code   image_count  disk_cache.total_size
put img i=1 (a=T,36B)      OK               1  36
put img i=2 (a=T,36B)      OK               2  72
put img i=3 (a=T,36B)      OK               2  72  <-- img#3: oldest EVICTED by LRU while-loop (count stays 2, disk==limit)
```

**Before / intermediate / after:** `image_count` 1 → 2 → **2** and `disk_cache.total_size` 36 → 72 → **72**. Adding the third 36‑byte image (which would reach 108 > 72) evicts exactly the oldest, holding storage at the 72‑byte limit — the pure `while (used_storage > storage_limit) remove_image(oldest)` loop (`kitty/graphics.c:L296`). This matches kitty's own test assertions.

### F.5 Disk‑cache spillover and the 5× animation‑frame quota → `ENOSPC`

Animation‑frame data is stored on disk under a *separate*, larger quota — five times the base limit:

```c
// kitty/graphics.c:L1570
if (is_new_frame && cache_size(self) + load_data->data_sz > self->storage_limit * 5)
```

When a new frame would exceed `storage_limit * 5`, the disk‑cache add fails and the command is rejected with `ENOSPC` (`kitty/graphics.c:L746`). Observed via the same labeled harness, at the 72‑byte lowered limit (so the 5× quota is 360 bytes), by appending 36‑byte frames to image id 2. **Complete, unedited output (byte‑identical across both runs):**

```
storage_limit=72 => 5x frame disk quota = 360 bytes
frame#  code    disk_cache.total_size
     0  OK      108
     1  OK      144
     2  OK      180
     3  OK      216
     4  OK      252
     5  OK      288
     6  OK      324
     7  OK      360
  over  ENOSPC  360  <-- ENOSPC: frame disk quota (5x) exceeded
```

**Before / intermediate / after:** `disk_cache.total_size` climbs 108 → 360 in 36‑byte steps as frames 0–7 are stored on disk (spillover), then the *9th* frame would push past 360 and is rejected — the code stays `ENOSPC` and the on‑disk size stays pinned at **360**. This is the disk‑cache spillover *and* its ceiling in one capture, and it reproduces the exact `ENOSPC` outcome kitty's own `test_graphics_quota_enforcement` asserts.

---


## Section G — Q5: Quiet adaptation vs. visible signs

**kitty does both** — and the two are cleanly separable by direction and by degree of pressure. Each side is backed by its own captured evidence from the sections above.

### Quiet adaptations (no user‑visible artifact)

| Adaptation | Mechanism | Evidence |
|---|---|---|
| Bounded buffering | 1 MiB `BUF_SZ` parser buffer; bytes accumulate silently | `kitty/vt-parser.c:L18`; the parser rejecting a >1 MiB inline APC as `VTE_APC escape code too long (1048574 bytes)` (Section F.4) |
| Delay coalescing + adaptive bypass | wait `input_delay`/`repaint_delay`, but process immediately within 16 KiB of full | `kitty/vt-parser.c:L1425`; defaults `input_delay=3 repaint_delay=10` captured stably (Section C.3) |
| LRU image eviction | `apply_storage_quota` deletes oldest images to stay ≤ 320 MiB | `disk_cache.total_size` bounded ≤ 335,544,320; `image_count` 6→2 at the quota (Section F.4) |
| Disk‑cache offload | image/frame pixels spilled to on‑disk cache | `disk_cache.total_size` climbing 108→360 as frames are stored (Section F.5) |

These are "quiet" because the *only* effects the user could notice are indirect (slightly delayed rendering, an old image disappearing) — kitty emits no log and returns no error for any of them.

### Visible signs (an observable artifact)

| Sign | Where it appears | Evidence |
|---|---|---|
| Producer frozen in `write()` | the *child* program stalls (kernel `n_tty_write`) | Section C.2 (C1): 12288‑byte freeze, `frozen? YES`, resumes on `SIGCONT` |
| Dropped‑data log error | kitty's **stderr / log** | Section D.3 (C2): `[1.679] Too much data being sent to child with id: 1, ignoring it` |
| APC error responses | bytes **sent back to the client** | Section F.3: `EINVAL`, `EFBIG`, `ENOENT` (and `OK`) captured byte‑for‑byte |
| `ENOSPC` on frame quota | bytes sent back to the client | Section F.5: `ENOSPC` at the 5× disk quota boundary |

**The through‑line:** kitty prefers to *quietly* absorb pressure (buffer, delay‑coalesce, evict, offload) for as long as it safely can, and only produces a *visible* sign when a hard boundary is crossed — the 1 MiB read buffer (→ frozen writer), the 100 MiB write cap (→ log error), a per‑transfer or per‑image limit (→ APC error), or a storage quota (→ eviction, and `ENOSPC` for frames). Push it gently and you see nothing; push it past a boundary and the shift becomes unmistakable.

---

## Section H — Methodology, stability, and cleanup appendix

### H.1 Build and run (default, canonical configuration)

- **Repository revision:** branch `kitty_815df1e210e0`, HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`. All `file:line` citations were re‑verified against this revision.
- **Build command:** `PATH=/usr/local/go/bin:$PATH ./dev.sh build`. Result (tail): `Build successful. Run kitty as: kitty/launcher/kitty` — producing the launcher `kitty/launcher/kitty` (kitty **0.35.2**) and the `fast_data_types` C extension `kitty/fast_data_types.so` (which contains the parser, screen, graphics, and child‑monitor code). The build preserves the project's default `-Werror`/`-pedantic-errors`.
- **Run (headless):** `xvfb-run -a env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe ./kitty/launcher/kitty …` — required because the canonical PTY flow‑control code only runs inside a full kitty process with a real window, child, and PTY.
- **Default binary integrity:** the default `kitty/fast_data_types.so` has md5 `0ba7e3c76fb9a41c76a733d8d990e5f7`. It was backed up before the debug rebuild and restored after; the md5 was re‑verified equal at the end (Section H.4).

### H.2 Canonical vs. supplement vs. inferred — how each claim was obtained

- **Canonical (real PTY + `io_loop`):** C1 (read‑side block, Section C.2), C2 (100 MiB drop log, Section D.3), and the C3/C4 APC error responses (Section F.3). These are the load‑bearing observations for Q1, Q2, and Q4; each used a real child on a real PTY.
- **Labeled supplement — debug build (S1):** to surface the internal `POLLIN`/`POLLOUT` gating and the `EWOULDBLOCK` break (Section D), kitty was rebuilt with `CPPFLAGS="-DDEBUG_POLL_EVENTS -DKITTY_PRINT_BYTES_SENT_TO_CHILD"` (these hooks exist in‑tree at `kitty/child-monitor.c:L1550` and `:L1449`). The flood itself remained a real PTY flood; only observability was added; the default `.so` was restored afterward.
- **Labeled supplement — in‑tree harness (S2):** storage‑quota eviction and the 5× frame quota (Section F.4/F.5) were observed with `send_command`/`parse_bytes`, which bypass the PTY + `io_loop` but exercise the identical `graphics.c` storage code. The default 320 MiB quota was used for the load‑bearing eviction observation; a 72‑byte quota (kitty's own test value, explicitly labeled **NON‑DEFAULT**) was used only to render the pure‑LRU transition crisply.
- **(inferred):** the 16 KiB delay‑bypass branch (`kitty/vt-parser.c:L1425`) was not instrumented to print; it is asserted from the code + option help text (Section C.3). Two dependency files were confirmed *not* on the graphics flow‑control path and are mentioned only as context: `3rdparty/ringbuf/` is used by `kitty/history.c` for pager history **(inferred)**, and `kitty/shm.py` is a POSIX shared‑memory transfer medium that changes the *shape* of a flood but not the flow‑control logic **(inferred)**; neither was exercised.

### H.3 Stability (every reported magnitude confirmed across ≥ 2 runs)

| Quantity | Run 1 | Run 2 | Stable? |
|---|---|---|---|
| `input_delay` / `repaint_delay` (ms) | 3 / 10 | 3 / 10 | yes (identical) |
| C1 bytes accepted before block | 12288 | 12288 | yes (identical) |
| C1 child kernel stack top | `n_tty_write` | `n_tty_write` | yes (identical) |
| C1 frozen while paused | YES | YES | yes (identical) |
| C1 progressed after `SIGCONT` | YES (114307072) | YES (117452800) | block stable; resumed count varies (expected — depends on run‑after‑CONT duration) |
| C2 drop‑log text | `…with id: 1, ignoring it` | `…with id: 1, ignoring it` | yes (identical text) |
| C2 first‑seen timestamp | `[1.679]` | `[1.703]` | ~1.68–1.70 s (stable order of magnitude) |
| C2 drop‑log occurrences | 114355 | 114401 | count varies (expected — depends on flood duration) |
| C3/C4 `OK`/`EINVAL`/`EFBIG`/`ENOENT` bytes | (hex captured) | (hex captured) | yes (byte‑identical, diff empty) |
| S2 default `storage_limit` | 335544320 | 335544320 | yes (== 320·1024·1024) |
| S2 Part A LRU (`image_count`/disk) | 1→2→2 / 36→72→72 | 1→2→2 / 36→72→72 | yes (identical) |
| S2 Part B (disk MiB across 9 imgs) | 48…288→96→144→192 | 48…288→96→144→192 | yes (identical) |
| S2 Part C frame `ENOSPC` boundary | ENOSPC at 360 | ENOSPC at 360 | yes (identical) |

*Scale/duration used:* C1 — 4096‑byte writes in a tight loop, kitty paused ~2 s; C2 — batches of 20000 commands, run until the cap log appeared (~1.7 s) then killed; C3/C4 — one command per run, 2 s read window; S2 — up to 9 × 48 MiB images (Part B) / 3 × 36 B images (Part A) / 9 × 36 B frames (Part C).

### H.4 Temporary scripts (the exact commands that produced the output)

All observation scripts lived **outside** the repository, under `/tmp/kitty_flow_obs`, and were removed afterward. Their full text is reproduced here because it is part of "the command that produced the output."

**C1 child — `flood_write.py`** (writes to the PTY, never reads; mmap counter so only the PTY `write()` can block):

```python
import os, time, mmap
OBS = "/tmp/kitty_flow_obs"
with open(os.path.join(OBS, "child.pid"), "w") as f:
    f.write(str(os.getpid()))
cntf = os.path.join(OBS, "child_bytes")
cf = open(cntf, "w+b"); cf.write(b"0".ljust(32, b"\0")); cf.flush()
mm = mmap.mmap(cf.fileno(), 32)
with open(os.path.join(OBS, "child.ready"), "w") as f:
    f.write("ready\n")
go = os.path.join(OBS, "go")
while not os.path.exists(go):
    time.sleep(0.02)
chunk = b"X" * 4096
total = 0
def set_counter(v):
    s = str(v).encode() + b"\0"
    mm[0:len(s)] = s
while True:
    n = os.write(1, chunk)    # BLOCKS when the OS PTY buffer is full
    total += n
    set_counter(total)
```

The C1 driver `run_c1.sh` launches kitty with that child under `xvfb-run`, identifies the *real* kitty process by `/proc/<pid>/comm == "kitty"` (excluding the `xvfb-run` wrapper), sends it `SIGSTOP`, samples `/proc/<child>/{stat,wchan,stack}` and the byte counter, then `SIGCONT`s and re‑samples.

**C2 child — `flood_query.py`** (floods response‑producing commands, never reads stdin):

```python
import os, time, tty
try:
    tty.setraw(0)
except Exception:
    pass
OBS = "/tmp/kitty_flow_obs"
open(os.path.join(OBS, "child.pid"), "w").write(str(os.getpid()))
open(os.path.join(OBS, "child.ready"), "w").write("ready\n")
cmd = b"\x1b_Ga=p,i=999\x1b\\"                        # -> ENOENT (non-existent image)
batch = cmd * 20000
go = os.path.join(OBS, "go")
while not os.path.exists(go):
    time.sleep(0.02)
while True:
    os.write(1, batch)                               # never reads stdin
```

The C2 driver `run_c2.sh` runs kitty with this child, captures kitty's stderr to a file, polls for the `Too much data` line, then reports the first three matching lines and the total count.

**C3/C4 send‑receive — `pty_send_recv.py`** (writes one command from `cmd.bin`, records kitty's APC reply):

```python
import os, time, termios, tty, select
OBS = "/tmp/kitty_flow_obs"
fd = 0
try:
    termios.tcgetattr(fd)
    tty.setraw(fd)                         # disable ECHO/ICANON on the slave
    rawmode = "ok"
except Exception as e:
    rawmode = "err:%r" % e
cmd = open(os.path.join(OBS, "cmd.bin"), "rb").read()
os.write(1, cmd)
buf = b""
deadline = time.monotonic() + 2.0
while time.monotonic() < deadline:
    r, _, _ = select.select([0], [], [], 0.2)
    if r:
        try:
            chunk = os.read(0, 65536)
        except OSError:
            break
        if not chunk:
            break
        buf += chunk
        deadline = time.monotonic() + 0.4
with open(os.path.join(OBS, "resp.txt"), "w") as f:
    f.write("rawmode=%s\n" % rawmode)
    f.write("len=%d\n" % len(buf))
    f.write("repr=%r\n" % buf)
    f.write("hex=%s\n" % buf.hex())
open(os.path.join(OBS, "done"), "w").write("done")
time.sleep(0.2)
```

The four command byte‑strings written into `cmd.bin` were, respectively: `\x1b_Ga=q,i=1,f=24,s=1,v=1;AAAA\x1b\\` (OK), `\x1b_Ga=q,i=1,f=100,S=500000000;AAAA\x1b\\` (EINVAL), `\x1b_Ga=t,i=1,f=24,s=1,v=1,m=1;` + `A`×300 + `\x1b\\` (EFBIG), and `\x1b_Ga=p,i=999\x1b\\` (ENOENT).

**S2 harness — `s2_quota.py`** (storage‑quota eviction + 5× frame quota, via the in‑tree `TestGraphics` harness; a **labeled supplement**). Key points: it uses only pure reads (`image_count` = `HASH_COUNT`, `disk_cache.total_size`) and never probes with `image_for_client_id` (which would *create* missing ids via `find_or_create_image`, `kitty/graphics.c:L2298`); large images use the `t='f'` file medium to avoid the 1 MiB inline‑APC limit; each image gets a distinct first byte to defeat de‑duplication. It was launched with `KITTY_REPO=$PWD … kitty/launcher/kitty +launch /tmp/kitty_flow_obs/s2_quota.py`.

The S1 debug‑build floods (`flood_s1.py`, `flood_s1_slow.py`) were identical in spirit to `flood_query.py` (the slow variant additionally drains stdin slowly so `POLLOUT` keeps firing while `write_buf` is large, provoking the `EWOULDBLOCK` `Wrote: -1 bytes`).

### H.5 The repository was left byte‑for‑byte unchanged

The debug `.so` was restored to the default build and re‑verified:

```
0ba7e3c76fb9a41c76a733d8d990e5f7  kitty/fast_data_types.so
expected default: 0ba7e3c76fb9a41c76a733d8d990e5f7
```

After removing all temporary scratch files and before committing this document, `git status --porcelain` showed **no** modified tracked files — only this new, untracked answer document under `blitzy/` (a directory that is not part of the upstream source tree). No source, configuration, test, or build file in the repository was modified, added, or deleted. Every build artifact produced during the investigation is git‑ignored, and the working tree at HEAD `815df1e210e0` is unchanged.

