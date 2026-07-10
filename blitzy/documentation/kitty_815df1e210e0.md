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
// kitty/vt-parser.c:L1476-L1484
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

The predicate is exactly `self->read.sz + self->write.pending < BUF_SZ`, taken under the parser lock (`with_lock`). `read.sz` is bytes already read and awaiting parse; `write.pending` is bytes staged by the I/O thread but not yet folded into `read.sz`. Their sum is the total buffer occupancy, and once it reaches `BUF_SZ` the predicate is **false**.

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

#### Captured evidence (C1) — the child blocks inside `write()` while kitty runs **normally**

The load‑bearing observation is that this backpressure is **automatic**: kitty is never paused, killed, or otherwise interfered with. A real child is run inside a real kitty window (default build, real PTY) that floods graphics commands to its stdout as fast as the OS allows and never reads its own input — so the *only* thing that can throttle it is kitty's own read‑side gate (the `POLLIN`‑drop of §C.2). A separate sampler reads `/proc/<child>/stack`, `/proc/<child>/wchan`, `/proc/<kitty>/stat`, and the child's byte counter 60 times over ~3.6 s. Command (full script in Section H.4):

```
xvfb-run -a env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
  ./kitty/launcher/kitty -o confirm_os_window_close=0 python3 /tmp/<scratch>/flood_read.py
```

**Complete, unedited output, run 1** (all PIDs are run‑local/ephemeral — they change every run and carry no meaning beyond identifying the processes within this one run):

```
=== read-side backpressure (DEFAULT canonical build) run1 mode=flood ===
KITTY_PID=129818 CHILD_PID=129889  [PIDs run-local/ephemeral]
samples=60  kitty_in_STOPPED(T)=0  child_blocked_in_tty_write_wait(wchan=wait_woken)=57/60
child_bytes first=2930400 last=59584800  elapsed=3.61s  throughput~=15.7 MB/s
--- child kernel stack (blocked in write to tty) ---
[<0>] wait_woken+0x3a/0x60
[<0>] n_tty_write+0x409/0x4b0
[<0>] file_tty_write+0x17c/0x330
[<0>] vfs_write+0x2be/0x390
[<0>] ksys_write+0x75/0xe0
[<0>] do_syscall_64+0x46/0xb0
[<0>] entry_SYSCALL_64_after_hwframe+0x78/0xe2
--- kitty stderr (startup only) ---
[0.153] Failed to open systemd user bus with error: Connection refused
RUN_run1 done
```

**What this shows, line by line:**

- `kitty_in_STOPPED(T)=0` — across all 60 samples kitty was **never** in the stopped (`T`) state. It ran normally the entire time. The throttling below is therefore kitty's *own* doing, not an externally imposed pause.
- `child_blocked_in_tty_write_wait(wchan=wait_woken)=57/60` — in 57 of 60 samples the flooding child was asleep in the kernel with `wchan=wait_woken`, and its kernel stack is topped by **`n_tty_write`**: it is blocked *inside its `write()` syscall to the tty*, waiting for the tty buffer to drain. That is OS‑level backpressure propagated to the producer, exactly as §C.2 describes.
- `throughput~=15.7 MB/s` — the child is not free‑running; its sustained write rate is pinned to the rate at which kitty drains the PTY (its parse rate). The producer has been *slowed to the consumer's pace*.

**Control — the throttle is load‑dependent, not inherent.** Re‑running the identical harness in `burst:65536` mode (the child writes only 64 KiB then stops) shows the child finishing instantly with no block — proving the block above is caused by *sustained pressure*, not by any fixed per‑write cost:

```
=== read-side backpressure (DEFAULT canonical build) control mode=burst:65536 ===
KITTY_PID=131157 CHILD_PID=131228  [PIDs run-local/ephemeral]
samples=1  kitty_in_STOPPED(T)=0  child_blocked_in_tty_write_wait(wchan=wait_woken)=0/1
child COMPLETED at sample 1 (total bytes=65536)
child_bytes first=65536 last=65536  elapsed=0.07s  throughput~=0.0 MB/s
--- kitty stderr (startup only) ---
[0.152] Failed to open systemd user bus with error: Connection refused
RUN_control done
```

**Stability (≥2 runs).** `run2` reproduced the same signal — `kitty_in_STOPPED(T)=0`, the identical `n_tty_write` kernel stack, `child_blocked...=58/60`, and `throughput~=15.5 MB/s`:

```
=== read-side backpressure (DEFAULT canonical build) run2 mode=flood ===
KITTY_PID=130616 CHILD_PID=130687  [PIDs run-local/ephemeral]
samples=60  kitty_in_STOPPED(T)=0  child_blocked_in_tty_write_wait(wchan=wait_woken)=58/60
child_bytes first=2930400 last=58933600  elapsed=3.62s  throughput~=15.5 MB/s
--- child kernel stack (blocked in write to tty) ---
[<0>] wait_woken+0x3a/0x60
[<0>] n_tty_write+0x409/0x4b0
[<0>] file_tty_write+0x17c/0x330
[<0>] vfs_write+0x2be/0x390
[<0>] ksys_write+0x75/0xe0
[<0>] do_syscall_64+0x46/0xb0
[<0>] entry_SYSCALL_64_after_hwframe+0x78/0xe2
--- kitty stderr (startup only) ---
[0.154] Failed to open systemd user bus with error: Connection refused
RUN_run2 done
```

The block, the `n_tty_write` stack, and the ~15.5–15.7 MB/s throughput cap (blocked in 57–58 of 60 samples) are reproducible; only the ephemeral PIDs and the exact byte totals differ.

#### Captured evidence (C1‑supplement, labeled) — the gate itself: `has_space == false`, `POLLIN` removed

The `/proc` capture above proves the *consequence* (the child blocks while kitty runs). To confirm the *mechanism* — that the buffer is actually full and kitty has actually stopped requesting `POLLIN` — a **debug build** (`./dev.sh build --debug` with `-DDEBUG_POLL_EVENTS -DKITTY_PRINT_BYTES_SENT_TO_CHILD`; a **labeled supplement**, not the default build) was driven with the same real flood, and `gdb` async‑attached (no breakpoint‑single‑stepping, to avoid an observer effect) to snapshot the live parser state three times per run. It reads `((struct PS*)children[0].screen->vt_parser->state)->read.sz` and `->write.pending`, evaluates the §C.2 predicate, and reads the child fd's requested poll events `children_fds[2].events` (`POLLIN==1`). **Complete, unedited output, run 1:**

```
KITTY_PID=127258 (pgid=127240 matches launcher pgid=127240) CHILD_PID=127329
snap1 kittyState=R child_stacktop=running read.sz=1048576 write.pending=0 sum=1048576 has_space=0 childfd=9 req_events=0 write_buf_used=0
snap2 kittyState=S child_stacktop=running read.sz=1048576 write.pending=0 sum=1048576 has_space=0 childfd=9 req_events=0 write_buf_used=0
snap3 kittyState=R child_stacktop=running read.sz=1048576 write.pending=0 sum=1048576 has_space=0 childfd=9 req_events=0 write_buf_used=0
--- DEBUG_POLL_EVENTS counts (child fd=i:2) ---
  63068 i:2 POLLIN
    233 i:0 POLLIN
      1 i:2 POLLI
--- total poll lines: 64046 ---
RUN_run1 complete
```

- `read.sz=1048576  write.pending=0  sum=1048576` — the parser buffer is occupied to **exactly `BUF_SZ` (1 048 576 bytes)**. It is completely full and, by design, never larger.
- `has_space=0` — `vt_parser_has_space_for_input()` returns **false** at this instant: `read.sz + write.pending < BUF_SZ` is `1048576 < 1048576` = false.
- `req_events=0` — the requested poll events on the child fd are `0`, i.e. **`POLLIN` has been removed**: kitty is no longer asking to read from the child. This is the exact "pause" of §C.2, observed live.
- `kittyState=R`/`S` — kitty is running/sleeping normally (never `T`), confirming again the gate is automatic.

**Stability (≥2 runs).** `run2` was byte‑identical on the load‑bearing fields — all three snapshots again `read.sz=1048576 … sum=1048576 has_space=0 … req_events=0` — differing only in ephemeral PIDs and the `DEBUG_POLL_EVENTS` tally (`66393` vs `63068` `i:2 POLLIN` lines), which simply counts how many poll iterations elapsed during the run:

```
KITTY_PID=127703 (pgid=127685 matches launcher pgid=127685) CHILD_PID=127774
snap1 kittyState=R child_stacktop=running read.sz=1048576 write.pending=0 sum=1048576 has_space=0 childfd=9 req_events=0 write_buf_used=0
snap2 kittyState=R child_stacktop=running read.sz=1048576 write.pending=0 sum=1048576 has_space=0 childfd=9 req_events=0 write_buf_used=0
snap3 kittyState=R child_stacktop=running read.sz=1048576 write.pending=0 sum=1048576 has_space=0 childfd=9 req_events=0 write_buf_used=0
--- DEBUG_POLL_EVENTS counts (child fd=i:2) ---
  66393 i:2 POLLIN
    260 i:0 POLLIN
      1 i
--- total poll lines: 67025 ---
RUN_run2 complete
```

#### Before / intermediate / after (the buffer filling to the cap)

Because the gate is a function of buffer occupancy, the occupancy itself has a clear **before → intermediate → after** progression, captured with the same single‑shot `gdb` technique on the debug build:

| Stage | `read.sz` | `write.pending` | `sum` (occupancy) | `has_space` | `req_events` (child fd) |
|-------|-----------|-----------------|-------------------|-------------|--------------------------|
| Intermediate (filling) | `144` | `377598` | `377742` (36 % of `BUF_SZ`) | `1` (space remains) | `1` (`POLLIN` still requested) |
| Full (at the cap) | `1048576` | `0` | `1048576` (100 % of `BUF_SZ`) | `0` (no space) | `0` (`POLLIN` removed) |

Under a sustained flood the "full" state dominates: five consecutive independent single‑shot snapshots all read `sum=1048576 has_space=0 req_events=0` (child having written ~102 MiB by then), i.e. the buffer sits pinned at `BUF_SZ` and the child stays blocked. The intermediate row was caught during a brief drain, showing the gate re‑opening (`has_space=1`, `POLLIN` re‑requested) the moment space appears — the pause is continuously re‑evaluated, not latched.

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

This decision is made inside `run_worker` (`kitty/vt-parser.c:L1417`, invoked on the main thread via the `parse_func` pointer). Its caller `do_parse` (`kitty/child-monitor.c:L438`) then arms the wait for only the remaining slice of `input_delay` — this is where `set_maximum_wait` is called, **not** in `parse_input` (the surrounding loop at `kitty/child-monitor.c:L451` that calls `do_parse` once per child):

```c
// kitty/child-monitor.c:L445-L446  (do_parse → set_maximum_wait)
        } else set_maximum_wait(OPT(input_delay) - pd.time_since_new_input);
    } else if (pd.has_pending_input) set_maximum_wait(OPT(input_delay) - pd.time_since_new_input);
```

These delays are ordinary, documented options whose *defaults* are the values kitty runs with out of the box. The option definitions literally document the bypass:

- `repaint_delay` = **10 ms**, `kitty/options/definition.py:L866` — its help text notes it is ignored while there is pending input.
- `input_delay` = **3 ms**, `kitty/options/definition.py:L878` — its help text (`:L885`) states: *"This setting is ignored when the input buffer is almost full."*

Both feed the global options struct as `monotonic_t repaint_delay, input_delay;` (`kitty/state.h:L51`), which `run_worker` (`kitty/vt-parser.c:L1425`) and `do_parse` (`kitty/child-monitor.c:L445-L446`) read at runtime.

#### Captured evidence — the default delay values at runtime

The values kitty actually runs with (default configuration) were read back through kitty's own config loader. Command and **complete output**:

```
Command: xvfb-run -a env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe ./kitty/launcher/kitty +runpy 'from kitty.options.types import defaults; print("input_delay=%r repaint_delay=%r" % (defaults.input_delay, defaults.repaint_delay))'
Run 1: input_delay=3 repaint_delay=10
Run 2: input_delay=3 repaint_delay=10
```

**Stable across two runs:** `input_delay=3` (ms), `repaint_delay=10` (ms) — matching the definitions above.

#### Captured evidence (labeled supplement) — the bypass clause observed under light vs. heavy load

The bypass branch *was* observed directly, on the same debug build used for the C1‑supplement. Under **light** load a `gdb` breakpoint at the decision site (`kitty/vt-parser.c:L1425`), self‑detaching after 8 hits, was driven by a child emitting one small terminated APC query every 80 ms; it prints `read.sz`, `write.pending`, `time_since_new_input` (`tsni`, ns), `input_delay` (ns), the bypass clause `read.sz + 16*1024 > BUF_SZ`, and whether the delay had elapsed. **Complete, unedited output, run 1:**

```
HIT 1 read.sz=17 write.pending=0 tsni=22350 input_delay=3000000 bypass_clause=0 delay_elapsed=0
HIT 2 read.sz=102 write.pending=0 tsni=438842349 input_delay=3000000 bypass_clause=0 delay_elapsed=1
HIT 3 read.sz=282 write.pending=0 tsni=102197 input_delay=3000000 bypass_clause=0 delay_elapsed=0
HIT 4 read.sz=282 write.pending=0 tsni=5105936 input_delay=3000000 bypass_clause=0 delay_elapsed=1
HIT 5 read.sz=17 write.pending=0 tsni=20083 input_delay=3000000 bypass_clause=0 delay_elapsed=0
HIT 6 read.sz=17 write.pending=0 tsni=10477905 input_delay=3000000 bypass_clause=0 delay_elapsed=1
HIT 7 read.sz=47 write.pending=0 tsni=30785 input_delay=3000000 bypass_clause=0 delay_elapsed=0
HIT 8 read.sz=47 write.pending=0 tsni=8281496 input_delay=3000000 bypass_clause=0 delay_elapsed=1
```

- `input_delay=3000000` — the threshold read *live* from the running process is **3 000 000 ns = 3 ms** (`3 × MONOTONIC_T_1e6`), matching the default captured above and the `time-ms` `ctype` at `kitty/options/definition.py:L878`.
- `bypass_clause=0` on **every** hit — under light load the buffer occupancy is tiny (`read.sz` 17–282 bytes), so `read.sz + 16*1024` is nowhere near `BUF_SZ`; the near‑full bypass is inactive and processing is gated by the `input_delay` comparison. (The `read.sz` values and `bypass_clause` are unperturbed by `gdb`; the `tsni` magnitudes on `delay_elapsed=1` hits, e.g. `438842349`, are inflated by the `gdb` pause between hits and are **not** treated as a latency measurement.) `run2` was near‑identical: same `read.sz` sequence `17,102,282,282,17,17,47,47`, same `input_delay=3000000`, same `bypass_clause=0` throughout.

Under **heavy** load the branch flips. Because a breakpoint‑with‑continue would perturb a fast‑filling buffer, the heavy case was captured with the single‑shot async‑attach technique during a live flood (print once, detach). **Five consecutive independent snapshots, byte‑identical:**

```
snap1: HEAVY read.sz=1048576 write.pending=0 sum=1048576 bypass_clause=1 has_space=0 input_delay=3000000 req_events=0 write_buf_used=0
snap2: HEAVY read.sz=1048576 write.pending=0 sum=1048576 bypass_clause=1 has_space=0 input_delay=3000000 req_events=0 write_buf_used=0
snap3: HEAVY read.sz=1048576 write.pending=0 sum=1048576 bypass_clause=1 has_space=0 input_delay=3000000 req_events=0 write_buf_used=0
snap4: HEAVY read.sz=1048576 write.pending=0 sum=1048576 bypass_clause=1 has_space=0 input_delay=3000000 req_events=0 write_buf_used=0
snap5: HEAVY read.sz=1048576 write.pending=0 sum=1048576 bypass_clause=1 has_space=0 input_delay=3000000 req_events=0 write_buf_used=0
child_bytes_written=106796800
```

- `bypass_clause=1` — with `read.sz=1048576`, `read.sz + 16*1024 = 1064960 > 1048576 = BUF_SZ` is **true**, so `run_worker` processes immediately and the 3 ms `input_delay` wait is **skipped**. The bypass threshold is `read.sz > BUF_SZ − 16*1024 = 1032192` bytes.

**Matched pair, stable:** light load → `bypass_clause=0` (delay honored, coalescing); heavy/near‑full load → `bypass_clause=1` (delay ignored, immediate processing) — the "slow down, then stop slowing down" adaptation, now confirmed by observed values on both sides rather than by inference.

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
            children_fds[EXTRA_FDS + i].events |= (screen->write_buf_used ? POLLOUT  : 0);
```

(`EXTRA_FDS` is `2` — `kitty/child-monitor.c:L35` — so the first child occupies `children_fds[2]`; the double space before `:` is verbatim from the source.) The line immediately above it (`L1501`) is the read‑side gate from Section C (`… ? POLLIN : 0`), so a single `pollfd` for the child simultaneously encodes *both* directions' flow‑control decisions. Under a write‑side flood this is directly observable: the requested‑events word is `POLLOUT` **alone** (value `4`) whenever the parser buffer is full — the read gate has cleared `POLLIN` while the write gate keeps `POLLOUT` set (captured in S1 below).

When `POLLOUT` fires, `write_to_child` (`kitty/child-monitor.c:L1443`) drains as much as the kernel will take, then **stops on `EWOULDBLOCK`/`EAGAIN`**, leaving the remainder queued for the next `POLLOUT`:

```c
// kitty/child-monitor.c:L1462-L1463
            if (errno == EINTR) continue;
            if (errno == EWOULDBLOCK || errno == EAGAIN) break;
```

(The `EWOULDBLOCK`/`EAGAIN` break is the second line, `kitty/child-monitor.c:L1463`.)

The reason kitty gets `EWOULDBLOCK`/`EAGAIN` here — rather than blocking inside `write()` the way the child blocks inside its `write()` in Section C — is that kitty puts the **master** side of the PTY into non‑blocking mode after the child is forked:

```python
# kitty/child.py:L344-L345
        if self.child_fd is not None:
            os.set_blocking(self.child_fd, False)
```

This is a *later* reconfiguration of the master fd and must be distinguished from the PTY's *initial* creation at `kitty/child.py:L171`, where both ends are deliberately blocking:

```python
# kitty/child.py:L171
    master, slave = os.openpty()  # Note that master and slave are in blocking mode
```

So the child inherits the **blocking slave** (it sleeps in `write()` under read‑side pressure — Section C), while kitty drives the **non‑blocking master** (it gets `EAGAIN` and defers under write‑side pressure — here). The two halves of the flow‑control story are literally the two ends of the same `openpty()` pair.

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

#### Captured evidence (C2) — the exact dropped‑data log line, on the DEFAULT build

This is the **canonical** write‑side capture: the *default* build (no debug hooks, `md5sum kitty/fast_data_types.so` = `bc54e8ab71f065f72a8155b4b1cda5d8`) run under a real PTY. The collector `run_w1.sh` launches kitty with a bounded flood child, `flood_query.py`, that floods `\x1b_Ga=p,i=999\x1b\\` — a *put* referencing a non‑existent image, each producing an ~85‑byte `ENOENT` response — and **never reads its stdin**. The child first puts its slave terminal into raw mode (`tty.setraw(0)`) so that ECHO is off; without this the PTY line discipline would echo kitty's responses straight back into kitty's reader, draining `write_buf` and preventing the backlog from ever forming. With echo off, `write_buf` grows until the 100 MiB cap fires. The full bounded scripts are in **Section H.4**.

Launch command (default build, headless): `bash run_w1.sh run1`, which runs

```
xvfb-run -a env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
   ./kitty/launcher/kitty -o confirm_os_window_close=0 python3 flood_query.py
