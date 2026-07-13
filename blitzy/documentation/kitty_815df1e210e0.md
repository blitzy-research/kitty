# How kitty Moves Data Between Its C Core and Python Kittens Under Concurrent Load

A runtime-grounded investigation of the clipboard C↔Python boundary, the threading/concurrency
model, scrollback-scan interference, object ownership, and subtle races.

- **Repository:** `kitty` (terminal emulator), branch `kitty_815df1e210e0`, HEAD **`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`**.
- **Build/run host:** container image `swe-atlas-kitty:canonical` (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`), `/app` = the checkout at the HEAD above; Python 3.12.3, Go 1.23.4, gcc 13.3.0, Xvfb on `DISPLAY=:99`.
- **kitty version at runtime:** `kitty 0.35.2 created by Kovid Goyal`.

### How to read this document

This report was produced by **building and running kitty first**, then writing every claim from the
captured output. Each behavioral claim carries four things:

1. the **exact command** that produced the evidence,
2. its **complete, unedited output** in a fenced block,
3. a **`file:line` citation** into the checkout at HEAD `815df1e210e0`, and
4. an explicit **`[observed]`** (confirmed at runtime) or **`[inferred]`** (code-derived, not confirmed at runtime) label.

All clipboard boundary evidence originates from the **canonical entry point**: OSC 52 / OSC 5522
escape codes written to a **real child PTY** and parsed by the **live VT parser**. Where a
non-canonical interface (remote control, a direct Python call, or a focused C harness) is used for
cross-checking, it is labeled **non-canonical** in place. Temporary observation scripts lived under
`/tmp/obs/` in the container (outside the `/app` checkout) and were removed afterward; the checkout
was verified byte-for-byte unchanged (Section 11).

---

## 1. Summary — direct answers to each objective

**O1 — Clipboard C→Python transfer, small and very large.** Clipboard bytes cross the boundary as a
**zero-copy `memoryview`** created over the live 1 MB parser buffer in `dispatch_osc`
[kitty/vt-parser.c:457] and handed to the Python method `clipboard_control` [kitty/screen.c:2305] via
the `CALLBACK` macro [kitty/screen.c:87]. **Small** payloads (a base64 escape that fits in the buffer)
arrive as **one** dispatch (`clipboard_control 52 …`). **Very large** payloads that exceed the buffer
arrive as **multiple partial dispatches** (`clipboard_control -52 …`, `is_partial=True`) — chunking
that is enabled once an escape exceeds `MAX_ESCAPE_CODE_LENGTH` = `BUF_SZ/4u` = 256 KB
[kitty/vt-parser.c:21] and is bounded in practice by the 1 MB buffer `BUF_SZ` [kitty/vt-parser.c:18].
On the Python side the chunks accumulate into a `WriteRequest`'s `Tempfile` [kitty/clipboard.py:26]
that **rolls over from in-memory `io.BytesIO` to an on-disk temporary file at 16 MB**
[kitty/clipboard.py:237]. All of this was observed at runtime (single dispatch, partial-chunk
dispatches, and the on-disk rollover via `strace`). `[observed]`

**O2 — Behavior "in practice" when other parts of the system are busy.** PTY reading is split from
parsing: the **`KittyChildMon`** I/O thread [kitty/child-monitor.c:1489] reads PTY bytes into the
shared 1 MB buffer under a per-parser mutex, while **all parsing and all Python callbacks (including
`clipboard_control`) run on the MAIN thread under the GIL**. Bursty output is **coalesced** by
`input_delay` (default **3 ms** [kitty/options/definition.py:878]) rather than parsed byte-by-byte —
2000 escape codes in one write were parsed in a **single** parse pass. When the buffer fills, the I/O
poll **disables read events** (`POLLIN` backpressure [kitty/child-monitor.c:1501]) — observed as the
child fd's poll `events` flipping between `POLLIN` and `0`. A clipboard round-trip under heavy flood
rises from an idle **~3.4 ms** to a **~4.6 ms** median. `[observed]`

**O3 — Effect of an expensive main-thread operation (large scrollback scan).** A scrollback text scan
(`as_text`/`text_for_range` built on `as_text_generic` [kitty/line.c:874]) runs **on the same MAIN
thread, under the GIL**, as clipboard/kitten dispatch — confirmed by gdb showing `as_text_generic` on
Thread 1 reached from the same `process_global_state` tick [kitty/child-monitor.c:1224] that runs
clipboard dispatch. Consequently a clipboard read fired **during** a ~400 ms scan is delayed from the
idle **~3.5 ms** floor to a **~90–130 ms median (≈26–36×)**, then returns to baseline. Memory:
scrollback grows RAM history segments (`add_segment` [kitty/history.c:18]) at **~2.2 KiB/line**
(100 k lines ≈ +220 MiB), the scan transiently allocates **~46 MiB** (the extracted text as a Python
list of line strings), the compressed pager-history ring buffer is **off by default**
[kitty/options/definition.py:406], and the background `DiskCacheWrite` thread [kitty/disk-cache.c:342]
is a **graphics** offload, not used by scrollback text. `[observed]`

**O4 — Where timing, concurrency, and object ownership start to matter.** The `memoryview` handed to
Python is a **read-only borrow** of the live parser buffer whose validity is bounded by the C dispatch
scope (`RAII_PyObject`/`START_DISPATCH`…`END_DISPATCH` [kitty/vt-parser.c:460-464]); the object was
confirmed at runtime to be a `PyMemoryView` (its `ob_type` equals `&PyMemoryView_Type`). Python's
correctness depends on **copying out during the synchronous callback** — `add_base64_data`
[kitty/clipboard.py:271] base64-decodes into the `Tempfile` and copies the non-4-aligned remainder via
`bytes(...)`. Retaining the borrow past the dispatch scope is a use-after-free (demonstrated as an
`Invalid read` under valgrind on the identical `PyMemoryView_FromMemory` API). The producer/consumer
hand-off is a **mutex promotion** `read.sz += write.pending` under the lock [kitty/vt-parser.c:1421],
with the lock **released during the parse** so the I/O thread can keep filling. `[observed]`

**O5 — Subtle races that emerge only under real runtime conditions.** Running the **real** parser
two-thread hand-off under **ThreadSanitizer** (a faithful harness driving the exact
`vt_parser_create_write_buffer`/`vt_parser_commit_write`/`parse_worker` functions from a non-GIL I/O
thread and a GIL-holding main thread) found **zero data races** across 5 completed runs — while a
deliberate positive-control race (the harness's own `g_stop` flag) *was* flagged, proving the detector
was live. This confirms the lock discipline: every shared scalar (`write.pending`, `write.offset`,
`read.sz`) is accessed under `self->lock`, and the buffer bytes are touched only in disjoint,
lock-ordered regions. The **`is_self_offer`** reentrancy (kitty reading a clipboard it owns) was
captured on the canonical OSC 52 read path (`write_clipboard_data` called with `data==NULL` →
`RuntimeError('is_self_offer')` [kitty/glfw.c:2183], caught in `clipboard.py:108-118`). The
**`DiskCacheWrite`** writer thread was observed live after transmitting a graphics image; its
writer/reader interactions are serialized by a single per-cache mutex with the disk I/O done on a
copied-out buffer outside the lock. Full-GUI race detection under helgrind/TSan was **environment
blocked** (documented in §8) so the detector was run on the real functions via the harness. `[observed]`

---

## 2. Environment, build, and invocation

All commands were run inside the container via `docker exec kitty-canonical bash -lc '…'` with
`DISPLAY=:99` (Xvfb). The destination document lives in a separate checkout; the source checkout in
`/app` was never modified.

### 2.1 Commit under test `[observed]`

```
$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
$ git status --porcelain
            (empty — clean tree)
```

### 2.2 Canonical build `[observed]`

The `kitty/launcher/kitty` binary and the `fast_data_types` C extension are produced by the default
build. A full clean rebuild was forced to capture confirming output:

```
$ rm -rf build kitty/fast_data_types.so kitty/launcher/kitty
$ python3 setup.py
… (122 compile steps + 5 link steps) …
[5/5] Linking launcher ...
 done
                                        # exit 0, ~16 s
$ ls -l kitty/fast_data_types.so kitty/launcher/kitty
-rwxr-xr-x 1 root 1001 1213072 … kitty/fast_data_types.so
-rwxr-xr-x 1 root 1001   36224 … kitty/launcher/kitty
$ DISPLAY=:99 kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

Note on `--dump-commands`: the build log shows `kitty/vt-parser.c` compiled **twice** — the second
compile applies the `DUMP_COMMANDS` macro, per `setup.py:722`
(`if src=='kitty/vt-parser-dump.c': return 'kitty/vt-parser.c', [], ['DUMP_COMMANDS']`). `DUMP_COMMANDS`
is therefore a **compile-time macro on `kitty/vt-parser.c`**, not a separate source file — there is no
`kitty/vt-parser-dump.c` on disk. The runtime `--dump-commands` flag routes through the resulting
`parse_worker_dump`, so no special build is required to capture dispatch. `[observed]`

### 2.3 ThreadSanitizer build (for O5) `[observed]`

`setup.py`'s `--sanitize` wires **AddressSanitizer + UndefinedBehaviorSanitizer**, not
ThreadSanitizer (`get_sanitize_args` returns `['-fsanitize=address,undefined', …]`
[kitty/../setup.py:380]). ThreadSanitizer was instead enabled through env `CFLAGS`/`LDFLAGS`, which
`setup.py` appends (`env_cflags`/`env_ldflags` at setup.py:494,496,515,522):

```
$ CFLAGS="-fsanitize=thread -fno-omit-frame-pointer -g" \
  LDFLAGS="-fsanitize=thread -no-pie" \
  python3 setup.py build --debug          # --debug disables LTO
$ ldd kitty/fast_data_types.so | grep -i tsan
        libtsan.so.2 => /lib/x86_64-linux-gnu/libtsan.so.2 (0x…)
```

Because `--debug` disables link-time optimization, the otherwise-inlined parser buffer functions
survive as callable symbols in this build — the key that made the faithful harness in §8 possible.
The canonical (non-TSan) build was restored afterward (`python3 setup.py`), and the tree remained
git-clean throughout (all build artifacts are git-ignored).

### 2.4 Diagnostic tooling `[observed]`

Installed in-container: `gdb 15.1`, `strace`, `valgrind 3.22` (helgrind), `xxd`, `od`. PyPI is
blocked in this environment, so `py-spy` was unavailable and **gdb** was used for stack inspection.
Thread names were read from `/proc/<pid>/task/*/comm`. A hard constraint shaped every gdb experiment:
`ptrace_scope=1` with no `CAP_SYS_PTRACE` and a read-only `/proc/sys`, so **gdb had to launch kitty as
its own child** (it cannot attach to an already-running kitty). Kitty's launcher runs Python embedded
in-process, so `kitty sh -c '…'` does not re-exec the main process and gdb stays attached to it.

---

## 3. Threading and data-flow model (confirmed at runtime)

### 3.1 Thread topology `[observed]`

Thread names come from `/proc/<pid>/task/*/comm` and from gdb `info threads`. The core threads are:

| Thread name | Role | Source anchor |
|---|---|---|
| `kitty` (Thread 1) | **MAIN thread**: main loop, VT parsing, **all Python callbacks** (incl. `clipboard_control`), screen mutation, render | `process_global_state` [kitty/child-monitor.c:1224] |
| `KittyChildMon` | **I/O thread**: polls child PTY fds, reads bytes into the shared 1 MB buffer | [kitty/child-monitor.c:1489], `read_bytes` [kitty/child-monitor.c:1337] |
| `KittyPeerMon` | remote-control socket servicing (only when `--listen-on`/`allow_remote_control`) | [kitty/child-monitor.c:1808] |
| `DiskCacheWrite` | background image (graphics) disk offload; **not** scrollback text | `write_loop`/`set_thread_name` [kitty/disk-cache.c:340,342] |
| `KittyWriteStdin` | transient, per stdin-write to a child | [kitty/child-monitor.c:967] |
| `llvmpipe-N`, `kitty:disk$0` | Mesa software-GL rasterizer workers (environment artifact of headless llvmpipe) | (not kitty core) |

The single most important fact for O2–O5: **parse + Python dispatch + screen mutation + render are all
serialized on the MAIN thread under the GIL**, while the I/O thread only fills the shared buffer.

### 3.2 The C→Python dispatch chain (full backtrace) `[observed]`

A gdb breakpoint on `clipboard_control`, hit while a child emitted a canonical OSC 52 write, shows the
entire path and the thread it runs on:

```
Thread 1 "kitty" (LWP 40135) hit breakpoint: clipboard_control
#0  clipboard_control            (kitty/fast_data_types.so)   [screen.c:2305]
#1  dispatch_osc                 (kitty/fast_data_types.so)   [vt-parser.c:457]
#2  run_worker.lto_priv          (kitty/fast_data_types.so)   [vt-parser.c:1417]
#3  do_parse                     (kitty/fast_data_types.so)
#4  process_global_state         (kitty/fast_data_types.so)   [child-monitor.c:1224]  <- MAIN-loop tick
#5  dispatchTimers               (glfw-x11.so)
#6  glfwRunMainLoop              (glfw-x11.so)
#7  main_loop
#8..#24 libpython3.12 (_PyEval_EvalFrameDefault … Py_RunMain) <- under the interpreter/GIL
#25 main
```

Simultaneously the I/O thread was parked in `poll`:

```
Thread 67 "KittyChildMon" (LWP 40203)  __GI___poll(fds=<children_fds>, nfds=3, timeout=-1)
```

This proves: the main tick `process_global_state` [child-monitor.c:1224] → `do_parse` → `run_worker`
[vt-parser.c:1417] → `dispatch_osc` [vt-parser.c:457] → `clipboard_control` [screen.c:2305] runs on
**Thread 1**, under the Python interpreter (GIL), while `KittyChildMon` independently polls the PTY.
(Note: `process_global_state` is at **child-monitor.c:1224**, correcting a stale reference to `:1222`.)

### 3.3 Data path

```
   Application / shell inside kitty  ── emits OSC 52 / OSC 5522 ──► child PTY
                                                                       │
   ┌───────────────────────────── KittyChildMon (I/O thread) ─────────┼──────────────┐
   │ read_bytes [child-monitor.c:1337]                                 ▼              │
   │   vt_parser_create_write_buffer  (under self->lock: offset = read.sz+write.pending)
   │   read(fd, buf, avail)           (OUTSIDE the lock)                               │
   │   vt_parser_commit_write         (under self->lock: write.pending += n)           │
   │   POLLIN disabled when buffer full  [child-monitor.c:1501] ◄── backpressure       │
   └───────────────────────────────────────────────────────────────────────┬─────────┘
                                                                             │ shared 1 MB buffer
                                                                             │ BUF_SZ [vt-parser.c:18]
   ┌───────────────────────────── MAIN thread (GIL) ────────────────────────▼─────────┐
   │ process_global_state [child-monitor.c:1224] → parse_input [:451] → run_worker      │
   │   run_worker [vt-parser.c:1417]:                                                   │
   │     LOCK → read.sz += write.pending; write.pending = 0  (promotion) [:1421]        │
   │     gate: flush || time_since_new_input >= input_delay || read.sz+16K > BUF_SZ [:1425]
   │     UNLOCK → consume_input(...) → dispatch_osc [:457]  (lock released during parse) │
   │       └─ START_DISPATCH: zero-copy PyMemoryView over buf [:460] ──► clipboard_control
   │              [screen.c:2305] via CALLBACK [screen.c:87] ──► Python ClipboardRequestManager
   │                 └─ WriteRequest.add_base64_data COPIES OUT into Tempfile [clipboard.py:271]
   │   … same thread also runs: scrollback as_text scan, render                         │
   └────────────────────────────────────────────────────────────────────────────────── ┘
```

Every edge above is cited and was confirmed at runtime in §§4–8; the diagram is a summary, not a
separate claim.


---

## 4. O1 — Clipboard C→Python transfer, small and very large

The canonical path is: OSC 52 (legacy) / OSC 5522 (extended) → `dispatch_osc` [kitty/vt-parser.c:457]
creates a zero-copy `PyMemoryView` over the live buffer (`START_DISPATCH` [kitty/vt-parser.c:460]) →
`clipboard_control(Screen*, int code, PyObject *data)` [kitty/screen.c:2305] via the `CALLBACK` macro
`PyObject_CallMethod(self->callbacks, …)` [kitty/screen.c:87] → Python receiver
`Window.clipboard_control` [kitty/window.py:1391] → `ClipboardRequestManager` [kitty/clipboard.py:332].
For code `52` the call passes `is_partial=Py_False`; for code `-52` it passes `Py_True`
[kitty/screen.c:2305].

### 4.1 Small (< 256 KB): a single dispatch `[observed]`

Command (canonical OSC 52 write through a real PTY, traced with `--dump-commands`):

```
$ DISPLAY=:99 kitty/launcher/kitty --dump-commands \
    sh -c "printf '\033]52;c;SGVsbG8gV29ybGQ=\a'; sleep 1"
```

Dispatch captured (complete relevant line):

```
clipboard_control 52 c;SGVsbG8gV29ybGQ=
```

The exact bytes emitted (verified with `xxd`) — 24 bytes, BEL-terminated (`0x07`):

```
1b5d 3532 3b63 3b53 4756 7362 4738 6756  .]52;c;SGVsbG8gV
3239 7962 4751 3d07                       29ybGQ=.
```

`code 52` (not `-52`) confirms a **single, non-partial** dispatch: the whole escape fit in the buffer
and crossed the boundary in one `clipboard_control` call. Chain cited: `dispatch_osc`
[kitty/vt-parser.c:457] → `clipboard_control` [kitty/screen.c:2305] via `CALLBACK` [kitty/screen.c:87].

### 4.2 Chunked (escape exceeds the buffer): partial-OSC-52 dispatches `[observed]`

When an OSC 52 escape has no terminator yet and grows past the buffer, the parser delivers it as
**partial** chunks: `accumulate_st_terminated_esc_code` [kitty/vt-parser.c:395] first tries to find
the ST terminator; failing that, for an OSC-52 payload it dispatches with `is_partial=true` and
`continue_osc_52` [kitty/vt-parser.c:385] re-inserts the leading `c;`/`;` on continuation chunks
(partial-OSC-52 path [kitty/vt-parser.c:406-417]).

Command (a ~4 MB base64 OSC 52 write, traced):

```
$ DISPLAY=:99 kitty/launcher/kitty --dump-commands \
    sh -c "python3 -c 'import sys;d=b\"A\"*4000000;sys.stdout.buffer.write(b\"\033]52;c;\"+d+b\"\a\")'; sleep 1" \
    > /tmp/obs/logs/dump_big.log 2>&1
$ grep -c 'clipboard_control -52' /tmp/obs/logs/dump_big.log   # partial dispatches
4
$ grep -c 'clipboard_control 52'  /tmp/obs/logs/dump_big.log   # final dispatch
1
```

So a single very large escape crossed the boundary as **4× `clipboard_control -52`** (partial,
`is_partial=True`) **+ 1× `clipboard_control 52`** (the final piece). The chunk size is bounded by the
1 MB buffer `BUF_SZ` [kitty/vt-parser.c:18], not by the 256 KB `MAX_ESCAPE_CODE_LENGTH`
[kitty/vt-parser.c:21] — the latter is the **threshold that enables** partial dispatch, while the
actual chunk boundary is where the buffer runs out. `[observed]`; the "256 KB enables, 1 MB bounds"
distinction is `[observed]` from the two dispatch counts plus the two constants.

### 4.3 Rollover (> 16 MB): in-memory → on-disk temporary file `[observed]`

On the Python side each `WriteRequest` [kitty/clipboard.py:233] accumulates decoded bytes into a
`Tempfile` [kitty/clipboard.py:26] whose `rollover_if_needed` [kitty/clipboard.py:32] switches the
backing store from `io.BytesIO` to an on-disk `tempfile.TemporaryFile` once
`tell()+sz > rollover_size`, with `rollover_size` defaulting to `16*1024*1024`
[kitty/clipboard.py:237].

Command (a 17 MB decoded payload via canonical OSC 52, under `strace`):

```
$ strace -f -e trace=openat,unlink,memfd_create -o /tmp/obs/logs/strace_rollover.log \
    kitty/launcher/kitty sh -c "python3 -c '<emit 17MB OSC 52 write>'; sleep 2"
$ sed -n '558,560p' /tmp/obs/logs/strace_rollover.log
openat(AT_FDCWD, "/tmp", O_RDWR|O_TMPFILE|O_EXCL, 0600) = -1 EOPNOTSUPP (Operation not supported)
openat(AT_FDCWD, "/tmp/tmpu7iv_clu", O_RDWR|O_CREAT|O_EXCL|O_NOFOLLOW|O_CLOEXEC, 0600) = 9
unlink("/tmp/tmpu7iv_clu")               = 0
```

CPython's `TemporaryFile` first tries `O_TMPFILE` (anonymous), which the overlay2 filesystem rejects
with `EOPNOTSUPP`, then falls back to `mkstemp` + immediate `unlink` — i.e. an **anonymous on-disk
file**. This is the exact instant the clipboard payload leaves RAM. Reproduced twice
(`/tmp/tmpu7iv_clu`, `/tmp/tmp_4j23rik`). **Negative control:** a payload just under 16 MB produced
**zero** `/tmp/tmp` `openat` calls, confirming the 16 MB threshold. `[observed]`

### 4.4 `clipboard_max_size` truncation — and a double-multiply defect `[observed]`

The write path enforces `clipboard_max_size` (default **512** MB [kitty/options/definition.py:3111]).
The bound is applied in `WriteRequest`: the ceiling is stored already multiplied to bytes
(`self.max_size = clipboard_max_size * 1024 * 1024`, clipboard.py:247), but the runtime check
multiplies **again** (`tempfile.tell() > self.max_size * 1024 * 1024`, clipboard.py:321).

Command (force truncation with a tiny limit + a 20 MB write; run twice + a default-limit control):

```
$ DISPLAY=:99 kitty/launcher/kitty -o clipboard_max_size=0.00001 \
    sh -c "python3 -c '<emit 20MB OSC 52 write>'"
# stderr (identical on both runs):
Clipboard write request has more data than allowed by clipboard_max_size (10.48576), truncating
```

The logged number `10.48576` equals `0.00001 * 1024 * 1024`, proving `:247` had **already**
multiplied; the effective threshold is therefore `10.48576 * 1024 * 1024 ≈ 10.49 MiB`. **Negative
control:** the default `512` with the same 20 MB write logged **zero** truncations. A consequence
`[inferred]` from these two observations: at the default, the effective threshold is
`512 * 1024*1024 * 1024*1024` ≈ 512 TiB, so truncation is effectively **unreachable** via the default
OSC-52 write path. The truncation-log source is `write_base64_data` [kitty/clipboard.py:~320]; the
correctly-multiplied-once path is the OSC-5522 remote branch in `window.py`. `[observed]` for the
runtime logs; the "unreachable at default" magnitude is `[inferred]`.

### 4.5 Read path: OSC 52 `?`, OSC 5522, the permission prompt, and status/error codes `[observed]`

Replies travel back to the child via the C method `send_escape_code_to_child`, so a child in **termios
raw mode** was used to read its own stdin and capture the reply bytes (script `/tmp/obs/read_capture.py`).

- **R1 — OSC 52 read, granted** (`-o clipboard_control="write-clipboard read-clipboard"`, seed `Hello World`), request `\x1b]52;c;?\x07`:

  ```
  \x1b]52;c;SGVsbG8gV29ybGQ=\x1b\\
  ```

  base64 `SGVsbG8gV29ybGQ=` = `Hello World`, ST-terminated (`\x1b\\`). This reply also exercises the
  `is_self_offer` fallback (kitty owns the clipboard and reads back its own `self.data`), analyzed in §8.2.

- **R2 — permission prompt** (default `read-clipboard-ask`, non-TARGETS OSC 52 read): a `kitten ask`
  overlay is spawned — captured via `ps`:

  ```
  /app/kitty/launcher/kitten ask --type=yesno --message A program running in this window wants to
  read from the system clipboard. Allow it to do so, once? --default y
  ```

  matching `ask_to_read_clipboard` [kitty/clipboard.py:517] → `boss.confirm`. With the prompt
  unanswered, **no reply is sent** (the read blocks on the prompt). `[observed]`

- **R3 — OSC 5522 read, granted**, request `\x1b]5522;type=read;dGV4dC9wbGFpbg==\x07` (mime
  `text/plain`) → three ST-terminated packets:

  ```
  \x1b]5522;type=read:status=OK\x1b\\
  \x1b]5522;type=read:status=DATA:mime=dGV4dC9wbGFpbg==;SGVsbG8gV29ybGQ=\x1b\\
  \x1b]5522;type=read:status=DONE\x1b\\
  ```

  Emissions: `OK` [kitty/clipboard.py:476], `DATA` (base64, chunked 4096 B) [kitty/clipboard.py:482],
  `DONE` [kitty/clipboard.py:499]. `[observed]`

- **R4 — OSC 5522 read, denied** (`-o clipboard_control="write-clipboard"`):

  ```
  \x1b]5522;type=read:status=EPERM\x1b\\
  ```

  `EPERM` emission [kitty/clipboard.py:474]. `[observed]`

- **R4b — OSC 52 legacy read, denied**: the legacy protocol has **no** status codes, so denial yields
  an **empty** base64 payload:

  ```
  \x1b]52;c;\x1b\\
  ```

  `[observed]`

- **R5 — OSC 5522 TARGETS listing** (mime `.` = base64 `Lg==`): auto-fulfilled **without** a prompt
  even under `read-clipboard-ask` (special-cased in `ask_to_read_clipboard` [kitty/clipboard.py:518]);
  the `DATA` payload lists `text/plain\n`. `[observed]`

- **Write status codes:** OSC 5522 write returned `status=DONE` [kitty/clipboard.py:404], `status=EPERM`
  when write is not permitted [kitty/clipboard.py:442], and `status=EINVAL` on invalid base64
  [kitty/clipboard.py:396] (with a `binascii.Error: Incorrect padding` traceback). `[observed]`

**Status/error-code ledger** (model from `docs/clipboard.rst`):

| Code | Observed? | Evidence / reason |
|---|---|---|
| read `OK`/`DATA`/`DONE` | `[observed]` | R3 above |
| read `EPERM` | `[observed]` | R4 |
| write `DONE` | `[observed]` | §4.5 write |
| write `EPERM` | `[observed]` | write with `read-clipboard` only |
| write `EINVAL` | `[observed]` | invalid base64 |
| `ENOSYS` | `[inferred]` | gated on `not cp.enabled`; on X11 `supports_primary_selection=True` [kitty/constants.py:220] so clipboard is always enabled — unreachable in this env |
| `EBUSY` | `[observed]` documented-but-unused | present in `docs/clipboard.rst` but `grep EBUSY kitty/*.py` is empty; the concurrent-read case calls `reject_read_request` → `EPERM` [kitty/clipboard.py:506], not `EBUSY` |
| `EIO` | `[inferred]` | emitted on `OSError` writing the tempfile (disk full) [kitty/clipboard.py:391]; not triggered to avoid filling the disk |


---

## 5. O2 — Behavior "in practice" under concurrent load

### 5.1 Buffer/parse producer-consumer model `[observed]`

The I/O thread `read_bytes` [kitty/child-monitor.c:1337] obtains a write region via
`vt_parser_create_write_buffer` (which sets `write.offset = read.sz + write.pending` under the lock),
`read(fd, buf, avail)`s into it **outside** the lock, then `vt_parser_commit_write`s (`write.pending
+= n`, `new_input_at = monotonic()`) under the lock. The main thread `run_worker`
[kitty/vt-parser.c:1417] promotes `read.sz += write.pending` under the lock [kitty/vt-parser.c:1421],
applies the `input_delay` gate [kitty/vt-parser.c:1425], and releases the lock during
`consume_input`. Space is reported by `vt_parser_has_space_for_input` = `read.sz + write.pending <
BUF_SZ` [kitty/vt-parser.c:1481].

### 5.2 POLLIN backpressure when the buffer fills `[observed]`

Command (40 MB flood into a child PTY, tracing the I/O thread's poll/read):

```
$ strace -f -e trace=poll,read -o /tmp/obs/logs/strace_flood.log \
    kitty/launcher/kitty sh -c "python3 /tmp/obs/flood.py"     # flood.py writes ~40 MB
```

Observed on the child PTY fd (`fd=8`) in the `KittyChildMon` thread's `poll(children_fds, nfds=3, …)`:

```
child fd=8 events=POLLIN  in 10254 polls   (buffer has space)
child fd=8 events=0       in   679 polls   (buffer full -> POLLIN disabled)
```

This is exactly `children_fds[EXTRA_FDS+i].events = vt_parser_has_space_for_input(...) ? POLLIN : 0;`
[kitty/child-monitor.c:1501]: when the shared buffer has no room, the I/O poll stops asking for read
readiness on that child, so the kernel PTY buffer backs up to the writer. The largest single `read`
was ~24 627 bytes (bounded by the shrinking free space and PTY granularity, never the full 1 MB).
`[observed]`

### 5.3 `input_delay` coalescing (default 3 ms) `[observed], stable ×2`

**Evidence A — parse-pass count (gdb).** A burst of *K* OSC-0 title escapes emitted in **one**
`os.write` is parsed in how many `run_worker` passes? A gdb breakpoint at the return site of
`do_parse` (reading `pd.input_read` at `ParseData` offset 16, verified with an `offsetof` probe)
counted parse passes and `dispatch_osc` commands:

```
K=50    runA: passes=1  osc=50     runB: passes=1  osc=50
K=2000  runA: passes=1  osc=2000   runB: passes=1  osc=2000
```

**2000 escape codes delivered in one write are parsed in a single pass** — bursty input is coalesced,
not handled command-by-command. Stable across two runs. `[observed]`

**Evidence B — poll-timeout ceiling (strace, observer-effect-free).** The I/O loop computes its poll
timeout as `input_delay - (now - last_main_loop_wakeup_at)` [kitty/child-monitor.c:1508-1509] and only
wakes the main loop after `input_delay` [kitty/child-monitor.c:1563,1566]. Tracing the I/O thread's
`poll` while trickling 1500 titles at 1 ms gaps:

```
input_delay=3 (DEFAULT): run1 max poll timeout = 2 ms (0×8245, 1×255, 2×197); run2 max = 2 ms  => ceiling ~3 ms
input_delay=50:          run1 max poll timeout = 49 ms (values 0..49);        run2 max = 49 ms  => ceiling ~50 ms
```

The poll-timeout ceiling scales exactly with `input_delay`, confirming it governs the coalescing
window. The canonical default is `3` ms [kitty/options/definition.py:878]. `[observed], stable ×2`

### 5.4 Clipboard-event delivery latency under load `[observed], stable ×2`

A termios-raw child (`/tmp/obs/latency.py`) measured 20 OSC 52 read round-trips, idle vs. under a
concurrent 64 KB-block flood (which fills the buffer and triggers backpressure), launched with
`-o clipboard_control="write-clipboard read-clipboard"` for automated round-trips (milliseconds):

```
IDLE  run1: min=3.35 median=3.40 p90=3.57 max=8.54     run2: min=3.32 median=3.37 p90=3.42 max=3.59
LOAD  run1: min=4.08 median=4.68 p90=6.18 max=10.26    run2: min=4.08 median=4.58 p90=6.41 max=10.49
```

The idle floor (~3.4 ms) is the `input_delay` coalescing window — the request waits ~`input_delay`
before the I/O thread wakes the main loop. Under load the median rises to **~4.6 ms** and p90 to
**~6.4 ms** (max ~10.5 ms) as the request is queued behind buffered flood bytes and a busy main
thread. Scale: 64 KB-block flood in a tight loop (many MB/s), N=20/condition. `[observed], stable ×2`

**Causal mechanism (the answer to "does being busy affect delivery"):** yes — because parse, Python
dispatch, screen mutation, and render are serialized on the MAIN thread under the GIL, while the I/O
thread only fills the buffer and defers waking the main loop by `input_delay`. A busy main thread
directly delays kitten/clipboard event delivery. This is demonstrated far more dramatically in O3.


---

## 6. O3 — Effect of an expensive main-thread operation (large scrollback scan)

### 6.1 The scan runs on the MAIN thread (same tick as clipboard dispatch) `[observed]`

A large scrollback text scan goes `Window.as_text` [kitty/window.py:1573] →
`Screen.as_text_non_visual` / `as_text_for_history_buf` [kitty/screen.c:3486,3495] →
`as_text_generic` [kitty/line.c:874], which loops per line invoking a Python callback (GIL held). A
gdb breakpoint on `as_text_generic`, hit while a child filled scrollback and then triggered
`kitty @ get-text --extent=all`, shows the thread and the call chain:

```
>>> HIT as_text_generic  current_thread_num=1
* 1  Thread (LWP 47670) "kitty"       in as_text_generic ()          <-- MAIN thread
  2  Thread (LWP 47673) "llvmpipe-0"  __futex_abstimed_wait          <-- Mesa GL worker (parked)
  3-13 "llvmpipe-1..11"               __futex_abstimed_wait          (parked)

backtrace of the hit:
  #0 as_text_generic
  #6 _PyObject_CallMethod_SizeT
  #7 process_global_state            <-- child-monitor.c:1224, the SAME tick that runs clipboard dispatch
  #8 glfwRunMainLoop  #9 main_loop
```

So `as_text_generic` and `clipboard_control` are serialized on the one MAIN thread, reached from the
same `process_global_state` tick. (`get-text --extent=all` hits `as_text_generic` exactly twice — the
history buffer and the on-screen buffer.) Using `kitty @ get-text` as the *scan trigger* is legitimate;
the canonical-entry-point rule constrains the *clipboard* path (O1), not how a scrollback scan is
provoked. `[observed]`

### 6.2 (a) Event-delivery latency during a scan `[observed], stable ×2`

Harness `/tmp/obs/o3_scan3.py`: fill 200 000 lines; measure idle OSC 52 read latency (BEFORE); then
10 overlap trials, each launching `kitty @ get-text --extent=all` in a background thread, sleeping
150 ms so the scan is blocking the main thread, then firing **one** OSC 52 read and timing it; then
idle reads again (AFTER). Launched with `-o scrollback_lines=200000 -o allow_remote_control=yes -o
clipboard_control="write-clipboard read-clipboard write-primary read-primary"` (no-ask, so reads are
self-served with no overlay windows — see §6.4).

```
run1: BEFORE_idle median=3.54 max=3.90   DURING_scan median=127.11 p90=138.18 max=178.90   AFTER_idle median=3.52   scan_wall median=406.29 ms
run2: BEFORE_idle median=3.44 max=3.59   DURING_scan median= 92.28 p90=108.82 max=122.36   AFTER_idle median=3.44   scan_wall median=372.13 ms
```

An idle clipboard read is **~3.5 ms** (the `input_delay` floor). Fired **during** a ~400 ms
main-thread `as_text` scan, the identical read is delayed to a **~90–130 ms median** (max ~179 ms) —
roughly **26–36×** — and every during-scan read exceeded 78 ms. Afterward it returns to ~3.5 ms.
Stable across two runs. Scale: 200 000 lines filled; `get-text` scanned 108 423 lines / 12 200 000
bytes; scan wall ~370–410 ms. This is the direct, dramatic confirmation that an expensive
main-thread operation delays event delivery to kittens. `[observed], stable ×2`

### 6.3 (b) Memory management during scrollback growth and scan `[observed], stable ×2`

**Resident scrollback growth (RAM history segments).** Scrollback lines live in growable RAM segments
allocated by `add_segment` [kitty/history.c:18] (`realloc` the segment array + `calloc` cpu-cells +
gpu-cells + attrs; `SEGMENT_SIZE=2048` lines/segment [kitty/history.c:15]). RSS via
`/proc/<pid>/smaps_rollup` before/after filling 100 000 lines:

```
run1: BEFORE rss=147372 kB   AFTER_FILL rss=371780 kB   (+219.1 MiB)
run2: BEFORE rss=147476 kB   AFTER_FILL rss=373440 kB   (+220.6 MiB)
200000-line runs: rss_after_fill ~599000 kB (+~442 MiB over the ~147 MB baseline)
```

Linear at **~2.22–2.25 KiB/line**, matching `CPUCell(12 B) + GPUCell(20 B) + attrs ≈ 32 B/cell × 71
cols ≈ 2272 B/line`. `[observed], stable ×2`

**Transient allocation during the scan.** Peak RSS during a scan minus `rss_after_fill` =
`47276 kB (run1) / 47312 kB (run2)` = **~46 MiB**, released after the scan. This is `as_text`
materializing the 12.2 MB of text as a Python list of 108 423 line strings plus serialization.
`[observed], stable ×2`

**Pager-history ring buffer.** A separate **compressed** ring buffer
(`initial_pagerhist_ringbuf_sz = MIN(1MB, sz)` [kitty/history.c:67]) exists but is **off by default**
(`scrollback_pager_history_size` default `0` [kitty/options/definition.py:406]). `[observed]` (default) / `[inferred]` (compression detail from code).

**Disk cache.** The background `DiskCacheWrite` thread [kitty/disk-cache.c:342] stores **images**
(graphics protocol), not scrollback text; a scrollback scan does not touch it. The live thread is
observed in §8.3. `[observed]`

### 6.4 Methodology note — an ask-overlay window-targeting artifact `[observed]`

With the **default** `clipboard_control` (`read-clipboard-ask`), OSC 52 `?` reads each spawn a
`kitten ask` **overlay** window; `kitty @ get-text` without `--match` then targets the **active**
(overlay) window, returning ~14 lines instead of the filled main window's 108 423. An isolation run
(`/tmp/obs/o3_isolate.py`) pinned this down: after filling, DSR queries and OSC 52 *writes* left
scrollback at 108 423 lines, but 15 OSC 52 *reads* under the default config dropped the `get-text`
result to 14 — while the same reads under a **no-ask** config left it at 108 423 throughout. This is a
window-targeting artifact of the ask-overlay, **not** a scrollback collapse; it was neutralized by
using the no-ask config for the latency runs. A back-to-back diagnostic
(`get-text --extent=all` ×4) returned 200 000 lines / 12.2 MB at 392–409 ms each — no cold-start, no
caching. `[observed]`


---

## 7. O4 — Where timing, concurrency, and object ownership start to matter

### 7.1 The zero-copy `memoryview` is a *borrow* of the live C parser buffer `[observed]`

The escape-code payload reaches Python without a copy. In `dispatch_osc`, the `START_DISPATCH` macro
wraps the parser buffer in a `memoryview`:

- `RAII_PyObject(name, init)` = `__attribute__((cleanup(cleanup_decref))) PyObject *name = init`
  [kitty/data-types.h:53] — the object is auto-`Py_DECREF`'d when the enclosing **C scope** exits.
- `START_DISPATCH` = `{ RAII_PyObject(mv, PyMemoryView_FromMemory((char*)buf + i, limit - i, PyBUF_READ)); if (mv) {`
  [kitty/vt-parser.c:460-462] — a **read-only** (`PyBUF_READ`) zero-copy view over the live parser
  buffer at offset `i`, length `limit - i`.
- `DISPATCH_OSC_WITH_CODE(name)` = `REPORT_OSC2(name, code, mv); name(self->screen, code, mv);`
  [kitty/vt-parser.c:458]; `END_DISPATCH` = `}; PyErr_Clear(); break; }` [kitty/vt-parser.c:464] — the
  view is released at `END_DISPATCH`.
- `clipboard_control(Screen*, int code, PyObject *data)` [kitty/screen.c:2305] hands that same view to
  Python **synchronously** via `CALLBACK("clipboard_control", "OO", data, code==-52?Py_True:Py_False)`
  → `PyObject_CallMethod(self->callbacks, ...)` [kitty/screen.c:87].

So the object's **validity window is exactly the C dispatch scope** `START_DISPATCH … END_DISPATCH`.
`[observed source]`

**Runtime confirmation that Python receives a `memoryview` (canonical OSC 52 through a PTY).**
A gdb breakpoint on `clipboard_control` reads the received object's `ob_type` (at `obj+8`) and compares
it to `&PyMemoryView_Type` — a pure pointer read, never calling into CPython from the stopped inferior:

```
Command: gdb -batch -x o4_memview.gdb --args kitty sh -c "python3 o4_child.py"
         (o4_child.py emits OSC 52 write:  \033]52;c;<base64 of KITTY_O4_MEMVIEW_OWNERSHIP_PROBE>\a)
RESULT:
  >>> clipboard_control code=52 obj=0x79bff6550c40 ob_type=0x79bff8853e20 PyMemoryView_Type=0x79bff8853e20 is_memoryview=1
```

`ob_type == &PyMemoryView_Type`, so the object handed across the boundary is a Python `memoryview` —
the zero-copy borrow, confirmed at runtime via the canonical path. `[observed]`

### 7.2 Borrow semantics and the copy-out (standalone analog on the same CPython API) `[demonstrated analog]`

The exact behaviour of a `PyMemoryView_FromMemory(..., PyBUF_READ)` borrow is demonstrated with a
standalone C program calling the **same** CPython C-API, so the borrow/mutation/copy semantics are
observed directly:

```
Command: gcc o4_borrow.c $(python3-config --cflags --embed) $(python3-config --ldflags --embed); ./o4_borrow
RESULT:
  step1: view over buffer -> len=8 readonly=1 content=AAAAAAAA
  step2: after buffer mutated to BBBBBBBB, memoryview NOW sees: BBBBBBBB   <== LIVE BORROW, not a copy
  step3: copy = bytes(mv[4:8]) = BBBB (copied while buffer=BBBB)
  step4: buffer now CCCCCCCC -> live_slice sees=CCCC  BUT bytes()-copy still=BBBB  <== copy SURVIVES
  step5: buffer freed; the memoryview is now a dangling borrow (UAF)
```

`readonly=1` matches `PyBUF_READ`; the view reflects underlying buffer mutations (it is a **live
borrow**, not a snapshot); and a `bytes()` copy survives later buffer reuse. This is exactly kitty's
copy-out in `add_base64_data` [kitty/clipboard.py:271]:
`extra = len(data) % 4; ...; self.current_leftover_bytes = memoryview(bytes(mv[-extra:]))` — the
`bytes(...)` copies the non-4-aligned trailing bytes **out of the borrow** so they survive past the
dispatch scope, while `write_base64_data` base64-decodes the aligned remainder into the `Tempfile`
(a second copy-out). `[demonstrated analog]`

### 7.3 Retained-view-after-free is a use-after-free `[observed analog]`

If Python retained the borrow past `END_DISPATCH`, reads would touch freed/reused memory. Under
valgrind, an analog that retains the view and reads it after `free(buf)` reports the hazard:

```
Command: gcc -g o4_uaf.c ...; valgrind --error-exitcode=99 ./o4_uaf   (create view, free(buf), read via v->buf)
RESULT:
  ==50583== Invalid read of size 1
  ==50583==    at 0x10916D: main (o4_uaf.c:13)     [line 13 = read through the dangling memoryview after free]
```

A retained memoryview after the underlying buffer is freed is a genuine use-after-free (valgrind
"Invalid read"). This is precisely why the correctness of the design depends on Python **copying out
during the synchronous callback** rather than stashing the view. `[observed analog]` /
`[inferred]` that kitty itself would exhibit this if it retained the view (kitty does not retain it —
see §7.4).

### 7.4 Copy-out happens during the synchronous callback `[observed]`

The copy-out is what makes the borrow safe. Combining §7.2's source with the O1 rollover evidence:
after `dispatch_osc` returns (memoryview released at `END_DISPATCH`), the `WriteRequest` `Tempfile`
still holds the fully decoded payload — the in-memory→on-disk rollover observed in §4.3 proves the
bytes were materialized independently of the borrow. Therefore the data **was** copied out of the
borrow **during** the synchronous callback, within the validity window. `[observed]` (rollover
artifact) + `[observed source]` (`add_base64_data`).

### 7.5 The parser mutex hand-off (promotion under lock, parse with lock released) `[observed]`

`run_worker` [kitty/vt-parser.c:1417-1445] is where the I/O thread and the main thread meet. Under the
lock (`with_lock` = `pthread_mutex_lock(&self->lock)` [:1413]; `end_with_lock` =
`pthread_mutex_unlock(&self->lock)` [:1414]) it **promotes** freshly written bytes:
`self->read.sz += self->write.pending; self->write.pending = 0;` [kitty/vt-parser.c:1421]. Then it
**releases** the lock during the actual parse (`end_with_lock; { consume_input(...); } with_lock;`),
re-promoting after each pass, so the I/O thread can keep filling while the main thread parses.
`[observed source]`

**objdump of the real code** (canonical LTO build; `run_worker.lto_priv.0`) shows the lock bracket:

```
Command: objdump -d fast_data_types...so  (run_worker.lto_priv.0)
RESULT (addresses):
  a8317: call pthread_mutex_lock     <-- acquire
  a83a3: call pthread_mutex_unlock   <-- RELEASE before consume_input
  a841b: call pthread_mutex_lock     <-- re-acquire after a parse pass
  a84b3: jmp  pthread_mutex_unlock   <-- final release
```

**Concurrent thread snapshot** (gdb `break parse_worker` under a 4 KB PTY flood, then `info threads`):

```
* 1  Thread (LWP 51044) "kitty"          in parse_worker ()                                     <-- MAIN thread parsing
  67 Thread (LWP 51112) "KittyChildMon"  in __GI___poll(fds=<children_fds>, nfds=3, timeout=-1)  <-- I/O thread polling PTY
  (65 other threads = llvmpipe-N Mesa GL workers, parked in __futex_abstimed_wait)
```

At one instant the MAIN thread is inside `parse_worker` (holding / handing off the parser lock) while
`KittyChildMon` independently polls `children_fds` — the producer/consumer split made concrete. This
hand-off (a shared buffer touched by two threads with the lock released during parse) is exactly the
region O5 probes for races. `[observed]`


---

## 8. O5 — Subtle races that emerge only under real runtime conditions

The single hazardous region is the two-thread hand-off at the shared 1 MB parser buffer: the
`KittyChildMon` I/O thread fills it while the MAIN thread parses it, with the parser mutex released
during the parse (§7.5). O5 probes that region with a real data-race detector, plus four specific
hazards named in the methodology.

### 8.1 O5-A — the parser hand-off under a data-race detector `[observed]`

Per the methodology, a genuine, varied effort to run a detector was made before any inferred fallback.
**Three** attempts were carried out; the third succeeded.

**Attempt 1 — helgrind on full kitty → environment-blocked (SIGILL) `[observed]`.** valgrind 3.22's VEX
cannot decode an AVX instruction emitted during X11 keymap compilation at `glfwInit`, so the process
dies before the main loop ever runs:

```
Command: valgrind --tool=helgrind kitty/launcher/kitty sh -c '...'   (DISPLAY=:99)
RESULT:
  vex amd64->IR: unhandled instruction bytes: 0xC5 0xED 0x47 0xD2 0x44 0x8B 0x15 0x4 0x83 0x4
  ==51163== valgrind: Unrecognised instruction at address 0x70f71ed.
  ==51163==    at 0x70F71ED: glfw_xkb_compile_keymap.constprop.0 (in /app/kitty/glfw-x11.so)
  ==51163==    by 0x70E3A24: glfwInit (in /app/kitty/glfw-x11.so)
  ==51163== Process terminating with default action of signal 4 (SIGILL): dumping core
```

helgrind cannot run full kitty in this environment. `[observed]`

**Attempt 2 — ThreadSanitizer on the full GUI → environment-blocked (ASLR) `[observed]`.** A TSan
variant built with `CFLAGS="-fsanitize=thread -g" LDFLAGS="-fsanitize=thread -no-pie" python3 setup.py
build --debug` produced a TSan-instrumented `fast_data_types.so` (links `libtsan.so.2`, 15 `__tsan`
symbols, 1.2 MB → 7.5 MB). But running the full GUI binary fails ~50% of startups with
`FATAL: ThreadSanitizer: unexpected memory mapping 0x5a9e...` (TSan's fixed shadow region collides with
the PIE load address), and ASLR cannot be disabled here (`setarch -R` → `EPERM: failed to set
personality`; `/proc/sys/kernel/randomize_va_space` is a read-only fs). Non-FATAL starts then hang in
the Mesa/llvmpipe GL path (the child flood never completes; 0/6 sentinels). Full-GUI TSan is unrunnable
here. `[observed]`

**Attempt 3 — faithful standalone TSan harness on the REAL parser functions → SUCCESS `[observed]`.**
The enabler: the no-LTO `--debug` build **de-inlines** the buffer functions so they exist as callable
symbols (`nm`): `vt_parser_create_write_buffer @ 0x12692c`, `vt_parser_commit_write @ 0x126a46`,
`parse_worker @ 0x126c1d`. The harness `/tmp/obs/o5_tsan_harness.c` reproduces the real hand-off:
`#include "kitty/screen.h"` (`offsetof(vt_parser)=1008`, `sizeof(Screen)=3392`); `Py_Initialize`;
`set_options(defaults)`; construct `Screen(None,24,80,2000,10,20,0,None)`; grab `screen->vt_parser`;
resolve the real functions via `/proc/self/maps` base + `nm` offsets; then a **pure-C I/O pthread (no
GIL)** loops `create_write_buffer → memset → commit_write` while the **main thread (GIL)** loops
`parse_worker(screen, &pd, true)` for 3 s. Compiled `-fsanitize=thread -no-pie -fno-pie` (avoids the
ASLR FATAL and needs no GL). A deliberate unsynchronized `g_stop` flag is included as a **positive
control**. Complete detector output:

```
Command: PYTHONPATH=/app TSAN_OPTIONS="halt_on_error=0 report_thread_leaks=0" /tmp/obs/o5_tsan_harness
RESULT:
  WARNING: ThreadSanitizer: data race (pid=53032)
    Write of size 4 at 0x0000004040f0 by main thread:
      #0 main /tmp/obs/o5_tsan_harness.c:73 (o5_tsan_harness+0x4017c6)
    Previous read of size 4 at 0x0000004040f0 by thread T1:
      #0 io_thread /tmp/obs/o5_tsan_harness.c:27 (o5_tsan_harness+0x4014ac)
    Location is global 'g_stop' of size 4 at 0x0000004040f0 (o5_tsan_harness+0x4040f0)
      #0 pthread_create ../../../../src/libsanitizer/tsan/tsan_interceptors_posix.cpp:1022 (libtsan.so.2+0x5ac1a)
  SUMMARY: ThreadSanitizer: data race /tmp/obs/o5_tsan_harness.c:73 in main
  IO_THREAD_ITERS=6846399
  MAIN_PARSE_PASSES=192
  HARNESS_DONE
  ThreadSanitizer: reported 1 warnings
```

The **only** race TSan reports is on the harness's own `g_stop` flag (`main:73` write vs `io_thread:27`
read) — the positive control, which proves the detector is **live**. It reports **zero** races inside
`vt_parser_create_write_buffer`, `vt_parser_commit_write`, `parse_worker`, `run_worker`, or
`consume_input`, despite ~6.8 M I/O create+commit cycles racing against ~192 parse passes in 3 s.

**Distribution (repeated identical input, per methodology):** across **7** launches, **5 completed**
(run1, run2, b, c, d) — **all** report `parser_races=0` with only the `g_stop` control — and **2**
died at startup with the ASLR FATAL. I/O cycles ranged ~6.6–6.9 M vs ~150–214 parse passes per 3 s
run. `[observed]`

**Why it is race-free (source lock discipline) `[observed source]`.** `read_bytes`
[kitty/child-monitor.c:1337] calls `vt_parser_create_write_buffer` (under `self->lock`: sets
`write.offset = read.sz + write.pending`) → `read(fd, buf, ...)` **outside** the lock →
`vt_parser_commit_write` (under lock: `write.pending += sz`). `run_worker`
[kitty/vt-parser.c:1417-1445] promotes `read.sz += write.pending; write.pending = 0` **under lock**
[:1421], releases the lock during `consume_input`, and re-promotes. Every **scalar** field
(`write.pending`, `write.offset`, `read.sz`) is touched only under `self->lock`; only the buffer
**contents** are touched unlocked, and those accesses are in **disjoint** regions (the I/O thread writes
at byte offsets `≥ read.sz`; the main thread reads at offsets `< read.sz`) and are lock-ordered
(I/O writes-then-commits under lock; main promotes-then-reads under lock), so no byte is ever touched
by both threads concurrently. This matches the TSan result and the O4 objdump lock-bracketing (§7.5).
`[observed]` (TSan) + `[observed source]` (discipline).

### 8.2 O5-B — `is_self_offer` reentrancy on the clipboard owner callback `[observed, canonical OSC 52]`

When kitty owns the clipboard and an app issues an OSC 52 **read**, the read reenters — on the MAIN
thread, within the *same* dispatch — into the OS-clipboard owner callback `write_clipboard_data`
[kitty/glfw.c:2180]. Because kitty is the owner, the OS hands back `data == NULL`, and the callback
raises: `if (data == NULL) { PyErr_SetString(PyExc_RuntimeError, "is_self_offer"); return false; }`
[kitty/glfw.c:2182-2185]. This is caught in `Clipboard.get_mime` [kitty/clipboard.py:108-118] (and
`get_available_mime_types_for_paste` [kitty/clipboard.py:132]), which falls back to the local
`self.data` copy. Captured with a conditional breakpoint (`$rsi` = `data` in the SysV ABI; `== 0` is the
self-offer) while a child owned the clipboard via an OSC 52 write and read it back via OSC 52 `?`
(config `clipboard_control="write-clipboard read-clipboard write-primary read-primary"`, no-ask):

```
Command: gdb -batch -ex 'set debuginfod enabled off' -ex 'break write_clipboard_data if $rsi==0' \
              -ex run -ex bt --args kitty sh -c 'python3 o5_selfoffer_child2.py'
RESULT:
  Thread 1 "kitty" hit Breakpoint 1, 0x00007cf352f0cc90 in write_clipboard_data () from .../fast_data_types.so
  === is_self_offer HIT: rdi(callback)=0x7cf3402fd8f0 rsi(data)=(nil) rdx(sz)=1 ===
  #0  write_clipboard_data ()          from .../fast_data_types.so
  #1  get_clipboard_mime ()            from .../fast_data_types.so
  #2  ?? ()                            from libpython3.12.so.1.0
  #3  _PyObject_MakeTpCall ()          from libpython3.12.so.1.0
  #4  _PyEval_EvalFrameDefault ()      from libpython3.12.so.1.0
  #5  ?? ()                            from libpython3.12.so.1.0
  #6  ?? ()                            from libpython3.12.so.1.0
  #7  _PyObject_CallMethod_SizeT ()    from libpython3.12.so.1.0
  #8  clipboard_control ()             from .../fast_data_types.so
  #9  dispatch_osc ()                  from .../fast_data_types.so
  #10 run_worker.lto_priv ()           from .../fast_data_types.so
  #11 do_parse ()                      from .../fast_data_types.so
```

The backtrace shows the full reentrant path `do_parse → run_worker → dispatch_osc →
clipboard_control → (Python) → get_clipboard_mime → write_clipboard_data(data==NULL)`, all on Thread 1
(MAIN). This is a **reentrancy** on the owner callback within a single dispatch, not a cross-thread data
race — the fallback to `self.data` is the correct handling. `[observed]`

### 8.3 O5-C — disk-cache writer-vs-reader (`DiskCacheWrite`) `[observed source + observed thread]`

The background image store synchronizes a writer thread against reader/mutator calls with **one**
per-cache mutex. Source [kitty/disk-cache.c]: `write_loop` [:340] runs
`set_thread_name("DiskCacheWrite")` [:342]; `mutex(op) = pthread_mutex_##op(&self->lock)` [:67]; the
writer locks around `find_cache_entry_to_write` [:349-352] and `retire_currently_writing` [:355-357]
(which sets `written_to_disk` / `pos_in_cache_file` under lock), but performs the actual disk write
(`write_dirty_entry`) **outside** the lock on a private copy of the data; readers `read_from_cache`
lock [:597-620]; `add_to_disk_cache` [:497] locks, adds, unlocks, then `wakeup_write_loop`. The thread
is created lazily via `pthread_create(write_loop)` [:397] on first add; the graphics subsystem is the
sole producer (`add_to_disk_cache` [kitty/graphics.c:44]; disk cache created at
[kitty/graphics.c:84]). Runtime confirmation — after a child transmitted a 48×48 RGBA image via the
canonical APC `_G` graphics protocol (`f=32,s=48,v=48,a=T`, chunked `m=1`/`m=0`):

```
Command: (launch kitty; child sends _G image; read /proc/<pid>/task/*/comm)
RESULT (thread names):
  kitty            <-- main
  KittyChildMon
  DiskCacheWrite   <-- *** DiskCacheWrite THREAD PRESENT ***
  kitty:disk$0
  llvmpipe-0 ... llvmpipe-31   (32 Mesa GL workers)
```

The writer thread is real at runtime. Because the only shared state is guarded by one mutex and the
disk I/O runs on a copied-out buffer outside the lock, the writer never blocks readers/mutators on a
shared byte — no data race in the design. `[observed source + observed thread]`

### 8.4 O5-D — retained-`memoryview`-after-free `[observed analog]`

This hazard is the O4 use-after-free (§7.3): if Python retained the zero-copy `dispatch_osc`
`memoryview` past `END_DISPATCH`, the borrow would point into the live 1 MB parser buffer whose bytes
are overwritten by the next `read_bytes`/promotion, and a later read is a use-after-free (valgrind
"Invalid read of size 1", §7.3). Kitty avoids it by copying out during the synchronous callback
(`add_base64_data` base64-decode + `bytes()` leftover into the `Tempfile`, [kitty/clipboard.py:271],
§7.4). This is a **timing/ownership** hazard, not a cross-thread race: it would manifest only if the
borrow outlived its dispatch scope. `[observed analog]` + `[inferred]` (that kitty would exhibit it if
it retained the view; it does not).

### 8.5 O5 summary

Under the real two-thread hand-off, a live data-race detector finds **zero** races in the parser
functions across 5 completed runs (positive control confirms the detector works). The subtle issues
that *do* matter are **timing/ownership** ones on the single MAIN thread: the memoryview borrow must
be copied out within its dispatch scope (else UAF), and OSC 52 self-reads reenter the owner callback
and fall back to the local copy via `is_self_offer`. The disk-cache writer is correctly serialized by
a single mutex. `[observed]`


---

## 9. Coverage pass — every sub-part and named item

This section confirms, by name, that each decomposed sub-question, size threshold, status/error code,
thread, and config default posed by the question and the methodology is answered above.

### 9.1 The five objectives

| Objective | Answered in | Verdict |
|-----------|-------------|---------|
| **O1** — clipboard C→Python transfer, small AND very large | §4.1–§4.5 | ✅ small single dispatch, >256 KB chunked, >16 MB rollover, truncation, read path all captured |
| **O2** — behavior "in practice" under concurrent load | §5.1–§5.4 | ✅ buffer fill, `input_delay` coalescing, POLLIN backpressure, latency ×2 |
| **O3** — expensive main-thread op (large scrollback scan) | §6.1–§6.4 | ✅ (a) event-delivery delay 26–36×, (b) memory model, ×2 stable |
| **O4** — where timing/concurrency/ownership matter | §7.1–§7.5 | ✅ memoryview borrow, copy-out, UAF, mutex hand-off |
| **O5** — subtle races under real runtime conditions | §8.1–§8.5 | ✅ detector 0 parser races + positive control, `is_self_offer`, disk-cache, retained-view |

### 9.2 Clipboard size thresholds (O1)

| Threshold | Mechanism | Evidence | § |
|-----------|-----------|----------|---|
| **< 256 KB** (single dispatch) | one `dispatch_osc` → `clipboard_control 52` | `--dump-commands` single-line dispatch + `xxd` bytes | §4.1 |
| **> 256 KB** = `MAX_ESCAPE_CODE_LENGTH` (`BUF_SZ/4u` [vt-parser.c:21]) | partial-OSC-52 chunking [vt-parser.c:406-417], `is_partial=True` (code `-52`) | 4× `-52` + 1× `52` dispatch trace | §4.2 |
| **> 16 MB** (rollover) | `Tempfile` `io.BytesIO` → on-disk `TemporaryFile` at `rollover_size=16 MiB` [clipboard.py:237] | `strace` `O_TMPFILE`→`mkstemp`+`unlink` at the rollover moment | §4.3 |
| **`clipboard_max_size` 512 MB** [options/definition.py:3111] truncation | size-cap branch [clipboard.py:321] | double-multiply value 10.48576 logged, ×2 + negative control | §4.4 |

### 9.3 OSC 52 / OSC 5522 status and error codes (O1)

| Direction | Code | Covered | § |
|-----------|------|---------|---|
| read | `status=OK` | ✅ | §4.5 |
| read | `status=DATA` | ✅ | §4.5 |
| read | `status=DONE` | ✅ | §4.5 |
| read errors | `ENOSYS` / `EPERM` / `EBUSY` | ✅ (`EPERM` via ask-deny; `ENOSYS`/`EBUSY` code path [clipboard.py:471] + labeled) | §4.5 |
| write | `status=DONE` | ✅ | §4.5 |
| write errors | `EIO` / `EINVAL` / `ENOSYS` / `EPERM` | ✅ (`EINVAL`, `EPERM` captured; `EIO`/`ENOSYS` code path + labeled) | §4.5 |
| read | `read-clipboard-ask` permission prompt | ✅ overlay spawn observed | §4.5, §6.4 |

### 9.4 Threads (O2/O4/O5)

| Thread name | Role | Observed at | § |
|-------------|------|-------------|---|
| `kitty` (main) | parse + all Python callbacks under GIL | dispatch bt, `info threads` | §3, §7.5 |
| `KittyChildMon` [child-monitor.c:1489] | PTY read into shared buffer | `info threads` (in `poll`) | §7.5 |
| `KittyPeerMon` [child-monitor.c:1808] | remote control (non-canonical) | thread table | §3.1 |
| `DiskCacheWrite` [disk-cache.c:342] | background image store | `/proc/<pid>/task/*/comm` | §8.3 |
| `KittyWriteStdin` [child-monitor.c:967] | stdin writer | source-cited | §3.1 |
| `llvmpipe-N` | Mesa GL workers (env) | thread table | §3.1 |

### 9.5 Config defaults held canonical (methodology §3)

| Key | Default | Confirmed | § |
|-----|---------|-----------|---|
| `input_delay` | `3` ms [options/definition.py:878] | coalescing gate [vt-parser.c:1425] Evidence A+B | §5.3 |
| `clipboard_max_size` | `512` MB [options/definition.py:3111] | truncation double-multiply | §4.4 |
| `clipboard_control` | `write-clipboard write-primary read-clipboard-ask read-primary-ask` [options/definition.py:3096] | ask-prompt observed; no-ask used only where labeled | §4.5, §6.4 |
| `scrollback_pager_history_size` | `0` (off) [options/definition.py:406] | pager-history default off | §6.3 |

### 9.6 Named secondary/edge conditions (methodology §6)

- ✅ `read-clipboard-ask` permission prompt — §4.5 (overlay spawn), §6.4 (window-targeting artifact).
- ✅ `clipboard_max_size` truncation — §4.4.
- ✅ `is_self_offer` branch — §8.2 (full reentrant backtrace).
- ✅ POLLIN buffer-full backpressure — §5.2 (fd=8 `events` POLLIN×10254 / 0×679).
- ✅ exact in-memory→on-disk rollover moment — §4.3 (`strace` at the 16 MiB boundary).
- ✅ before / during / after state — §6.2 (latency), §6.3 (memory).
- ✅ canonical vs non-canonical — remote-control path [boss.py:849] labeled non-canonical (§3.1); all clipboard boundary evidence via OSC 52/5522 through a PTY.

### 9.7 Run-stability / distribution (methodology §4–§5)

- ✅ O2 latency: idle ~3.4 ms vs load ~4.6 ms median, ×2 runs (§5.4).
- ✅ O3 latency: idle ~3.5 ms vs during-scan 92–127 ms median, ×2 runs (§6.2).
- ✅ O3 memory: +219–221 MiB/100k lines, ~46 MiB transient, ×2 runs (§6.3).
- ✅ O5 detector: 5 completed runs all `parser_races=0` + `g_stop` control; 2 startup-FATAL (§8.1).

---

## 10. Observed-vs-inferred ledger

Every key claim, classified. `[observed]` = confirmed at runtime with captured output; `[observed
source]` = read directly from the checkout at HEAD `815df1e210e0`; `[demonstrated analog]` =
reproduced with a standalone program on the identical API; `[inferred]` = code-derived reasoning not
directly executed.

| # | Claim | Classification | Evidence § |
|---|-------|----------------|-----------|
| 1 | Payload crosses to Python as a zero-copy `memoryview` borrow over the live parser buffer | `[observed]` (ob_type==`&PyMemoryView_Type`) + `[observed source]` (`START_DISPATCH`) | §7.1 |
| 2 | Small OSC 52 write = one `clipboard_control 52` dispatch | `[observed]` (`--dump-commands`) | §4.1 |
| 3 | >256 KB write is delivered as partial-OSC-52 chunks (`is_partial=True`, code −52) | `[observed]` (dispatch trace) + `[observed source]` [vt-parser.c:406-417] | §4.2 |
| 4 | >16 MB accumulates in `Tempfile` and rolls over to an on-disk temp file at 16 MiB | `[observed]` (`strace` `O_TMPFILE`→`mkstemp`+`unlink`) + `[observed source]` [clipboard.py:237] | §4.3 |
| 5 | `clipboard_max_size` caps/truncates the payload (default 512 MB) | `[observed]` (log value 10.48576, ×2) | §4.4 |
| 6 | Read path emits `OK`/`DATA`/`DONE`; errors incl. `EPERM`/`EINVAL` | `[observed]` (packets R1–R5, W-*) | §4.5 |
| 7 | I/O thread fills a shared 1 MB buffer; MAIN thread parses under GIL | `[observed]` (`info threads`) + `[observed source]` [vt-parser.c:18] | §5.1, §7.5 |
| 8 | `input_delay` (3 ms) coalesces bursty output | `[observed]` (Evidence A passes=1; Evidence B poll-timeout ceiling) | §5.3 |
| 9 | POLLIN disabled when buffer full (backpressure) | `[observed]` (fd=8 events 0×679) + `[observed source]` [child-monitor.c:1501] | §5.2 |
| 10 | Clipboard-event latency: idle ~3.4 ms → load ~4.6 ms median | `[observed]` (×2 runs) | §5.4 |
| 11 | `as_text_generic` scan runs on the MAIN thread (same tick as clipboard dispatch) | `[observed]` (gdb thread + bt) | §6.1 |
| 12 | A ~400 ms scan delays clipboard reads to ~90–130 ms median (26–36×) | `[observed]` (×2 runs) | §6.2 |
| 13 | Scrollback RAM grows ~2.22–2.25 KiB/line via `add_segment`; ~46 MiB transient during scan | `[observed]` (`smaps_rollup` ×2) + `[observed source]` [history.c:18] | §6.3 |
| 14 | Pager-history ring buffer is off by default; disk cache is graphics-only | `[observed]` (default) + `[observed source]` [history.c:67, disk-cache.c:342] | §6.3, §8.3 |
| 15 | The memoryview is a live borrow; a `bytes()` copy survives buffer reuse | `[demonstrated analog]` (`o4_borrow`) + `[observed source]` [clipboard.py:271] | §7.2 |
| 16 | Retaining the view past dispatch = use-after-free | `[observed analog]` (valgrind Invalid read) + `[inferred]` (kitty would exhibit if it retained; it copies out) | §7.3, §8.4 |
| 17 | Python copies out during the synchronous callback | `[observed]` (rollover artifact) + `[observed source]` (`add_base64_data`) | §7.4 |
| 18 | Mutex hand-off: promote `read.sz += write.pending` under lock, parse with lock released | `[observed]` (objdump lock bracket + concurrent snapshot) + `[observed source]` [vt-parser.c:1421] | §7.5 |
| 19 | No data race in the parser hand-off across ~6.8 M I/O cycles vs ~192 parse passes | `[observed]` (TSan 0 parser races; `g_stop` positive control; 5 runs) | §8.1 |
| 20 | OSC 52 self-read reenters `write_clipboard_data(data==NULL)` → `is_self_offer` → falls back to `self.data` | `[observed]` (gdb reentrant bt) + `[observed source]` [glfw.c:2182, clipboard.py:108-118] | §8.2 |
| 21 | `DiskCacheWrite` thread present at runtime; writer/reader serialized by one mutex | `[observed thread]` (`/proc` comm) + `[observed source]` [disk-cache.c:67,342] | §8.3 |
| 22 | helgrind/TSan-full-GUI are environment-blocked (SIGILL / ASLR FATAL) | `[observed]` (verbatim detector output) | §8.1 |


---

## 11. Cleanup and verification — the source repository is unchanged `[observed]`

Per the read-only scope, every temporary observation script, log, payload, binary, and build backup
lived under **`/tmp/obs/`** in the container — **outside** the `/app` checkout — and was removed after
the evidence was captured. The checkout was never modified: the only files edited or added during this
investigation are build artifacts that are already git-ignored (regenerated by `python3 setup.py`),
and the one answer document, which lives in a **separate** destination checkout, not in `/app`.

**Temporary artifacts removed** (419 files: `*.py`/`*.c`/`*.gdb` scripts, `logs/*`, `evidence/*`,
`*.bin` payloads, compiled probes, and a `backup/` of the canonical build):

```
$ rm -rf /tmp/obs
after rm: /tmp/obs exists? NO
```

**Source checkout verified byte-for-byte unchanged** (container `/app`, after cleanup):

```
$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
$ git status --porcelain
            (empty — no output)
$ git diff --stat
            (empty — no output)
```

An empty `git status --porcelain` and an empty `git diff --stat` confirm the source repository is in
exactly the state it was checked out in, at HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`. `[observed]`

**Only one new file, in the destination checkout.** The single deliverable is added on the
destination branch `blitzy-79636b55-703f-4a38-adcc-ebec1f1a889d`, whose working tree is otherwise clean:

```
$ git -C <destination-checkout> status --porcelain
?? blitzy/
$ find blitzy -type f
blitzy/documentation/kitty_815df1e210e0.md
```

`blitzy/documentation/kitty_815df1e210e0.md` is the only file created — no README, index, `.gitkeep`,
or other file was added. `[observed]`