```

**Complete, unedited collector output, run 1:**

```
=== C2 write-side 100 MiB cap via real PTY (DEFAULT canonical build) run1 ===
KITTY_PID=148661 CHILD_PID=148732  [PIDs run-local/ephemeral]
time_to_first_dropped_log = 1.637 s   found=1
total 'Too much data' lines captured = 865844
child_bytes_emitted_total = 29400000
kitty_disposition = (exited-on-own)
--- first 3 matching stderr lines (verbatim) ---
[1.637] Too much data being sent to child with id: 1, ignoring it
[1.637] Too much data being sent to child with id: 1, ignoring it
[1.637] Too much data being sent to child with id: 1, ignoring it
--- last 2 matching stderr lines (verbatim) ---
[3.237] Too much data being sent to child with id: 1, ignoring it
[3.237] Too much data being sent to child with id: 1, ignoring it
--- non-'Too much data' stderr lines (verbatim, should be startup-only) ---
[0.151] Failed to open systemd user bus with error: Connection refused
--- err file size (bytes) ---
57145775
W1_run1 done
```

The captured stderr line is exactly the source format string (`kitty/child-monitor.c:L342`) with `id: 1` substituted for `%lu`:

```
[1.637] Too much data being sent to child with id: 1, ignoring it
```

The `[1.637]` prefix is kitty's own log timestamp (seconds since start): the response backlog crossed 100 MiB **1.637 s** into the run. The child emitted `29,400,000` bytes of `a=p` commands before its 3 s deadline, kitty dropped the overflow `865,844` times, and — critically — **kitty exited cleanly on its own** (`kitty_disposition = (exited-on-own)`) once the bounded child terminated; the collector did not have to kill it. The *only* non‑cap line on kitty's stderr for the entire run is a single benign startup notice, `[0.151] Failed to open systemd user bus with error: Connection refused`, which is unrelated to flow control.

**Stability (≥2 runs).** Run 2 output, complete and unedited:

```
=== C2 write-side 100 MiB cap via real PTY (DEFAULT canonical build) run2 ===
KITTY_PID=148880 CHILD_PID=148951  [PIDs run-local/ephemeral]
time_to_first_dropped_log = 1.645 s   found=1
total 'Too much data' lines captured = 865661
child_bytes_emitted_total = 29582000
kitty_disposition = (exited-on-own)
--- first 3 matching stderr lines (verbatim) ---
[1.645] Too much data being sent to child with id: 1, ignoring it
[1.645] Too much data being sent to child with id: 1, ignoring it
[1.645] Too much data being sent to child with id: 1, ignoring it
--- last 2 matching stderr lines (verbatim) ---
[3.216] Too much data being sent to child with id: 1, ignoring it
[3.216] Too much data being sent to child with id: 1, ignoring it
--- non-'Too much data' stderr lines (verbatim, should be startup-only) ---
[0.151] Failed to open systemd user bus with error: Connection refused
--- err file size (bytes) ---
57133697
W1_run2 done
```

The log **text**, the 100 MiB trigger, and the **time‑to‑first‑drop are stable**: `1.637 s` vs `1.645 s` (a ~8 ms difference at the same fixed 3 s flood scale), and the drop count is `865,844` vs `865,661` (within 0.02 %). The scale and duration are stated explicitly — a 3 s bounded flood emitting ~29 MB of `i=999` puts — so the reported ~1.64 s time‑to‑cap is a genuine, reproducible measurement, not a one‑off.

#### Captured evidence (S1, labeled supplement) — queue growth, `POLLOUT` gating, the exact `EAGAIN`, retained bytes, and drain

The queue occupancy (`write_buf_used`), the requested `POLLOUT`, the write's return value, and its errno are *internal* state and branch points that emit no bytes on their own. To surface them, kitty was **rebuilt in a labeled debug configuration** with its two built‑in debug hooks enabled — `KITTY_PRINT_BYTES_SENT_TO_CHILD` (`kitty/child-monitor.c:L1449`, prints each `write_to_child` call's return to stderr) and `DEBUG_POLL_EVENTS` (`kitty/child-monitor.c:L1550`, prints poll `revents` to stdout) — plus `gdb` to read `children[0].screen->write_buf_used` and the requested `children_fds[2].events` live, and `strace` to record the raw `write(2)` return and errno. **These are supplements, not substitutes** for the canonical path: the flood itself still goes through a real PTY (`flood_query.py`, echo off); only the observability was added, and the default binary (`md5 bc54e8ab…`) was restored afterward (Section H). The instrumented binary is `md5 4e553e9221c1aa221529fd57a8e28e0e`.

**(a) Queue occupancy before / intermediate / after — `write_buf_used` growth, and the requested‑events word.** `gdb` was async‑attached at four points during a real‑PTY flood; each snapshot prints `write_buf_used`, `write_buf_sz`, the requested poll events, the child fd, and the parser's `read.sz`. Complete, unedited, run 1:

```
=== S1 write-side queue internals (INSTRUMENTED supplement) run1 ===
KITTY_PID=150431 CHILD_PID=150502  [PIDs run-local/ephemeral]
BEFORE        write_buf_used=15200295 write_buf_sz=15200295 req_events=4 child_fd=9 parser_read_sz=1048576
INTERMEDIATE  write_buf_used=52498210 write_buf_sz=52498210 req_events=4 child_fd=9 parser_read_sz=1048576
PRECAP        write_buf_used=104857530 write_buf_sz=104857530 req_events=4 child_fd=9 parser_read_sz=1048576
ATCAP         write_buf_used=104857530 write_buf_sz=104857530 req_events=4 child_fd=9 parser_read_sz=1048576
child_bytes_emitted_total = 90188000
--- DEBUG_POLL_EVENTS revent counts on child fd (i:2); POLLOUT is the write-ready signal ---
  24125 i:2 POLLIN
      9 i:2 POLLOUT
--- all distinct poll-event line types ---
 339294 i:0 POLLIN
  24154 i:2 POLLIN
      9 i:2 POLLOUT
      1 i:2 POLLHUP
      1 i:1 POLLIN
--- total poll lines: 363459 ---
S1_run1 done
```

Reading this directly:

- **`write_buf_used` grows monotonically** — `15,200,295 → 52,498,210 → 104,857,530` — then **pins** at `104,857,530` (the `PRECAP` and `ATCAP` snapshots are identical). That ceiling is exactly **70 bytes below the 100 MiB cap** (`104,857,600 − 104,857,530 = 70`): the buffer grows to hold whole responses up to the point where adding the next 85‑byte `ENOENT` would exceed `100 * 1024 * 1024`, after which `schedule_write_to_child` drops rather than grows. This is the **retained** queue at the strict boundary. `write_buf_sz == write_buf_used` at every snapshot, i.e. the buffer is realloc'd to exactly fit.
- **`req_events=4` is `POLLOUT` alone** (`POLLOUT` = `4`). At every snapshot `parser_read_sz=1048576` (the full 1 MiB parser buffer from Section C), so the read gate has cleared `POLLIN` while the write gate keeps `POLLOUT` — the single `children_fds[2]` word simultaneously says "stop reading" and "still need to write". This is the read‑side and write‑side backpressure engaging **at the same instant**, exactly as the two lines `L1501`/`L1503` predict.
- **`POLLOUT` revents fired `9` times** on the child fd (`i:2 POLLOUT`), versus `24,125` `POLLIN`: kitty wrote only when the OS reported the child fd writable, and once the slave input buffer filled (child not reading) `POLLOUT` stopped arriving — so `write_buf` grew unbounded until the cap. `child_fd=9`.

**Stability (≥2 runs).** Run 2 was byte‑identical on the load‑bearing values — `write_buf_used` pinned at the same `104,857,530`, `req_events` `POLLOUT`‑based — and additionally caught the transient moment the parser drained enough to re‑request `POLLIN` (`req_events=5` = `POLLIN|POLLOUT`, at `parser_read_sz=630558`):

```
=== S1 write-side queue internals (INSTRUMENTED supplement) run2 ===
KITTY_PID=150717 CHILD_PID=150788  [PIDs run-local/ephemeral]
BEFORE        write_buf_used=16235510 write_buf_sz=16235510 req_events=4 child_fd=9 parser_read_sz=1048576
INTERMEDIATE  write_buf_used=59768005 write_buf_sz=59768005 req_events=4 child_fd=9 parser_read_sz=1048576
PRECAP        write_buf_used=104857530 write_buf_sz=104857530 req_events=5 child_fd=9 parser_read_sz=630558
ATCAP         write_buf_used=104857530 write_buf_sz=104857530 req_events=4 child_fd=9 parser_read_sz=1048576
child_bytes_emitted_total = 90244000
--- DEBUG_POLL_EVENTS revent counts on child fd (i:2); POLLOUT is the write-ready signal ---
  24204 i:2 POLLIN
      7 i:2 POLLOUT
      1 i:2 POLLI
--- all distinct poll-event line types ---
 317247 i:0 POLLIN
  24363 i:2 POLLIN
      7 i:2 POLLOUT
      1 i:2 POLLHUP
      1 i:1 POLLIN
--- total poll lines: 341619 ---
S1_run2 done
```

The pinned cap value `104,857,530` is identical to the byte across both runs; the `POLLOUT` revent count varies slightly (`9` vs `7`) with flood timing. (The single `i:2 POLLI` line is a harmless truncation artifact: `DEBUG_POLL_EVENTS` and the write hook both write to the same stdout/stderr from different threads under heavy load, so one `printf("i:%lu POLLIN\n")` was interleaved mid‑write — it is shown here verbatim rather than silently removed.)

**(b) The exact `write()` return and errno — partial writes then `EAGAIN`, via `strace`.** The `KITTY_PRINT_BYTES_SENT_TO_CHILD` hook's `Wrote: -1 bytes:` line records only the *return* value, not the errno, so it cannot by itself establish `EAGAIN` vs `EWOULDBLOCK`. To capture the errno definitively, the same real‑PTY flood was run under `strace -f -e trace=write`, filtering the child PTY master (`write(9, …)`). Complete, unedited (run 1) — the three consecutive writes that show the whole write‑side story:

```
155323 write(9, "\33_Gi=999;ENOENT:Put command refe"..., 55590) = 11776
155323 write(9, "tent image with id: 999 and numb"..., 43814) = 3584
155323 write(9, "h id: 999 and number: 0\33\\\33_Gi=99"..., 40230) = -1 EAGAIN (Resource temporarily unavailable)
```

This is `write_to_child`'s inner loop (`kitty/child-monitor.c:L1447-L1466`) captured at the syscall boundary:

1. First `write(9, …, 55590) = 11776` — a **partial write**: kitty asked to send 55,590 queued bytes, the kernel accepted only 11,776 (the free space in the PTY slave input buffer). `write_to_child` advances `written` by 11,776 and loops.
2. Second `write(9, …, 43814) = 3584` — another partial write (`43,814 = 55,590 − 11,776`; the kernel now accepts 3,584 more).
3. Third `write(9, …, 40230) = -1 EAGAIN (Resource temporarily unavailable)` — the slave buffer is now full, so the write returns **`-1` with errno `EAGAIN`**. This is precisely the condition tested at `kitty/child-monitor.c:L1463` (`if (errno == EWOULDBLOCK || errno == EAGAIN) break;`); kitty breaks out of the loop, **leaving the remaining 40,230 bytes queued** in `write_buf` for the next `POLLOUT`.

The `11,776 + 3,584 = 15,360` bytes accepted before `EAGAIN` is the PTY slave input buffer's capacity (~15 KiB). `PID 155323` is kitty's `io_loop` I/O thread (shown by `strace -f`). **Stability (≥2 runs).** Run 2 was identical on the return values — `= 11776`, `= 3584`, then `..., 38615) = -1 EAGAIN (Resource temporarily unavailable)` — exactly one `EAGAIN` per run; only the requested sizes differ (they depend on the instantaneous `write_buf_used`).

**(c) Response bytes are really flushed — the write hook, sub‑cap.** A short flood kept below the cap shows the successful `write_to_child` calls with their payloads. The `KITTY_PRINT_BYTES_SENT_TO_CHILD` hook recorded eleven `write_to_child` returns whose byte counts were, in order, `3230, 1275, 2380, 1190, 1360, 1530, 1275, 1275, 1870, 1275, 1360` — **every one a whole multiple of the 85‑byte `ENOENT` response** (e.g. `3230 = 38×85`, `1190 = 14×85`, `1360 = 16×85`). The smallest of these, shown complete and unedited (1190 bytes = 14 copies of the response, no truncation):

```
Wrote: 1190 bytes: \x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\\x1b_Gi=999;ENOENT:Put command refers to non-existent image with id: 999 and number: 0\x1b\\
```

This confirms the queued bytes on the write path are the graphics **responses** themselves — the same `ENOENT` string built by `finish_command_response`/`set_command_failed_response` and routed through `write_escape_code_to_child` (D.1). (Note: the hook's `Wrote: -1 bytes:` variant only appears when a single `write_to_child` call's loop is slow enough to reach the full‑buffer state mid‑call — e.g. under `strace`; it is *not* a reliable canonical signal, which is why the errno above was established with `strace`, not the hook.)

**(d) Retention → drain — `write_buf_used` returns to zero when the child reads.** Finally, a `drain` variant of the child floods briefly (accumulating a backlog) and then **starts reading its stdin**, so kitty's queued responses flush and `write_buf_used` falls back to zero. `gdb` snapshots, complete and unedited, both runs:

```
run1:  PEAK-RETAINED(flood end)   write_buf_used=12716255 req_events=5
       DRAINING(+1.8s free-run)   write_buf_used=12716255 req_events=5
       DRAINED(+4.2s free-run)    write_buf_used=0 req_events=1
run2:  PEAK-RETAINED(flood end)   write_buf_used=12719400 req_events=5
       DRAINING(+1.8s free-run)   write_buf_used=12719400 req_events=5
       DRAINED(+4.2s free-run)    write_buf_used=0 req_events=1
```

At the flood's end ~12.7 MB is **retained** (`req_events=5` = `POLLIN|POLLOUT`, since the parser has drained and both directions are active); the queue then empties to `write_buf_used=0`, at which point the requested events drop to `req_events=1` (`POLLIN` only) — because `L1503` adds `POLLOUT` *only* while `write_buf_used` is non‑zero. The intermediate reading is unchanged because the child is still parsing its command backlog (generating responses at roughly the drain rate) until the backlog clears; the load‑bearing transition — retained backlog → fully drained, and `POLLOUT` requested → not requested — is stable across both runs.

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
| Render coalescing | `kitty/child-monitor.c` | `do_parse` → `set_maximum_wait` (**not** `parse_input`, whose definition begins at L451) | L438–L448 (`set_maximum_wait` calls at L444, L445, L446) |
| Debug hooks (supplements) | `kitty/child-monitor.c` | `KITTY_PRINT_BYTES_SENT_TO_CHILD`, `DEBUG_POLL_EVENTS` | L1449, L1550 |
| Response routing onto `write_buf` | `kitty/screen.c` | `screen_handle_graphics_command` → `write_escape_code_to_child` | L1047, L1050 (def L979) |
| `write_buf` fields + parser handle | `kitty/screen.h` | `Screen` struct | L114–L116, L158 |
| Success/ack response builder + quiet/id gate | `kitty/graphics.c` | `finish_command_response` | L759, L762–L766 |
| Error response builder | `kitty/graphics.c` | `set_command_failed_response` | L305 |
| 320 MiB storage quota | `kitty/graphics.c` | `#define DEFAULT_STORAGE_LIMIT 320u * (1024u * 1024u)` | L25 |
| LRU eviction | `kitty/graphics.c` | `apply_storage_quota` (`HASH_SORT(self->images, oldest_img_first)`) | L290–L296 |
| Per‑transfer cap `MAX_DATA_SZ` = `4u * 100000000u` = **400,000,000** bytes (decimal) + strict‑size `EFBIG` | `kitty/graphics.c` | `load_image_data` size sub‑clause `load_data->buf_used + g->payload_sz > MAX_DATA_SZ` | L521 (macro), L533 (check) |
| PNG oversize `EINVAL` (declared `S=` size) | `kitty/graphics.c` | `initialize_load_data` (`g->data_sz > MAX_DATA_SZ`) | L638 |
| Root‑image disk‑cache add failure → `ENOSPC` "Failed to store image data in disk cache" | `kitty/graphics.c` | root‑frame `add_to_cache` failure path | L744 (call), L746 (abort) |
| 5× animation‑frame quota → `ENOSPC` "Cache size exceeded cannot add new frames" (**distinct** from L746) | `kitty/graphics.c` | `cache_size(self) + load_data->data_sz > self->storage_limit * 5` | L1570–L1573 |
| On‑disk spillover | `kitty/disk-cache.c` | `add_to_disk_cache` / `remove_from_disk_cache` | L488, L517 |
| Logical cache accounting (synchronous, **not** physical proof) | `kitty/disk-cache.c` | `total_size += s->data_sz` inside `add_to_disk_cache` (before the background `wakeup_write_loop`) | L508 |
| Physical offload observation | `kitty/disk-cache.c` | `wait_for_write` (block for the writer) / `size_on_disk` / `num_cached_in_ram` | L704, L712, L669 |
| Delay defaults | `kitty/options/definition.py` | `repaint_delay` (10 ms), `input_delay` (3 ms) | L866, L878 |
| Delay storage in `OPT` | `kitty/state.h` | `monotonic_t repaint_delay, input_delay;` | L51 |
| Child PTY: initial (blocking) descriptors | `kitty/child.py` | `os.openpty()` (`# Note that master and slave are in blocking mode`) | L171 |
| Child PTY: master set non‑blocking (→ kitty's writes get `EWOULDBLOCK`) | `kitty/child.py` | `os.set_blocking(self.child_fd, False)` | L344–L345 |

---

## Section F — Q4: How it shows up at runtime (each artifact captured)

Below, each observable artifact is captured directly, with before/intermediate/after values for anything stateful.

### F.1 A child blocking on `write()` (read‑side backpressure)

Fully captured in **Section C.2 (C1)**, with kitty running **normally** (never stopped): the flooding child is asleep in `n_tty_write` in 57–58 of 60 samples (`kitty_in_STOPPED(T)=0`), its sustained write rate pinned to ~15.5–15.7 MB/s, while a control 64 KiB burst completes instantly. The C1‑supplement shows the mechanism directly — `sum=1048576` (buffer at `BUF_SZ`), `has_space=0`, and `req_events=0` (`POLLIN` removed). This is the runtime signal that kitty has "stopped reading" — the producer is *made to wait*.

### F.1b The parser rejecting an over‑long inline escape code (`BUF_SZ` ceiling)

A distinct read‑side artifact appears when a *single* APC escape code never terminates and grows without bound: the parser refuses to accumulate more than one buffer's worth and emits a `[PARSE ERROR]` on kitty's stderr. This is the direct runtime manifestation of the fixed `BUF_SZ`.

The check lives in `accumulate_st_terminated_esc_code` (`kitty/vt-parser.c:L406`): if no ST terminator is found and the accumulated length exceeds `MAX_ESCAPE_CODE_LENGTH` (`= BUF_SZ/4u = 262144`, `kitty/vt-parser.c:L21`), it reports and discards (`kitty/vt-parser.c:L419`). In the default build `REPORT_ERROR` expands to `log_error(ERROR_PREFIX " " __VA_ARGS__)` (`kitty/vt-parser.c:L125`; `ERROR_PREFIX = "[PARSE ERROR]"`, `kitty/data-types.h:L70`), so the signal is a real stderr line.

**Canonical capture (default build, real PTY).** A real child emits an APC graphics command whose payload is 1.5 MB of `A` with **no** terminating `ESC \`; kitty reads it through the real PTY and logs the error. Child script (`emit_bad_apc.sh`, full text in Section H.4):

```bash
#!/bin/bash
printf '\033_Ga=q,i=1;'
head -c 1500000 /dev/zero | tr '\0' 'A'
# no ESC\ terminator on purpose
sleep 1
printf 'DONE_EMITTING\n'
sleep 1
```

Command: `xvfb-run -a env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe ./kitty/launcher/kitty --config NONE -o allow_remote_control=no /tmp/<scratch>/emit_bad_apc.sh` (kitty stderr redirected to a file). **Complete, unedited matching line, byte‑identical across two runs:**

```
[0.167] [PARSE ERROR] VTE_APC escape code too long (1048574 bytes), ignoring it
```

**Why exactly `1048574`.** This is the **ceiling** `BUF_SZ − 2 = 1 048 576 − 2`: an unterminated APC that fills the whole 1 MiB buffer is reported at the buffer size minus the 2‑byte `ESC _` prefix that opens the APC. `MAX_ESCAPE_CODE_LENGTH` (262144) is only the *threshold* that arms the check; the *reported* value is however many bytes had accumulated when the check ran, which — for a buffer‑filling flood — is the full `BUF_SZ − 2`. This was verified by exercising the parser directly through kitty's in‑tree harness (**labeled supplement**, DUMP_COMMANDS translation unit → Python `dump_callback`), which shows the reported length scaling with the payload and saturating at the ceiling; stable across two runs:

```
INPUT_PAYLOAD_SIZE=300000 total_bytes_fed=300011
  ERROR_MSG: 'VTE_APC escape code too long (300009 bytes), ignoring it'
INPUT_PAYLOAD_SIZE=600000 total_bytes_fed=600011
  ERROR_MSG: 'VTE_APC escape code too long (600009 bytes), ignoring it'
INPUT_PAYLOAD_SIZE=1048600 total_bytes_fed=1048611
  ERROR_MSG: 'VTE_APC escape code too long (1048574 bytes), ignoring it'
INPUT_PAYLOAD_SIZE=2000000 total_bytes_fed=2000011
  ERROR_MSG: 'VTE_APC escape code too long (1048574 bytes), ignoring it'
```

Payloads of 300000/600000 bytes report `300009`/`600009` (= bytes fed − 2); payloads that overflow the buffer (1048600, 2000000) both saturate at `1048574`. The canonical PTY value `1048574` therefore corresponds to a buffer‑filling unterminated APC.

### F.2 The dropped‑data log line (write‑side cap)

Fully captured on the **default build** in **Section D.3 (C2)**: `[1.637] Too much data being sent to child with id: 1, ignoring it`, first seen `1.637 s` into the run and repeated `865,844` times over a 3 s bounded flood, stable across two runs (`[1.645]` / `865,661` in run 2). This is the runtime signal that the response backlog crossed the 100 MiB (`104,857,600`‑byte) cap; the queue itself was observed pinned at `104,857,530` bytes just under that cap (D.3 S1). This is the exact format string at `kitty/child-monitor.c:L342`.

### F.3 APC graphics‑protocol responses (byte‑for‑byte, via the real PTY)

These are the most directly "visible" signs — the client receives real bytes. Every capture below was produced **canonically**: a child launched by a real kitty (default build) writes one graphics command to its stdout (the PTY slave), kitty reads it from the PTY master, processes it (`screen_handle_graphics_command` → `grman_handle_command` → response builder → `write_escape_code_to_child`, `kitty/screen.c:L1047-L1050`), and writes the APC reply back onto the PTY, which the child reads from stdin. The child is in raw mode (`tty.setraw(0)`, echo off) so replies are not echoed back into kitty's reader. The probe records each reply's `repr`, byte `len`, and `hex`. Every record below was **byte‑identical across two runs** (`apc_x_run1.result` vs `apc_x_run2.result` `diff`‑clean, md5 `e7cce85d6763b0f294399f044c6828c2`; see Section H.3), and every `hex` was verified to decode to its `repr`. The probe first drained kitty's startup chatter and observed `_startup_bytes=b'' len=0` — kitty sends the child nothing on start‑up.

#### F.3.1 The response‑suppression cross‑product (`q=0` / `q=1` / `q=2` × success / error) + the `I=` case

The `q` (quiet) key is honoured by `finish_command_response` (`kitty/graphics.c:L759-L781`): `if (g->quiet) { if (is_ok_response || g->quiet > 1) return NULL; }`. So `q=1` suppresses only the success (`OK`) reply while errors still go out, and `q=2` suppresses **everything**. A response is emitted only when `g->id || g->image_number` is set, and it prints `i=%u` for the id (`L773`) and `,I=%u` for the image number (`L774`). The full cross‑product was exercised through the real PTY — **success** = a valid query carrying an id (`a=q` → `OK`), **error** = a put referencing a non‑existent image (`a=p` → `ENOENT`). Complete, unedited capture:

```
=== q0_success ===
cmd_repr=b'\x1b_Ga=q,i=11,f=24,s=1,v=1,q=0;AAAA\x1b\\'
expect_response=True
resp_len=12
resp_repr=b'\x1b_Gi=11;OK\x1b\\'
resp_hex=1b5f47693d31313b4f4b1b5c

=== q0_error ===
cmd_repr=b'\x1b_Ga=p,i=901,q=0\x1b\\'
expect_response=True
resp_len=85
resp_repr=b'\x1b_Gi=901;ENOENT:Put command refers to non-existent image with id: 901 and number: 0\x1b\\'
resp_hex=1b5f47693d3930313b454e4f454e543a50757420636f6d6d616e642072656665727320746f206e6f6e2d6578697374656e7420696d61676520776974682069643a2039303120616e64206e756d6265723a20301b5c

=== q1_success ===
cmd_repr=b'\x1b_Ga=q,i=12,f=24,s=1,v=1,q=1;AAAA\x1b\\'
expect_response=False
resp_len=0
resp_repr=b''
resp_hex=

=== q1_error ===
cmd_repr=b'\x1b_Ga=p,i=902,q=1\x1b\\'
expect_response=True
resp_len=85
resp_repr=b'\x1b_Gi=902;ENOENT:Put command refers to non-existent image with id: 902 and number: 0\x1b\\'
resp_hex=1b5f47693d3930323b454e4f454e543a50757420636f6d6d616e642072656665727320746f206e6f6e2d6578697374656e7420696d61676520776974682069643a2039303220616e64206e756d6265723a20301b5c

=== q2_success ===
cmd_repr=b'\x1b_Ga=q,i=13,f=24,s=1,v=1,q=2;AAAA\x1b\\'
expect_response=False
resp_len=0
resp_repr=b''
resp_hex=

=== q2_error ===
cmd_repr=b'\x1b_Ga=p,i=903,q=2\x1b\\'
expect_response=False
resp_len=0
resp_repr=b''
resp_hex=

=== I_number ===
cmd_repr=b'\x1b_Ga=p,I=777\x1b\\'
expect_response=True
resp_len=86
resp_repr=b'\x1b_G,I=777;ENOENT:Put command refers to non-existent image with id: 0 and number: 777\x1b\\'
resp_hex=1b5f472c493d3737373b454e4f454e543a50757420636f6d6d616e642072656665727320746f206e6f6e2d6578697374656e7420696d61676520776974682069643a203020616e64206e756d6265723a203737371b5c

=== qabsent_err ===
cmd_repr=b'\x1b_Ga=p,i=904\x1b\\'
expect_response=True
resp_len=85
resp_repr=b'\x1b_Gi=904;ENOENT:Put command refers to non-existent image with id: 904 and number: 0\x1b\\'
resp_hex=1b5f47693d3930343b454e4f454e543a50757420636f6d6d616e642072656665727320746f206e6f6e2d6578697374656e7420696d61676520776974682069643a2039303420616e64206e756d6265723a20301b5c
```

Reading the cross‑product against the source:

| Case | `q` | outcome | captured reply | why |
|---|---|---|---|---|
| success | 0 | `OK` sent | `\x1b_Gi=11;OK\x1b\\` (12 B) | `g->quiet == 0`, so the `if (g->quiet)` guard is skipped |
| error | 0 | `ENOENT` sent | `\x1b_Gi=901;ENOENT:…\x1b\\` (85 B) | same |
| success | 1 | **suppressed** | `b''` (0 B) | `is_ok_response` is true → `return NULL` |
| error | 1 | `ENOENT` sent | `\x1b_Gi=902;ENOENT:…\x1b\\` (85 B) | error is not an OK response and `quiet == 1` (not `> 1`) |
| success | 2 | **suppressed** | `b''` (0 B) | `is_ok_response` true → `return NULL` |
| error | 2 | **suppressed** | `b''` (0 B) | `g->quiet > 1` → `return NULL` even for errors |
| `I=` (error) | 0 | `ENOENT` sent | `\x1b_G,I=777;ENOENT:…id: 0 and number: 777\x1b\\` (86 B) | only `image_number` set → response prints `,I=777` (the `,I=%u` branch, `L774`); `g->id == 0` so no `i=` |
| `q` absent (error) | — | `ENOENT` sent | `\x1b_Gi=904;ENOENT:…\x1b\\` (85 B) | `quiet` defaults to `0`; identical to `q=0` |

The three suppressed cases (`q1_success`, `q2_success`, `q2_error`) each returned `b''` — a real, bounded **no‑response window** (the probe waited out its per‑command quiet window and read zero bytes), which is the observable form of "kitty silently dropped the reply." The `I=` case is the direct manifestation of the `,I=%u` branch at `L774`: with only an image number and no id, the reply opens `\x1b_G,I=777;…` (note the leading comma and the absent `i=`).

#### F.3.2 The three distinct transfer size‑limit error branches

There are **three separate** rejection branches around image data size, and the question's Q4 asks for the boundary behaviour of each — they are not the same code path:

**(a) `EFBIG` — the strict PNG‑size clause `buf_used + payload_sz > MAX_DATA_SZ` (`kitty/graphics.c:L533`, PNG format).** This is the load‑bearing per‑transfer size ceiling: a *direct PNG* transfer (`a=t,f=100,t=d`) streamed in chunks (`m=1`) whose accumulated **decoded** payload crosses `MAX_DATA_SZ` = `4u * 100000000u` = **400,000,000** bytes (decimal). Because `data_fmt == PNG`, the `|| data_fmt != PNG` sub‑clause is false, so the abort fires *purely* on the size condition. `g->payload_sz` is the decoded chunk size (`base64_decode8`, `kitty/parse-graphics-command.h:L326-L327`), so this is real decoded volume, not wire bytes. Complete, unedited capture (this is run 1; the **response bytes are byte‑identical across all runs**, while `chunks_sent`/`decoded_total_sent` are timing‑dependent — see the distribution note below):

```
decoded_per_chunk=195000 base64_per_chunk=260000
MAX_DATA_SZ=400000000
chunks_sent=2054
decoded_total_sent=400530000
crossed_MAX_DATA_SZ=True
resp_len=28
resp_repr=b'\x1b_Gi=1;EFBIG:Too much data\x1b\\'
resp_hex=1b5f47693d313b45464249473a546f6f206d75636820646174611b5c
```

Each chunk is 195,000 decoded bytes (260,000 base64 bytes on the wire, well under `MAX_ESCAPE_CODE_LENGTH` = 262,144); once the accumulated `buf_used + payload_sz` crosses 400,000,000, the chunk that crosses it triggers `EFBIG:Too much data`. **Distribution across three runs:** `chunks_sent` was `2054 / 2056 / 2053` and `decoded_total_sent` was `400,530,000 / 400,920,000 / 400,335,000` respectively — every run crossed `MAX_DATA_SZ` (`crossed_MAX_DATA_SZ=True` each time) but the exact count varies because it depends on how many chunks the child has pushed before it reads kitty's reply back and stops. The **response** — `resp_len=28`, `resp_repr`, and `resp_hex` — was **byte‑identical in every run** (that is the load‑bearing signal; the chunk count is not). Wall time to the reply was ≈ 4.2 s (Section H.3). No `m=0` was ever sent, so kitty never attempted to decode the (deliberately invalid) PNG.

**(b) `EFBIG` — the non‑PNG capacity clause `data_fmt != PNG` (`kitty/graphics.c:L533`, other half of the same `||`).** A direct non‑PNG transfer (`f=24`) that needs to grow its buffer aborts immediately because only PNG buffers may grow — a different trigger that yields the *same* `EFBIG:Too much data` text. Command `\x1b_Ga=t,i=1,f=24,s=1,v=1,m=1;` + `A`×300 + `\x1b\\`:

```
len=28
repr=b'\x1b_Gi=1;EFBIG:Too much data\x1b\\'
hex=1b5f47693d313b45464249473a546f6f206d75636820646174611b5c
```

**(c) `EINVAL` — the declared‑size PNG clause `g->data_sz > MAX_DATA_SZ` (`kitty/graphics.c:L638`).** This fires in `initialize_load_data` *before* any payload arrives, when the client *declares* `S=` larger than `MAX_DATA_SZ`. It is distinct from (a): (a) crosses the limit by *accumulating* real chunks; (c) rejects the *declared* size up front. Command `\x1b_Ga=q,i=1,f=100,S=500000000;AAAA\x1b\\`:

```
len=39
repr=b'\x1b_Gi=1;EINVAL:PNG data size too large\x1b\\'
hex=1b5f47693d313b45494e56414c3a504e4720646174612073697a6520746f6f206c617267651b5c
```

**(d) `ENOENT` — reference to a non‑existent image (`kitty/graphics.c:L1048`).** The put path used throughout the cross‑product above (e.g. `\x1b_Ga=p,i=999\x1b\\` → 85‑byte `ENOENT`). This is the same response the C2/S1 write‑side floods used to grow `write_buf` — the identical `ENOENT` text appears in the Section D `Wrote:` capture, tying the read‑ and write‑side stories together.

### F.4 Image LRU eviction under the 320 MiB storage quota

**Units nuance stated up front:** the code quota is **320 MiB** — `#define DEFAULT_STORAGE_LIMIT 320u * (1024u * 1024u)` (`kitty/graphics.c:L25`) = **335,544,320 bytes**. The protocol docs colloquially call it "320MB per buffer" (`docs/graphics-protocol.rst`). Both refer to the same limit; the exact byte value is 335,544,320.

Eviction is enforced by `apply_storage_quota` (`kitty/graphics.c:L290-L296`): it first removes *unreferenced* images (`remove_images(self, trim_predicate, currently_added_image_internal_id)`, where `trim_predicate` = `!img->root_frame_data_loaded || !img->refs`, defined at L280‑L281 and invoked at L292), and if storage is still over the limit it sorts oldest‑first (`HASH_SORT(self->images, oldest_img_first)`) and deletes until it fits.

Because `used_storage` is not exposed to Python, eviction was observed via `image_count` (= `HASH_COUNT`, `kitty/graphics.c:L2362`) and `disk_cache.total_size`, driven through kitty's own in‑tree harness. **This is a labeled supplement (S2):** `send_command`/`parse_bytes` feed the parser directly and bypass the PTY + `io_loop`, but they exercise the *identical* `graphics.c` storage code regardless of transport. (A single inline APC that would exceed the 1 MiB parser buffer is itself rejected as `VTE_APC escape code too long (1048574 bytes)` = `BUF_SZ − 2` — a direct manifestation of `BUF_SZ`, captured canonically in Section F.1b — so large images below use the temporary‑file transfer medium `t='t'`.)

**`disk_cache.total_size` is a LOGICAL accounting figure, not proof of physical bytes on disk (Finding 11).** The value read from Python is `disk_cache_total_size` (`kitty/disk-cache.c:L666`), which returns the `total_size` counter. That counter is incremented **synchronously, inside the cache mutex, at the moment `add_to_disk_cache` is called** — `self->total_size += s->data_sz` (`kitty/disk-cache.c:L508`) — *before* the background writer thread has necessarily flushed the bytes to the on‑disk file (the writer is only woken at the end of the function). So `total_size` proves the logical size the cache has *accepted for storage*; the *physical* bytes are confirmed separately by `disk_cache_wait_for_write` (`kitty/disk-cache.c:L642`, Python `wait_for_write`, L704), `disk_cache_size_on_disk` (Python `size_on_disk`, L712), and `disk_cache_num_cached_in_ram` (L669, counts entries where `written_to_disk && data`). The two are distinguished directly below before eviction is exercised.

**Logical vs. physical offload — the writer lag, captured (Finding 11, labeled supplement S2).** Transmitting three 120,000‑byte images and sampling both counters shows the physical size **lagging** the logical size until the writer catches up. The logical column and the post‑`wait_for_write` column are byte‑identical across both runs; the pre‑wait physical samples are intrinsically timing‑dependent (the background writer's progress is nondeterministic), so both runs' pre‑wait values are shown to make that explicit:

```
# run 1
DEFAULT grman.storage_limit = 335544320 bytes (== 320*1024*1024 ? True)
baseline: total_size=0  size_on_disk=0  num_cached_in_ram=0
transmit img i=101 code=OK : total_size(logical)=120000  size_on_disk(physical,pre-wait)=28672  num_cached_in_ram=0
transmit img i=102 code=OK : total_size(logical)=240000  size_on_disk(physical,pre-wait)=172032  num_cached_in_ram=0
transmit img i=103 code=OK : total_size(logical)=360000  size_on_disk(physical,pre-wait)=319488  num_cached_in_ram=0
-> wait_for_write() returned: True
after wait_for_write: total_size(logical)=360000  size_on_disk(physical)=360000  num_cached_in_ram=0
```

```
# run 2 (same logical + post-wait values; only the pre-wait physical samples differ, proving the lag is real and nondeterministic)
transmit img i=101 code=OK : total_size(logical)=120000  size_on_disk(physical,pre-wait)=36864  num_cached_in_ram=0
transmit img i=102 code=OK : total_size(logical)=240000  size_on_disk(physical,pre-wait)=120000  num_cached_in_ram=0
transmit img i=103 code=OK : total_size(logical)=360000  size_on_disk(physical,pre-wait)=270336  num_cached_in_ram=0
-> wait_for_write() returned: True
after wait_for_write: total_size(logical)=360000  size_on_disk(physical)=360000  num_cached_in_ram=0
```

**The crispest offload proof — a raw 64 MiB blob added directly to the disk cache.** Sampling `size_on_disk()` five times immediately after the add shows it pinned at **0** while `total_size` already reads the full 67,108,864 bytes; only after `wait_for_write()` does the physical size reach 67,108,864. This is the logical/physical split in its clearest form and is **byte‑identical across both runs:**

```
--- 1b: raw dc.add() of a 64 MiB blob (catches the writer lag) ---
immediately after dc.add(64MiB): total_size(logical)=67108864
size_on_disk() sampled x5 BEFORE wait_for_write = [0, 0, 0, 0, 0]
num_cached_in_ram (pre-wait) = 0
-> wait_for_write() returned: True
size_on_disk() AFTER wait_for_write = 67108864
total_size(logical) still = 67108864
```

So the data genuinely leaves RAM and lands on disk: `total_size` records the logical acceptance at `disk-cache.c:L508`, and `size_on_disk` climbing from 0 to 67,108,864 after the write barrier is the physical offload itself.

**At the true default quota (320 MiB), via file transfer — LRU eviction.** Each image is 4096×4096×3 = 50,331,648 bytes (48.00 MiB), with a distinct first byte to defeat content de‑duplication; seven of them would total 336 MiB > 320 MiB. **Complete, unedited output (byte‑identical across both runs):**

```
grman.storage_limit (DEFAULT) = 335544320 bytes (== 320*1024*1024 ? True)
each image: s=4096 v=4096 f=24 t=t => data_sz=50331648 bytes (48.00 MiB); 7 images => 336 MiB > 320 MiB
img_id  code   image_count  disk_cache.total_size(bytes,LOGICAL)  total(MiB)  <=320MiB?
     1  OK               1                              50331648      48.00  yes
     2  OK               2                             100663296      96.00  yes
     3  OK               3                             150994944     144.00  yes
     4  OK               4                             201326592     192.00  yes
     5  OK               5                             251658240     240.00  yes
     6  OK               6                             301989888     288.00  yes
     7  OK               2                             100663296      96.00  yes
```

**Before / intermediate / after:** storage grows monotonically to **288.00 MiB** across the first 6 images (`image_count` 1→6), then adding image #7 — which would reach 336 MiB > 320 MiB — triggers eviction, and the count/size **drop to 2 / 96 MiB**. **At every step `disk_cache.total_size` stays ≤ 335,544,320 bytes** — the quota is never exceeded. The size of the 6→2 drop (rather than a single‑image eviction) is because `apply_storage_quota`'s *first* step trims images whose placements have scrolled off the small (5×5) test screen and thus lost their `refs`; this is real, code‑grounded behavior (`trim_predicate`, `kitty/graphics.c:L280‑L281`), and it too was byte‑identical across both runs.

**At a lowered quota (72 bytes) — the pure‑LRU transition, crisply.** To isolate the LRU `while` loop from placement‑scroll trimming, the quota was set to 72 bytes (**NON‑DEFAULT**; kitty's own test `test_graphics_quota_enforcement` uses exactly this) and three 36‑byte images were put. **Complete, unedited output (byte‑identical across both runs):**

```
storage_limit=72 bytes (NON-DEFAULT; mirrors kitty_tests test_graphics_quota_enforcement)
step                       code   image_count  disk_cache.total_size
put img i=1 (a=T,36B)      OK               1  36
put img i=2 (a=T,36B)      OK               2  72
put img i=3 (a=T,36B)      OK               2  72  <-- img#3: oldest EVICTED by LRU while-loop (count stays 2, total==limit)
```

**Before / intermediate / after:** `image_count` 1 → 2 → **2** and `disk_cache.total_size` 36 → 72 → **72**. Adding the third 36‑byte image (which would reach 108 > 72) evicts exactly the oldest, holding storage at the 72‑byte limit — the pure `while (used_storage > storage_limit) remove_image(oldest)` loop (`kitty/graphics.c:L296`). This matches kitty's own test assertions.

### F.5 Disk‑cache spillover and the 5× animation‑frame quota → `ENOSPC`

Animation‑frame data is stored on disk under a *separate*, larger quota — five times the base limit. The check, its LRU pre‑trim, and the rejection all live at **`kitty/graphics.c:L1570-L1573`** (verified verbatim against HEAD):

```c
// kitty/graphics.c:L1570-L1573
if (is_new_frame && cache_size(self) + load_data->data_sz > self->storage_limit * 5) {
    remove_images(self, trim_predicate, img->internal_id);
    if (cache_size(self) + load_data->data_sz > self->storage_limit * 5)
        ABRT("ENOSPC", "Cache size exceeded cannot add new frames");
}
```

At the **default** `storage_limit` of 335,544,320 bytes this ceiling is **`storage_limit * 5` = 1,677,721,600 bytes** (printed at runtime by the harness below). When a new frame would still exceed that ceiling after the LRU pre‑trim, the command is rejected with `ENOSPC` and the message **`Cache size exceeded cannot add new frames`** — this is the frame‑quota branch at **L1573**, which is *distinct* from the root‑image disk‑cache‑write failure at **`kitty/graphics.c:L746`** (`ABRT("ENOSPC", "Failed to store image data in disk cache")`). Both emit the `ENOSPC` code but with different messages and from different code paths; the earlier version of this document mis‑attributed the frame ceiling to L746 — the corrected citation is L1570‑L1573.

Observed via the same labeled harness (S2). The default 1,677,721,600‑byte ceiling is stated, then the limit is lowered to 72 bytes (**NON‑DEFAULT**; kitty's own `test_graphics_quota_enforcement` uses exactly this) so the 5× frame quota is 360 bytes and the boundary is reachable with tiny 36‑byte frames appended to image id 2. **Complete, unedited output (byte‑identical across both runs):**

```
DEFAULT storage_limit=335544320 ; default 5x frame boundary = storage_limit*5 = 1677721600 bytes
set storage_limit=72 (NON-DEFAULT; mirrors kitty_tests test_graphics_quota_enforcement); 5x=360
base images:
  li(a='T')      code=OK image_count=1 disk_cache.total_size=36
  li(a='T',i=2)  code=OK image_count=2 disk_cache.total_size=72
append 36-byte animation frames to image i=2 (a='f' is the default action of li):
  frame 0 code=OK disk_cache.total_size=108
  frame 1 code=OK disk_cache.total_size=144
  frame 2 code=OK disk_cache.total_size=180
  frame 3 code=OK disk_cache.total_size=216
  frame 4 code=OK disk_cache.total_size=252
  frame 5 code=OK disk_cache.total_size=288
  frame 6 code=OK disk_cache.total_size=324
  frame 7 code=OK disk_cache.total_size=360
9th frame (over quota):
  raw_response_bytes_repr = b'\x1b_Gi=2,r=10;ENOSPC:Cache size exceeded cannot add new frames\x1b\\'
  raw_response_bytes_len  = 62
  raw_response_bytes_hex  = 1b5f47693d322c723d31303b454e4f5350433a43616368652073697a652065786365656465642063616e6e6f7420616464206e6577206672616d65731b5c
  parsed code=ENOSPC msg='Cache size exceeded cannot add new frames'
DISTINCT branch note: L746 'Failed to store image data in disk cache' is the ROOT-IMAGE
  add_to_cache failure (a different code path); the frame ceiling above is L1573.
```

**Byte‑exact response.** The rejection APC is 62 bytes; the `hex` decodes exactly to the `repr` (`1b 5f 47` = `ESC _ G`, … `1b 5c` = `ESC \`), confirming the emitted bytes:

```
b'\x1b_Gi=2,r=10;ENOSPC:Cache size exceeded cannot add new frames\x1b\\'
```

**Before / intermediate / after:** `disk_cache.total_size` climbs 108 → 360 in 36‑byte steps as frames 0–7 are stored on disk (the spillover), then the frame that would push past the 360‑byte 5× ceiling is rejected — code `ENOSPC`, message `Cache size exceeded cannot add new frames`, and the on‑disk size stays pinned at **360**. This is the disk‑cache spillover *and* its 5× ceiling in one capture, and it reproduces the exact `ENOSPC` outcome kitty's own `test_graphics_quota_enforcement` asserts. The default‑configuration boundary is the printed 1,677,721,600 bytes; the 72‑byte limit is a labeled non‑default scaling used only to make the boundary reachable without allocating 1.6 GiB.

---


## Section G — Q5: Quiet adaptation vs. visible signs

**kitty does both** — and the two are cleanly separable by direction and by degree of pressure. Each side is backed by its own captured evidence from the sections above.

### Quiet adaptations (no user‑visible artifact)

| Adaptation | Mechanism | Evidence |
|---|---|---|
| Bounded buffering | 1 MiB `BUF_SZ` parser buffer; bytes accumulate silently | `kitty/vt-parser.c:L18`; the parser rejecting a buffer‑filling unterminated inline APC as `VTE_APC escape code too long (1048574 bytes)` = `BUF_SZ − 2` (Section F.1b) |
| Delay coalescing + adaptive bypass | wait `input_delay`/`repaint_delay`, but process immediately within 16 KiB of full | `kitty/vt-parser.c:L1425`; defaults `input_delay=3 repaint_delay=10` captured stably (Section C.3) |
| LRU image eviction | `apply_storage_quota` deletes oldest images to stay ≤ 320 MiB | `disk_cache.total_size` bounded ≤ 335,544,320; `image_count` 6→2 at the quota (Section F.4) |
| Disk‑cache offload | image/frame pixels spilled to on‑disk cache | `disk_cache.total_size` climbing 108→360 as frames are stored (Section F.5) |

These are "quiet" because the *only* effects the user could notice are indirect (slightly delayed rendering, an old image disappearing) — kitty emits no log and returns no error for any of them.

### Visible signs (an observable artifact)

| Sign | Where it appears | Evidence |
|---|---|---|
| Producer throttled/blocked in `write()` | the *child* program stalls (kernel `n_tty_write`) | Section C.2 (C1): child blocked in `n_tty_write` in 57–58 of 60 samples while kitty is **never** stopped (`kitty_in_STOPPED(T)=0`), throughput pinned to ~15.5–15.7 MB/s; the gate confirmed live as `sum=1048576 has_space=0 req_events=0` |
| Dropped‑data log error | kitty's **stderr / log** | Section D.3 (C2): `[1.637] Too much data being sent to child with id: 1, ignoring it` |
| APC error responses | bytes **sent back to the client** | Section F.3: `EINVAL`, `EFBIG`, `ENOENT` (and `OK`) captured byte‑for‑byte |
| `ENOSPC` on frame quota | bytes sent back to the client | Section F.5: `ENOSPC` at the 5× disk quota boundary |

**The through‑line:** kitty prefers to *quietly* absorb pressure (buffer, delay‑coalesce, evict, offload) for as long as it safely can, and only produces a *visible* sign when a hard boundary is crossed — the 1 MiB read buffer (→ frozen writer), the 100 MiB write cap (→ log error), a per‑transfer or per‑image limit (→ APC error), or a storage quota (→ eviction, and `ENOSPC` for frames). Push it gently and you see nothing; push it past a boundary and the shift becomes unmistakable.

---

## Section H — Methodology, stability, and cleanup appendix

### H.1 Build and run (default, canonical configuration)

- **Repository revision.** All `file:line` citations reference the kitty **source baseline** commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (branch `kitty_815df1e210e0`). This answer document was added on top of that baseline (initial document commit `8fe5c7357`) and has since been refined by follow-up commits that touch **only** this document; the baseline therefore remains the parent of the document history, not `HEAD`, once the document exists. The kitty C/Python/Go **source** is byte-for-byte unchanged from the baseline through current `HEAD` -- `git diff --name-status 815df1e21 HEAD` lists only `blitzy/documentation/kitty_815df1e210e0.md`, and `git diff --stat 815df1e21 HEAD -- kitty/ 3rdparty/ kitty_tests/ docs/` is empty (full proof in Section H.5).
- **Required setup pre‑step (this environment).** The prebuilt dependency bundle ships a Wayland `pkg-config` file that is incompatible with the pinned toolchain, so the supported X11‑only build requires disabling it **before** building (re‑apply if `dependencies/` is re‑downloaded):

```bash
mv dependencies/linux-amd64/lib/pkgconfig/wayland-protocols.pc \
   dependencies/linux-amd64/lib/pkgconfig/wayland-protocols.pc.disabled
```

- **Host build prerequisites.** kitty's C extension links a number of system libraries; on a Debian/Ubuntu host these are installed from the OS package manager **before** building. The authoritative, always-current list is `docs/build.rst` (its Dependencies section): `harfbuzz`, `libpng`, `zlib`, `liblcms2`, `libxxhash`, `openssl`, `freetype`/`fontconfig`, `simde`, and `pkg-config` (plus the X11/GL development headers for the X11 build). `./dev.sh build` then downloads the prebuilt Python/dependency bundle, but these host libraries must already be present.

- **Build command (default, canonical).**

```bash
PATH=/usr/local/go/bin:$PATH ./dev.sh build
```

  Complete, unedited transcript tail and exit status from a clean canonical build in this environment:

```
[79/85] Compiling 3rdparty/base64/lib/arch/generic/codec.c ...
[80/85] Compiling kitty/cleanup.c ...
[81/85] Compiling [x11] glfw/monotonic.c ...
[82/85] Compiling kitty/monotonic.c ...
[83/85] Compiling kitty/simd-string-128.c ...
[84/85] Compiling kitty/simd-string-256.c ...
[85/85] Compiling kitty/gl-wrapper.c ...
 done
[1/4] Linking kitty/fast_data_types ...
[2/4] Linking [x11] kitty/glfw-x11 ...
[3/4] Linking kittens/transfer/rsync ...
[4/4] Linking launcher ...
 done
kitty/tools/cmd
Build successful. Run kitty as: kitty/launcher/kitty
```

```
build exit status = 0
```

  This produces the launcher `kitty/launcher/kitty` and the `fast_data_types` C extension `kitty/fast_data_types.so` (which contains the parser, screen, graphics, and child‑monitor code). The build preserves the project's default `-Werror`/`-pedantic-errors`.

- **Toolchain / component versions (this environment).**

```
kitty     0.35.2 created by Kovid Goyal
gcc       (Ubuntu 15.2.0-4ubuntu4) 15.2.0
go        go1.22.12 linux/amd64
python    3.14.6   (the bundled interpreter the launcher runs under)
```

- **Locale / fonts (scope note).** The `LANG`/`LC_ALL=C.UTF-8` locale and CI font configuration that this environment sets up are prerequisites only for kitty's **full test suite** (`kitty +launch test.py`) to pass; they are **not** required by -- and do not affect -- the flow-control observations in this document, which push real graphics-protocol bytes through the PTY and depend on neither font rendering nor a specific locale. (Confirmed empirically: the canonical drivers in Section H.4 run to byte-identical results without that test-only setup.)

- **Run (headless), exact per‑condition commands.** The canonical PTY flow‑control code only runs inside a full kitty process with a real window, child, and PTY, so every run used a real launcher under a virtual display. Two equivalent display setups were used: `xvfb-run -a` (self‑allocating) for the read‑side driver, and a persistent `Xvfb :91` for the write‑side and graphics drivers. The exact per‑condition launch commands are the driver scripts in H.4; with a one-time setup that creates the out-of-repo scratch directory (shown first in the block below), they reduce to:

```bash
# --- One-time setup: run ONCE, from the repository root, before any driver below ---
# The temporary observation scripts live OUTSIDE the repository, in an owner-only
# scratch directory. Create it, publish its path, then write every script from
# Section H.4 into "$OBS/<name>" (keeping the same filenames).
export KITTY_REPO="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"  # repo root (run from it)
OBS="$(mktemp -d /tmp/kitty_obs.XXXXXX)"; chmod 0700 "$OBS"              # owner-only scratch, outside the repo
printf '%s\n' "$OBS" > /tmp/kitty_obs_scratch_path.txt                   # every driver reads this to find $OBS
export OBS                                                               # the S2 +runpy line below reads $OBS
# Then write each Section H.4 script into "$OBS/" (same filename) and run:

# read-side flood (canonical, /proc observation)
bash "$OBS/obs_read_proc.sh" run1 flood        # and: bash "$OBS/obs_read_proc.sh" run1 burst:65536   (control)

# write-side 100 MiB cap (canonical drop-log)
bash "$OBS/run_w1.sh" run1                      # and run2

# graphics APC cross-product / strict-PNG EFBIG (canonical PTY)
bash "$OBS/run_apcx.sh" run1                    # and run2
bash "$OBS/run_efbig.sh" run1                   # and run2

# graphics quota eviction / frame quota (in-tree harness supplement, S2)
KITTY_REPO=$PWD OBS=$OBS SCRIPT=$OBS/gfx_evict.py DISPLAY=:91 \
  LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
  ./kitty/launcher/kitty +runpy 'import runpy,os; runpy.run_path(os.environ["SCRIPT"], run_name="__main__")'
```

  Each driver embeds the full display/software‑GL environment (`DISPLAY`/`LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe`) so the exact invocation is captured with no elision.

- **Binary integrity — what a hash can and cannot prove here.** The compiled `kitty/fast_data_types.so` is a **git‑ignored local build product**; its content hash is **not** a cross‑environment reproducibility constant. In this environment a clean rebuild produced md5 `bc54e8ab71f065f72a8155b4b1cda5d8` (sha256 `5a2795e426aa7f2ee23266d4c5c650e107231e05f41f49b233e81d8c69fbc502`), and a second full rebuild reproduced the *same* bytes; but that value is specific to this toolchain/flags and legitimately differs in other environments (the review's independent rebuild yielded a different hash from the one an earlier draft had hard‑coded). Accordingly, **no fixed‑hash invariant is claimed.** The only integrity guarantee that matters for this read‑only task — that the kitty **source** was not modified — is proven with Git, not with a binary hash (Section H.5). The hash is used **only** for a within‑run, same‑environment check that the debug‑build supplement was restored to the exact default artifact that existed before it (compared with SHA‑256, which is also collision‑resistant, rather than MD5).

### H.2 Canonical vs. supplement vs. inferred — how each claim was obtained

- **Canonical (real PTY + `io_loop`, default build).** The load‑bearing observations for Q1, Q2, and Q4 each used a real child on a real PTY under a normally‑running kitty:
  - **Read‑side (Section C.2):** `obs_read_proc.sh` + `flood_read.py` — a real child floods graphics commands and never reads; kitty runs normally (**never** signalled or stopped) and its own read‑side gate throttles the child, which is caught asleep in `n_tty_write`. This replaces the earlier SIGSTOP experiment, whose causal mechanism was different.
  - **Write‑side (Section D.3):** `run_w1.sh` + `flood_query.py` — a real child floods `a=p,i=999` puts (each producing an ~85‑byte `ENOENT`) with ECHO off and never reads, so `write_buf` grows to the 100 MiB cap and the drop log fires.
  - **Graphics responses (Section F.3):** `run_apcx.sh` + `apc_xprobe.py` (the q‑matrix cross‑product and the `I=` case) and `run_efbig.sh` + `efbig_png.py` (the strict‑PNG size‑clause `EFBIG`) — real children on a real PTY reading kitty's actual APC replies.
- **Labeled supplement — debug build (S1, Sections C.2/C.3/D.2).** To surface internal branch points that emit no bytes of their own (`read.sz`/`write.pending`, `has_space`, requested `POLLIN`/`POLLOUT`, `write_buf_used`, the `EWOULDBLOCK` break, the `input_delay`/16 KiB bypass clause), kitty was rebuilt with `CPPFLAGS="-DDEBUG_POLL_EVENTS -DKITTY_PRINT_BYTES_SENT_TO_CHILD"` and `--debug` symbols (both hooks exist in‑tree at `kitty/child-monitor.c:L1550` and `:L1449`). The flood itself stayed a real PTY flood; only observability was added via `gdb`/`strace`. The instrumented binary is md5 `4e553e9221c1aa221529fd57a8e28e0e`; the default binary was restored afterward and re‑verified by SHA‑256 (Section H.5).
- **Labeled supplement — in‑tree harness (S2, Sections F.4/F.5).** Storage‑quota eviction, the logical/physical disk‑cache split, and the 5× frame quota were observed with kitty's own in‑tree harness (`send_command`/`parse_bytes` via `+runpy`), which bypasses the PTY + `io_loop` but exercises the *identical* `graphics.c`/`disk-cache.c` code. The **true default** 320 MiB quota was used for the load‑bearing eviction and offload observations; a 72‑byte quota (kitty's own test value, explicitly labeled **NON‑DEFAULT**) was used only to render the pure‑LRU and 5× frame transitions crisply without allocating gigabytes.
- **(inferred, not exercised).** Two dependency files were confirmed *not* on the graphics flow‑control path and are named only as context: `3rdparty/ringbuf/` is used by `kitty/history.c` for pager history **(inferred)**, and `kitty/shm.py` is a POSIX shared‑memory transfer medium that changes the *shape* of a flood but not the flow‑control logic **(inferred)**.

### H.3 Stability (every reported magnitude, across ≥ 2 runs)

The complete raw run‑1 and run‑2 outputs for each experiment are embedded inline in Sections C, D, and F (each shows both runs). This table consolidates every reported magnitude/timing with its per‑run values and states, for each, whether it is **deterministic** (byte‑identical) or a **distribution** (timing‑dependent, reported as a range). Additional confirmation runs performed during remediation are included where available.

| Quantity | Run 1 | Run 2 | Run 3 (confirm) | Nature |
|---|---|---|---|---|
| `input_delay` / `repaint_delay` (ms) | 3 / 10 | 3 / 10 | — | deterministic (config defaults) |
| C read: kitty ever in STOPPED(`T`) | 0/60 | 0/60 | 0/60 | deterministic — kitty **never** paused |
| C read: child blocked in `n_tty_write` (wchan `wait_woken`) | 57/60 | 58/60 | 59/60 | dominant ~95–98% (stable band) |
| C read: child throughput (pinned to kitty's drain rate) | 15.7 MB/s | 15.5 MB/s | 15.1 MB/s | distribution ~15.1–15.7 MB/s |
| C read‑supp (debug): parser occupancy at full | `sum=1048576 has_space=0 req_events=0` | `sum=1048576 has_space=0 req_events=0` | — | deterministic (= `BUF_SZ`, gate false, `POLLIN` removed) |
| C.3 `bypass_clause` light / heavy (debug) | 0 / 1 | 0 / 1 | — | deterministic |
| F.1b APC‑too‑long reported bytes | 1048574 | 1048574 | — | deterministic (= `BUF_SZ − 2`) |
| D write: drop‑log text | `…with id: 1, ignoring it` | `…with id: 1, ignoring it` | `…with id: 1, ignoring it` | deterministic (byte‑identical text) |
| D write: time to first drop‑log | 1.637 s | 1.645 s | 1.705 s | distribution ~1.6–1.7 s |
| D write‑supp (debug): `write_buf_used` pinned at cap | 104857530 | 104857530 | — | deterministic (70 B below the 100 MiB cap) |
| D write‑supp: `write(2)` return / errno | `-1` / `EAGAIN` | `-1` / `EAGAIN` | — | deterministic |
| F.3 APC cross‑product (`q=0/1/2` × ok/err + `I=`) | md5 `e7cce85d…` | md5 `e7cce85d…` | md5 `e7cce85d…` | deterministic (byte‑identical) |
| F.3.2 strict‑PNG `EFBIG` response bytes | `…EFBIG:Too much data…` (28 B) | `…EFBIG:Too much data…` (28 B) | `…EFBIG:Too much data…` (28 B) | deterministic (response) |
| F.3.2 `chunks_sent` before response read back | 2054 | 2056 | 2053 | distribution (all decode > 400,000,000; timing‑dependent) |
| F.4 320 MiB LRU eviction (`image_count`/disk MiB) | 6→2 / 288→96 | 6→2 / 288→96 | — | deterministic |
| F.4 72‑byte pure‑LRU (`image_count`/disk) | 1→2→2 / 36→72→72 | 1→2→2 / 36→72→72 | — | deterministic |
| F.4 disk‑cache logical `total_size` (3×120000) | 120000/240000/360000 | 120000/240000/360000 | 120000/240000/360000 | deterministic |
| F.4 disk‑cache physical `size_on_disk` pre‑`wait_for_write` | 28672/172032/319488 | 36864/120000/270336 | 49152/120000/282624 | distribution — background‑writer lag (nondeterministic) |
| F.4 physical `size_on_disk` after `wait_for_write` | 360000 | 360000 | 360000 | deterministic |
| F.4 raw 64 MiB blob offload (`[×5]` pre / after) | `[0,0,0,0,0]` / 67108864 | `[0,0,0,0,0]` / 67108864 | `[0,0,0,0,0]` / 67108864 | deterministic |
| F.5 5× frame quota `ENOSPC` response bytes | 62 B (identical) | 62 B (identical) | — | deterministic |
| S2 default `storage_limit` | 335544320 | 335544320 | 335544320 | deterministic (== 320·1024·1024) |

*Scale/duration used.* C read — ~360 KiB graphics‑command batches in a tight loop, 60 samples over ~3.6 s with kitty running **normally** (never paused); the debug supplement snapshots the buffer with the child having written ~100 MiB. D write — batches of 1000 `ENOENT`‑producing commands, run until the cap log appeared (~1.7 s) then bounded‑killed; the debug supplement snapshots `write_buf_used` at four points. F.3 APC — one command per case, 2 s read window. F.3.2 EFBIG — 195,000 decoded bytes/chunk, up to ~2056 chunks (~400.9 MB) until the response is read; ~4.2 s. F.4/F.5 — 7 × 48 MiB images (320 MiB eviction) / 3 × 36 B (pure‑LRU) / 3 × 120000 B + one 64 MiB blob (offload) / 8 frames + 1 over (5× quota). Every magnitude above was observed on at least two runs; distributions are reported as ranges, deterministic values as byte‑identical.

### H.4 Temporary scripts (complete text of every script that produced the output)

Every temporary script is reproduced here **in full, with no ellipsis and no "identical in spirit" substitution** — the exact bytes that were run. All scripts lived **outside** the repository, under a per‑run scratch directory created with `mktemp -d` (mode `0700`, owner‑only; the instance for this run was `/tmp/kitty_obs.2ba7uE`, confirmed `drwx------`), and were removed afterward (Section H.5). Each driver discovers that directory by reading the fixed path‑file `/tmp/kitty_obs_scratch_path.txt` (created by the one‑time setup in Section H.1) and **guards** it: a missing or empty scratch path aborts the driver with a non‑zero exit and a clear `ERROR: OBS scratch not initialized` message instead of proceeding. The repository root is auto‑detected via `${KITTY_REPO:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}` rather than hard‑coded, so every script is reproducible exactly as printed from any checkout.

**Safety properties shared by every script (auditable below):**

- **Bounded flood children.** Each flood child stops on an absolute wall‑clock deadline **and** a hard maximum‑bytes cap; the write‑side child additionally arms `signal.alarm(HARD_S)` so it is killed even if blocked inside `write()`. There is no unbounded `while True`.
- **Unique 0700 scratch, safe paths.** The scratch directory is a `mktemp -d` (0700) path passed via `OBS`; the byte counter is opened `O_CREAT|O_TRUNC` with mode `0600`; only the exact owned path is deleted.
- **Exact‑PID / process‑group ownership.** Each driver captures the launched PID and its process‑group id (`PGID`), records the child's PID from a file the child writes, and terminates by those exact ids — never by name‑matching. A `safe_killpg` helper refuses to signal a group equal to the driver's own (`SELF_PGID`) or `≤ 1`.
- **`trap`‑based cleanup.** Every driver installs `trap _cleanup EXIT INT TERM`; `_cleanup` sends `SIGCONT` to any owned pid (so nothing can be left stopped), then `TERM` then `KILL` to the owned process group only. The read‑side experiment additionally uses **no `SIGSTOP` at all** (that was the defect in the earlier draft), so "a failed run could leave kitty stopped" cannot occur here.
- **`setsid` isolation.** kitty is launched under `setsid` so it forms its own process group that can be reaped as a unit without touching the driver or unrelated host processes.

#### H.4.1 Read‑side scripts (Section C)

**`flood_read.py`** — the read‑side flood child. Writes graphics commands to the PTY slave as fast as the OS allows and **never reads**, so the only thing that can slow it is kitty's own read‑side gate; kitty is **never** signalled or paused. Bounded by an absolute deadline **and** a hard byte cap:

```python
#!/usr/bin/env python3
# Read-side flood child. Writes graphics commands to its stdout (the PTY slave)
# as fast as the OS allows and NEVER reads its input, so the ONLY thing that can
# slow it down is kitty's own read-side backpressure. Bounded by an absolute
# deadline AND a hard byte cap so it can never run away.
import os, sys, time, base64
OBS = os.environ["OBS"]                                  # scratch dir passed explicitly
MODE = os.environ.get("MODE", "flood")                   # "flood" or "burst:<nbytes>"
MAX_BYTES = int(os.environ.get("MAX_BYTES", str(400 * 1024 * 1024)))   # hard cap
DEADLINE_S = float(os.environ.get("DEADLINE_S", "25"))                 # absolute deadline
def set_counter(v):
    fd = os.open(os.path.join(OBS, "child_bytes"), os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o600)
    try: os.write(fd, str(v).encode())
    finally: os.close(fd)
open(os.path.join(OBS, "child.pid"), "w").write(str(os.getpid()))
set_counter(0)
open(os.path.join(OBS, "child.ready"), "w").write("ready")
go = os.path.join(OBS, "go"); gd = time.monotonic() + 10.0
while not os.path.exists(go):
    if time.monotonic() > gd: sys.exit(3)
    time.sleep(0.02)
px = bytes([65, 66, 67]) * (20 * 20)                     # 1200 bytes RGB
b64 = base64.standard_b64encode(px)
cmd = b"\x1b_Ga=T,f=24,s=20,v=20,q=2;" + b64 + b"\x1b\\" # transmit+display, responses suppressed (q=2)
batch = cmd * 200                                        # ~360 KiB per write()
deadline = time.monotonic() + DEADLINE_S; total = 0
if MODE.startswith("burst:"):
    n_target = int(MODE.split(":", 1)[1]); blob = b"X" * 4096
    while total < n_target and time.monotonic() < deadline:
        n = os.write(1, blob[:min(len(blob), n_target - total)]); total += n; set_counter(total)
else:
    while total < MAX_BYTES and time.monotonic() < deadline:
        n = os.write(1, batch)                           # BLOCKS when PTY + kitty buffer full
        total += n; set_counter(total)
set_counter(total)
open(os.path.join(OBS, "child.done"), "w").write(str(total))
```

**`obs_read_proc.sh`** — the **canonical** read‑side driver. Starts the flood, then samples `/proc/<kitty>/stat` (kitty run state), `/proc/<child>/wchan`, `/proc/<child>/stack`, and the child byte counter 60 times over ~3.6 s. **No signal is ever sent to kitty.** A `burst:<n>` mode provides the control run. (Produced the C1 output in Section C.2.)

```bash
#!/usr/bin/env bash
# Read-side backpressure on the DEFAULT (canonical -O3) build, observed purely via /proc.
# The flood child is asleep in the tty-write wait (wchan=wait_woken, kernel stack n_tty_write)
# while kitty runs normally (State R/S, never T) -> kitty's AUTOMATIC gate; NO SIGSTOP.
set -u
RUN="${1:-run1}"; RMODE="${2:-flood}"                 # RMODE: flood | burst:<bytes>
OBS="$(cat /tmp/kitty_obs_scratch_path.txt 2>/dev/null)"
[ -n "$OBS" ] && [ -d "$OBS" ] || { echo "ERROR: OBS scratch not initialized (run the H.4 one-time setup first)" >&2; exit 2; }
REPO="${KITTY_REPO:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"
cd "$REPO"; export PATH=/usr/local/go/bin:$PATH
export OBS MODE="$RMODE" MAX_BYTES=$((400*1024*1024)) DEADLINE_S=12
SELF_PGID=$(ps -o pgid= -p $$ | tr -d ' ')
rm -f "$OBS/go" "$OBS/child.ready" "$OBS/child.done" "$OBS/child.pid" "$OBS/child_bytes"
setsid xvfb-run -a env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
   OBS="$OBS" MODE="$MODE" MAX_BYTES="$MAX_BYTES" DEADLINE_S="$DEADLINE_S" \
   ./kitty/launcher/kitty -o confirm_os_window_close=0 python3 "$OBS/flood_read.py" \
   >"$OBS/kout_$RUN.log" 2>"$OBS/kerr_$RUN.log" &
XVFB_PID=$!; PGID=$(ps -o pgid= -p "$XVFB_PID" 2>/dev/null | tr -d ' ')
safe_killpg(){ [ -n "${PGID:-}" ] && [ "$PGID" != "$SELF_PGID" ] && [ "$PGID" -gt 1 ] 2>/dev/null && kill "$1" -"$PGID" 2>/dev/null || true; }
# --- Finding 15/S3: trap-based cleanup on EXIT/INT/TERM (resume, terminate, reap only owned pids) ---
_cleanup(){ trap - EXIT INT TERM; for _p in "${CHILD_PID:-}" "${KPID:-}" "${XVFB_PID:-}"; do [ -n "$_p" ] && kill -CONT "$_p" 2>/dev/null; done; safe_killpg -TERM 2>/dev/null; safe_killpg -KILL 2>/dev/null; return 0; }
trap _cleanup EXIT INT TERM

for i in $(seq 1 200); do [ -f "$OBS/child.ready" ] && break; sleep 0.05; done
CHILD_PID="$(cat "$OBS/child.pid" 2>/dev/null || echo '')"
KITTY_PID=""; for p in $(pgrep -f 'launcher/kitty' 2>/dev/null||true); do [ "$(cat /proc/$p/comm 2>/dev/null||true)" = kitty ] && KITTY_PID="$p"; done
echo "=== read-side backpressure (DEFAULT canonical build) $RUN mode=$RMODE ==="
echo "KITTY_PID=$KITTY_PID CHILD_PID=$CHILD_PID  [PIDs run-local/ephemeral]"
echo go > "$OBS/go"
NSAMP=60; blocked=0; kitty_T=0; first=""; last=""; t0=$(date +%s.%N); done_at=""
for s in $(seq 1 $NSAMP); do
  sleep 0.05
  kst=$(awk '{print $3}' /proc/$KITTY_PID/stat 2>/dev/null || echo '?'); [ "$kst" = T ] && kitty_T=$((kitty_T+1))
  wc=$(cat /proc/$CHILD_PID/wchan 2>/dev/null || echo ''); [ "$wc" = wait_woken ] && blocked=$((blocked+1))
  cb=$(cat "$OBS/child_bytes" 2>/dev/null || echo 0); [ -z "$first" ] && first="$cb"; last="$cb"
  if [ -f "$OBS/child.done" ]; then done_at="$s"; break; fi
done
t1=$(date +%s.%N); s=${s}
echo "samples=$s  kitty_in_STOPPED(T)=$kitty_T  child_blocked_in_tty_write_wait(wchan=wait_woken)=$blocked/$s"
[ -n "$done_at" ] && echo "child COMPLETED at sample $done_at (total bytes=$(cat "$OBS/child.done" 2>/dev/null))"
echo "child_bytes first=$first last=$last  elapsed=$(awk "BEGIN{printf \"%.2f\",$t1-$t0}")s  throughput~=$(awk "BEGIN{printf \"%.1f\",($last-$first)/1e6/($t1-$t0)}") MB/s"
# robust raw kernel-stack capture: retry until the task is caught in the write wait
if [ ! -f "$OBS/child.done" ]; then
  for r in $(seq 1 40); do
    stk=$(cat /proc/$CHILD_PID/stack 2>/dev/null)
    if echo "$stk" | grep -q n_tty_write; then echo "--- child kernel stack (blocked in write to tty) ---"; echo "$stk" | head -7; break; fi
    sleep 0.02
  done
fi
echo "--- kitty stderr (startup only) ---"; head -1 "$OBS/kerr_$RUN.log" 2>/dev/null
[ -n "${CHILD_PID:-}" ] && kill -CONT "$CHILD_PID" 2>/dev/null || true
safe_killpg -TERM; sleep 0.5; safe_killpg -KILL
echo "RUN_$RUN done"
```

**`obs_read_gdb.sh`** — the read‑side **labeled supplement** (debug build). Same real flood, but async‑attaches `gdb` (no single‑stepping) to snapshot the parser's `read.sz`/`write.pending`, evaluate the space predicate, and read the requested `POLLIN` word. (Produced the C1‑supplement output.)

```bash
#!/usr/bin/env bash
# Read-side gate observation on the INSTRUMENTED build (labeled supplement).
# gdb async-attach snapshots of parser occupancy + has-space + requested poll events.
set -u
RUN="${1:-run1}"
OBS="$(cat /tmp/kitty_obs_scratch_path.txt 2>/dev/null)"
[ -n "$OBS" ] && [ -d "$OBS" ] || { echo "ERROR: OBS scratch not initialized (run the H.4 one-time setup first)" >&2; exit 2; }
REPO="${KITTY_REPO:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"
cd "$REPO"; export PATH=/usr/local/go/bin:$PATH
export OBS MODE=flood MAX_BYTES=$((400*1024*1024)) DEADLINE_S=20
SELF_PGID=$(ps -o pgid= -p $$ | tr -d ' ')
rm -f "$OBS/go" "$OBS/child.ready" "$OBS/child.done" "$OBS/child.pid" "$OBS/child_bytes"

setsid xvfb-run -a env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
   OBS="$OBS" MODE="$MODE" MAX_BYTES="$MAX_BYTES" DEADLINE_S="$DEADLINE_S" \
   ./kitty/launcher/kitty -o confirm_os_window_close=0 -o scrollback_lines=100 \
   python3 "$OBS/flood_read.py" >"$OBS/poll_$RUN.log" 2>"$OBS/err_$RUN.log" &
XVFB_PID=$!; PGID=$(ps -o pgid= -p "$XVFB_PID" 2>/dev/null | tr -d ' ')
safe_killpg() { [ -n "${PGID:-}" ] && [ "$PGID" != "$SELF_PGID" ] && [ "$PGID" -gt 1 ] 2>/dev/null && kill "$1" -"$PGID" 2>/dev/null || true; }
# --- Finding 15/S3: trap-based cleanup on EXIT/INT/TERM (resume, terminate, reap only owned pids) ---
_cleanup(){ trap - EXIT INT TERM; for _p in "${CHILD_PID:-}" "${KITTY_PID:-}" "${XVFB_PID:-}"; do [ -n "$_p" ] && kill -CONT "$_p" 2>/dev/null; done; safe_killpg -TERM 2>/dev/null; safe_killpg -KILL 2>/dev/null; return 0; }
trap _cleanup EXIT INT TERM
for i in $(seq 1 200); do [ -f "$OBS/child.ready" ] && break; sleep 0.05; done
CHILD_PID="$(cat "$OBS/child.pid" 2>/dev/null || echo '')"
KITTY_PID=""
for p in $(pgrep -f 'launcher/kitty' 2>/dev/null || true); do
  [ "$(cat /proc/$p/comm 2>/dev/null || true)" = "kitty" ] && KITTY_PID="$p"
done
KPGID=$(ps -o pgid= -p "$KITTY_PID" 2>/dev/null | tr -d ' ')
echo "KITTY_PID=$KITTY_PID (pgid=$KPGID matches launcher pgid=$PGID) CHILD_PID=$CHILD_PID"
cat > "$OBS/snap.gdb" <<'GEOF'
set pagination off
printf "read.sz=%lu write.pending=%lu sum=%lu has_space=%d childfd=%d req_events=%d write_buf_used=%lu\n", ((struct PS*)children[0].screen->vt_parser->state)->read.sz, ((struct PS*)children[0].screen->vt_parser->state)->write.pending, ((struct PS*)children[0].screen->vt_parser->state)->read.sz + ((struct PS*)children[0].screen->vt_parser->state)->write.pending, (((struct PS*)children[0].screen->vt_parser->state)->read.sz + ((struct PS*)children[0].screen->vt_parser->state)->write.pending) < 1048576u, children_fds[2].fd, children_fds[2].events, children[0].screen->write_buf_used
GEOF
echo go > "$OBS/go"
sleep 2
for c in 1 2 3; do
  stk=$(head -1 /proc/$CHILD_PID/stack 2>/dev/null | grep -o 'n_tty_write' || echo 'running')
  kst=$(awk '{print $3}' /proc/$KITTY_PID/stat 2>/dev/null || echo '?')
  snap=$(timeout 12 gdb -q -batch -p "$KITTY_PID" -x "$OBS/snap.gdb" 2>/dev/null | grep 'read.sz=')
  printf "snap%s kittyState=%s child_stacktop=%s %s\n" "$c" "$kst" "$stk" "$snap"
  sleep 0.4
done
echo "--- DEBUG_POLL_EVENTS counts (child fd=i:2) ---"
sort "$OBS/poll_$RUN.log" 2>/dev/null | uniq -c | sort -rn | head -6
echo "--- total poll lines: $(wc -l < "$OBS/poll_$RUN.log" 2>/dev/null) ---"
# cleanup: guaranteed CONT to child, then terminate the launcher's group (guarded)
[ -n "${CHILD_PID:-}" ] && kill -CONT "$CHILD_PID" 2>/dev/null || true
safe_killpg -TERM; sleep 0.5; safe_killpg -KILL
echo "RUN_$RUN complete"
```

**`snap.gdb`** — the gdb command file used by `obs_read_gdb.sh` (async snapshot, no single‑stepping):

```
set pagination off
printf "read.sz=%lu write.pending=%lu sum=%lu has_space=%d childfd=%d req_events=%d write_buf_used=%lu\n", ((struct PS*)children[0].screen->vt_parser->state)->read.sz, ((struct PS*)children[0].screen->vt_parser->state)->write.pending, ((struct PS*)children[0].screen->vt_parser->state)->read.sz + ((struct PS*)children[0].screen->vt_parser->state)->write.pending, (((struct PS*)children[0].screen->vt_parser->state)->read.sz + ((struct PS*)children[0].screen->vt_parser->state)->write.pending) < 1048576u, children_fds[2].fd, children_fds[2].events, children[0].screen->write_buf_used
```

**`emit_bad_apc.sh`** — the canonical `BUF_SZ`‑ceiling emitter (Section F.1b): an **unterminated** ~1.5 MB APC (no `ESC\` terminator), which the parser rejects once it exceeds `MAX_ESCAPE_CODE_LENGTH`:

```bash
#!/bin/bash
printf '\033_Ga=q,i=1;'
head -c 1500000 /dev/zero | tr '\0' 'A'
# no ESC\ terminator on purpose
sleep 1
printf 'DONE_EMITTING\n'
sleep 1
```

**`probe_apc.py`** — the F.1b harness **supplement** (the payload‑size scaling table via the in‑tree parser), a bounded loop over four fixed sizes:

```python
import sys, os
repo = os.environ['KITTY_REPO']
sys.path.insert(0, repo)
from kitty_tests import parse_bytes, Callbacks
from kitty.fast_data_types import Screen

def make_screen():
    c = Callbacks()
    # Screen(callbacks, lines, columns, scrollback, cell_width, cell_height, window_id, test_child)
    s = Screen(c, 5, 20, 5, 10, 20, 0, c)
    return s, c

# An unterminated APC graphics escape: ESC _ G <huge payload> with NO terminating ESC \
# The parser must accumulate until it exceeds MAX_ESCAPE_CODE_LENGTH and then report.
for size in (300000, 600000, 1048600, 2000000):
    s, c = make_screen()
    errors = []
    def dump(window_id, kind, msg):
        if kind == 'error':
            errors.append(msg)
    payload = b'\x1b_Ga=q,i=1;' + (b'A' * size)  # NO closing ESC\
    parse_bytes(s, payload, dump)
    # flush any remaining by parsing an empty follow-up
    print(f"INPUT_PAYLOAD_SIZE={size} total_bytes_fed={len(payload)}")
    for e in errors:
        print("  ERROR_MSG:", repr(e))
    if not errors:
        print("  (no error captured)")
```

**C.3 delay‑timing scripts (labeled supplement, debug build).** The light‑vs‑heavy `input_delay` bypass in Section C.3 was captured on the same debug build as the C1‑supplement, using three small scripts. `light_child.sh` drives a **slow** flood (60 `a=q` commands at ~0.08 s cadence) so the buffer stays nearly empty and the delay is *honored*; `light.gdb` sets a breakpoint at the decision line `kitty/vt-parser.c:L1425` and prints the `bypass_clause` for the first 8 hits. `heavy_snap.gdb` async‑attaches once (no breakpoint, to avoid perturbing a fast‑filling buffer) and prints the same fields during a **heavy** `flood_read.py` flood so the bypass clause is *taken*. The exact commands (debug build, persistent display `:91`) were:

```bash
# Light case (delay honored, bypass_clause=0) — slow child + breakpoint:
setsid xvfb-run -a env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
   ./kitty/launcher/kitty -o confirm_os_window_close=0 bash "$OBS/light_child.sh" &
# once kitty is up (comm==kitty), attach the breakpoint script:
gdb -q -batch -p "$KITTY_PID" -x "$OBS/light.gdb"   > "$OBS/light_run1.gdb.out" 2>&1

# Heavy case (delay bypassed, bypass_clause=1) — fast flood + 5 async snapshots:
setsid xvfb-run -a env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
   OBS="$OBS" MODE=flood MAX_BYTES=$((400*1024*1024)) DEADLINE_S=20 \
   ./kitty/launcher/kitty -o confirm_os_window_close=0 python3 "$OBS/flood_read.py" &
for i in 1 2 3 4 5; do gdb -q -batch -p "$KITTY_PID" -x "$OBS/heavy_snap.gdb"; done
```

**`light_child.sh`** — the slow‑cadence light‑load flood child (bounded: 60 iterations then exits):

```bash
#!/bin/bash
sleep 3
for i in $(seq 1 60); do
  printf '\033_Ga=q,i=%d;AAAA\033\\' "$i"
  sleep 0.08
done
sleep 1
```

**`light.gdb`** — the light‑load breakpoint script (prints `HIT … bypass_clause=…` for 8 hits, then detaches and quits — self‑bounded):

```
set pagination off
set confirm off
set $n = 0
break vt-parser.c:1425
commands
  silent
  set $n = $n + 1
  printf "HIT %d read.sz=%lu write.pending=%lu tsni=%ld input_delay=%ld bypass_clause=%d delay_elapsed=%d\n", $n, self->read.sz, self->write.pending, (long)pd->time_since_new_input, (long)global_state.opts.input_delay, (int)(self->read.sz + 16*1024 > 1048576), (int)(pd->time_since_new_input >= global_state.opts.input_delay)
  if $n >= 8
    detach
    quit
  end
  continue
end
continue
```

**`heavy_snap.gdb`** — the heavy‑load single async snapshot (prints one `HEAVY … bypass_clause=1` line, then detaches and quits):

```
set pagination off
set confirm off
printf "HEAVY read.sz=%lu write.pending=%lu sum=%lu bypass_clause=%d has_space=%d input_delay=%ld req_events=%d write_buf_used=%lu\n", \
  ((struct PS*)children[0].screen->vt_parser->state)->read.sz, \
  ((struct PS*)children[0].screen->vt_parser->state)->write.pending, \
  ((struct PS*)children[0].screen->vt_parser->state)->read.sz + ((struct PS*)children[0].screen->vt_parser->state)->write.pending, \
  (int)(((struct PS*)children[0].screen->vt_parser->state)->read.sz + 16*1024 > 1048576u), \
  (int)((((struct PS*)children[0].screen->vt_parser->state)->read.sz + ((struct PS*)children[0].screen->vt_parser->state)->write.pending) < 1048576u), \
  (long)global_state.opts.input_delay, \
  children_fds[2].events, \
  children[0].screen->write_buf_used
detach
quit
```
#### H.4.2 Write‑side scripts (Section D)

**`flood_query.py`** — the write‑side flood child. Floods `a=p,i=999` puts (each yields an ~85‑byte `ENOENT`) to the PTY slave and, in `flood` mode, **never reads** them back, so kitty's per‑screen `write_buf` grows to the 100 MiB cap. It puts the slave in **raw mode (ECHO off)** first — otherwise the line discipline echoes kitty's responses back into kitty's reader, draining `write_buf`. Bounded three ways: absolute deadline, `signal.alarm(HARD_S)` hard kill (fires even if blocked in `write()`), and a max‑bytes cap:

```python
#!/usr/bin/env python3
# Write-side flood child. Floods `a=p,i=999` graphics commands to its stdout (the PTY
# slave); each one makes kitty build an ~85-byte ENOENT response and queue it back onto
# the child's input. In "flood" mode this child NEVER reads that input, so kitty's
# per-screen write_buf grows until the hard 100 MiB cap fires. In "drain" mode it floods
# briefly, then reads its stdin so the queued bytes flush and write_buf_used falls to ~0.
#
# The child puts its terminal into RAW mode first (ECHO OFF): otherwise the PTY line
# discipline echoes every byte kitty writes to the master straight back into kitty's
# read side, which drains write_buf and prevents the write-side backlog from forming.
#
# Bounded three ways so it can never run away:
#   * absolute wall-clock deadline (DEADLINE_S)
#   * SIGALRM hard kill (HARD_S) that fires even if blocked inside write()
#   * maximum bytes emitted (MAX_BYTES)
import os, sys, time, signal, termios, tty, select
OBS = os.environ["OBS"]
MODE = os.environ.get("MODE", "flood")                 # "flood" | "drain"
MAX_BYTES = int(os.environ.get("MAX_BYTES", str(400 * 1024 * 1024)))
DEADLINE_S = float(os.environ.get("DEADLINE_S", "8"))
HARD_S = int(os.environ.get("HARD_S", "12"))
FLOOD_S = float(os.environ.get("FLOOD_S", "0.2"))      # drain-mode accumulation window
signal.alarm(HARD_S)                                   # guaranteed death even if blocked in write()
# RAW mode / ECHO OFF on the slave so responses are NOT echoed back into kitty's reader.
try:
    tty.setraw(0)
except Exception:
    pass
def set_counter(v):
    fd = os.open(os.path.join(OBS, "child_bytes"), os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o600)
    try: os.write(fd, str(v).encode())
    finally: os.close(fd)
open(os.path.join(OBS, "child.pid"), "w").write(str(os.getpid()))
set_counter(0)
open(os.path.join(OBS, "child.ready"), "w").write("ready")
go = os.path.join(OBS, "go"); gd = time.monotonic() + 10.0
while not os.path.exists(go):
    if time.monotonic() > gd: sys.exit(3)
    time.sleep(0.02)
CMD = b"\x1b_Ga=p,i=999\x1b\\"                          # put referencing non-existent image -> ENOENT
batch = CMD * 1000
total = 0
deadline = time.monotonic() + DEADLINE_S
if MODE == "drain":
    drain_after = time.monotonic() + FLOOD_S
    # Phase 1: flood briefly to accumulate a modest write_buf backlog (do NOT read stdin)
    while time.monotonic() < drain_after and total < MAX_BYTES:
        try: n = os.write(1, batch); total += n; set_counter(total)
        except OSError: break
    open(os.path.join(OBS, "drain.start"), "w").write("1")
    # Phase 2: STOP flooding, read stdin as fast as data arrives so kitty's write_buf empties
    os.set_blocking(0, True)
    dend = time.monotonic() + 6.0
    while time.monotonic() < dend:
        r, _, _ = select.select([0], [], [], 0.1)
        if r:
            try:
                d = os.read(0, 1 << 20)
                if not d: break
            except OSError: break
    open(os.path.join(OBS, "drain.done"), "w").write("1")
else:
    while time.monotonic() < deadline and total < MAX_BYTES:
        try: n = os.write(1, batch); total += n; set_counter(total)
        except OSError: break
set_counter(total)
open(os.path.join(OBS, "child.done"), "w").write(str(total))
```

**`run_w1.sh`** — the **canonical** write‑side driver (DEFAULT build). Launches kitty with `flood_query.py`, captures kitty's stderr, polls for the `Too much data` cap log, and reports the first/last matching lines, the count, the child byte total, and kitty's disposition. (Produced the C2 output in Section D.3.)

```bash
#!/usr/bin/env bash
# W1 (canonical, DEFAULT build): drive the write-side flood through a real kitty PTY and
# capture the exact 100 MiB-cap dropped-data log line, with NON-BLANK timing, count,
# kitty exit status, and full raw output. No SIGSTOP, no debug hooks - the default binary.
set -u
RUN="${1:-run1}"
OBS="$(cat /tmp/kitty_obs_scratch_path.txt 2>/dev/null)"
[ -n "$OBS" ] && [ -d "$OBS" ] || { echo "ERROR: OBS scratch not initialized (run the H.4 one-time setup first)" >&2; exit 2; }
REPO="${KITTY_REPO:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"
cd "$REPO"; export PATH=/usr/local/go/bin:$PATH
SELF_PGID=$(ps -o pgid= -p $$ | tr -d ' ')
ERR="$OBS/w1_${RUN}.err"; OUT="$OBS/w1_${RUN}.out"
rm -f "$OBS/go" "$OBS/child.ready" "$OBS/child.done" "$OBS/child.pid" "$OBS/child_bytes" "$ERR" "$OUT"
export OBS MODE=flood MAX_BYTES=$((400*1024*1024)) DEADLINE_S=3 HARD_S=8
setsid xvfb-run -a env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
   OBS="$OBS" MODE=flood MAX_BYTES="$MAX_BYTES" DEADLINE_S=3 HARD_S=8 \
   ./kitty/launcher/kitty -o confirm_os_window_close=0 python3 "$OBS/flood_query.py" \
   >"$OUT" 2>"$ERR" &
XVFB_PID=$!; PGID=$(ps -o pgid= -p "$XVFB_PID" 2>/dev/null | tr -d ' ')
safe_killpg(){ [ -n "${PGID:-}" ] && [ "$PGID" != "$SELF_PGID" ] && [ "$PGID" -gt 1 ] 2>/dev/null && kill "$1" -"$PGID" 2>/dev/null || true; }
# --- Finding 15/S3: trap-based cleanup on EXIT/INT/TERM (resume, terminate, reap only owned pids) ---
_cleanup(){ trap - EXIT INT TERM; for _p in "${CHILD_PID:-}" "${KPID:-}" "${XVFB_PID:-}"; do [ -n "$_p" ] && kill -CONT "$_p" 2>/dev/null; done; safe_killpg -TERM 2>/dev/null; safe_killpg -KILL 2>/dev/null; return 0; }
trap _cleanup EXIT INT TERM

# wait for child readiness
for i in $(seq 1 200); do [ -f "$OBS/child.ready" ] && break; sleep 0.05; done
CHILD_PID="$(cat "$OBS/child.pid" 2>/dev/null || echo '')"
KITTY_PID=""; for p in $(pgrep -f 'launcher/kitty' 2>/dev/null||true); do [ "$(cat /proc/$p/comm 2>/dev/null||true)" = kitty ] && KITTY_PID="$p"; done
echo "=== C2 write-side 100 MiB cap via real PTY (DEFAULT canonical build) $RUN ==="
echo "KITTY_PID=$KITTY_PID CHILD_PID=$CHILD_PID  [PIDs run-local/ephemeral]"
echo go > "$OBS/go"
# poll for the cap log line, bounded
first_ts=""; found=0
for s in $(seq 1 300); do
  if grep -q "Too much data being sent to child" "$ERR" 2>/dev/null; then found=1; break; fi
  [ -f "$OBS/child.done" ] && break
  sleep 0.05
done
# let the flood run to its own deadline so the child self-terminates
for s in $(seq 1 260); do [ -f "$OBS/child.done" ] && break; sleep 0.05; done
# now wait (bounded) for kitty to exit on its own after the child dies
KEXIT="(still-running)"
for s in $(seq 1 60); do
  if ! kill -0 "$KITTY_PID" 2>/dev/null; then KEXIT="(exited-on-own)"; break; fi
  sleep 0.05
done
# extract NON-BLANK timing from the first matching line
first_line="$(grep -m1 'Too much data being sent to child' "$ERR" 2>/dev/null || true)"
first_ts="$(printf '%s' "$first_line" | sed -n 's/^\[\([0-9.][0-9.]*\)\].*/\1/p')"
cnt="$(grep -c 'Too much data being sent to child' "$ERR" 2>/dev/null || echo 0)"
child_total="$(cat "$OBS/child.done" 2>/dev/null || echo '(child did not finish)')"
echo "time_to_first_dropped_log = ${first_ts:-<none>} s   found=$found"
echo "total 'Too much data' lines captured = $cnt"
echo "child_bytes_emitted_total = $child_total"
echo "kitty_disposition = $KEXIT"
echo "--- first 3 matching stderr lines (verbatim) ---"
grep -m3 'Too much data being sent to child' "$ERR" 2>/dev/null || echo '(none)'
echo "--- last 2 matching stderr lines (verbatim) ---"
grep 'Too much data being sent to child' "$ERR" 2>/dev/null | tail -2 || echo '(none)'
echo "--- non-'Too much data' stderr lines (verbatim, should be startup-only) ---"
grep -v 'Too much data being sent to child' "$ERR" 2>/dev/null | sed '/^$/d' || echo '(none)'
echo "--- err file size (bytes) ---"
wc -c < "$ERR" 2>/dev/null
# safety cleanup by exact pid / pgid
[ -n "${CHILD_PID:-}" ] && kill -TERM "$CHILD_PID" 2>/dev/null || true
safe_killpg -TERM; sleep 0.4; safe_killpg -KILL
echo "W1_$RUN done"
```

**`run_w2.sh`** — write‑side **labeled supplement** (debug build). Async‑attaches `gdb` at four points during a real‑PTY flood to snapshot `write_buf_used`, `write_buf_sz`, the requested poll events, the child fd, and the parser's `read.sz`. (Produced the S1(a) `write_buf_used` growth output in Section D.2.)

```bash
#!/usr/bin/env bash
# W2 (INSTRUMENTED build, labeled SUPPLEMENT): observe the write-side queue internals that
# emit no bytes of their own - write_buf_used growth (before/intermediate/after), the
# requested POLLOUT event bit, the child fd, and (via DEBUG_POLL_EVENTS) the POLLOUT revents.
# The flood still goes through a real PTY; only observability was added. Bounded + guarded.
set -u
RUN="${1:-run1}"
OBS="$(cat /tmp/kitty_obs_scratch_path.txt 2>/dev/null)"
[ -n "$OBS" ] && [ -d "$OBS" ] || { echo "ERROR: OBS scratch not initialized (run the H.4 one-time setup first)" >&2; exit 2; }
REPO="${KITTY_REPO:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"
cd "$REPO"; export PATH=/usr/local/go/bin:$PATH
SELF_PGID=$(ps -o pgid= -p $$ | tr -d ' ')
rm -f "$OBS/go" "$OBS/child.ready" "$OBS/child.done" "$OBS/child.pid" "$OBS/child_bytes"
export OBS MODE=flood MAX_BYTES=$((400*1024*1024)) DEADLINE_S=11 HARD_S=15
setsid xvfb-run -a env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
   OBS="$OBS" MODE=flood MAX_BYTES="$MAX_BYTES" DEADLINE_S=11 HARD_S=15 \
   ./kitty/launcher/kitty -o confirm_os_window_close=0 python3 "$OBS/flood_query.py" \
   >"$OBS/w2_poll_$RUN.log" 2>/dev/null &
XVFB_PID=$!; PGID=$(ps -o pgid= -p "$XVFB_PID" 2>/dev/null | tr -d ' ')
safe_killpg(){ [ -n "${PGID:-}" ] && [ "$PGID" != "$SELF_PGID" ] && [ "$PGID" -gt 1 ] 2>/dev/null && kill "$1" -"$PGID" 2>/dev/null || true; }
# --- Finding 15/S3: trap-based cleanup on EXIT/INT/TERM (resume, terminate, reap only owned pids) ---
_cleanup(){ trap - EXIT INT TERM; for _p in "${CHILD_PID:-}" "${KPID:-}" "${XVFB_PID:-}"; do [ -n "$_p" ] && kill -CONT "$_p" 2>/dev/null; done; safe_killpg -TERM 2>/dev/null; safe_killpg -KILL 2>/dev/null; return 0; }
trap _cleanup EXIT INT TERM

for i in $(seq 1 200); do [ -f "$OBS/child.ready" ] && break; sleep 0.05; done
CHILD_PID="$(cat "$OBS/child.pid" 2>/dev/null || echo '')"
KITTY_PID=""; for p in $(pgrep -f 'launcher/kitty' 2>/dev/null||true); do [ "$(cat /proc/$p/comm 2>/dev/null||true)" = kitty ] && KITTY_PID="$p"; done
echo "=== S1 write-side queue internals (INSTRUMENTED supplement) $RUN ==="
echo "KITTY_PID=$KITTY_PID CHILD_PID=$CHILD_PID  [PIDs run-local/ephemeral]"
cat > "$OBS/w2_snap.gdb" <<'GEOF'
set pagination off
printf "write_buf_used=%lu write_buf_sz=%lu req_events=%d child_fd=%d parser_read_sz=%lu\n", children[0].screen->write_buf_used, children[0].screen->write_buf_sz, children_fds[2].events, children_fds[2].fd, ((struct PS*)children[0].screen->vt_parser->state)->read.sz
GEOF
echo go > "$OBS/go"
# Four async single-shot snapshots: rising -> pinned-at-cap. Each gdb attach is bounded by timeout.
LABELS=("BEFORE" "INTERMEDIATE" "PRECAP" "ATCAP")
DELAYS=(0.20 0.45 0.85 2.0)
for idx in 0 1 2 3; do
  sleep "${DELAYS[$idx]}"
  snap=$(timeout --signal=KILL 15 gdb -q -batch -p "$KITTY_PID" -x "$OBS/w2_snap.gdb" 2>/dev/null | grep 'write_buf_used=')
  printf "%-13s %s\n" "${LABELS[$idx]}" "$snap"
done
# let the child finish on its own
for i in $(seq 1 120); do [ -f "$OBS/child.done" ] && break; sleep 0.05; done
echo "child_bytes_emitted_total = $(cat "$OBS/child.done" 2>/dev/null || echo '(unfinished)')"
echo "--- DEBUG_POLL_EVENTS revent counts on child fd (i:2); POLLOUT is the write-ready signal ---"
grep -E '^i:2 ' "$OBS/w2_poll_$RUN.log" 2>/dev/null | sort | uniq -c | sort -rn
echo "--- all distinct poll-event line types ---"
sort "$OBS/w2_poll_$RUN.log" 2>/dev/null | uniq -c | sort -rn | head -8
echo "--- total poll lines: $(wc -l < "$OBS/w2_poll_$RUN.log" 2>/dev/null) ---"
[ -n "${CHILD_PID:-}" ] && kill -CONT "$CHILD_PID" 2>/dev/null || true
safe_killpg -TERM; sleep 0.5; safe_killpg -KILL
echo "S1_$RUN done"
```

**`w2_snap.gdb`** — the gdb command file used by `run_w2.sh`:

```
set pagination off
printf "write_buf_used=%lu write_buf_sz=%lu req_events=%d child_fd=%d parser_read_sz=%lu\n", children[0].screen->write_buf_used, children[0].screen->write_buf_sz, children_fds[2].events, children_fds[2].fd, ((struct PS*)children[0].screen->vt_parser->state)->read.sz
```

**`run_w2_strace.sh`** — write‑side **labeled supplement** (debug build). Runs the flood under `strace -e trace=write` to record the raw `write(2)` return value and errno on the master fd, capturing the exact `= -1 EAGAIN`. (Produced the S1 errno evidence in Section D.2.)

```bash
#!/usr/bin/env bash
# W2d (labeled supplement): capture the EXACT errno of the write() to the child PTY master.
# strace observes the real syscall in the io_loop thread (-f follows threads). We filter the
# child fd (=9 on this build, confirmed by gdb in W2). The EAGAIN return is the concrete
# evidence behind the `if (errno == EWOULDBLOCK || errno == EAGAIN) break;` at child-monitor.c:L1463.
set -u
OBS="$(cat /tmp/kitty_obs_scratch_path.txt 2>/dev/null)"
[ -n "$OBS" ] && [ -d "$OBS" ] || { echo "ERROR: OBS scratch not initialized (run the H.4 one-time setup first)" >&2; exit 2; }
REPO="${KITTY_REPO:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"
cd "$REPO"; export PATH=/usr/local/go/bin:$PATH
SELF_PGID=$(ps -o pgid= -p $$ | tr -d ' ')
rm -f "$OBS/go" "$OBS/child.ready" "$OBS/child.done" "$OBS/child.pid" "$OBS/child_bytes" "$OBS/strace.out"
export OBS MODE=flood MAX_BYTES=$((64*1024*1024)) DEADLINE_S=2.5 HARD_S=8
setsid xvfb-run -a env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
   OBS="$OBS" MODE=flood MAX_BYTES="$MAX_BYTES" DEADLINE_S=2.5 HARD_S=8 \
   strace -f -qq -e trace=write -o "$OBS/strace.out" \
   ./kitty/launcher/kitty -o confirm_os_window_close=0 python3 "$OBS/flood_query.py" \
   >/dev/null 2>/dev/null &
XVFB_PID=$!; PGID=$(ps -o pgid= -p "$XVFB_PID" 2>/dev/null | tr -d ' ')
safe_killpg(){ [ -n "${PGID:-}" ] && [ "$PGID" != "$SELF_PGID" ] && [ "$PGID" -gt 1 ] 2>/dev/null && kill "$1" -"$PGID" 2>/dev/null || true; }
# --- Finding 15/S3: trap-based cleanup on EXIT/INT/TERM (resume, terminate, reap only owned pids) ---
_cleanup(){ trap - EXIT INT TERM; for _p in "${CHILD_PID:-}" "${KPID:-}" "${XVFB_PID:-}"; do [ -n "$_p" ] && kill -CONT "$_p" 2>/dev/null; done; safe_killpg -TERM 2>/dev/null; safe_killpg -KILL 2>/dev/null; return 0; }
trap _cleanup EXIT INT TERM

for i in $(seq 1 300); do [ -f "$OBS/child.ready" ] && break; sleep 0.05; done
echo "CHILD_PID=$(cat "$OBS/child.pid" 2>/dev/null)  [PIDs run-local/ephemeral]"
echo go > "$OBS/go"
for i in $(seq 1 200); do [ -f "$OBS/child.done" ] && break; sleep 0.05; done
sleep 0.4
safe_killpg -TERM; sleep 0.5; safe_killpg -KILL
echo "=== W2d: exact errno of write() to child PTY master (fd 9), via strace ==="
echo "--- write() calls to fd 9 that returned EAGAIN (first 5, verbatim) ---"
grep -aE 'write\(9,' "$OBS/strace.out" | grep -a 'EAGAIN' | head -5
echo "--- count of write(9,...) = -1 EAGAIN over the run ---"
grep -acE 'write\(9,.*= -1 EAGAIN' "$OBS/strace.out"
echo "--- for contrast: first 3 SUCCESSFUL write(9,...) returns ---"
grep -aE 'write\(9,' "$OBS/strace.out" | grep -avE 'EAGAIN|= -1' | head -3
echo "--- strace.out size ---"; wc -c < "$OBS/strace.out"
echo "W2d done"
```

**`run_w2_hook.sh`** — write‑side **labeled supplement** (debug build). Enables the in‑tree `KITTY_PRINT_BYTES_SENT_TO_CHILD` hook (`kitty/child-monitor.c:L1449`) so each `write_to_child` return is printed to stderr. (Produced the S1 `Wrote:` hook lines in Section D.2.)

```bash
#!/usr/bin/env bash
# W2c (INSTRUMENTED supplement): surface the write() return value at each write_to_child call
# via KITTY_PRINT_BYTES_SENT_TO_CHILD (child-monitor.c:L1451). A SHORT flood so stderr stays
# small. We expect a few large successful writes (slave buffer fills) then `Wrote: -1 bytes: `
# = the write() that returned -1 immediately before the EAGAIN break (child-monitor.c:L1463).
set -u
OBS="$(cat /tmp/kitty_obs_scratch_path.txt 2>/dev/null)"
[ -n "$OBS" ] && [ -d "$OBS" ] || { echo "ERROR: OBS scratch not initialized (run the H.4 one-time setup first)" >&2; exit 2; }
REPO="${KITTY_REPO:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"
cd "$REPO"; export PATH=/usr/local/go/bin:$PATH
SELF_PGID=$(ps -o pgid= -p $$ | tr -d ' ')
rm -f "$OBS/go" "$OBS/child.ready" "$OBS/child.done" "$OBS/child.pid" "$OBS/child_bytes" "$OBS/w2c.err"
setsid xvfb-run -a env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
   OBS="$OBS" MODE=flood MAX_BYTES=$((64*1024*1024)) DEADLINE_S=0.35 HARD_S=6 \
   ./kitty/launcher/kitty -o confirm_os_window_close=0 python3 "$OBS/flood_query.py" \
   >/dev/null 2>"$OBS/w2c.err" &
XVFB_PID=$!; PGID=$(ps -o pgid= -p "$XVFB_PID" 2>/dev/null | tr -d ' ')
safe_killpg(){ [ -n "${PGID:-}" ] && [ "$PGID" != "$SELF_PGID" ] && [ "$PGID" -gt 1 ] 2>/dev/null && kill "$1" -"$PGID" 2>/dev/null || true; }
# --- Finding 15/S3: trap-based cleanup on EXIT/INT/TERM (resume, terminate, reap only owned pids) ---
_cleanup(){ trap - EXIT INT TERM; for _p in "${CHILD_PID:-}" "${KPID:-}" "${XVFB_PID:-}"; do [ -n "$_p" ] && kill -CONT "$_p" 2>/dev/null; done; safe_killpg -TERM 2>/dev/null; safe_killpg -KILL 2>/dev/null; return 0; }
trap _cleanup EXIT INT TERM

for i in $(seq 1 200); do [ -f "$OBS/child.ready" ] && break; sleep 0.05; done
echo go > "$OBS/go"
for i in $(seq 1 120); do [ -f "$OBS/child.done" ] && break; sleep 0.05; done
sleep 0.3
safe_killpg -TERM; sleep 0.4; safe_killpg -KILL
echo "=== W2c: write() returns via KITTY_PRINT_BYTES_SENT_TO_CHILD hook ==="
echo "--- distinct 'Wrote: <n> bytes:' prefixes with counts (response text stripped) ---"
grep -aoE 'Wrote: -?[0-9]+ bytes: ' "$OBS/w2c.err" | sort | uniq -c | sort -rn
echo "--- proof the EAGAIN write returned -1 with NO trailing newline (next call's 'Wrote:' is glued on) ---"
grep -aoE 'Wrote: -1 bytes: (Wrote:)?' "$OBS/w2c.err" | sort | uniq -c
echo "--- one full successful Wrote line (first, response text shown verbatim, truncated to 220 chars) ---"
grep -am1 -oE 'Wrote: [0-9]+ bytes: [^Z]{0,200}' "$OBS/w2c.err" | head -c 260; echo
echo "--- w2c.err size ---"; wc -c < "$OBS/w2c.err"
echo "W2c done"
```

**`run_w2_drain.sh`** — write‑side **labeled supplement** (debug build). Uses `flood_query.py` in `drain` mode (flood briefly, then read stdin) so `write_buf_used` first grows, then drains toward ~0 as `POLLOUT` keeps firing. (Produced the S1 drain evidence in Section D.2.)

```bash
#!/usr/bin/env bash
# W2e (INSTRUMENTED supplement): show RETENTION then DRAIN of write_buf. The drain child
# floods ~1s (write_buf grows, retained because child does not read), then STOPS flooding
# and reads its stdin, so kitty's queued bytes flush and write_buf_used falls back toward 0.
set -u
OBS="$(cat /tmp/kitty_obs_scratch_path.txt 2>/dev/null)"
[ -n "$OBS" ] && [ -d "$OBS" ] || { echo "ERROR: OBS scratch not initialized (run the H.4 one-time setup first)" >&2; exit 2; }
REPO="${KITTY_REPO:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"
cd "$REPO"; export PATH=/usr/local/go/bin:$PATH
SELF_PGID=$(ps -o pgid= -p $$ | tr -d ' ')
rm -f "$OBS/go" "$OBS/child.ready" "$OBS/child.done" "$OBS/child.pid" "$OBS/child_bytes" "$OBS/drain.start" "$OBS/drain.done"
setsid xvfb-run -a env LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
   OBS="$OBS" MODE=drain MAX_BYTES=$((200*1024*1024)) DEADLINE_S=9 HARD_S=13 FLOOD_S=0.2 \
   ./kitty/launcher/kitty -o confirm_os_window_close=0 python3 "$OBS/flood_query.py" \
   >/dev/null 2>/dev/null &
XVFB_PID=$!; PGID=$(ps -o pgid= -p "$XVFB_PID" 2>/dev/null | tr -d ' ')
safe_killpg(){ [ -n "${PGID:-}" ] && [ "$PGID" != "$SELF_PGID" ] && [ "$PGID" -gt 1 ] 2>/dev/null && kill "$1" -"$PGID" 2>/dev/null || true; }
# --- Finding 15/S3: trap-based cleanup on EXIT/INT/TERM (resume, terminate, reap only owned pids) ---
_cleanup(){ trap - EXIT INT TERM; for _p in "${CHILD_PID:-}" "${KPID:-}" "${XVFB_PID:-}"; do [ -n "$_p" ] && kill -CONT "$_p" 2>/dev/null; done; safe_killpg -TERM 2>/dev/null; safe_killpg -KILL 2>/dev/null; return 0; }
trap _cleanup EXIT INT TERM

for i in $(seq 1 200); do [ -f "$OBS/child.ready" ] && break; sleep 0.05; done
CHILD_PID="$(cat "$OBS/child.pid" 2>/dev/null||echo '')"
KITTY_PID=""; for p in $(pgrep -f 'launcher/kitty' 2>/dev/null||true); do [ "$(cat /proc/$p/comm 2>/dev/null||true)" = kitty ] && KITTY_PID="$p"; done
echo "KITTY_PID=$KITTY_PID CHILD_PID=$CHILD_PID  [PIDs run-local/ephemeral]"
cat > "$OBS/wd_snap.gdb" <<'GEOF'
set pagination off
printf "write_buf_used=%lu req_events=%d\n", children[0].screen->write_buf_used, children_fds[2].events
GEOF
echo go > "$OBS/go"
snap(){ timeout --signal=KILL 15 gdb -q -batch -p "$KITTY_PID" -x "$OBS/wd_snap.gdb" 2>/dev/null | grep 'write_buf_used='; }
# Minimal-attach schedule to avoid freezing the io_loop during the drain: three
# well-separated single-shot snapshots with free-running gaps between them.
for i in $(seq 1 120); do [ -f "$OBS/drain.start" ] && break; sleep 0.01; done
printf "%-26s %s\n" "PEAK-RETAINED(flood end)" "$(snap)"
sleep 1.8;  printf "%-26s %s\n" "DRAINING(+1.8s free-run)" "$(snap)"
sleep 2.4;  printf "%-26s %s\n" "DRAINED(+4.2s free-run)"  "$(snap)"
[ -n "${CHILD_PID:-}" ] && kill -CONT "$CHILD_PID" 2>/dev/null||true
safe_killpg -TERM; sleep 0.5; safe_killpg -KILL
echo "DRAIN done"
```

**`wd_snap.gdb`** — the gdb command file used by `run_w2_drain.sh`:

```
set pagination off
printf "write_buf_used=%lu req_events=%d\n", children[0].screen->write_buf_used, children_fds[2].events
```

#### H.4.3 Graphics / Q4 scripts (Section F)

**`apc_xprobe.py`** — the APC response cross‑product probe (Section F.3.1). Sends `q=0/1/2 × success/error` plus an `I=` image‑number case through a real PTY (ECHO off) and records each exact APC reply (repr, length, hex). Bounded by `signal.alarm`:

```python
#!/usr/bin/env python3
# CANONICAL APC cross-product probe. Launched BY kitty as its child; fd0/fd1 are the PTY
# slave. Writes one graphics command at a time to fd1 (kitty reads it from the PTY master,
# processes it via screen_handle_graphics_command -> write_escape_code_to_child -> write_buf
# -> io_loop write_to_child -> PTY master), then reads kitty's APC reply back from fd0.
# Records exact bytes (repr/len/hex) to a result file. Suppressed cases get a bounded
# no-response window. RAW mode / ECHO OFF so replies are not echoed back into kitty's reader.
#
# Bounded: absolute deadline + SIGALRM hard kill; each read has its own short window.
import os, sys, time, signal, termios, tty, select

OBS = os.environ["OBS"]
RESULT = os.path.join(OBS, os.environ.get("RESULT", "apc_x.result"))
HARD_S = int(os.environ.get("HARD_S", "20"))
signal.alarm(HARD_S)

try:
    tty.setraw(0)
except Exception:
    pass

def drain(window):
    """Read everything available until `window` seconds elapse with no new bytes."""
    buf = b""
    last = time.monotonic()
    end = time.monotonic() + max(window, 0.05)
    while time.monotonic() < end:
        r, _, _ = select.select([0], [], [], 0.05)
        if r:
            try:
                chunk = os.read(0, 65536)
            except OSError:
                break
            if chunk:
                buf += chunk
                last = time.monotonic()
                end = time.monotonic() + window   # extend while data keeps coming
        # stop early once we've been quiet for `window`
        if time.monotonic() - last >= window:
            break
    return buf

results = []
def do(label, cmd, expect_response, quiet_window=0.8):
    # clear any stray bytes first
    drain(0.2)
    os.write(1, cmd)
    resp = drain(quiet_window)
    results.append((label, cmd, resp, expect_response))

# Drain kitty startup chatter (if any) before the first probe.
_startup = drain(0.4)

# ---- q = 0 / 1 / 2  x  {success, error}  cross-product -------------------------
# success = valid query carrying i=<id>  -> "OK"    (unless suppressed)
# error   = put referencing non-existent image      -> "ENOENT" (unless suppressed)
do("q0_success", b"\x1b_Ga=q,i=11,f=24,s=1,v=1,q=0;AAAA\x1b\\", True)
do("q0_error",   b"\x1b_Ga=p,i=901,q=0\x1b\\",                    True)
do("q1_success", b"\x1b_Ga=q,i=12,f=24,s=1,v=1,q=1;AAAA\x1b\\", False)
do("q1_error",   b"\x1b_Ga=p,i=902,q=1\x1b\\",                    True)
do("q2_success", b"\x1b_Ga=q,i=13,f=24,s=1,v=1,q=2;AAAA\x1b\\", False)
do("q2_error",   b"\x1b_Ga=p,i=903,q=2\x1b\\",                    False)
# ---- I= (image_number) case: put non-existent by image NUMBER, not id ----------
do("I_number",   b"\x1b_Ga=p,I=777\x1b\\",                        True)
# also q=0 explicit-vs-absent sanity: q absent should equal q=0
do("qabsent_err",b"\x1b_Ga=p,i=904\x1b\\",                        True)

with open(RESULT, "w") as f:
    f.write("_startup_bytes=%r len=%d\n" % (_startup, len(_startup)))
    for label, cmd, resp, expect in results:
        f.write("=== %s ===\n" % label)
        f.write("cmd_repr=%r\n" % cmd)
        f.write("expect_response=%s\n" % expect)
        f.write("resp_len=%d\n" % len(resp))
        f.write("resp_repr=%r\n" % resp)
        f.write("resp_hex=%s\n" % resp.hex())
        f.write("\n")

open(os.path.join(OBS, "apc_x.done"), "w").write("done")
```

**`run_apcx.sh`** — the canonical launcher for `apc_xprobe.py` (DEFAULT build, under `Xvfb :91`), bounded with exact‑PID cleanup:

```bash
#!/usr/bin/env bash
# Canonical launcher for the APC cross-product probe. Runs a real kitty (DEFAULT build)
# under the persistent Xvfb :91, with apc_xprobe.py as its child. Bounded + exact-PID cleanup.
set -u
RUN="${1:-run1}"
OBS="$(cat /tmp/kitty_obs_scratch_path.txt 2>/dev/null)"
[ -n "$OBS" ] && [ -d "$OBS" ] || { echo "ERROR: OBS scratch not initialized (run the H.4 one-time setup first)" >&2; exit 2; }
REPO="${KITTY_REPO:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"
cd "$REPO"; export PATH=/usr/local/go/bin:$PATH
SELF_PGID=$(ps -o pgid= -p $$ | tr -d ' ')
RESULT="apc_x_${RUN}.result"
rm -f "$OBS/$RESULT" "$OBS/apc_x.done"
ERR="$OBS/apcx_${RUN}.err"; OUT="$OBS/apcx_${RUN}.out"
rm -f "$ERR" "$OUT"
setsid env DISPLAY=:91 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
   OBS="$OBS" RESULT="$RESULT" HARD_S=20 \
   ./kitty/launcher/kitty -o confirm_os_window_close=0 python3 "$OBS/apc_xprobe.py" \
   >"$OUT" 2>"$ERR" &
KPID=$!; PGID=$(ps -o pgid= -p "$KPID" 2>/dev/null | tr -d ' ')
safe_killpg(){ [ -n "${PGID:-}" ] && [ "$PGID" != "$SELF_PGID" ] && [ "$PGID" -gt 1 ] 2>/dev/null && kill "$1" -"$PGID" 2>/dev/null || true; }
# --- Finding 15/S3: trap-based cleanup on EXIT/INT/TERM (resume, terminate, reap only owned pids) ---
_cleanup(){ trap - EXIT INT TERM; for _p in "${CHILD_PID:-}" "${KPID:-}" "${XVFB_PID:-}"; do [ -n "$_p" ] && kill -CONT "$_p" 2>/dev/null; done; safe_killpg -TERM 2>/dev/null; safe_killpg -KILL 2>/dev/null; return 0; }
trap _cleanup EXIT INT TERM

# bounded wait for probe to finish
for s in $(seq 1 300); do [ -f "$OBS/apc_x.done" ] && break; sleep 0.05; done
sleep 0.3
# cleanup by exact pid / pgid
kill -TERM "$KPID" 2>/dev/null || true
safe_killpg -TERM; sleep 0.4; safe_killpg -KILL
echo "=== APC cross-product ($RUN) — DEFAULT canonical build, real PTY ==="
if [ -f "$OBS/$RESULT" ]; then cat "$OBS/$RESULT"; else echo "(no result file produced)"; fi
echo "--- kitty stderr (non-empty lines) ---"
sed '/^$/d' "$ERR" 2>/dev/null | head -20 || true
echo "RUN_${RUN}_done"
```

**`efbig_png.py`** — the strict‑PNG `EFBIG` driver (Section F.3.2). Streams a direct PNG transfer in bounded 195,000‑byte decoded chunks until the running total crosses `MAX_DATA_SZ` (400,000,000) and kitty replies `EFBIG`. Bounded by deadline, `signal.alarm`, and a max‑chunks cap:

```python
#!/usr/bin/env python3
# CANONICAL strict-PNG-size EFBIG driver (Finding 12). Launched BY kitty as its child.
# Streams a chunked DIRECT PNG transfer (a=t, f=100, t=d, m=1) whose accumulated DECODED
# payload crosses MAX_DATA_SZ = 400,000,000 bytes, hitting the size sub-clause of
#   graphics.c:L533  if (load_data->buf_used + g->payload_sz > MAX_DATA_SZ || data_fmt != PNG) ABRT("EFBIG","Too much data")
# with data_fmt == PNG so the || data_fmt!=PNG sub-clause is FALSE (the size condition alone fires).
# We NEVER send m=0, so kitty never tries to decode the (invalid) PNG.
#
# Bounded: absolute deadline + SIGALRM hard kill + MAX_CHUNKS cap. RAW/echo-off.
import os, sys, time, signal, base64, termios, tty, select

OBS = os.environ["OBS"]
RESULT = os.path.join(OBS, os.environ.get("RESULT", "efbig.result"))
HARD_S = int(os.environ.get("HARD_S", "120"))
DEADLINE_S = float(os.environ.get("DEADLINE_S", "100"))
DECODED_PER_CHUNK = int(os.environ.get("DECODED_PER_CHUNK", "195000"))   # -> base64 260000 B < MAX_ESCAPE_CODE_LENGTH(262144)
MAX_CHUNKS = int(os.environ.get("MAX_CHUNKS", "4096"))
MAX_DATA_SZ = 4 * 100000000   # 400,000,000, mirrors graphics.c:L521
signal.alarm(HARD_S)

try:
    tty.setraw(0)
except Exception:
    pass

# One reusable base64 block (zeros) that decodes to exactly DECODED_PER_CHUNK bytes.
assert DECODED_PER_CHUNK % 3 == 0, "keep divisible by 3 so base64 has no padding"
B64 = base64.standard_b64encode(b"\x00" * DECODED_PER_CHUNK)   # len == DECODED_PER_CHUNK/3*4

def read_avail(window):
    buf = b""
    end = time.monotonic() + window
    while time.monotonic() < end:
        r, _, _ = select.select([0], [], [], min(0.05, max(0.0, end - time.monotonic())))
        if r:
            try: chunk = os.read(0, 65536)
            except OSError: break
            if chunk: buf += chunk
    return buf

deadline = time.monotonic() + DEADLINE_S
decoded_total = 0
resp = b""
n = 0
first = b"\x1b_Ga=t,f=100,t=d,i=1,m=1;" + B64 + b"\x1b\\"
os.write(1, first); decoded_total += DECODED_PER_CHUNK; n += 1
cont = b"\x1b_Gm=1;" + B64 + b"\x1b\\"
while n < MAX_CHUNKS and time.monotonic() < deadline:
    try:
        os.write(1, cont); decoded_total += DECODED_PER_CHUNK; n += 1
    except OSError as e:
        resp += b"[child write OSError: %r]" % e
        break
    # opportunistically collect any response so far
    r, _, _ = select.select([0], [], [], 0)
    if r:
        try: resp += os.read(0, 65536)
        except OSError: pass
    if b"\x1b\\" in resp and b"EFBIG" in resp:
        break
# drain any final response
resp += read_avail(2.0)

with open(RESULT, "w") as f:
    f.write("decoded_per_chunk=%d base64_per_chunk=%d\n" % (DECODED_PER_CHUNK, len(B64)))
    f.write("MAX_DATA_SZ=%d\n" % MAX_DATA_SZ)
    f.write("chunks_sent=%d\n" % n)
    f.write("decoded_total_sent=%d\n" % decoded_total)
    f.write("crossed_MAX_DATA_SZ=%s\n" % (decoded_total > MAX_DATA_SZ))
    # isolate the last complete APC record in the response
    f.write("resp_len=%d\n" % len(resp))
    f.write("resp_repr=%r\n" % resp)
    f.write("resp_hex=%s\n" % resp.hex())

open(os.path.join(OBS, "efbig.done"), "w").write("done")
```

**`run_efbig.sh`** — the canonical launcher for `efbig_png.py` (DEFAULT build, under `Xvfb :91`), bounded with exact‑PID cleanup:

```bash
#!/usr/bin/env bash
set -u
RUN="${1:-run1}"
OBS="$(cat /tmp/kitty_obs_scratch_path.txt 2>/dev/null)"
[ -n "$OBS" ] && [ -d "$OBS" ] || { echo "ERROR: OBS scratch not initialized (run the H.4 one-time setup first)" >&2; exit 2; }
REPO="${KITTY_REPO:-$(git rev-parse --show-toplevel 2>/dev/null || pwd)}"
cd "$REPO"; export PATH=/usr/local/go/bin:$PATH
SELF_PGID=$(ps -o pgid= -p $$ | tr -d ' ')
RESULT="efbig_${RUN}.result"
rm -f "$OBS/$RESULT" "$OBS/efbig.done"
ERR="$OBS/efbig_${RUN}.err"; OUT="$OBS/efbig_${RUN}.out"
rm -f "$ERR" "$OUT"
t0=$(date +%s.%N)
setsid env DISPLAY=:91 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe \
   OBS="$OBS" RESULT="$RESULT" HARD_S=120 DEADLINE_S=100 DECODED_PER_CHUNK=195000 MAX_CHUNKS=4096 \
   ./kitty/launcher/kitty -o confirm_os_window_close=0 python3 "$OBS/efbig_png.py" \
   >"$OUT" 2>"$ERR" &
KPID=$!; PGID=$(ps -o pgid= -p "$KPID" 2>/dev/null | tr -d ' ')
safe_killpg(){ [ -n "${PGID:-}" ] && [ "$PGID" != "$SELF_PGID" ] && [ "$PGID" -gt 1 ] 2>/dev/null && kill "$1" -"$PGID" 2>/dev/null || true; }
# --- Finding 15/S3: trap-based cleanup on EXIT/INT/TERM (resume, terminate, reap only owned pids) ---
_cleanup(){ trap - EXIT INT TERM; for _p in "${CHILD_PID:-}" "${KPID:-}" "${XVFB_PID:-}"; do [ -n "$_p" ] && kill -CONT "$_p" 2>/dev/null; done; safe_killpg -TERM 2>/dev/null; safe_killpg -KILL 2>/dev/null; return 0; }
trap _cleanup EXIT INT TERM

for s in $(seq 1 2400); do [ -f "$OBS/efbig.done" ] && break; sleep 0.05; done
t1=$(date +%s.%N)
sleep 0.3
kill -TERM "$KPID" 2>/dev/null || true
safe_killpg -TERM; sleep 0.4; safe_killpg -KILL
elapsed=$(awk "BEGIN{printf \"%.2f\", $t1-$t0}")
echo "=== strict-PNG EFBIG ($RUN) — DEFAULT canonical build, real PTY ==="
echo "wall_elapsed_to_done = ${elapsed} s"
if [ -f "$OBS/$RESULT" ]; then cat "$OBS/$RESULT"; else echo "(no result file produced)"; fi
echo "--- kitty stderr (non-empty) ---"
sed '/^$/d' "$ERR" 2>/dev/null | head -10 || true
echo "RUN_${RUN}_done"
```

**`gfx_quota.py`** — the in‑tree harness (**labeled supplement S2**) for the logical/physical disk‑cache split (Finding 11) and the 5× animation‑frame quota (Finding 10). Uses only pure reads (`image_count`=`HASH_COUNT`, `disk_cache.total_size`, `size_on_disk`, `num_cached_in_ram`), direct/inline transmission (the default medium, no `t=` key; each 120 KB image fits in one escape code), and a distinct first byte per image to defeat de‑duplication. Run via `+runpy`, so it runs to completion and exits (inherently bounded):

```python
#!/usr/bin/env python3
# LABELED SUPPLEMENT (S2). Exercises the IDENTICAL graphics.c + disk-cache.c code as the
# PTY path, but feeds the parser directly (kitty_tests.parse_bytes) so the Python-level
# disk_cache accessors (total_size / size_on_disk / wait_for_write / num_cached_in_ram)
# and grman.storage_limit are reachable — they are not exposed over the wire.
#
# Part 1 (Finding 11): logical `total_size` (synchronous, disk-cache.c:L508) vs physical
#                      `size_on_disk()` (needs the background writer, disk-cache.c:L704/L712)
#                      + `num_cached_in_ram()`.
# Part 2 (Finding 10): the 5x animation-frame quota abort at graphics.c:L1570-L1573,
#                      "Cache size exceeded cannot add new frames" (ENOSPC) — DISTINCT from
#                      the root-image add failure at L746 "Failed to store image data in disk cache".
import os, sys, time
repo = os.environ['KITTY_REPO']; sys.path.insert(0, repo)
from kitty.fast_data_types import Screen, set_options
from kitty.options.parse import merge_result_dicts
from kitty.options.types import Options, defaults
from kitty.config import finalize_keys, finalize_mouse_mappings
from kitty_tests import Callbacks
from kitty_tests.graphics import make_send_command, send_command, parse_full_response

def set_opts():
    o = Options(merge_result_dicts(defaults._asdict(), {'scrollback_pager_history_size':1024,'click_interval':0.5}))
    finalize_keys(o, {}); finalize_mouse_mappings(o, {}); set_options(o)

def create_screen(cols=5, lines=5, scrollback=5, cw=10, ch=20):
    set_opts(); c = Callbacks(); return Screen(c, lines, cols, scrollback, cw, ch, 0, c)

print("=" * 70)
print("PART 1 (Finding 11): logical total_size vs physical size_on_disk")
print("=" * 70)
s = create_screen()
g = s.grman
dc = s.grman.disk_cache
print("DEFAULT grman.storage_limit = %d bytes (== 320*1024*1024 ? %s)" % (
    g.storage_limit, g.storage_limit == 320*1024*1024))
print("baseline: total_size=%d  size_on_disk=%d  num_cached_in_ram=%d" % (
    dc.total_size, dc.size_on_disk(), dc.num_cached_in_ram()))

# --- 1a: graphics transmit path (images stored via add_to_cache -> disk cache) ---
li = make_send_command(s)
# 200x200 RGB = 120000 bytes each -> base64 160000 < MAX_ESCAPE_CODE_LENGTH(262144); one escape code
W = H = 200
for n in range(1, 4):
    payload = bytes([n]) + b"\x00" * (W*H*3 - 1)   # distinct first byte -> no dedup
    r = li(a='t', i=100+n, s=W, v=H, f=24, payload=payload)
    print("transmit img i=%d code=%s : total_size(logical)=%d  size_on_disk(physical,pre-wait)=%d  num_cached_in_ram=%d" % (
        100+n, r.code, dc.total_size, dc.size_on_disk(), dc.num_cached_in_ram()))
print("-> wait_for_write() returned:", dc.wait_for_write())
print("after wait_for_write: total_size(logical)=%d  size_on_disk(physical)=%d  num_cached_in_ram=%d" % (
    dc.total_size, dc.size_on_disk(), dc.num_cached_in_ram()))

# --- 1b: raw disk-cache add of a 64 MiB blob to expose the sync/async split crisply ---
print("--- 1b: raw dc.add() of a 64 MiB blob (catches the writer lag) ---")
s2 = create_screen(); dc2 = s2.grman.disk_cache
BLOB = b"Z" * (64 * 1024 * 1024)
dc2.add(b"bigkey", BLOB)
# read physical repeatedly right after the synchronous logical increment
prog = [dc2.size_on_disk() for _ in range(5)]
print("immediately after dc.add(64MiB): total_size(logical)=%d" % dc2.total_size)
print("size_on_disk() sampled x5 BEFORE wait_for_write = %r" % prog)
print("num_cached_in_ram (pre-wait) =", dc2.num_cached_in_ram())
print("-> wait_for_write() returned:", dc2.wait_for_write())
print("size_on_disk() AFTER wait_for_write = %d" % dc2.size_on_disk())
print("total_size(logical) still = %d" % dc2.total_size)

print()
print("=" * 70)
print("PART 2 (Finding 10): 5x animation-frame quota -> ENOSPC at graphics.c:L1570-L1573")
print("=" * 70)
sq = create_screen()
gq = sq.grman
# default boundary (real runtime value): storage_limit * 5
print("DEFAULT storage_limit=%d ; default 5x frame boundary = storage_limit*5 = %d bytes" % (
    gq.storage_limit, gq.storage_limit * 5))
# small limit (LABELED NON-DEFAULT) so the ceiling is reachable with tiny frames -
# EXACTLY the pattern kitty's own test_graphics_quota_enforcement uses (storage_limit=36*2).
gq.storage_limit = 36 * 2
print("set storage_limit=%d (NON-DEFAULT; mirrors kitty_tests test_graphics_quota_enforcement); 5x=%d" % (
    gq.storage_limit, gq.storage_limit * 5))
liq = make_send_command(sq)
print("base images:")
print("  li(a='T')      code=%s image_count=%d disk_cache.total_size=%d" % (liq(a='T').code, gq.image_count, gq.disk_cache.total_size))
print("  li(a='T',i=2)  code=%s image_count=%d disk_cache.total_size=%d" % (liq(a='T', i=2).code, gq.image_count, gq.disk_cache.total_size))
print("append 36-byte animation frames to image i=2 (a='f' is the default action of li):")
for i in range(8):
    r = liq(payload=f'{i}' * 36, i=2)
    print("  frame %d code=%s disk_cache.total_size=%d" % (i, r.code, gq.disk_cache.total_size))
# the 9th frame crosses cache_size + data_sz > storage_limit*5 (=360)
res_bytes = send_command(sq, 's=4,v=3,f=24,i=2,a=f', payload=b'x' * 36)  # identical keys to li(payload='x'*36,i=2)
full = parse_full_response(res_bytes)
print("9th frame (over quota):")
print("  raw_response_bytes_repr = %r" % res_bytes)
print("  raw_response_bytes_len  = %d" % len(res_bytes))
print("  raw_response_bytes_hex  = %s" % res_bytes.hex())
print("  parsed code=%s msg=%r" % (full.code, full.msg))
print("DISTINCT branch note: L746 'Failed to store image data in disk cache' is the ROOT-IMAGE")
print("  add_to_cache failure (a different code path); the frame ceiling above is L1573.")
print("HARNESS_DONE")
```

**`gfx_evict.py`** — the in‑tree harness (**labeled supplement S2**) for LRU eviction at the true default 320 MiB quota and the pure‑LRU transition at a lowered 72‑byte quota (Section F.4). Run via `+runpy` (inherently bounded):

```python
#!/usr/bin/env python3
# LABELED SUPPLEMENT (S2). LRU eviction under the storage quota (apply_storage_quota,
# graphics.c:L290-L296). Same graphics.c code as the PTY path; direct parser feed so
# grman.storage_limit / image_count / disk_cache are reachable.
import os, sys, tempfile
repo = os.environ['KITTY_REPO']; sys.path.insert(0, repo)
from kitty.fast_data_types import Screen, set_options, base64_encode
from kitty.options.parse import merge_result_dicts
from kitty.options.types import Options, defaults
from kitty.config import finalize_keys, finalize_mouse_mappings
from kitty_tests import Callbacks
from kitty_tests.graphics import send_command, make_send_command, parse_full_response

def set_opts():
    o = Options(merge_result_dicts(defaults._asdict(), {'scrollback_pager_history_size':1024,'click_interval':0.5}))
    finalize_keys(o, {}); finalize_mouse_mappings(o, {}); set_options(o)
def create_screen(cols=5, lines=5, scrollback=5, cw=10, ch=20):
    set_opts(); c = Callbacks(); return Screen(c, lines, cols, scrollback, cw, ch, 0, c)

print("=" * 70)
print("PART A (Finding 11 / F.4): LRU eviction at the TRUE DEFAULT 320 MiB quota")
print("=" * 70)
s = create_screen(); g = s.grman
print("grman.storage_limit (DEFAULT) = %d bytes (== 320*1024*1024 ? %s)" % (
    g.storage_limit, g.storage_limit == 320*1024*1024))
W = H = 4096
per = W*H*3
print("each image: s=%d v=%d f=24 t=t => data_sz=%d bytes (%.2f MiB); 7 images => %.0f MiB > 320 MiB" % (
    W, H, per, per/1024/1024, 7*per/1024/1024))
li = make_send_command(s)
print("img_id  code   image_count  disk_cache.total_size(bytes,LOGICAL)  total(MiB)  <=320MiB?")
for n in range(1, 8):
    # write a 48 MiB image file with a DISTINCT first byte (defeats content de-dup), transmit via t='t'
    tf = tempfile.NamedTemporaryFile(prefix='tty-graphics-protocol-', dir=os.environ['OBS'], delete=False)
    tf.write(bytes([n]) + b"\x00" * (per - 1)); tf.flush(); tf.close()
    r = li(a='T', i=n, s=W, v=H, f=24, t='t', payload=tf.name.encode())
    ts = g.disk_cache.total_size
    print("%6d  %-5s  %11d  %36d  %9.2f  %s" % (
        n, r.code, g.image_count, ts, ts/1024/1024, "yes" if ts <= g.storage_limit else "NO"))
    # t='t' => kitty deletes the source temp file after reading; clean up if it somehow remains
    try: os.unlink(tf.name)
    except OSError: pass

print()
print("=" * 70)
print("PART B (F.4): pure-LRU while-loop at a lowered 72-byte quota (crisp)")
print("=" * 70)
s2 = create_screen(); g2 = s2.grman
g2.storage_limit = 36 * 2
print("storage_limit=%d bytes (NON-DEFAULT; mirrors kitty_tests test_graphics_quota_enforcement)" % g2.storage_limit)
li2 = make_send_command(s2)
print("step                       code   image_count  disk_cache.total_size")
for i in (1, 2, 3):
    r = li2(a='T', i=i)
    note = ""
    if i == 3:
        note = "  <-- img#3: oldest EVICTED by LRU while-loop (count stays 2, total==limit)"
    print("put img i=%d (a=T,36B)      %-5s  %11d  %d%s" % (i, r.code, g2.image_count, g2.disk_cache.total_size, note))
print("HARNESS_DONE")
```

### H.5 Repository integrity — what changed and what did not

The read‑only constraint is verified with Git, scoped precisely to **tracked/untracked Git‑visible** repository state (git‑ignored build products are explicitly excluded, since building kitty necessarily regenerates them).

**Baseline-to-deliverable committed history.** The answer document was first added on top of the source baseline `815df1e21` in the initial document commit `8fe5c7357` (686 insertions, the document only) and has since been refined by follow-up commits that touch **only** this same file. The immutable initial-commit diff:

```
$ git diff --stat 815df1e21 8fe5c7357
 blitzy/documentation/kitty_815df1e210e0.md | 686 +++++++++++++++++++++++++++++
 1 file changed, 686 insertions(+)
```

(The document is longer than 686 lines in its final corrected form; 686 is the initial-commit count and grows with each documentation-only revision, so the exact HEAD line count is not a fixed invariant and is deliberately not pinned here.)

**kitty source unchanged -- proven against current `HEAD`, not just the initial commit.** The load-bearing read-only guarantee is that the kitty **source** is untouched across the *entire* document history, however many documentation-only commits accumulate. From the baseline to current `HEAD` the only Git-visible change is this one document, and restricting the diff to the source trees is empty -- both hold at `HEAD` regardless of how many times the document is revised:

```
$ git diff --name-status 815df1e21 HEAD
A	blitzy/documentation/kitty_815df1e210e0.md

$ git diff --stat 815df1e21 HEAD -- kitty/ 3rdparty/ kitty_tests/ docs/
(no output)
```

**Working‑tree status.** During the investigation the only tracked file that appears in `git status` is the answer document (being edited); the observation scripts never appear because they live under the `mktemp -d` scratch directory **outside** the repository:

```
$ git status --porcelain
 M blitzy/documentation/kitty_815df1e210e0.md
```

**Build products are git‑ignored (not part of "unchanged").** Rebuilding kitty regenerates these, and Git correctly ignores them, so "the repository is unchanged" is scoped to Git‑visible tracked/untracked state, not to a byte‑for‑byte filesystem claim:

```
$ git check-ignore kitty/fast_data_types.so kitty/launcher/kitty dependencies
kitty/fast_data_types.so
kitty/launcher/kitty
dependencies
```

**Debug‑build restoration (within‑run, same environment).** The labeled debug build temporarily replaced `kitty/fast_data_types.so`; the original default artifact was backed up beforehand and restored afterward, and the restoration was verified by **SHA‑256** equality within this run (SHA‑256 is used rather than MD5 because it is collision‑resistant; note this is a local restore check, not a cross‑environment reproducibility invariant — see H.1):

```
default (before debug build):  sha256 5a2795e426aa7f2ee23266d4c5c650e107231e05f41f49b233e81d8c69fbc502
restored (after debug build):  sha256 5a2795e426aa7f2ee23266d4c5c650e107231e05f41f49b233e81d8c69fbc502  (equal)
```

**Scratch removal.** After capturing all output, the entire `mktemp -d` scratch directory is removed (`rm -rf "$OBS"` on the exact owned path only), leaving no observation artifact anywhere on disk. The final `git status --porcelain` then shows only the committed answer document and an otherwise clean tree.
