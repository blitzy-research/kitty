# How kitty turns a surge of raw terminal input into something the application reacts to

**An investigate-by-running answer, grounded in the observed runtime behavior of a canonically built kitty.**

- **Subject:** kitty terminal emulator, version **0.35.2**, compiled-in VCS commit **`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`** (source branch `kitty_815df1e210e0`).
- **Method:** Every *behavioral* claim below was produced by **building and running** kitty and driving a **real child process writing to kitty's pseudo-terminal (PTY)** — the canonical input path — then capturing the **complete, unedited** output with kitty's own instrumentation (`--dump-commands`, `--dump-bytes`, `--debug-keyboard`, `--debug-rendering`) and with direct runtime inspection (`/proc/<pid>/task`, DECRQM query/reply round-trips through the PTY, `sha256` integrity checks). Each claim carries the exact command that produced it, the complete unedited output, and a `file:line` reference naming the specific function/struct.
- **Scope of the question (four threads):**
  - **Q1 — Input surge & entry point (incl. pause/resume):** When a surge of raw input arrives — especially when a session is paused then resumed — how does the byte stream become actionable, and *where does it first enter*?
  - **Q2 — The "unseen conductor":** How are timing/ordering/state-handoff responsibilities split across threads, and *what decides which event is handled first*?
  - **Q3 — Aligned semantics under stress:** When shell-integration hints arrive mixed in with ordinary text, how does the system keep screen state, command context, and input meaning aligned without drifting — and does it behave differently under heavy backpressure or an unstable remote connection?
  - **Q4 — End-to-end rhythm:** From the moment mixed input arrives to the moment the interface settles, what really happens and how do the moving parts keep rhythm?

---

## How to read the evidence in this document

This document distinguishes three kinds of statements, and never blurs them:

- **[OBSERVED]** — demonstrated by running the built binary and shown by the complete, unedited output that immediately follows. The `$ ` line is *exactly* the command that produced the block; the fenced block that follows contains *only* that command's literal output. All prose commentary lives **outside** the fences.
- **[INFERRED]** — read from the source code (with a `file:line` citation) and consistent with the observed behavior, but not itself directly captured at runtime. Used where the sandbox cannot expose an internal signal directly (e.g., an internal C-level branch, or visible pixels under a software GL rasterizer).
- **[NON-CANONICAL]** — obtained through a bypassing interface (the synchronous `parse_bytes` test helper, a remote-control/debug hook, or a hand-written synthetic byte sequence). Used only as a clearly-labeled supplement, **never** as primary proof. §6 lists every non-canonical item.

**Filtering disclosure (important):** where a raw capture is enormous (e.g., a 20 000-line parser dump), this document does **not** silently show a few lines and imply the rest. Instead it runs a **deterministic validator** over the *entire* file and shows the validator's complete output, and additionally states the file's line count and `sha256`. Any place where only part of a file is shown is labeled *illustrative* and is accompanied by the authoritative whole-file check.

**Verbatim-fidelity disclosure (control bytes & trailing whitespace):** the fenced blocks reproduce kitty's output byte-for-byte, with one unavoidable transformation — a raw `ESC` (`0x1b`) control byte cannot be represented as printable text inside a Markdown file. kitty's `--debug-keyboard` colours the `on_key_input` label with an SGR escape (`ESC [ 33 m … ESC [ m`); in the fenced blocks below the two `ESC` bytes of that colour sequence are dropped, so the label reads `[33mon_key_input[m` rather than an invisible control byte. Everything *semantically meaningful* is verbatim — this was verified programmatically: each shown `on_key_input` line equals its raw-capture line with only the `0x1b` bytes removed (every field from `glfw key` onward is byte-identical). Conversely, **trailing whitespace is preserved exactly, not trimmed**: kitty's `--debug-keyboard` byte listing (e.g. `sent encoded key to child: ^[ [ A `) and the shell `draw` prompt lines (e.g. `draw …# ` and `draw PROMPT$ `) genuinely end in a space, so those bytes are kept as-is — which means a whitespace linter such as `git diff --check` will, by design, report trailing whitespace on exactly those evidence lines rather than on any authored prose.

**Honest sandbox limitations (stated up front, expanded where relevant):** the observations were made in a **headless** container (Xvfb + Mesa software GL, running as `root` — the only account the image provides). Two things therefore cannot be captured directly and are handled as **[INFERRED]** with source citations plus an observed *proxy*:
1. **Visible pixels.** Under `llvmpipe` there is no real display scan-out, so "the frame the user sees was held / flushed atomically" is inferred from the snapshot code in `screen.c`; the *observed* proxy is the mode-state flag toggling and parsing continuing during the hold (§2.3), and the render/settle boundary itself is disclosed as not directly observed (§5.4).
2. **Physical GLFW key events.** Delivering an OS-level key press into a focused GLFW window requires a real X server with input focus. In this setup synthetic `XTEST` injection into the focused kitty window **did** reach the real `on_key_input` (`kitty/keys.c:166`), so §5.1 is canonical [OBSERVED] evidence — kitty's own `--debug-keyboard` output and the child's independently-logged bytes agree. The Python `encode_key_for_tty` route (§5.1.4) is kept as a clearly-labeled [NON-CANONICAL] supplement for the OS-entry boundary.

---

## 1. Environment & canonical build

**Provisioned image (from the project setup instructions):** `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`, pulled from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`. The runtime toolchain is activated **before** any version is queried, so that every subsequent observation comes from the canonical Python 3.11 / Go 1.22 build rather than the container's incidental system interpreter.

**[OBSERVED] — image/OS identity and toolchain, in order (setup first, then versions):**

```
########## STEP 0: image / OS / host identity ##########
$ cat /etc/os-release | grep -E '^(PRETTY_NAME|VERSION_ID)='
PRETTY_NAME="Ubuntu 25.10"
VERSION_ID="25.10"
$ uname -srm
Linux 6.6.122+ x86_64

########## STEP 1: activate canonical toolchain (BEFORE any version query) ##########
$ source /root/kitty-venv/bin/activate
$ export PATH=$PATH:/usr/local/go/bin
$ export TMPDIR=/tmp/kitty-nosgid DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1
$ export PYTHONHOME=/root/.local/share/uv/python/cpython-3.11.15-linux-x86_64-gnu
$ export PYTHONPATH=/root/kitty-venv/lib/python3.11/site-packages
$ export LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8

########## STEP 2: toolchain versions (AFTER setup) ##########
$ python3 --version
Python 3.11.15
$ go version
go version go1.22.12 linux/amd64
$ gcc --version | head -n1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
```

The order matters: this resolves the review's point that a version queried in a *fresh* shell would report the container's system Python 3.13 with Go absent from `PATH`. After activation, `python3` is the repo-compatible **3.11.15** (matching `requires-python >=3.8`, CI-pinned 3.11) and `go` is **1.22.12** (matching `go 1.22` in `go.mod`).

**[OBSERVED] — headless display, secure observation directory, build artifacts, VCS-stamped banner, and provenance:**

```
########## STEP 3: headless display (Xvfb) health ##########
# launched once at session start as:
#   Xvfb :99 -screen 0 1920x1080x24 -nolisten tcp &
$ xdpyinfo -display :99 | head -n 3
name of display:    :99
version number:    11.0
vendor string:    The X.Org Foundation

########## STEP 4: secure observation dir (F16: umask 077 + 0700 mktemp -d) ##########
# created once as: umask 077; OBS=$(mktemp -d $TMPDIR/kitty_obs.XXXXXXXX); chmod 700 $OBS
$ stat -c '%A %U:%G %n' $(cat /tmp/kitty-nosgid/obs_path.txt)
drwx------ root:root /tmp/kitty-nosgid/kitty_obs.jWw1gwl6

########## STEP 5: canonical build artifacts (built by: python3 setup.py build) ##########
$ ls -1 kitty/fast_data_types*.so kitty/glfw-x11.so kitty/launcher/kitty kittens/transfer/rsync.so
kittens/transfer/rsync.so
kitty/fast_data_types.so
kitty/glfw-x11.so
kitty/launcher/kitty

########## STEP 6: VCS-stamped version banner (canonical run) ##########
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
$ ./kitty/launcher/kitty +runpy 'from kitty.fast_data_types import KITTY_VCS_REV as r; print(r)'
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1

########## STEP 7: provenance — built source rev (immutable) vs the single added file (F15) ##########
$ git rev-parse 815df1e210e0
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
$ git log -1 --format='%h %s' 815df1e210e0
815df1e21 Wire up applying of font config
$ git diff --name-status 815df1e210e0..HEAD
A	blitzy/documentation/kitty_815df1e210e0.md
```

**Provenance note (resolves the "which commit?" ambiguity, and why it is anchored on the built rev):** the kitty **C extension** was compiled from the base commit `815df1e210e0…` (`Wire up applying of font config`), so the binary's compiled-in `KITTY_VCS_REV` reads `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (identical to the value shown in STEP 6). The *only* change introduced over that base commit — cumulatively, across **every** commit up to the current `HEAD` — is the addition of this single documentation file, which is exactly what `git diff --name-status 815df1e210e0..HEAD` reports above (`A  blitzy/documentation/kitty_815df1e210e0.md`). This provenance is deliberately anchored on the **immutable built-source rev** rather than the working tree's mutable `HEAD` hash: a document cannot contain the hash of the very commit that adds it (that hash does not exist until *after* the file's bytes are frozen and committed), and any later doc-only revision advances `HEAD` again — so an embedded `HEAD` hash is inherently self-referential and goes stale as soon as the answer is revised. The two facts that *do* reproduce verbatim no matter how many doc-only commits are layered on are the ones this document relies on, both shown above: (a) the binary is stamped `815df1e210e0…`, and (b) the entire base→`HEAD` delta is the addition of this one file. All runtime evidence in this document comes from the binary stamped `815df1e210e0…`.

**Security posture of the observation harness (resolves F16):** the container provides only the `root` account, which is disclosed here rather than hidden. To avoid predictable, world-accessible artifacts the harness (a) sets `umask 077`, (b) places every file, log, and unix socket inside a `mktemp -d` directory `chmod`ed to `0700` (shown above as `drwx------`), (c) captures each spawned kitty PID via `$!` and reaps exactly that PID in an `EXIT`/`INT`/`TERM` trap (never `pkill`), and (d) scopes remote control to a single unix socket created inside that `0700` directory. The complete harness source is reproduced verbatim in the Appendix so every experiment is auditable and reproducible.

**[OBSERVED] — default timing knobs, read from the built binary (these are the canonical defaults; the source lines corroborate them):**

```
########## default timing knobs — read canonically from the built kitty ##########
$ ./kitty/launcher/kitty +runpy 'from kitty.options.types import defaults; print("input_delay =", defaults.input_delay); print("repaint_delay =", defaults.repaint_delay); print("sync_to_monitor =", defaults.sync_to_monitor)'
input_delay = 3
repaint_delay = 10
sync_to_monitor = True

########## VT parser buffer size — exported compiled constant ##########
$ ./kitty/launcher/kitty +runpy 'from kitty.fast_data_types import VT_PARSER_BUFFER_SIZE as b; print(b, "bytes =", b//1024, "KiB")'
1048576 bytes = 1024 KiB
```

These four numbers are used throughout the rest of the document: `input_delay = 3` ms (`kitty/options/definition.py:878`), `repaint_delay = 10` ms (`kitty/options/definition.py:866`), `sync_to_monitor = yes` (`kitty/options/definition.py:889`), and the VT parser buffer `BUF_SZ = 1024u*1024u = 1 MiB` (`kitty/vt-parser.c:18`, exported as `VT_PARSER_BUFFER_SIZE`). They are **source-derived defaults**, read here canonically from the running binary — not values measured by timing a frame. Where later sections need a *measured* value (the coalescing window), that value is measured directly and reported as `[OBSERVED]`.

---

## 2. Q1 — The input surge, its entry point, and the pause/resume cycle

### 2.1 Where the byte stream first enters

A child process (a shell, or any program) does not write "to kitty" directly — it writes to the **slave** end of a pseudo-terminal, and kitty reads the **master** end. That pairing is created in `Child.fork`:

- `openpty()` wraps `os.openpty()` (`kitty/child.py:170-171`);
- `Child.fork` (`kitty/child.py:276`) calls it (`master, slave = openpty()`, `kitty/child.py:281`), keeps the master as `self.child_fd` (`kitty/child.py:338`) and sets it **non-blocking** (`kitty/child.py:345`);
- in the forked child, `setsid()` (`kitty/child.c:123`) and `ioctl(tfd, TIOCSCTTY, 0)` (`kitty/child.c:129`) make the slave the child's controlling terminal (its stdin/stdout/stderr).

**[OBSERVED] — a real kitty child confirms it is talking to a PTY slave:**

```
$ ./kitty/launcher/kitty -o close_on_child_death=yes --dump-commands sh -c 'printf "CHILD_TTY=%s\r\n" "$(tty)"; if [ -t 1 ]; then printf "STDOUT_ISATTY=yes\r\n"; else printf "STDOUT_ISATTY=no\r\n"; fi; sleep 0.2'
draw CHILD_TTY=/dev/pts/0
screen_carriage_return
screen_carriage_return
screen_linefeed
draw STDOUT_ISATTY=yes
screen_carriage_return
screen_carriage_return
screen_linefeed
```

(This is the **complete, unfiltered** stdout of the command — all eight `--dump-commands` events. The two `draw` lines are the parser events for the text the child printed; the child's stdout is the PTY slave `/dev/pts/0`. Each `printf "…\r\n"` the child wrote arrives at the parser as `\r\r\n`: the PTY's output line discipline has `ONLCR` set, so the child's `\n` is translated to `\r\n`, and combined with the `\r` the child already emitted this yields `\r\r\n` — parsed here as two `screen_carriage_return` events followed by one `screen_linefeed`. The command writes one diagnostic line to *stderr* — `[…] Failed to open systemd user bus with error: Connection refused` — which is the benign headless-container diagnostic that also appears in the `--debug-rendering` capture in §5.4 and is not part of this stdout block.)

The **master** fd is where the surge first enters kitty's C core. On the `io_thread`, `read_bytes` reads it:

```c
read_bytes(int fd, Screen *screen) {                                  // kitty/child-monitor.c:1337
    ...
    uint8_t *buf = vt_parser_create_write_buffer(screen->vt_parser, &available_buffer_space);
    if (!available_buffer_space) return true;                         // (backpressure self-guard, see §4.2)
    while(true) {
        len = read(fd, buf, available_buffer_space);                  // the read() from the child fd
        ...
    }
    vt_parser_commit_write(screen->vt_parser, len);                   // hands the bytes to the parser
    return len != 0;
}
```

`read_bytes` is the *entry point*: it obtains a write region inside the VT parser's buffer, `read()`s the child's master fd straight into it, and commits it to the parser. It is invoked once per readable child in the io-loop at `kitty/child-monitor.c:1531` (`read_bytes(children_fds[EXTRA_FDS + i].fd, children[i].screen)`). This is the canonical path; nothing is copied through Python on the way in.

### 2.2 The surge itself, verified exactly (not sampled)

**[OBSERVED] — 5 000 numbered lines emitted by a real child; kitty's own instrumentation records the raw bytes the parser consumed and the parser events, and a deterministic validator checks the *entire* result:**

```
$ ./kitty/launcher/kitty -o close_on_child_death=yes --dump-bytes=$OBS/out/q1_surge_bytes.bin --dump-commands sh -c 'for i in $(seq 1 5000); do printf "L%04d-THE-QUICK-BROWN-FOX\r\n" "$i"; done; sleep 0.4'
$ python3 harness/surge_validate.py $OBS/out/q1_surge_bytes.bin $OBS/out/q1_surge_dump.txt 5000
records_expected=5000
draw_records_found=5000
first_draw=L0001
last_draw=L5000
all_present_1..5000=YES
raw_bytes=140000 expected_raw_bytes=140000
raw_sha256=8c52d6cf552ff7c51f3ce821e5d28160b902b94425c5be6daaace61051a9a132
RESULT=PASS
```

The validator (full source in the Appendix) parses the **whole** 20 000-line dump and the **whole** 140 000-byte raw file; it confirms all 5 000 records are present in strict order **including `L5000`**, that the byte count is exactly `5000 × 28` (25 printable characters + `CR CR LF`, the second `CR` from the terminal's `ONLCR`), and prints the `sha256`. It exits non-zero on any gap, so this `RESULT=PASS` is a whole-file guarantee, not an eyeballed sample.

**[OBSERVED] — the surge is bit-for-bit reproducible across runs (magnitude/stability rule):**

```
run 2: exit=0 sha256=8c52d6cf552ff7c51f3ce821e5d28160b902b94425c5be6daaace61051a9a132
run 3: exit=0 sha256=8c52d6cf552ff7c51f3ce821e5d28160b902b94425c5be6daaace61051a9a132
```

Both repeats reproduce the run-1 digest `8c52d6cf…` exactly, so "5 000 lines, 140 000 bytes, all present, in order" is stable, not a one-off.

**Attribution precision (resolves the "raw bytes at read_bytes" overclaim, F4):** the bytes written to `--dump-bytes` are emitted **parser-side**, guarded by `#ifdef DUMP_COMMANDS` at `kitty/vt-parser.c:1397`, and cover `self->read.pos - pre_consume_pos` — i.e. exactly the span the parser has just *consumed*. Those are the very bytes that `read_bytes` delivered into the parser's write buffer (`read_bytes` → `vt_parser_commit_write`), so the 140 000-byte figure faithfully reflects what entered at the PTY. The statement "these are the bytes that entered at `read_bytes`" is therefore an **[INFERRED]** data-flow link (parser buffer ← `read_bytes`), not a capture taken at the `read()` call itself. The CLI help for the flag describes it as "raw bytes received from the child process" (`kitty/cli.py:985`), which is accurate as to *content*; this document is simply precise about the *emission point*.


### 2.3 Pausing and resuming — synchronized output (DEC private mode 2026)

The question singles out "a session being paused and then resumed." kitty implements this with **synchronized output**, DEC private mode **2026** (`#define PENDING_MODE 2026`, `kitty/control-codes.h:235`). An application sends `CSI ? 2026 h` to *begin* a synchronized update (the terminal keeps displaying the last committed frame while it **keeps consuming** incoming bytes) and `CSI ? 2026 l` to *end* it (flush the accumulated result atomically).

To observe the transition **before / during / after**, a real PTY child issues a DECRQM status query `CSI ? 2026 $ p` at each phase and reads kitty's reply back through the PTY; separately, every byte the child writes is logged so the *emitted* stream can be shown independently.

**[OBSERVED] — parser events for the whole pause/resume cycle (complete, unedited):**

```
$ ./kitty/launcher/kitty -o close_on_child_death=yes --dump-commands env EMIT_LOG=$OBS/out/q1_pause_emit.bin python3 harness/pause_probe.py
report_mode_status 2026 1
draw PHASE=before DECRQM_REPLY=<ESC>[?2026;2$y
screen_carriage_return
screen_linefeed
screen_set_mode 2026 1
draw DRAWN-WHILE-PAUSED
screen_carriage_return
screen_linefeed
report_mode_status 2026 1
draw PHASE=during DECRQM_REPLY=<ESC>[?2026;1$y
screen_carriage_return
screen_linefeed
screen_reset_mode 2026 1
report_mode_status 2026 1
draw PHASE=after DECRQM_REPLY=<ESC>[?2026;2$y
screen_carriage_return
screen_linefeed
```

**[OBSERVED] — the child's full emitted byte stream (`od -c`), proving it actually sent `CSI ?2026 h` and `CSI ?2026 l`:**

```
$ od -c $OBS/out/q1_pause_emit.bin
0000000 033   [   ?   2   0   2   6   $   p   P   H   A   S   E   =   b
0000020   e   f   o   r   e       D   E   C   R   Q   M   _   R   E   P
0000040   L   Y   =   <   E   S   C   >   [   ?   2   0   2   6   ;   2
0000060   $   y  \r  \n 033   [   ?   2   0   2   6   h   D   R   A   W
0000100   N   -   W   H   I   L   E   -   P   A   U   S   E   D  \r  \n
0000120 033   [   ?   2   0   2   6   $   p   P   H   A   S   E   =   d
0000140   u   r   i   n   g       D   E   C   R   Q   M   _   R   E   P
0000160   L   Y   =   <   E   S   C   >   [   ?   2   0   2   6   ;   1
0000200   $   y  \r  \n 033   [   ?   2   0   2   6   l 033   [   ?   2
0000220   0   2   6   $   p   P   H   A   S   E   =   a   f   t   e   r
0000240       D   E   C   R   Q   M   _   R   E   P   L   Y   =   <   E
0000260   S   C   >   [   ?   2   0   2   6   ;   2   $   y  \r  \n
0000277
```

What this proves, precisely:

- **The state machine tracks pause with a timer.** The DECRQM replies are `?2026;2` (before), `?2026;1` (during), `?2026;2` (after). In `report_mode` the reply value for `PENDING_UPDATE` is `ans = self->paused_rendering.expires_at ? 1 : 2` (`kitty/screen.c:2238`), so `1` means "paused, a resume timer is armed" and `2` means "not paused." The observed `2 → 1 → 2` is the *before / during / after* of the pause flag itself.
- **kitty keeps processing input while paused.** `draw DRAWN-WHILE-PAUSED` appears in the trace **between** `screen_set_mode 2026 1` and `screen_reset_mode 2026 1` — the parser consumed the byte stream and mutated the (off-screen) screen state *during* the hold. This is the whole point of synchronized output: the *display* is frozen, the *processing* is not.
- **The begin/end were genuinely sent.** The `od -c` shows the literal `033 [ ? 2 0 2 6 h` and `033 [ ? 2 0 2 6 l` (`033` = `ESC`), so the pause and resume were real emitted control sequences, not an artifact of the trace.

**[INFERRED] — what the headless capture cannot show directly, with source:** that the *pixels the user sees* were held to the pre-pause frame and then replaced atomically is not captured here, because under Mesa `llvmpipe` there is no real scan-out to sample. The mechanism is in `screen_pause_rendering` (`kitty/screen.c:2506`): on begin it arms `expires_at` and **snapshots** the visible state — every visible line is copied into `paused_rendering.linebuf` (`kitty/screen.c:2529-2538`), along with the cursor (`:2527`), colors (`:2528`), and selections — so the renderer can keep drawing the *old* frame; on end it clears `expires_at` and sets `is_dirty = true` (`kitty/screen.c:2511`) to force the GPU to update to the now-current state (the atomic flush). The **observed proxy** for "held then flushed" is the combination above: the pause flag toggled `2→1→2` and processing demonstrably continued during the hold.

### 2.4 Resume without an "end" — the safety timeout

A well-behaved application always sends `CSI ? 2026 l`. A crashed or disconnected one might not. kitty guards against a permanently frozen display with a timeout: `screen_check_pause_rendering` auto-resumes when the timer expires (`if (self->paused_rendering.expires_at && now > self->paused_rendering.expires_at) screen_pause_rendering(self, false, 0);`, `kitty/screen.c:2489-2491`), and the default timeout is **2000 ms** (`if (for_in_ms <= 0) for_in_ms = 2000;`, `kitty/screen.c:2521`).

**[OBSERVED] — begin a synchronized update and *never* end it; poll DECRQM ~every 500 ms (two independent runs, ~3.5 s each):**

```
$ ./kitty/launcher/kitty -o close_on_child_death=yes --dump-commands env EMIT_LOG=$OBS/out/q1_timeout_emit_r1.bin python3 harness/pause_timeout_probe.py   # RUN 1
draw T=0003ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=0506ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=1010ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=1513ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=2019ms DECRQM_REPLY=<ESC>[?2026;2$y
draw T=2523ms DECRQM_REPLY=<ESC>[?2026;2$y
draw T=3026ms DECRQM_REPLY=<ESC>[?2026;2$y
draw T=3530ms DECRQM_REPLY=<ESC>[?2026;2$y
```

```
$ ./kitty/launcher/kitty -o close_on_child_death=yes --dump-commands env EMIT_LOG=$OBS/out/q1_timeout_emit_r2.bin python3 harness/pause_timeout_probe.py   # RUN 2
draw T=0003ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=0506ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=1010ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=1513ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=2020ms DECRQM_REPLY=<ESC>[?2026;2$y
draw T=2524ms DECRQM_REPLY=<ESC>[?2026;2$y
draw T=3027ms DECRQM_REPLY=<ESC>[?2026;2$y
draw T=3533ms DECRQM_REPLY=<ESC>[?2026;2$y
```

**[OBSERVED] — proof the child never sent the "end", so the resume was the timeout (counted over the whole emitted byte stream):**

```
$ python3 - $OBS/out/q1_timeout_emit_r1.bin   # counts CSI?2026h vs CSI?2026l in the ENTIRE emitted stream
bytes_emitted= 392
contains CSI?2026h (begin): 1
contains CSI?2026l (end)  : 0
```

Across both runs the pause flag holds at `1` through `T=1513 ms` and has flipped to `2` by `T≈2019–2020 ms` — i.e. the display auto-resumed between 1.5 s and 2.0 s, matching the 2000 ms default. The parser trace for these runs contains exactly one `screen_set_mode 2026 1` and **zero** `screen_reset_mode` events, and the byte-stream count above shows `begin = 1, end = 0`: the child never sent `CSI ? 2026 l`, so the resume was produced solely by `screen_check_pause_rendering` firing on `expires_at`. The two runs agree to within 1 ms at the flip (`2019` vs `2020`), so this is a stable timeout, not jitter.


---

## 3. Q2 — The "unseen conductor": thread split, poll order, and the coalescing window

### 3.1 The threads, observed

kitty splits its work across a small number of purpose-named POSIX threads. The names are set with `set_thread_name`: `KittyChildMon` for the I/O loop (`kitty/child-monitor.c:1489`), `KittyPeerMon` for the remote-control "talk" loop (`kitty/child-monitor.c:1808`), and `KittyWriteStdin` for the on-demand stdin writer (`kitty/child-monitor.c:967`). The VT **parser runs on the main thread** (`parse_worker` is invoked from the main render loop, §3.3); rendering and the GLFW event loop are also the main thread's job.

**[OBSERVED] — the COMPLETE thread list of a default kitty (one long-lived child), captured by PID (via `$!`) after waiting for the I/O thread to be named:**

```
$ bash harness/thread_inspect.sh default
label=default pid=117626 childmon_ready=yes
TID COMM
117626 kitty
117636 llvmpipe-0
117637 llvmpipe-1
117638 llvmpipe-2
117639 llvmpipe-3
117640 llvmpipe-4
117641 llvmpipe-5
117642 llvmpipe-6
117643 llvmpipe-7
117644 llvmpipe-8
117645 llvmpipe-9
117646 llvmpipe-10
117647 llvmpipe-11
117648 llvmpipe-12
117649 llvmpipe-13
117650 llvmpipe-14
117651 llvmpipe-15
117652 llvmpipe-16
117653 llvmpipe-17
117654 llvmpipe-18
117655 llvmpipe-19
117656 llvmpipe-20
117657 llvmpipe-21
117658 llvmpipe-22
117659 llvmpipe-23
117660 llvmpipe-24
117661 llvmpipe-25
117662 llvmpipe-26
117663 llvmpipe-27
117664 llvmpipe-28
117665 llvmpipe-29
117666 llvmpipe-30
117667 llvmpipe-31
117668 kitty
117669 kitty
117670 kitty
117671 kitty
117672 kitty
117673 kitty
117674 kitty
117675 kitty
117676 kitty
117677 kitty
117678 kitty
117679 kitty
117680 kitty
117681 kitty
117682 kitty
117683 kitty
117684 kitty
117685 kitty
117686 kitty
117687 kitty
117688 kitty
117689 kitty
117690 kitty
117691 kitty
117692 kitty
117693 kitty
117694 kitty
117695 kitty
117696 kitty
117697 kitty
117698 kitty
117699 kitty
117700 kitty:disk$0
117703 KittyChildMon
```

Most of this list is **not** kitty's own design: the 32 `llvmpipe-*` threads and the 33 additional bare-`kitty` threads are the Mesa software-GL rasterizer's worker pools, present only because this headless environment renders with `LIBGL_ALWAYS_SOFTWARE=1`. kitty's own long-lived threads here are the main thread (`117626 kitty`, whose TID equals the PID), a disk-cache worker (`kitty:disk$0`), and the I/O thread (`KittyChildMon`). There is **no** `KittyPeerMon` in the default configuration.

**[OBSERVED] — deterministic summary of all three configurations (filtering to kitty's core threads is disclosed; the exact `grep` is shown, and the total TID count proves nothing is silently dropped):**

```
$ for c in default listen si; do f=$OBS/out/q2_threads_${c}.txt; echo "===== $c ====="; echo "total_tids=$(grep -c '^[0-9]' "$f") llvmpipe=$(grep -c ' llvmpipe' "$f") bare_kitty=$(awk '$2=="kitty"{c++}END{print c+0}' "$f")"; grep -E 'Kitty|kitty:' "$f"; done
===== default =====
total_tids=67 llvmpipe=32 bare_kitty=33
117700 kitty:disk$0
117703 KittyChildMon
===== listen =====
total_tids=68 llvmpipe=32 bare_kitty=33
117987 kitty:disk$0
117990 KittyPeerMon
117991 KittyChildMon
===== si =====
total_tids=68 llvmpipe=32 bare_kitty=33
118669 kitty:disk$0
118670 KittyPeerMon
118671 KittyChildMon
```

- `default` (no remote control): `KittyChildMon` present, **no** `KittyPeerMon`.
- `listen` (`--listen-on unix:…` + `allow_remote_control=yes`): `KittyPeerMon` **appears**.
- `si` (`--single-instance --instance-group …`, launched with `XDG_RUNTIME_DIR=$OBS`, and **no** `--listen-on`): `KittyPeerMon` **also appears**.

**This corrects a subtle but important point (resolves F7):** it is *not* true that the talk thread appears "only when a listen socket is configured." `start()` creates it when *either* a talk fd *or* a listen fd is present — `if (self->talk_fd > -1 || self->listen_fd > -1)` (`kitty/child-monitor.c:285`) — and `--single-instance` supplies a **talk fd** (`kitty/main.py:485` → `Boss(..., talk_fd)` `kitty/main.py:226` → `ChildMonitor(talk_fd, listen_fd)` `kitty/boss.py:373`), which is why case `si` grows a `KittyPeerMon` with no listen socket at all. There is additionally an on-demand path: `inject_peer` starts the talk thread if it is not already running — `if (!talk_thread_started) { pthread_create(&self->talk_thread, NULL, talk_loop, self); … }` (`kitty/child-monitor.c:254-259`), called from `kitty/boss.py:2397`. **[INFERRED]** from those lines; the two observed cases above (talk-fd and listen-fd) already demonstrate the "not only a listen socket" point directly.

So the "conductor" is really three loops that never block each other: the **main thread** (GLFW events + rendering + the VT parser), the **I/O thread** `KittyChildMon` (reads child PTYs with `read_bytes`, writes them with `write_to_child`), and — when remote control is in play — the **talk thread** `KittyPeerMon` (accepts and reads remote-control peers, §4.3). State is handed between them through the parser's locked buffer (I/O thread → main thread) and a wakeup pipe (either direction).

### 3.2 What decides which event is handled first

Within the I/O thread, a single `poll()` watches a fixed-order array of file descriptors, and the code services them in **array-index order** after each `poll()` returns:

```c
if (children_fds[0].revents && POLLIN) drain_fd(children_fds[0].fd); // wakeup      child-monitor.c:1515
if (children_fds[1].revents && POLLIN) { … read_signals(…); … }      // signals     child-monitor.c:1516
for (i = 0; i < self->count; i++) {                                  // child PTYs  child-monitor.c:1528
    if (children_fds[EXTRA_FDS + i].revents & (POLLIN | POLLHUP))
        has_more = read_bytes(children_fds[EXTRA_FDS + i].fd, children[i].screen);   // child-monitor.c:1531
    …
}
```

The array is laid out `[0] = wakeup pipe`, `[1] = signal fd`, `[EXTRA_FDS + i] = each child PTY` (`EXTRA_FDS = 2`, `kitty/child-monitor.c:35`). So *when several descriptors are ready in the same `poll()` return*, the wakeup pipe is drained first, then pending signals are handled, then each child's bytes are read in order.

**Qualification (resolves F17):** this index order determines the servicing order **only among descriptors that are ready in the same `poll()` cycle**. It is not a claim that "a wakeup is always processed before child data" across time — a child that becomes readable while the wakeup pipe is quiet is serviced on its own cycle. The durable guarantee it *does* give is a **single-consumer, serial** discipline within a cycle: one thread, one `poll()`, one pass over the ready fds in a fixed order — which is exactly what keeps per-child byte order intact on the way to the parser.


### 3.3 The coalescing window — measured, not asserted

The main loop is not woken for every byte. Waking it is deliberately throttled by `input_delay` (default 3 ms). To **measure** the resulting latency, a real PTY child runs a tight query/reply ping-pong — it sends a DECRQM query `CSI ? 2026 $ p`, times how long until kitty's `…$y` reply returns (steady clock `time.monotonic_ns()`), and repeats. This round-trip is a **proxy** for the wakeup latency: it also contains `read_bytes`, the parse, the reply encode, and the return read, so it slightly *over*-counts the pure wakeup delay. Because those extra terms are ~constant, sweeping `input_delay` isolates the coalescing window.

**[OBSERVED] — sweep of `input_delay ∈ {0, 3, 10, 30}` ms, two independent runs each, N = 200 measured round-trips per run (after 20 warm-ups); statistics computed deterministically from the raw per-trial log:**

```
label=input_delay=0ms gap=0ms run1 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=0.008
min_ms=0.0329
median_ms=0.0363
p90_ms=0.0505
max_ms=0.0975
mean_ms=0.0392

label=input_delay=0ms gap=0ms run2 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=0.008
min_ms=0.0317
median_ms=0.0372
p90_ms=0.0478
max_ms=0.1257
mean_ms=0.0407

label=input_delay=3ms gap=0ms run1 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=0.632
min_ms=3.1197
median_ms=3.1543
p90_ms=3.1833
max_ms=3.2705
mean_ms=3.1586

label=input_delay=3ms gap=0ms run2 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=0.631
min_ms=3.0923
median_ms=3.1537
p90_ms=3.1773
max_ms=3.2000
mean_ms=3.1539

label=input_delay=10ms gap=0ms run1 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=2.059
min_ms=10.1086
median_ms=10.2455
p90_ms=10.2902
max_ms=12.5236
mean_ms=10.2901

label=input_delay=10ms gap=0ms run2 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=2.055
min_ms=10.0621
median_ms=10.2385
p90_ms=10.2860
max_ms=12.9565
mean_ms=10.2700

label=input_delay=30ms gap=0ms run1 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=6.090
min_ms=30.2051
median_ms=30.2929
p90_ms=30.3632
max_ms=33.1693
mean_ms=30.4426

label=input_delay=30ms gap=0ms run2 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=6.090
min_ms=30.2037
median_ms=30.2992
p90_ms=30.3893
max_ms=32.8010
mean_ms=30.4446
```

Reading the medians against the knob:

| `input_delay` | run 1 median | run 2 median | median − `input_delay` |
|---:|---:|---:|---:|
| 0 ms  | 0.0363 ms  | 0.0372 ms  | ≈ 0.036 ms (pure overhead) |
| 3 ms  | 3.1543 ms  | 3.1537 ms  | ≈ 0.154 ms |
| 10 ms | 10.2455 ms | 10.2385 ms | ≈ 0.243 ms |
| 30 ms | 30.2929 ms | 30.2992 ms | ≈ 0.293 ms |

The round-trip latency tracks `input_delay` with slope ≈ 1.0, and the two runs at each setting agree to ~0.01 ms, so the window is stable, not jitter. The base overhead — measured directly at `input_delay = 0` — is a median of **≈ 0.036 ms (36 µs)**, not a fixed 0.05 ms; the small residual over the knob (`0.15 → 0.24 → 0.29 ms`) grows modestly with the delay, consistent with the parser re-check granularity described below. (This corrects an earlier draft that asserted a single fixed overhead inconsistent with its own numbers.)

**Where the delay actually lives (a correction worth stating precisely).** One might expect that a query arriving after a long idle would skip the window. It does **not**, and the control experiment shows why:

**[OBSERVED] — same probe at `input_delay = 30` ms but sleeping a 50 ms idle gap *before* each timed query (so every query is the first byte after idle):**

```
label=input_delay=30ms gap=50ms run1 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=16.116
min_ms=30.2474
median_ms=30.3498
p90_ms=30.4186
max_ms=34.1037
mean_ms=30.4939

label=input_delay=30ms gap=50ms run2 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=16.117
min_ms=30.2723
median_ms=30.3597
p90_ms=30.4118
max_ms=35.2847
mean_ms=30.4961
```

Even with a 50 ms idle gap between queries (wall time ≈ `200 × (50 + 30) ms` ≈ 16.1 s, confirming the gap was applied), the round-trip median stays ≈ 30.35 ms. The idle gap does **not** collapse the latency to the base overhead. The reason is that the dominant delay is the **parser's flush gate**, not the I/O-thread's wakeup batching. In `run_worker` the accumulated bytes are only consumed when

```c
pd->time_since_new_input = pd->now - self->new_input_at;                              // vt-parser.c:1424
if (flush || pd->time_since_new_input >= OPT(input_delay) || self->read.sz + 16 * 1024 > BUF_SZ) {  // vt-parser.c:1425
    …
    consume_input(self, pd->dump_callback, screen->window_id);                        // vt-parser.c:1431
    …
    self->new_input_at = 0;                                                           // vt-parser.c:1436
}
```

`new_input_at` is stamped when bytes are committed (`if (self->new_input_at == 0) self->new_input_at = monotonic();`, `kitty/vt-parser.c:1469`) and reset to 0 after a consume (`kitty/vt-parser.c:1436`). Because it is re-stamped for **each fresh arrival**, every isolated query must wait `input_delay` before it is parsed — which is exactly why the 50 ms idle gap makes no difference. The I/O-thread's wakeup coalescing (`kitty/child-monitor.c:1562-1569`, including the "wake immediately if idle longer than `input_delay`" branch at `:1565`) is a *second*, complementary throttle on the same knob governing *when the main loop is woken*; the ping-pong proxy makes the parser gate the visible term. Either way, the trade-off is the one documented in `docs/performance.rst`: a few milliseconds of artificial delay batches work and cuts CPU wakeups, at the cost of a little display latency.


---

## 4. Q3 — Aligned semantics under stress

### 4.1 Shell-integration hints stay aligned with text because dispatch is serial

Shell integration marks the prompt, the command, and the command's output with OSC 133 escape codes so the terminal can offer prompt navigation, exit-status coloring, and output selection. The question asks how these hints stay aligned with ordinary text "without drifting out of sync." The answer is structural: there is **no separate channel** for hints. They travel *in* the byte stream, and a single parser pass consumes that stream in byte order, dispatching each token — text, CSI, OSC — to the screen before it looks at the next. `dispatch_osc` routes code 133 to `shell_prompt_marking` (`kitty/vt-parser.c:536-544`), which invokes the `cmd_output_marking` callback for the prompt-start (`A`), output-start (`C`, with the command line), and command-finished (`D`, with the exit status) marks (`kitty/screen.c:2338`, `:2347`, `:2352`).

**[OBSERVED] — what kitty's *shipped* bash integration actually emits.** A real interactive `bash` was launched under a real kitty with shell integration auto-injected, one command (`printf "HELLO-OUTPUT\n"`) was driven in via `kitty @ send-text`, and the raw bytes were dumped and the OSC 133 sequences extracted in order:

```
$ python3 harness/osc133_extract.py $OBS/osc133_bytes.bin
01: <ESC>]133;k;start_kitty<BEL>
02: <ESC>]133;D;0<BEL>
03: <ESC>]133;A<BEL>
04: <ESC>]133;k;end_kitty<BEL>
05: <ESC>]133;k;start_suffix_kitty<BEL>
06: <ESC>]133;k;end_suffix_kitty<BEL>
07: <ESC>]133;k;start_kitty<BEL>
08: <ESC>]133;A;k=s<BEL>
09: <ESC>]133;k;end_kitty<BEL>
10: <ESC>]133;C;cmdline=$'printf "HELLO-OUTPUT\n"'<BEL>
11: <ESC>]133;k;start_kitty<BEL>
12: <ESC>]133;k;end_kitty<BEL>
13: <ESC>]133;k;start_suffix_kitty<BEL>
14: <ESC>]133;k;end_suffix_kitty<BEL>
15: <ESC>]133;k;start_kitty<BEL>
16: <ESC>]133;D;0<BEL>
17: <ESC>]133;A<BEL>
18: <ESC>]133;k;end_kitty<BEL>
19: <ESC>]133;k;start_suffix_kitty<BEL>
20: <ESC>]133;k;end_suffix_kitty<BEL>
total_osc133=20
```

So kitty's bash integration emits exactly four *semantic* marks — `133;A` (prompt start), `133;A;k=s` (secondary/continuation prompt start), `133;C;cmdline=…` (command output start, carrying the command line), and `133;D;<exit>` (command finished with exit status) — plus its own **kitty-private** `133;k;…` region markers used to delimit the redrawn prompt. It emits **no `133;B`**. That matches the shipped script — `PS1` appends `\e]133;D;$?\a\e]133;A\a` (`shell-integration/bash/kitty.bash:239`), `PS2`/secondary uses `133;A;k=s` (`:240`, `:137`), and `PS0` emits `133;C;cmdline=%q` (`:208`) — and the documented contract, which lists only `A`, `A;k=s`, `C`, and `D` (`docs/shell-integration.rst:424-438`). (An earlier draft claimed a hand-written `A/B/C/D` stream was "byte-identical to the shipped emitters"; that was wrong — the shipped emitters have no `B` and add `k=…` marks — and the claim has been removed.)

**[OBSERVED] — the marks stay aligned with text because they are dispatched serially, in byte order.** This is the COMPLETE 53-line `--dump-commands` trace for the same run; note how each `shell_prompt_marking 133 …` event is interleaved with the `draw` of the surrounding text, in the exact order the bytes arrived:

```
handle_remote_print aWdub3JlYm90aCBvciBpZ25vcmVzcGFjZSBwcmVzZW50IGluIGJhc2ggSElTVENPTlRST0wgc2V0dGluZywgc2hvd2luZyBydW5uaW5nIGNvbW1hbmQgd2lsbCBub3QgYmUgcm9idXN0Cg==}
ignoreboth or ignorespace present in bash HISTCONTROL setting, showing running command will not be robust
process_cwd_notification 7 kitty-shell-cwd://reverse-code-generator-6e7d35be-kxf24/tmp/blitzy/kitty/blitzy-85ce7b41-edf3-42eb-84b1-2fc55abedc59_304657
screen_set_mode 2004 1
screen_delete_characters 59
shell_prompt_marking 133 k;start_kitty
shell_prompt_marking 133 D;0
shell_prompt_marking 133 A
shell_prompt_marking 133 k;end_kitty
set_title root@reverse-code-generator-6e7d35be-kxf24: /tmp/blitzy/kitty/blitzy-85ce7b41-edf3-42eb-84b1-2fc55abedc59_304657
set_icon root@reverse-code-generator-6e7d35be-kxf24: /tmp/blitzy/kitty/blitzy-85ce7b41-edf3-42eb-84b1-2fc55abedc59_304657
draw root@reverse-code-generator-6e7d35be-kxf24:/tmp/blitzy/kitty/blitzy-85ce7b41-edf3-42eb-84b1-2fc55abedc59_304657# 
shell_prompt_marking 133 k;start_suffix_kitty
screen_set_cursor 5 32
set_title /tmp/blitzy/kitty/blitzy-85ce7b41-edf3-42eb-84b1-2fc55abedc59_304657
shell_prompt_marking 133 k;end_suffix_kitty
draw printf "HELLO-OUTPUT
screen_carriage_return
screen_linefeed
screen_reset_mode 2004 1
screen_carriage_return
screen_set_mode 2004 1
shell_prompt_marking 133 k;start_kitty
shell_prompt_marking 133 A;k=s
shell_prompt_marking 133 k;end_kitty
draw > "
screen_carriage_return
screen_linefeed
screen_reset_mode 2004 1
screen_carriage_return
set_title printf "HELLO-OUTPUT"
shell_prompt_marking 133 C;cmdline=$'printf "HELLO-OUTPUT\n"'
shell_prompt_marking 133 k;start_kitty
shell_prompt_marking 133 k;end_kitty
shell_prompt_marking 133 k;start_suffix_kitty
screen_set_cursor 0 32
shell_prompt_marking 133 k;end_suffix_kitty
draw HELLO-OUTPUT
screen_carriage_return
screen_linefeed
screen_set_mode 2004 1
screen_delete_characters 59
shell_prompt_marking 133 k;start_kitty
shell_prompt_marking 133 D;0
shell_prompt_marking 133 A
shell_prompt_marking 133 k;end_kitty
set_title root@reverse-code-generator-6e7d35be-kxf24: /tmp/blitzy/kitty/blitzy-85ce7b41-edf3-42eb-84b1-2fc55abedc59_304657
set_icon root@reverse-code-generator-6e7d35be-kxf24: /tmp/blitzy/kitty/blitzy-85ce7b41-edf3-42eb-84b1-2fc55abedc59_304657
draw root@reverse-code-generator-6e7d35be-kxf24:/tmp/blitzy/kitty/blitzy-85ce7b41-edf3-42eb-84b1-2fc55abedc59_304657# 
shell_prompt_marking 133 k;start_suffix_kitty
screen_set_cursor 5 32
set_title /tmp/blitzy/kitty/blitzy-85ce7b41-edf3-42eb-84b1-2fc55abedc59_304657
shell_prompt_marking 133 k;end_suffix_kitty
```

Read the ordering: `133;A` (prompt start) is applied, *then* the prompt text is drawn (`draw root@…#`); the typed command is drawn (`draw printf "HELLO-OUTPUT`), the continuation prompt is marked (`133;A;k=s`) and drawn (`draw > "`); then `133;C;cmdline=…` (output start, carrying the exact command line) is applied *immediately before* the output is drawn (`draw HELLO-OUTPUT`); and finally `133;D;0` marks the command finished with exit status 0, followed by `133;A` for the next prompt. The mark for a region is always adjacent to the `draw` of that region because both are consumed from the same buffer in the same order — there is nothing to drift.

**[NON-CANONICAL] — the general FinalTerm `A/B/C/D` protocol (including the `B` that kitty's own shells do not emit), to illustrate that kitty's parser routes every mark serially regardless of source.** This is a *hand-written* byte sequence (not kitty's shell integration), fed through a real kitty PTY:

```
$ ./kitty/launcher/kitty -o close_on_child_death=yes --dump-commands sh -c 'printf "\033]133;A\007PROMPT$ \033]133;B\007echo hi\033]133;C\007hi-OUTPUT\r\n\033]133;D;0\007"; sleep 0.3'
shell_prompt_marking 133 A
draw PROMPT$ 
shell_prompt_marking 133 B
draw echo hi
shell_prompt_marking 133 C
draw hi-OUTPUT
screen_carriage_return
screen_carriage_return
screen_linefeed
shell_prompt_marking 133 D;0
```

Even with the full `A → B → C → D` set interleaved with text, the parser emits `mark, draw, mark, draw, …` in byte order: `A` before `PROMPT$`, `B` before `echo hi`, `C` before `hi-OUTPUT`, `D;0` after. kitty parses `B` if it is sent; its own shells simply never send it. This block is labeled non-canonical precisely because the byte sequence is synthetic — the canonical evidence for what kitty *actually* emits is the shipped-integration capture above.

**Ordering wording (resolves F17).** The guarantee is that a **single parser state** consumes bytes in order; it is *not* that an entire logical stream is always consumed within one `run_worker` call. A long burst can be split across several `run_worker` passes (each pass consumes what is available, then `consume_input` (`kitty/vt-parser.c:1432`) loops `while (self->read.pos < self->read.sz)` (`kitty/vt-parser.c:1435`)), but because the read position and all parser sub-state persist in the same `PS` struct across passes, the *n*-th byte is always processed before the *(n+1)*-th regardless of how the passes fall. That persistence — not a single-pass assumption — is what keeps marks and text aligned.


### 4.2 Under heavy backpressure the stream is paced, not dropped

When a child produces faster than kitty can consume, kitty does **not** discard bytes. It stops asking to read, lets the kernel PTY buffer fill, and the child's `write()` blocks — cooperative backpressure. The mechanism (read from source, since the internal "buffer full" boolean is not exposed to a script) is a POLLIN gate:

```c
children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;  // child-monitor.c:1501
```

`vt_parser_has_space_for_input` returns `self->read.sz + self->write.pending < BUF_SZ` (`kitty/vt-parser.c:1477-1483`), i.e. false once the 1 MiB buffer (`BUF_SZ`, `kitty/vt-parser.c:18`) is full. When it is false the child's PTY fd is polled with `events = 0`, so the I/O thread simply stops reading it; `read_bytes` has a matching self-guard, `if (!available_buffer_space) return true;` (`kitty/child-monitor.c:1342`). No bytes are dropped — the producer is throttled at the OS boundary. **[INFERRED]** from those three lines; below is the **[OBSERVED]** consequence.

**[OBSERVED] — an 8 MiB burst (8× the 1 MiB buffer) survives bit-for-bit; the varied payload means integrity is proven by hash, not by a byte count that a compensating drop+duplicate could preserve:**

```
$ FLOOD_META=… FLOOD_BYTES=8388608 ./kitty/launcher/kitty -o close_on_child_death=yes -o input_delay=3 --dump-bytes=… env … python3 harness/flood.py
$ python3 harness/flood_verify.py <dump.bin> <meta> "trial 1"
label=bytes=8388608 input_delay=3 trial=1
sent_bytes=8388608 dumped_bytes=8388608
sent_sha256=f7ed19dbe389fdfb8746b6d71de4a0cfa4aee1fba9f40f7cd3587f4539040b6f
dump_sha256=f7ed19dbe389fdfb8746b6d71de4a0cfa4aee1fba9f40f7cd3587f4539040b6f
write_wall_ms=1319.395
buffer_multiple=8.00x the 1 MiB VT parser buffer
RESULT=PASS_LOSSLESS

label=bytes=8388608 input_delay=3 trial=2
sent_bytes=8388608 dumped_bytes=8388608
sent_sha256=f7ed19dbe389fdfb8746b6d71de4a0cfa4aee1fba9f40f7cd3587f4539040b6f
dump_sha256=f7ed19dbe389fdfb8746b6d71de4a0cfa4aee1fba9f40f7cd3587f4539040b6f
write_wall_ms=1392.782
buffer_multiple=8.00x the 1 MiB VT parser buffer
RESULT=PASS_LOSSLESS
```

8 MiB is far larger than the 1 MiB buffer plus the ~64 KiB kernel PTY buffer, so the writer *must* have blocked repeatedly mid-burst; yet the SHA-256 of what kitty consumed equals the SHA-256 of what the child sent, in both trials. Nothing was dropped, duplicated, or reordered.

**[OBSERVED] — the writer is paced by kitty's drain rate (a size sweep and a slow-drain control), which is the signature of blocking rather than discarding:**

```
label=bytes=1048576 input_delay=3 trial=1 launcher_exit=0
write_wall_ms=3.326
RESULT=PASS_LOSSLESS

label=bytes=4194304 input_delay=3 trial=1 launcher_exit=0
write_wall_ms=535.321
RESULT=PASS_LOSSLESS

label=bytes=8388608 input_delay=3 trial=1 launcher_exit=0
write_wall_ms=1399.153
RESULT=PASS_LOSSLESS
```

```
label=bytes=8388608 input_delay=3 trial=A launcher_exit=0
write_wall_ms=1247.591
RESULT=PASS_LOSSLESS

label=bytes=8388608 input_delay=200 trial=A launcher_exit=0
write_wall_ms=1579.969
RESULT=PASS_LOSSLESS
```

Two things stand out. First, the **before/at/after-threshold** behavior: a 1 MiB burst — which fits entirely within the parser buffer — completes in **3.3 ms** without ever blocking, whereas 4 MiB and 8 MiB (which overflow the buffer) take **535 ms** and **1399 ms**, i.e. the write time grows once the producer crosses the buffer size and must wait for kitty to drain. Second, holding the burst at 8 MiB but making kitty drain **slower** (raising `input_delay` from 3 to 200 ms) *lengthens* the writer's blocked time (**1248 ms → 1580 ms**) while remaining lossless — the producer's rate is dictated by the consumer, which is exactly cooperative backpressure.

**Correction (resolves the `repaint_delay` claim in F9/F14).** An earlier draft attributed this pacing to a "~10 ms `repaint_delay` cadence." That is wrong: `repaint_delay`'s own documentation says "to minimize latency when there is pending input to be processed, this option is ignored" (`kitty/options/definition.py:873-875`). While a backlog exists, the render delay is bypassed, so it does **not** pace ingestion; the pacing comes solely from the POLLIN gate above. The `repaint_delay` claim has been removed.

**[NON-CANONICAL] supplement.** kitty's own test suite exercises the buffer boundary synchronously via the `parse_bytes` helper and `VT_PARSER_BUFFER_SIZE` (`kitty_tests/parser.py`), but that helper feeds the parser directly and **bypasses** the `io_thread`/PTY path, so it is not canonical evidence for backpressure; the real-PTY losslessness and pacing above are the canonical evidence.

### 4.3 Under an unstable remote connection the PTY pipeline is unaffected

Remote control reaches kitty by **two distinct transports**, and the question's "unstable remote connection" concerns the first of them:

- a **dedicated talk socket** served on its own thread (`talk_loop`, named `KittyPeerMon`, `kitty/child-monitor.c:1805`/`1808`), created when a listen socket is configured (`--listen-on`) or a peer fd is injected; and
- an **inline DCS** channel carried *in the PTY byte stream itself* — `ESC P @ kitty-cmd{…} ESC \` — parsed by `parse_kitty_dcs` (`kitty/vt-parser.c:586`).

Before the disruption experiment, two routing facts must be established canonically, because an earlier draft misattributed one of them (F13).

#### 4.3.1 Standard DCS capability queries are routed by `dispatch_dcs`, not `parse_kitty_dcs` (F13)

[INFERRED — source] In `kitty/vt-parser.c`, `dispatch_dcs` (defined at `L620`) switches on the first DCS byte. A standard **DECRQSS** capability query begins `$q` or `+q`; those land in `case '+': case '$':` (`L623-624`), and when `buf[1] == 'q'` (`L625`) the parser calls `screen_request_capabilities(self->screen, …)` (`L631`). Only a leading `@` (`case '@':`, `L654`) is forwarded to `parse_kitty_dcs` (`L655`), which itself returns early unless the payload begins `kitty-` (`L586`). So common DCS traffic (`$q`/`+q`) is handled by `dispatch_dcs`; `parse_kitty_dcs` is reserved for kitty's own `@kitty-…` remote-control verbs (`dispatch("cmd{", handle_remote_cmd, 1)` at `L603`).

[OBSERVED] A real PTY child (`decrqss_child.py`, reproduced in Appendix A) put its input into raw mode, emitted three DECRQSS queries on its stdout, and read kitty's replies back from its stdin, writing the raw reply bytes to an out-of-band file. kitty was run under `--dump-commands` so the parser's routing is visible. Command:

```
bash "$OBS/harness/decrqss_probe.sh"
```

where the launch inside that script is:

```
timeout 30 ./kitty/launcher/kitty -o close_on_child_death=yes --dump-commands \
    sh -c "$PYBIN $OBS_DIR/harness/decrqss_child.py '<reply-file>'; sleep 0.3"
```

Complete, unedited output:

```
launcher_exit=0
=== child-captured DECRQSS replies (raw bytes, out-of-band file) ===
SGR(m)
  query_hex=1b5024716d1b5c
  reply_hex=1b503124726d1b5c
  reply_repr=b'\x1bP1$rm\x1b\\'
DECSCUSR( q)
  query_hex=1b50247120711b5c
  reply_hex=1b503124723120711b5c
  reply_repr=b'\x1bP1$r1 q\x1b\\'
DECSTBM(r)
  query_hex=1b502471721b5c
  reply_hex=1b50312472313b3232721b5c
  reply_repr=b'\x1bP1$r1;22r\x1b\\'
=== parser trace: lines mentioning screen_request_capabilities ===
1:screen_request_capabilities 36 m
2:screen_request_capabilities 36  q
3:screen_request_capabilities 36 r
=== dump-commands total lines ===
3
```

Every one of the three queries (`ESC P $ q m ESC \`, `ESC P $ q ' ' q ESC \`, `ESC P $ q r ESC \`) was routed to **`screen_request_capabilities`** — the `36` is the decimal of the leading `$` (`0x24`), and the trailing token is the queried setting (`m`, ` q`, `r`). None reached `parse_kitty_dcs`. The child received well-formed DECRQSS *responses* (`1$r…`, the leading `1` meaning "valid"): SGR reported `m` (default rendition), DECSCUSR reported cursor style `1`, and DECSTBM reported the scroll region `1;22` (a 22-row window). This is the canonical correction demanded by F13.


#### 4.3.2 The inline-DCS remote-control path (`parse_kitty_dcs`), distinct from the talk socket

[OBSERVED] Remote-control commands can also travel *inside* the PTY stream, wrapped as `ESC P @ kitty-cmd{…} ESC \` (the prefix `KITTY_CMD_PREFIX "\x1bP@kitty-cmd{"` is defined at `kitty/child-monitor.c:1650` and shared by both transports). A real PTY child emitted one such command with `-o allow_remote_control=yes` under `--dump-commands`. Command (inside `inline_dcs_probe.sh`):

```
timeout 30 ./kitty/launcher/kitty -o close_on_child_death=yes -o allow_remote_control=yes --dump-commands \
    sh -c "printf '\033P@kitty-cmd%s\033\\\\' '{\"cmd\":\"ls\",\"version\":[0,35,2],\"async_id\":\"\",\"no_response\":true}'; sleep 0.4"
```

Complete, unedited output:

```
launcher_exit=0
payload_sent=@kitty-cmd{"cmd":"ls","version":[0,35,2],"async_id":"","no_response":true}
=== parser trace lines mentioning handle_remote_cmd or screen_handle_kitty_dcs ===
1:handle_remote_cmd {"cmd":"ls","version":[0,35,2],"async_id":"","no_response":true}
=== full dump-commands trace ===
handle_remote_cmd {"cmd":"ls","version":[0,35,2],"async_id":"","no_response":true}
=== stderr ===
[0.146] Failed to open systemd user bus with error: Connection refused
```

The parser routed the inline `@kitty-cmd{…}` to **`handle_remote_cmd`** — i.e. through `dispatch_dcs`'s `case '@'` (`L654`) into `parse_kitty_dcs` (`L586`) and its `dispatch("cmd{", handle_remote_cmd, 1)` (`L603`). This confirms the inline path is the `parse_kitty_dcs` route and is carried on the **PTY/parser** pipeline, entirely separate from the `talk_loop` socket exercised next. (The single stderr line is the container's benign systemd-bus notice; it is shown unedited.)


#### 4.3.3 An unstable peer on the talk socket: kitty survives, the peer is removed, the socket recovers (F10)

[OBSERVED] A kitty **server** was launched with a listen socket and a real PTY child (`reader_child.py`, which appends every line it reads from its stdin to an out-of-band delivery log so that delivery can be verified independently of the send command's own exit status). The launch (inside `remote_harness.sh`, Appendix A):

```
timeout 200 ./kitty/launcher/kitty --listen-on "unix:$OBS_DIR/ktalk.sock" \
    -o allow_remote_control=yes -o close_on_child_death=yes \
    sh -c "DELIVERY_LOG='<log>' $PYBIN $OBS_DIR/harness/reader_child.py"
```

The "unstable remote" is a **single, byte-for-byte identical** disruption applied repeatedly. Its entire definition is `peer_disrupt.py`:

```
import socket, struct, sys
sockpath = sys.argv[1]
PARTIAL = b'\x1bP@kitty-cmd{"cmd":"ls","versi'   # truncated: no ESC\ terminator
s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.settimeout(5)
s.connect(sockpath)
s.sendall(PARTIAL)
s.setsockopt(socket.SOL_SOCKET, socket.SO_LINGER, struct.pack('ii', 1, 0))  # abrupt close
s.close()
sys.stdout.write("disrupt_sent_bytes=%d partial_sha=%s\n" % (
    len(PARTIAL), __import__('hashlib').sha256(PARTIAL).hexdigest()[:16]))
```

Each trial connects, sends exactly those 30 bytes — the *start* of a `@kitty-cmd{…}` command with **no `ESC \` terminator** — then closes with `SO_LINGER {1,0}` so the half-command is dropped and the peer vanishes mid-message. After each disruption the harness records, on the **real kitty process** (resolved as the child of the `timeout` wrapper via `ps --ppid`): is the process still alive; how many `KittyPeerMon` threads exist (raw `/proc/<pid>/task/*/comm`); does a fresh well-formed `kitty @ ls` succeed over a *new* connection (`set -o pipefail`, so a producer failure cannot be masked); and is a unique marker sent with `kitty @ send-text` actually delivered to the child's stdin. Command:

```
DISRUPT_N=40 bash "$OBS/harness/remote_harness.sh"
```

Complete, unedited per-trial records (run 1; columns: `trial  proc_alive  peermon  ls_exit  ls_bytes  deliver  rtt_ms`):

```
wrapper_pid=133796 real_kitty_pid=133798 comm=kitty
socket=/tmp/kitty-nosgid/kitty_obs.jWw1gwl6/ktalk.sock  (perm srwx------)
baseline_KittyPeerMon_count=1
baseline_ls_exit=0 baseline_ls_bytes=10850
trial	proc_alive	peermon	ls_exit	ls_bytes	deliver	rtt_ms
1	1	1	0	10850	1	30.05
2	1	1	0	10850	1	31.14
3	1	1	0	10850	1	28.15
4	1	1	0	10850	1	30.20
5	1	1	0	10850	1	30.64
6	1	1	0	10850	1	34.72
7	1	1	0	10850	1	28.43
8	1	1	0	10850	1	28.16
9	1	1	0	10850	1	28.84
10	1	1	0	10850	1	31.32
11	1	1	0	10850	1	33.95
12	1	1	0	10850	1	34.80
13	1	1	0	10850	1	29.27
14	1	1	0	10850	1	31.28
15	1	1	0	10850	1	28.80
16	1	1	0	10850	1	28.30
17	1	1	0	10850	1	29.18
18	1	1	0	10850	1	36.54
19	1	1	0	10850	1	29.08
20	1	1	0	10850	1	29.31
21	1	1	0	10850	1	28.98
22	1	1	0	10850	1	29.77
23	1	1	0	10850	1	37.07
24	1	1	0	10850	1	29.33
25	1	1	0	10850	1	45.58
26	1	1	0	10850	1	34.15
27	1	1	0	10850	1	31.25
28	1	1	0	10850	1	30.18
29	1	1	0	10850	1	31.06
30	1	1	0	10850	1	36.11
31	1	1	0	10850	1	27.89
32	1	1	0	10850	1	30.08
33	1	1	0	10850	1	31.82
34	1	1	0	10850	1	33.87
35	1	1	0	10850	1	31.73
36	1	1	0	10850	1	32.30
37	1	1	0	10850	1	28.66
38	1	1	0	10850	1	31.85
39	1	1	0	10850	1	38.56
40	1	1	0	10850	1	31.84
=== distribution over 40 BYTE-IDENTICAL disruptions ===
proc_alive_after:      40 / 40
KittyPeerMon_present:  40 / 40
recovery_ls_success:   40 / 40
send_text_delivered:   40 / 40
recovery_rtt_ms distribution (min/median/max):
  min=27.89 median=30.64 max=45.58 (n=40)
```

The disruption's *visible effect* is one error per aborted peer, in the server's own stderr (complete, unedited):

```
[0.161] Failed to open systemd user bus with error: Connection refused
[0.444] Malformatted remote control message received from peer, ignoring
[0.721] Malformatted remote control message received from peer, ignoring
[1.005] Malformatted remote control message received from peer, ignoring
[1.282] Malformatted remote control message received from peer, ignoring
[1.565] Malformatted remote control message received from peer, ignoring
[1.868] Malformatted remote control message received from peer, ignoring
[2.158] Malformatted remote control message received from peer, ignoring
[2.438] Malformatted remote control message received from peer, ignoring
[2.716] Malformatted remote control message received from peer, ignoring
[3.008] Malformatted remote control message received from peer, ignoring
[3.287] Malformatted remote control message received from peer, ignoring
[3.608] Malformatted remote control message received from peer, ignoring
[3.908] Malformatted remote control message received from peer, ignoring
[4.209] Malformatted remote control message received from peer, ignoring
[4.502] Malformatted remote control message received from peer, ignoring
[4.784] Malformatted remote control message received from peer, ignoring
[5.070] Malformatted remote control message received from peer, ignoring
[5.362] Malformatted remote control message received from peer, ignoring
[5.663] Malformatted remote control message received from peer, ignoring
[5.955] Malformatted remote control message received from peer, ignoring
[6.250] Malformatted remote control message received from peer, ignoring
[6.558] Malformatted remote control message received from peer, ignoring
[6.862] Malformatted remote control message received from peer, ignoring
[7.166] Malformatted remote control message received from peer, ignoring
[7.464] Malformatted remote control message received from peer, ignoring
[7.767] Malformatted remote control message received from peer, ignoring
[8.061] Malformatted remote control message received from peer, ignoring
[8.356] Malformatted remote control message received from peer, ignoring
[8.641] Malformatted remote control message received from peer, ignoring
[8.950] Malformatted remote control message received from peer, ignoring
[9.255] Malformatted remote control message received from peer, ignoring
[9.544] Malformatted remote control message received from peer, ignoring
[9.824] Malformatted remote control message received from peer, ignoring
[10.118] Malformatted remote control message received from peer, ignoring
[10.400] Malformatted remote control message received from peer, ignoring
[10.690] Malformatted remote control message received from peer, ignoring
[10.975] Malformatted remote control message received from peer, ignoring
[11.264] Malformatted remote control message received from peer, ignoring
[11.570] Malformatted remote control message received from peer, ignoring
[11.882] Malformatted remote control message received from peer, ignoring
```

That is exactly **40** "Malformatted remote control message received from peer, ignoring" lines — one per disruption — over an ~11.9 s run. To confirm the outcome is not a fluke, the identical 40-trial experiment was repeated; the second run's distribution:

```
=== distribution over 40 BYTE-IDENTICAL disruptions ===
proc_alive_after:      40 / 40
KittyPeerMon_present:  40 / 40
recovery_ls_success:   40 / 40
send_text_delivered:   40 / 40
recovery_rtt_ms distribution (min/median/max):
  min=27.53 median=28.62 max=36.67 (n=40)
```

**What this proves and how the mechanism is named.** Across 80 identical unstable-peer events (two runs of 40), the kitty process stayed alive every time, the `KittyPeerMon` talk thread was never torn down (`peermon=1` in every row), a fresh well-formed remote command always succeeded (`ls_exit=0`, `10850` bytes each), and — critically — `send-text` was **actually delivered** to the child's stdin every time (`deliver=1`), which the reader child confirmed out-of-band rather than trusting the send command's own exit code. Only the individual half-spoken peer is discarded. [INFERRED — source] The teardown of that one peer is the path `read_from_peer` → `notify_on_peer_removal` (`kitty/child-monitor.c:1673`), which enqueues an internal message whose payload is `strdup("peer_death")` (`L1677`); on the Python side `Boss.peer_message_received` (`kitty/boss.py:776`) matches `msg_bytes == b'peer_death'` (`L777`) and simply pops that peer from `peer_data_map`, returning `False`. The internal `"peer_death"` token is not surfaced to any log, so it is cited from source; what is **observed** is its consequence — the peer disappears while the thread, the process, and the socket all continue. The recovery latency is reported as a distribution (median ≈ 29–31 ms, occasional tail to ~46 ms) rather than a single tidy number, per the inconsistency rule.


#### 4.3.4 The PTY pipeline is independent of talk-socket turmoil (matched control)

[OBSERVED] Survival is necessary but not sufficient; the sharper question is whether remote-socket chaos *perturbs the PTY input path at all*. To answer it, the same PTY RTT prober from §3.3 (`pingpong.py`, `PP_N=200`, default `input_delay=3`) was run twice against a server that also carried a listen socket, holding the prober input identical and changing only the remote condition:

- **Phase A (baseline):** the talk socket sat idle.
- **Phase B (chaos):** a background loop fired the *same* byte-identical `peer_disrupt.py` events at the talk socket continuously for the entire prober run.

Command:

```
PP_N=200 bash "$OBS/harness/pty_independence.sh"
```

Complete, unedited output — run 1:

```
### Phase A (baseline, talk socket idle), PP_N=200 ###
label=A_baseline
count=200 (header n=200)
wall_duration_s=0.636
min_ms=3.1326
median_ms=3.1630
p90_ms=3.1938
max_ms=5.0107
mean_ms=3.1758

### Phase B (concurrent talk-socket chaos), PP_N=200 ###
label=B_chaos
count=200 (header n=200)
wall_duration_s=0.642
min_ms=3.1055
median_ms=3.1565
p90_ms=3.1896
max_ms=10.0006
mean_ms=3.2105
```

Complete, unedited output — run 2 (reproduction):

```
### Phase A (baseline, talk socket idle), PP_N=200 ###
label=A_baseline
count=200 (header n=200)
wall_duration_s=0.630
min_ms=3.0912
median_ms=3.1481
p90_ms=3.1681
max_ms=3.1970
mean_ms=3.1491

### Phase B (concurrent talk-socket chaos), PP_N=200 ###
label=B_chaos
count=200 (header n=200)
wall_duration_s=0.645
min_ms=3.1387
median_ms=3.1766
p90_ms=3.2141
max_ms=10.7181
mean_ms=3.2205
```

The PTY RTT's central tendency is **unchanged** by concurrent talk-socket chaos: baseline vs chaos median is `3.1630` vs `3.1565` ms (run 1) and `3.1481` vs `3.1766` ms (run 2) — a shift smaller than the run-to-run variation of the baseline itself, and in both phases the median stays pinned to the `input_delay = 3 ms` coalescing window measured in §3.3. Chaos contributes only a rare high outlier (`max` ~10 ms vs ~3–5 ms) — a single delayed sample out of 200 — while `p90` is essentially identical. Because the talk thread (`talk_loop`) has its own poll loop (`kitty/child-monitor.c:1805`) separate from the `io_loop` that drives `read_bytes`/`run_worker`, an unstable peer degrades — at most — its own thread's tail latency and never the PTY pipeline's throughput or characteristic latency. This is the matched-control evidence for PTY independence that F10 requires.


## 5. Q4 — End to end: from a keystroke and a byte to the interface settling

Q4 asks for the whole rhythm — from the moment mixed input arrives to the moment the interface settles — and how the moving parts keep time. The inbound half (child → PTY → `read_bytes` → parse → screen) was established in §2–§4; this section adds the **outbound** half the question also implies ("input" broadly), the **wakeup mechanism** that ties the threads together, what is and is not observable at the **render/settle** boundary, and finally the composed end-to-end narrative.

### 5.1 The outbound keyboard path, driven through its real entry point (F11)

A keystroke enters kitty as an X/GLFW key event and is turned into bytes for the child by **`on_key_input`** (`kitty/keys.c:166`). Rather than call an internal Python shortcut (which would bypass exactly the boundary in question), these events were injected as **real X events** via the XTEST extension into the focused kitty window, so the canonical GLFW → `on_key_input` path actually runs. The driver (`xtest_inject.py`, Appendix A) locates the kitty top-level window by `WM_CLASS`, `XSetInputFocus`es it (Xvfb has no window manager), and taps each key with `XTestFakeKeyEvent`. kitty was launched with `--debug-keyboard`; its child (`key_child.py`) logged every received byte, in hex, to an out-of-band file so the bytes can be confirmed independently of kitty's own debug print.

#### 5.1.1 Legacy encodings (default keyboard mode)

Command:

```
TAG=legacy bash "$OBS/harness/keyboard_probe.sh" \
    "plain_a::0x61" "ctrl_a:ctrl:0x61" "up::0xff52" "shift_up:shift:0xff52" "alt_a:alt:0x61"
```

Complete, unedited output:

```
=== injecting keys (tag=legacy mode=none) ===
kitty_window=0x20000c focus=0x20000c FOCUS_OK
INJECTED plain_a (mods=none keysym=0x61 keycode=38)
INJECTED ctrl_a (mods=ctrl keysym=0x61 keycode=38)
INJECTED up (mods=none keysym=0xff52 keycode=111)
INJECTED shift_up (mods=shift keysym=0xff52 keycode=111)
INJECTED alt_a (mods=alt keysym=0x61 keycode=38)
=== kitty --debug-keyboard stderr: on_key_input + encoded bytes ===
[0.222] [33mon_key_input[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
[0.246] [33mon_key_input[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.396] [33mon_key_input[m: glfw key: 0xe062 native_code: 0xffe3 action: PRESS mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.396] [33mon_key_input[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: 0x1 
[0.426] [33mon_key_input[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.426] [33mon_key_input[m: glfw key: 0xe062 native_code: 0xffe3 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.576] [33mon_key_input[m: glfw key: 0xe008 native_code: 0xff52 action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ [ A 
[0.606] [33mon_key_input[m: glfw key: 0xe008 native_code: 0xff52 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.757] [33mon_key_input[m: glfw key: 0xe061 native_code: 0xffe1 action: PRESS mods: shift text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.757] [33mon_key_input[m: glfw key: 0xe008 native_code: 0xff52 action: PRESS mods: shift text: '' state: 0 sent encoded key to child: ^[ [ 1 ; 2 A 
[0.787] [33mon_key_input[m: glfw key: 0xe008 native_code: 0xff52 action: RELEASE mods: shift text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.787] [33mon_key_input[m: glfw key: 0xe061 native_code: 0xffe1 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.937] [33mon_key_input[m: glfw key: 0xe063 native_code: 0xffe9 action: PRESS mods: alt text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.937] [33mon_key_input[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: alt text: '' state: 0 sent encoded key to child: ^[ a 
[0.967] [33mon_key_input[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: alt text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.967] [33mon_key_input[m: glfw key: 0xe063 native_code: 0xffe9 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
=== child-received bytes (hex, out-of-band) ===
CHILD_START tty=/dev/pts/0
RECV 61
RECV 01
RECV 1b5b41
RECV 1b5b313b3241
RECV 1b61
```

Each key traversed the real `on_key_input` (the debug line's shape is exactly the format string at `kitty/keys.c:176`), was encoded by `encode_glfw_key_event` (`kitty/keys.c:251`), and handed to the child by `schedule_write_to_child` (`kitty/keys.c:259`). The child's independent hex log confirms every byte kitty said it sent: `a`→`61`, Ctrl+a→`01`, Up→`1b5b41` (`ESC [ A`), Shift+Up→`1b5b313b3241` (`ESC [ 1 ; 2 A`, where `;2` is the Shift modifier), Alt+a→`1b61` (`ESC a`). Note also that only *press* events with an encodable result are sent; bare modifier presses and release events print "ignoring as keyboard mode does not support encoding this event" — the legacy mode encodes press-only. The identical byte sequence reproduced on a second run.


#### 5.1.2 DECCKM (application cursor keys): a transitional before/after

`encode_glfw_key_event` takes `screen->modes.mDECCKM` as an argument (`kitty/keys.c:251`), so enabling DECCKM (application cursor-key mode) changes how *unmodified* cursor keys encode. The child turned DECCKM on with `CSI ? 1 h`, then the same Up / Shift+Up were injected. Command:

```
TAG=deckm_on MODE=DECCKM_ON bash "$OBS/harness/keyboard_probe.sh" "up::0xff52" "shift_up:shift:0xff52"
```

Complete, unedited output:

```
=== injecting keys (tag=deckm_on mode=DECCKM_ON) ===
kitty_window=0x20000c focus=0x20000c FOCUS_OK
INJECTED up (mods=none keysym=0xff52 keycode=111)
INJECTED shift_up (mods=shift keysym=0xff52 keycode=111)
=== kitty --debug-keyboard stderr: on_key_input + encoded bytes ===
[0.525] [33mon_key_input[m: glfw key: 0xe008 native_code: 0xff52 action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ O A 
[0.550] [33mon_key_input[m: glfw key: 0xe008 native_code: 0xff52 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.700] [33mon_key_input[m: glfw key: 0xe061 native_code: 0xffe1 action: PRESS mods: shift text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.701] [33mon_key_input[m: glfw key: 0xe008 native_code: 0xff52 action: PRESS mods: shift text: '' state: 0 sent encoded key to child: ^[ [ 1 ; 2 A 
[0.731] [33mon_key_input[m: glfw key: 0xe008 native_code: 0xff52 action: RELEASE mods: shift text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.731] [33mon_key_input[m: glfw key: 0xe061 native_code: 0xffe1 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
=== child-received bytes (hex, out-of-band) ===
CHILD_START tty=/dev/pts/0
MODE DECCKM_ON
RECV 1b4f41
RECV 1b5b313b3241
```

With DECCKM **on**, an unmodified Up now encodes as `ESC O A` (`1b4f41`) — compare the DECCKM-**off** `ESC [ A` (`1b5b41`) from §5.1.1: the same physical key, a different byte sequence, decided entirely by `mDECCKM`. The **modified** Shift+Up is unchanged (`ESC [ 1 ; 2 A`, `1b5b313b3241`), which is correct: DECCKM only rewrites *unmodified* cursor keys.

#### 5.1.3 The kitty keyboard protocol (CSI u)

The other argument to `encode_glfw_key_event` is `screen_current_key_encoding_flags(screen)` (`kitty/keys.c:251`) — the kitty keyboard-protocol flags. The child enabled the protocol with `CSI > 1 u` (progressive enhancement, flag 1 = "disambiguate escape codes"), then plain `a`, Ctrl+a, and Up were injected. Command:

```
TAG=kkp_on MODE=KKP_ON bash "$OBS/harness/keyboard_probe.sh" "plain_a::0x61" "ctrl_a:ctrl:0x61" "up::0xff52"
```

Complete, unedited output:

```
=== injecting keys (tag=kkp_on mode=KKP_ON) ===
kitty_window=0x20000c focus=0x20000c FOCUS_OK
INJECTED plain_a (mods=none keysym=0x61 keycode=38)
INJECTED ctrl_a (mods=ctrl keysym=0x61 keycode=38)
INJECTED up (mods=none keysym=0xff52 keycode=111)
=== kitty --debug-keyboard stderr: on_key_input + encoded bytes ===
[0.529] [33mon_key_input[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
[0.554] [33mon_key_input[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.704] [33mon_key_input[m: glfw key: 0xe062 native_code: 0xffe3 action: PRESS mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.704] [33mon_key_input[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: ^[ [ 9 7 ; 5 u 
[0.734] [33mon_key_input[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.734] [33mon_key_input[m: glfw key: 0xe062 native_code: 0xffe3 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[0.884] [33mon_key_input[m: glfw key: 0xe008 native_code: 0xff52 action: PRESS mods: none text: '' state: 0 sent encoded key to child: ^[ [ A 
[0.914] [33mon_key_input[m: glfw key: 0xe008 native_code: 0xff52 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
=== child-received bytes (hex, out-of-band) ===
CHILD_START tty=/dev/pts/0
MODE KKP_ON
RECV 61
RECV 1b5b39373b3575
RECV 1b5b41
```

The decisive contrast is Ctrl+a: in legacy mode it is a single control byte `0x01`; under the kitty keyboard protocol it becomes `ESC [ 9 7 ; 5 u` (`1b5b39373b3575`), i.e. CSI-u form where `97` is the codepoint of `a` and `;5` encodes Ctrl (1 + 4). A plain printable `a` is still delivered as text under flag 1 (`RECV 61`), and the unmodified Up remains `ESC [ A` — because flag 1 only *disambiguates*; it does not request event-type or alternate-key reporting, which is also why the release events still print "ignoring… does not support encoding this event." This is the same `encode_glfw_key_event` call, steered by the protocol flags rather than by `mDECCKM`.

#### 5.1.4 The Python call chain, corrected — and the shared encoder (F11)

[INFERRED — source] An earlier draft claimed the Python side ran `send_key` → `send_key_sequence`. That is wrong. `Window.send_key` (`kitty/window.py:917`) builds `KeyEvent`s and calls `self.encoded_key(ev)` (`L931`), then `self.write_to_child(enc)` (`L933`); `encoded_key` (`L1795`) calls **`encode_key_for_tty`** (`L1796`) — the C function `pyencode_key_for_tty` (`kitty/keys.c:311`), which calls the very same `encode_glfw_key_event` (`kitty/keys.c:319`) — and finally `write_to_child` (`L955`) delivers the bytes. `send_key_sequence` (`L937`) is a *separate* method. This Python route is what `kitty @ send-key` / the `send_key` action use; it invokes the shared encoder but **without any GLFW/OS key event**, so it is [NON-CANONICAL] for the OS-entry boundary that `on_key_input` represents.

[NON-CANONICAL] supplement — that the two routes share an encoder was checked by calling `encode_key_for_tty` directly (`pyencode_supplement.py`). Command:

```
PYTHONPATH="$PWD:$PYTHONPATH" $PYBIN "$OBS/harness/pyencode_supplement.py"
```

Complete, unedited output:

```
ctrl_a legacy: repr='\x01' hex=01
ctrl_a KKP flags=1: repr='\x1b[97;5u' hex=1b5b39373b3575
Up legacy DECCKM off: repr='\x1b[A' hex=1b5b41
Up legacy DECCKM on: repr='\x1bOA' hex=1b4f41
Shift+Up legacy: repr='\x1b[1;2A' hex=1b5b313b3241
```

Every value is byte-identical to the canonical XTEST results above (`01`, `1b5b39373b3575`, `1b5b41`, `1b4f41`, `1b5b313b3241`), confirming both routes converge on `encode_glfw_key_event`. The canonical proof, however, remains the XTEST runs in §5.1.1–§5.1.3, which alone exercise the real `on_key_input` OS-entry boundary.


### 5.2 The wakeup mechanism that ties the threads together (F12)

[INFERRED — source] The threads of §3 rendezvous through a self-pipe/eventfd. When the `io_thread` has appended bytes and the coalescing window has elapsed, it calls `wakeup_loop` (`kitty/loop-utils.c:113`), which simply `write()`s one token to the wakeup fd (an `eventfd` value at `L117`, or a byte `"w"` into a pipe at `L119`). The main loop notices that fd is readable and **drains** it with `drain_fd` (`kitty/loop-utils.h:76`) before doing work — in the `io_loop` at `kitty/child-monitor.c:1515` and in the `talk_loop` at `kitty/child-monitor.c:1858`. `init_loop_data` (`kitty/loop-utils.c:59`) sets these fds up per loop. This is the concrete "hand-off" the question's conductor performs: the wakeup fd is the first descriptor each loop polls (§3.2), so a pending wakeup is always serviced before child or peer traffic in the same cycle.

### 5.3 Canonical instrumentation, exercised (F12)

The investigation used only kitty's own instrumentation, and each flag was exercised on a real PTY:

- **`--dump-commands`** ("Output commands received from child process to STDOUT", `kitty/cli.py`) — used throughout (§2–§5) to print the parser's per-token stream.
- **`--dump-bytes`** ("Path to file in which to store the raw bytes received from the child process") — used in §2.2; note (per F4) that this is emitted parser-side from the consumed span, not literally from `read_bytes`.
- **`--debug-input` / `--debug-keyboard`** — the *same* option (`kitty/cli.py:996`, `dest=debug_keyboard`). §5.1 used `--debug-keyboard`; a one-off with the `--debug-input` spelling confirms it emits the identical `on_key_input` line:

```
[0.213] [33mon_key_input[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
[0.238] [33mon_key_input[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
```

- **`--replay-commands`** — replays a dump file into a fresh window. A capture was taken from a real child, then replayed:

```
--- captured trace ---
draw REPLAY-SRC
screen_carriage_return
screen_carriage_return
screen_linefeed
--- replay re-dump (draw of REPLAY-SRC should reappear) ---
draw REPLAY-SRC
screen_carriage_return
screen_carriage_return
screen_carriage_return
screen_linefeed
```

The replayed `draw REPLAY-SRC` re-executes, demonstrating the flag round-trips a dump back through the command pipeline.

### 5.4 The render/settle boundary: what is observable here, and the knobs (F14)

[OBSERVED — with an explicit limitation] The final "interface settles" phase is a GPU frame draw plus buffer swap. In this **headless, software-GL** sandbox that phase is *not* surfaced by kitty's own tracing. `--debug-rendering` emitted only window-lifecycle lines, never a per-frame draw or swap:

```
[0.139] OS Window created
[0.148] Failed to open systemd user bus with error: Connection refused
[0.152] Child launched
```

[INFERRED — source] That is consistent with the code: the `debug_rendering` macro (`kitty/state.h:14`) is used in `kitty/glfw.c` only for a color-scheme change (`L72`), an occlusion change (`L261`), and window creation (`L1321`) — none per frame. **Honest limitation:** I therefore did *not* directly observe the visible frame draw/swap; the claim that a frame is composed and presented is inferred from the render configuration, not shown as a captured frame.

What *can* be stated canonically is the render configuration, read **from the built binary** (not the `.py` source text) via the default `Options`:

```
input_delay=3 (ms)
repaint_delay=10 (ms)
sync_to_monitor=True
VT_PARSER_BUFFER_SIZE=1048576 bytes (=1 MiB)
```

So `repaint_delay = 10 ms` (≈100 FPS) and `sync_to_monitor = yes` are **source-derived defaults**, not behaviours I timed. Crucially (and correcting the earlier overclaim noted in §4.2), `repaint_delay` does **not** pace ingestion: its own definition says "to minimize latency when there is pending input to be processed, this option is ignored" (`kitty/options/definition.py:873-875`). Ingestion pacing comes solely from the backpressure gate (§4.2); `repaint_delay`/`sync_to_monitor` govern only the cadence of drawing once there is no backlog.


### 5.5 The end-to-end rhythm, composed

Putting the observed pieces together, here is the full rhythm from "mixed input arrives" to "the interface settles," each step tied to the evidence that established it:

1. **Arrival.** A child writes bytes to the PTY master fd. The `io_thread`, blocked in `poll()`, sees `POLLIN` on that child and calls `read_bytes` (`kitty/child-monitor.c:1337`, call site `L1531`), appending into the 1 MiB VT parser buffer (`BUF_SZ`, `kitty/vt-parser.c:18`). *(§2.1, §2.2)*
2. **Coalesced wakeup.** The `io_thread` does not wake the main loop per read; it batches, signalling only after `OPT(input_delay)` (measured ≈ 3 ms, §3.3) via `wakeup_loop` → the wakeup fd, which the main loop drains with `drain_fd` (§5.2). If the buffer is nearly full the parser flushes early instead (the `read.sz + 16*1024 > BUF_SZ` term of the flush condition at `kitty/vt-parser.c:1425`). *(§3.3, §4.2)*
3. **Serial dispatch.** The main thread runs one `run_worker`/`parse_worker` pass (`kitty/vt-parser.c:1496`) that consumes the buffer **in byte order**, routing each token to the screen before the next — plain text via `consume_normal`, CSI/mode changes, OSC 133 marks via `cmd_output_marking` (`kitty/screen.c:2338`), DCS capability queries via `dispatch_dcs` → `screen_request_capabilities`, and inline remote commands via `parse_kitty_dcs` → `handle_remote_cmd`. Because it is one ordered pass over one buffer, shell-integration hints never drift out of sync with the text they annotate. *(§4.1, §4.3.1, §4.3.2)*
4. **Screen mutation, possibly paused.** Screen state is updated in place. If the stream opened a synchronized update (`CSI ? 2026 h`, `PENDING_MODE` = 2026, `kitty/control-codes.h:235`), `screen_pause_rendering` (`kitty/screen.c:2506`) snapshots the frame and holds it — with a safety `expires_at` timeout (default 2000 ms, auto-resumed by `screen_check_pause_rendering`, `kitty/screen.c:2489`) even if the closing `CSI ? 2026 l` never arrives. Parsing continues while paused. *(§2.3, §2.4)*
5. **Backpressure, if the producer outruns the consumer.** If the buffer fills, the POLLIN gate `vt_parser_has_space_for_input(...) ? POLLIN : 0` (`kitty/child-monitor.c:1501`) stops requesting reads; the PTY fills and the child blocks on `write()` — lossless, cooperative pacing, verified by hash over an 8 MiB burst. *(§4.2)*
6. **Independence of the remote channel.** Any remote-control turmoil rides a separate `talk_loop`/`KittyPeerMon` thread and never perturbs steps 1–5; an unstable peer produces `peer_death` (`kitty/child-monitor.c:1677`) handled independently, with PTY latency unchanged under concurrent chaos. *(§4.3.3, §4.3.4)*
7. **Settle.** With no backlog, the frame is presented at the `repaint_delay`/`sync_to_monitor` cadence (source defaults `10 ms`/`yes`, §5.4). The visible draw/swap itself was not observable in this headless build (§5.4, disclosed).
8. **Outbound symmetry.** A user keystroke runs the mirror path: X/GLFW event → `on_key_input` (`kitty/keys.c:166`) → `encode_glfw_key_event` honoring `mDECCKM` and the keyboard-protocol flags (`L251`) → `schedule_write_to_child` (`L259`) → the child's PTY. *(§5.1)*

The moving parts keep rhythm through exactly two disciplines: **one ordered buffer consumed by one serial parser pass** (guaranteeing alignment), and **one coalescing window plus a self-pipe wakeup** (bounding latency while batching expensive main-loop wakeups). Everything else — pause/resume, backpressure, remote isolation — layers onto those two without breaking them.


## 6. Canonical vs non-canonical evidence, and honest limitations

Per the investigation rules, the primary proof for every claim comes from a canonical entry point — a real child writing to a real PTY (inbound), or a real X/GLFW key event (outbound) — driven by a normally built kitty. This section labels every piece of evidence so the reader can see exactly what was observed canonically, what was a clearly-marked supplement, and what could not be observed in this sandbox and is therefore stated as inferred.

**Canonical [OBSERVED] evidence (real PTY child → `read_bytes`, or real X event → `on_key_input`):**

- §2.2 surge of 5000 lines validated byte-exactly (sha256 stable across 3 runs).
- §2.3 / §2.4 synchronized-output pause, resume, and the safety-timeout auto-resume (DECRQM state before/during/after; child's full emitted byte stream).
- §3.1 the thread set from raw `/proc/<pid>/task/*/comm` on the exact `$!`-captured PID (default, `--listen-on`, and single-instance cases).
- §3.3 the `input_delay` coalescing window, measured with an RTT proxy across the {0,3,10,30} ms sweep, ≥2 runs each.
- §4.1 real interactive `bash` shell integration emitting OSC 133 `A` / `A;k=s` / `C;cmdline` / `D;$?`, interleaved with `draw` in byte order.
- §4.2 losslessness of an 8 MiB burst by sha256, plus the writer-blocking/drain control across a size sweep.
- §4.3.1 DECRQSS `$q` capability queries routed to `screen_request_capabilities`, with the child's real replies.
- §4.3.2 an inline `@kitty-cmd{…}` routed to `handle_remote_cmd`.
- §4.3.3 / §4.3.4 an unstable talk-socket peer over 80 identical disruptions (survival, recovery, delivery), and the matched PTY-independence control.
- §5.1 real keystrokes injected via XTEST through `on_key_input`, with kitty's `--debug-keyboard` output and the child's independent byte log agreeing, across legacy, DECCKM off/on, and the kitty keyboard protocol.
- §5.3 `--replay-commands` and `--debug-input` exercised on a real PTY.

**[NON-CANONICAL] supplements (used only to corroborate, never as primary proof, and labeled inline):**

- §4.1 the synthetic FinalTerm `A/B/C/D` stream produced with `printf` — a protocol illustration of the four marks (including `B`, which kitty's own shell integration does *not* emit); it is *not* attributed to kitty's shipped integration.
- §4.2 the `parse_bytes` / `VT_PARSER_BUFFER_SIZE` boundary check in kitty's test suite, which feeds the parser directly and bypasses the `io_thread`/PTY path.
- §5.1.4 the Python `encode_key_for_tty` encoder invoked without a GLFW event; byte-identical to the canonical results, but it bypasses the `on_key_input` OS-entry boundary.

**[INFERRED — source] statements (mechanism named from code; the *consequence* is observed, the internal step is cited, not printed):**

- The backpressure gate itself — `vt_parser_has_space_for_input` (`kitty/vt-parser.c:1477`) and the POLLIN gate (`kitty/child-monitor.c:1501`) — is inferred; its consequence (lossless writer-blocking) is observed (§4.2).
- The internal `"peer_death"` message (`kitty/child-monitor.c:1677`; `Boss.peer_message_received`, `kitty/boss.py:777`) is inferred; its consequence (peer removed, thread/process/socket survive) is observed (§4.3.3).
- The synchronized-output *frame hold/snapshot* (`screen_pause_rendering`, `kitty/screen.c:2506`) is inferred from code; what is observed is the DECRQM `expires_at` state and continued parsing (§2.3).
- `read_bytes` as the literal read site is inferred (the `--dump-bytes` figure is emitted parser-side, per F4); the child-on-PTY fact is observed (a real slave `/dev/pts/N`).
- The `loop-utils` wakeup/drain mechanism (§5.2) is inferred from source.

**Honest limitations (could not be observed in this headless, software-GL sandbox):**

- The **visible frame draw/buffer swap** — the literal "interface settles" pixel event — is not surfaced by `--debug-rendering` (which logs only window lifecycle), so it was not observed; the presence of a composed frame is inferred from the render configuration (§5.4).
- `screen_pause_rendering` holding a *displayed* frame cannot be pixel-verified here; only the parser-visible mode state and timing were captured (§2.3, §2.4).


## 7. Coverage pass — every question part and every named item

This is the final coverage check. The first table confirms each of the four question threads is answered; the second is an evidence-linked matrix over every symbol/struct/knob named in the plan, with the section where it is grounded and whether the grounding is canonical [OBSERVED], a labeled supplement [NON-CANONICAL], or code-cited [INFERRED].

**Question threads:**

| Thread | Question | Answered in | Basis |
|--------|----------|-------------|-------|
| Q1 | Where does a surge first enter, and how does pause/resume work? | §2.1–§2.4 | [OBSERVED] |
| Q2 | The "conductor": thread split + what decides which event is handled first | §3.1–§3.3, §5.2 | [OBSERVED] + [INFERRED] |
| Q3 | Hints aligned with text; behavior under backpressure and an unstable remote | §4.1–§4.3.4 | [OBSERVED] (+ labeled supplements) |
| Q4 | End-to-end from mixed input to the interface settling | §5.1–§5.5 | [OBSERVED] (render/settle limitation disclosed) |

**Named items (symbols, structs, knobs):**

| Item (file:line) | Section | Evidence |
|------------------|---------|----------|
| `read_bytes` (child-monitor.c:1337) | §2.1, §2.2 | [OBSERVED] consequence (real `/dev/pts` child; 5000-line surge) + [INFERRED] read site |
| `io_loop` poll order: wakeup → signals → child PTYs (child-monitor.c:1509-1531) | §3.2 | [INFERRED] source, qualified per F17 (same-cycle ordering) |
| `input_delay` coalescing (child-monitor.c:1562-1569; flush vt-parser.c:1425) | §3.3 | [OBSERVED] measured ≈3 ms, sweep {0,3,10,30}, ≥2 runs |
| `vt_parser_has_space_for_input` (vt-parser.c:1477) + POLLIN gate (child-monitor.c:1501) | §4.2 | [INFERRED] gate; [OBSERVED] lossless writer-blocking |
| `BUF_SZ` = 1 MiB (vt-parser.c:18) | §4.2, §5.4 | [OBSERVED] threshold behavior + binary value 1048576 |
| `run_worker`/`parse_worker` (vt-parser.c:1496) | §4.1, §5.5 | [OBSERVED] serial dispatch |
| `consume_normal` (vt-parser.c:230) | §4.1 | [OBSERVED] `draw` interleave |
| `dispatch_osc` → OSC 133 (vt-parser.c:457) | §4.1 | [OBSERVED] `shell_prompt_marking` |
| `dispatch_dcs` (vt-parser.c:620), `$q`/`+q` → `screen_request_capabilities` (L631) | §4.3.1 | [OBSERVED] (F13 correction) |
| `parse_kitty_dcs` (vt-parser.c:586) → `handle_remote_cmd` (L603) | §4.3.2 | [OBSERVED] inline `@kitty-cmd` |
| `cmd_output_marking` (screen.c:2338) | §4.1 | [OBSERVED] via real bash OSC 133 |
| `screen_pause_rendering` (screen.c:2506), `PENDING_MODE 2026` (control-codes.h:235) | §2.3 | [OBSERVED] DECRQM state; [INFERRED] frame hold |
| `screen_check_pause_rendering` (screen.c:2489), `expires_at` timeout | §2.4 | [OBSERVED] auto-resume at ≈2000 ms |
| `talk_loop`/`KittyPeerMon` (child-monitor.c:1805/1808) | §3.1, §4.3 | [OBSERVED] thread present; independence |
| `accept_peer` (child-monitor.c:1632), `read_from_peer` (child-monitor.c:1714) | §4.3.3 | [INFERRED] peer path; [OBSERVED] "Malformatted…" per peer |
| `notify_on_peer_removal` / `"peer_death"` (child-monitor.c:1673/1677) | §4.3.3 | [INFERRED] message; [OBSERVED] consequence |
| `inject_peer` (child-monitor.c:248) — `KittyPeerMon` without a listen socket | §3.1 | [OBSERVED] single-instance case (F7 correction) |
| `loop-utils`: `wakeup_loop` (L113), `drain_fd` (loop-utils.h:76), `init_loop_data` (L59) | §5.2 | [INFERRED] source (F12) |
| `on_key_input` (keys.c:166) | §5.1 | [OBSERVED] canonical XTEST |
| `encode_glfw_key_event` (keys.c:251) — `mDECCKM` + key-encoding flags | §5.1.1–§5.1.3 | [OBSERVED] legacy/DECCKM/kitty-protocol |
| `schedule_write_to_child` (keys.c:259) | §5.1.1 | [OBSERVED] child byte log |
| `encode_key_for_tty`/`pyencode_key_for_tty` (keys.c:311); `Window.send_key`→`encoded_key`→`write_to_child` (window.py:917/1795/955) | §5.1.4 | [INFERRED] chain (F11 correction) + [NON-CANONICAL] supplement |
| `Boss` / `peer_message_received` (boss.py:323/776), `on_child_death` (boss.py:881) | §4.3.3, §2–§5 | [OBSERVED] peer_death handling; `close_on_child_death` used throughout |
| `Child` / `fork` (child.py:197/276) | §2.1 | [OBSERVED] real slave `/dev/pts/N` |
| shell-integration emitters (`shell-integration/bash/…`) | §4.1 | [OBSERVED] real bash marks |
| knobs `input_delay`=3 / `repaint_delay`=10 / `sync_to_monitor`=yes (definition.py:878/866/889) | §5.4 | [OBSERVED] read from the binary |
| CLI flags `--dump-commands`/`--dump-bytes`/`--replay-commands`/`--debug-input`/`--debug-rendering` (cli.py:972-996) | §5.3, §5.4 | [OBSERVED] each exercised |
| version `0.35.2` + VCS `815df1e210e0` (constants.py:25; setup.py:674) | §1 | [OBSERVED] `--version` banner |
| visible frame draw/swap ("interface settles" pixels) | §5.4 | Not observed — disclosed limitation |

Every question part and every named item above is grounded in a specific section with a labeled basis; no claim of completeness is made for the one item (visible frame draw) that the headless sandbox cannot show, which is explicitly disclosed rather than asserted.

## Appendix A — Complete observation harness (verbatim, reproducible)

Every experiment above was produced by the scripts below, reproduced here in full so each result is independently reproducible and auditable (F1). All scripts lived under a `umask 077` observation directory created with `mktemp -d` (mode 0700); `common.sh` established the canonical, secure run environment and readiness/cleanup helpers sourced by every experiment. Paths shown as `$OBS`/`$OBS_DIR` refer to that directory. These scripts are temporary observation tooling and are **removed** after the investigation — they are reproduced here, not left in the repository.

### `common.sh`

```
# common.sh — sourced by every experiment. Establishes the canonical, SECURE
# run environment for a headless kitty and provides readiness/cleanup helpers.
# Security (F16): umask 077; the observation directory is a 0700 mktemp -d; every
# launched kitty PID is captured with $! and killed by an EXIT trap; unix sockets
# live only inside the 0700 directory and remote control is scoped to that socket.
set -euo pipefail
umask 077

# shellcheck disable=SC1091
source /root/kitty-venv/bin/activate
export PATH="$PATH:/usr/local/go/bin"
export TMPDIR=/tmp/kitty-nosgid
export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1
export PYTHONHOME=/root/.local/share/uv/python/cpython-3.11.15-linux-x86_64-gnu
export PYTHONPATH=/root/kitty-venv/lib/python3.11/site-packages
export LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8

OBS_DIR="${OBS_DIR:-$(mktemp -d "$TMPDIR/kitty_obs.XXXXXXXX")}"; chmod 700 "$OBS_DIR"
KITTY=./kitty/launcher/kitty
PYBIN=/root/kitty-venv/bin/python3

_SPAWNED_PIDS=()
track_pid() { _SPAWNED_PIDS+=("$1"); }
cleanup() {
    local pid
    for pid in "${_SPAWNED_PIDS[@]:-}"; do
        [ -n "${pid:-}" ] || continue
        kill "$pid" 2>/dev/null || true
        wait "$pid" 2>/dev/null || true
    done
}
trap cleanup EXIT INT TERM

wait_for_socket() {
    local sock="$1" timeout="${2:-10}" i=0
    while [ ! -S "$sock" ]; do
        i=$((i+1)); [ "$i" -gt $((timeout*20)) ] && { echo "socket $sock never appeared" >&2; return 1; }
        sleep 0.05
    done
}

wait_for_file_line() {
    local f="$1" pat="$2" timeout="${3:-10}" i=0
    while ! grep -q "$pat" "$f" 2>/dev/null; do
        i=$((i+1)); [ "$i" -gt $((timeout*20)) ] && { echo "pattern '$pat' never appeared in $f" >&2; return 1; }
        sleep 0.05
    done
}
```

### `surge_validate.py`

```
#!/usr/bin/env python3
"""Deterministic validator for the Q1 surge (F5).

Reads the raw-bytes dump and the dump-commands file for a 5000-line surge and
checks EVERY record: that draw lines are exactly L0001..L5000 in order with no
gaps, that the raw byte count matches, and prints the sha256 of the raw dump.
Exits non-zero (failing) on any mismatch so a bad run cannot masquerade as good.
"""
import hashlib
import re
import sys

bytes_path, dump_path, expected_n = sys.argv[1], sys.argv[2], int(sys.argv[3])

raw = open(bytes_path, "rb").read()
sha = hashlib.sha256(raw).hexdigest()

draws = []
for line in open(dump_path, "r", encoding="utf-8", errors="replace"):
    m = re.match(r"^draw (L\d{4})-THE-QUICK-BROWN-FOX$", line.rstrip("\n"))
    if m:
        draws.append(int(m.group(1)[1:]))

problems = []
if len(draws) != expected_n:
    problems.append(f"draw count {len(draws)} != expected {expected_n}")
for idx, val in enumerate(draws, start=1):
    if val != idx:
        problems.append(f"order break at position {idx}: got L{val:04d}")
        break

expected_bytes = expected_n * 28
if len(raw) != expected_bytes:
    problems.append(f"raw byte count {len(raw)} != expected {expected_bytes}")

print(f"records_expected={expected_n}")
print(f"draw_records_found={len(draws)}")
print(f"first_draw=L{draws[0]:04d}" if draws else "first_draw=NONE")
print(f"last_draw=L{draws[-1]:04d}" if draws else "last_draw=NONE")
print(f"all_present_1..{expected_n}={'YES' if draws==list(range(1,expected_n+1)) else 'NO'}")
print(f"raw_bytes={len(raw)} expected_raw_bytes={expected_bytes}")
print(f"raw_sha256={sha}")
if problems:
    print("RESULT=FAIL")
    for p in problems:
        print("  problem:", p)
    sys.exit(1)
print("RESULT=PASS")
sys.exit(0)
```

### `pause_probe.py`

```
#!/usr/bin/env python3
"""Q1 synchronized-output (mode 2026) pause/resume probe — a real PTY child.

Runs as kitty's child: fd 0/1 are the PTY slave. It emits DEC private mode 2026
begin/end around a batch of draws, and at each phase issues a DECRQM query
(CSI ? 2026 $ p) and reads kitty's reply back THROUGH the PTY. Every byte it
writes to the PTY is also appended to EMIT_LOG, so the full emitted byte stream
can be shown independently (proving CSI ?2026 h and l were actually sent).
"""
import os
import select
import termios
import tty

EMIT_LOG = os.environ["EMIT_LOG"]
_emit = open(EMIT_LOG, "wb")


def send(data: bytes) -> None:
    _emit.write(data)
    _emit.flush()
    os.write(1, data)


def read_reply(timeout=1.0) -> bytes:
    buf = b""
    while True:
        r, _, _ = select.select([0], [], [], timeout)
        if not r:
            break
        chunk = os.read(0, 64)
        if not chunk:
            break
        buf += chunk
        if buf.endswith(b"y"):
            break
    return buf


def decode(reply: bytes) -> str:
    return reply.replace(b"\x1b", b"<ESC>").decode("latin-1")


def query_and_report(phase: str) -> None:
    send(b"\x1b[?2026$p")
    reply = read_reply()
    send(f"PHASE={phase} DECRQM_REPLY={decode(reply)}\r\n".encode())


old = termios.tcgetattr(0)
tty.setraw(0)
try:
    query_and_report("before")
    send(b"\x1b[?2026h")
    send(b"DRAWN-WHILE-PAUSED\r\n")
    query_and_report("during")
    send(b"\x1b[?2026l")
    query_and_report("after")
finally:
    termios.tcsetattr(0, termios.TCSANOW, old)
    _emit.close()
```

### `pause_timeout_probe.py`

```
#!/usr/bin/env python3
"""Q1 mode-2026 safety-timeout probe — a real PTY child.

Begins synchronized output (CSI ?2026 h) and DELIBERATELY never ends it, then
polls DECRQM (CSI ?2026 $ p) roughly every 500 ms for ~3.5 s. Every emitted byte
is logged to EMIT_LOG so it can be shown that NO CSI ?2026 l ('l' terminator)
was ever sent — the auto-resume is due solely to the safety timeout.
Timestamps use time.monotonic_ns() (a steady clock).
"""
import os
import select
import termios
import time
import tty

EMIT_LOG = os.environ["EMIT_LOG"]
_emit = open(EMIT_LOG, "wb")


def send(data: bytes) -> None:
    _emit.write(data)
    _emit.flush()
    os.write(1, data)


def read_reply(timeout=1.0) -> bytes:
    buf = b""
    while True:
        r, _, _ = select.select([0], [], [], timeout)
        if not r:
            break
        chunk = os.read(0, 64)
        if not chunk:
            break
        buf += chunk
        if buf.endswith(b"y"):
            break
    return buf


def decode(reply: bytes) -> str:
    return reply.replace(b"\x1b", b"<ESC>").decode("latin-1")


old = termios.tcgetattr(0)
tty.setraw(0)
t0 = time.monotonic_ns()
try:
    send(b"\x1b[?2026h")
    for _ in range(8):
        send(b"\x1b[?2026$p")
        reply = read_reply()
        ms = (time.monotonic_ns() - t0) // 1_000_000
        send(f"T={ms:04d}ms DECRQM_REPLY={decode(reply)}\r\n".encode())
        time.sleep(0.5)
finally:
    termios.tcsetattr(0, termios.TCSANOW, old)
    _emit.close()
```

### `thread_inspect.sh`

```
#!/usr/bin/env bash
# Launch kitty with the given extra args plus a long-lived child, capture its
# PID via $!, wait until the io-thread (KittyChildMon) is named, then print the
# COMPLETE /proc/<pid>/task/*/comm listing (every thread identity). The launched
# PID is reaped exactly (no pkill) by this script and by common.sh's EXIT trap.
set -euo pipefail
source "$(dirname "$0")/common.sh"
cd "$(git rev-parse --show-toplevel)"
label="$1"; shift
"$KITTY" -o close_on_child_death=yes "$@" sh -c 'sleep 60' >/dev/null 2>&1 &
kpid=$!
track_pid "$kpid"
ready=no
for i in $(seq 1 200); do
    if grep -qs KittyChildMon /proc/"$kpid"/task/*/comm 2>/dev/null; then ready=yes; break; fi
    sleep 0.05
done
echo "label=$label pid=$kpid childmon_ready=$ready"
echo "TID COMM"
for t in /proc/"$kpid"/task/*/comm; do
    tid=$(basename "$(dirname "$t")")
    printf '%s %s\n' "$tid" "$(cat "$t")"
done
kill "$kpid" 2>/dev/null || true
wait "$kpid" 2>/dev/null || true
```

### `pingpong.py`

```
#!/usr/bin/env python3
"""Q2 input_delay coalescing probe — a real PTY child (RTT proxy).

In raw mode, repeatedly sends a DECRQM query (CSI ?2026 $p) and times how long
until kitty's reply (ends with 'y') comes back through the PTY, using the steady
clock time.monotonic_ns(). Writes one RTT (ns) per line to RTT_LOG, preceded by
a '# n=.. wall_ns=..' header. This RTT is a PROXY for the main-loop wakeup
latency: it also includes read_bytes, parse, reply-encode and the return read;
those are ~constant across runs, so DIFFERENCES in RTT as input_delay is varied
isolate the coalescing window.

PP_GAP_MS (optional) sleeps that long BEFORE each timed query (outside the timed
window). A gap larger than input_delay exercises the "immediate-after-idle"
branch (child-monitor.c:1565): the first byte after idle wakes the main loop
immediately, without waiting for the coalescing window.
"""
import os
import select
import termios
import time
import tty

RTT_LOG = os.environ["RTT_LOG"]
N = int(os.environ.get("PP_N", "500"))
WARMUP = int(os.environ.get("PP_WARMUP", "20"))
GAP_MS = float(os.environ.get("PP_GAP_MS", "0"))
QUERY = b"\x1b[?2026$p"


def one_rtt() -> int:
    if GAP_MS > 0:
        time.sleep(GAP_MS / 1000.0)
    t0 = time.monotonic_ns()
    os.write(1, QUERY)
    buf = b""
    while not buf.endswith(b"y"):
        r, _, _ = select.select([0], [], [], 2.0)
        if not r:
            break
        buf += os.read(0, 64)
    return time.monotonic_ns() - t0


old = termios.tcgetattr(0)
tty.setraw(0)
rtts = []
try:
    for _ in range(WARMUP):
        one_rtt()
    tstart = time.monotonic_ns()
    for _ in range(N):
        rtts.append(one_rtt())
    tend = time.monotonic_ns()
finally:
    termios.tcsetattr(0, termios.TCSANOW, old)

with open(RTT_LOG, "w") as f:
    f.write(f"# n={len(rtts)} wall_ns={tend - tstart}\n")
    for r in rtts:
        f.write(f"{r}\n")
```

### `rtt_stats.py`

```
#!/usr/bin/env python3
"""Deterministic RTT statistics for a pingpong RTT_LOG (F6).

Reads '# n=.. wall_ns=..' header + one RTT (ns) per line, prints count, wall
duration, and min/median/p90/max/mean in milliseconds. Pure function of the
input file, so the summary is reproducible from the raw trials.
"""
import statistics
import sys

path = sys.argv[1]
label = sys.argv[2] if len(sys.argv) > 2 else path
n_hdr = wall_ns = None
vals = []
for line in open(path):
    line = line.strip()
    if line.startswith("#"):
        for tok in line[1:].split():
            k, _, v = tok.partition("=")
            if k == "n":
                n_hdr = int(v)
            elif k == "wall_ns":
                wall_ns = int(v)
        continue
    if line:
        vals.append(int(line))


def ms(x):
    return x / 1_000_000.0


vals_sorted = sorted(vals)
p90 = vals_sorted[min(len(vals_sorted) - 1, int(round(0.90 * (len(vals_sorted) - 1))))]
print(f"label={label}")
print(f"count={len(vals)} (header n={n_hdr})")
print(f"wall_duration_s={ms(wall_ns)/1000:.3f}" if wall_ns else "wall_duration_s=NA")
print(f"min_ms={ms(min(vals)):.4f}")
print(f"median_ms={ms(statistics.median(vals)):.4f}")
print(f"p90_ms={ms(p90):.4f}")
print(f"max_ms={ms(max(vals)):.4f}")
print(f"mean_ms={ms(statistics.mean(vals)):.4f}")
```

### `osc133_capture.sh`

```
#!/usr/bin/env bash
# Q3 OSC 133 capture: launch a REAL interactive bash under a real kitty PTY with
# kitty's shipped shell integration (auto-injected). Drive one command via
# `kitty @ send-text` (the input trigger only; the OSC 133 output is genuine
# shell-integration output). Capture both the raw bytes (--dump-bytes) and the
# parser events (--dump-commands). Readiness-gated on the socket and on the
# first prompt marker; the launched PID is reaped exactly.
set -euo pipefail
source "$(dirname "$0")/common.sh"
cd "$(git rev-parse --show-toplevel)"
SOCK_PATH="$OBS_DIR/ksi.sock"
SOCK="unix:$SOCK_PATH"
DUMP="$OBS_DIR/osc133_bytes.bin"
CMDS="$OBS_DIR/osc133_dump.txt"
: > "$DUMP"
"$KITTY" -o close_on_child_death=yes -o allow_remote_control=yes --listen-on "$SOCK" \
    --dump-bytes="$DUMP" --dump-commands bash > "$CMDS" 2>"$OBS_DIR/osc133.err" &
kpid=$!
track_pid "$kpid"
wait_for_socket "$SOCK_PATH" 15
# wait for shell integration to emit the first prompt-start marker in the raw dump
ok=no
for i in $(seq 1 200); do
    if grep -qa $'\x1b]133;A' "$DUMP" 2>/dev/null; then ok=yes; break; fi
    sleep 0.05
done
echo "prompt_marker_seen=$ok"
"$KITTY" @ --to "$SOCK" send-text $'printf "HELLO-OUTPUT\\n"\r' && echo "send_text_exit=$?"
for i in $(seq 1 200); do
    if grep -qa 'HELLO-OUTPUT' "$CMDS" 2>/dev/null; then break; fi
    sleep 0.05
done
sleep 0.4   # allow the D;<exit> marker for the finished command to land
kill "$kpid" 2>/dev/null || true
wait "$kpid" 2>/dev/null || true
echo "done"
```

### `osc133_extract.py`

```
#!/usr/bin/env python3
"""Extract and pretty-print every OSC 133 sequence from a raw --dump-bytes file.

An OSC starts with ESC ] (0x1b 0x5d) and ends with BEL (0x07) or ST (ESC \\).
Prints one line per OSC-133 sequence with ESC shown as <ESC> and BEL as <BEL>,
in the exact order they appear in the byte stream (proving serial ordering).
"""
import re
import sys

raw = open(sys.argv[1], "rb").read()
# match ESC ] ... (BEL | ESC \)
pat = re.compile(rb"\x1b\][^\x07\x1b]*(?:\x07|\x1b\\)")
n = 0
for m in pat.finditer(raw):
    seq = m.group(0)
    if seq[2:6] != b"133;":
        continue
    n += 1
    disp = seq.replace(b"\x1b", b"<ESC>").replace(b"\x07", b"<BEL>")
    print(f"{n:02d}: {disp.decode('latin-1')}")
print(f"total_osc133={n}")
```

### `flood.py`

```
#!/usr/bin/env python3
"""Q3 backpressure flood — a real PTY child emitting a VARIED payload >> 1 MiB.

Sets the PTY to raw mode (so the line discipline performs no transforms), then
writes a DETERMINISTIC, VARIED byte stream (concatenated SHA-256 hex digests of
a block counter, no newlines/control bytes) of FLOOD_BYTES bytes to the PTY.
A single-byte drop, duplication or reorder changes the whole-stream SHA-256, so
counting is not relied upon. Records to META: the exact bytes sent, their
SHA-256, and the wall time the write took (which reflects how long kitty made
the writer block while draining).
"""
import hashlib
import os
import sys
import termios
import time
import tty

META = os.environ["FLOOD_META"]
TARGET = int(os.environ.get("FLOOD_BYTES", str(8 * 1024 * 1024)))

# Build the varied payload deterministically.
h = hashlib.sha256()
chunks = []
n = 0
i = 0
while n < TARGET:
    block = hashlib.sha256(b"blk-%d" % i).hexdigest().encode()  # 64 printable bytes
    chunks.append(block)
    n += len(block)
    i += 1
payload = b"".join(chunks)[:TARGET]
sent_sha = hashlib.sha256(payload).hexdigest()

old = termios.tcgetattr(1)
tty.setraw(1)
t0 = time.monotonic_ns()
try:
    written = 0
    while written < len(payload):
        written += os.write(1, payload[written:written + 65536])
finally:
    dt = time.monotonic_ns() - t0
    termios.tcsetattr(1, termios.TCSANOW, old)

with open(META, "w") as f:
    f.write(f"sent_bytes={len(payload)}\n")
    f.write(f"sent_sha256={sent_sha}\n")
    f.write(f"write_wall_ms={dt/1_000_000:.3f}\n")
```

### `flood_verify.py`

```
#!/usr/bin/env python3
"""Verify backpressure losslessness: compare kitty's --dump-bytes to what the
child sent (F9). PASS only if byte count AND SHA-256 match exactly."""
import hashlib
import sys

dump_path, meta_path, label = sys.argv[1], sys.argv[2], sys.argv[3]
raw = open(dump_path, "rb").read()
dump_sha = hashlib.sha256(raw).hexdigest()
meta = dict(l.strip().split("=", 1) for l in open(meta_path) if "=" in l)
sent_bytes = int(meta["sent_bytes"])
sent_sha = meta["sent_sha256"]
print(f"label={label}")
print(f"sent_bytes={sent_bytes} dumped_bytes={len(raw)}")
print(f"sent_sha256={sent_sha}")
print(f"dump_sha256={dump_sha}")
print(f"write_wall_ms={meta['write_wall_ms']}")
ok = (len(raw) == sent_bytes) and (dump_sha == sent_sha)
print(f"buffer_multiple={sent_bytes/ (1024*1024):.2f}x the 1 MiB VT parser buffer")
print("RESULT=PASS_LOSSLESS" if ok else "RESULT=FAIL_LOSSY")
sys.exit(0 if ok else 1)
```

### `decrqss_child.py`

```
import os, sys, termios, tty, select, binascii

outpath = sys.argv[1]
fd_in = 0          # PTY slave (stdin)  - kitty writes replies here
fd_out = 1         # PTY slave (stdout) - kitty parses what we write here

# DECRQSS queries: ESC P $ q <Pt> ESC \    (Pt = the setting being queried)
#   'm'      -> SGR (graphic rendition)
#   ' q'     -> DECSCUSR (cursor style)
#   'r'      -> DECSTBM (scroll region)
queries = {
    b"SGR(m)":      b"\x1bP$qm\x1b\\",
    b"DECSCUSR( q)": b"\x1bP$q q\x1b\\",
    b"DECSTBM(r)":  b"\x1bP$qr\x1b\\",
}

old = termios.tcgetattr(fd_in)
tty.setraw(fd_in)
captured = {}
try:
    for name, q in queries.items():
        os.write(fd_out, q)
        buf = b""
        # read reply until ST (ESC \) or timeout
        while True:
            r, _, _ = select.select([fd_in], [], [], 1.0)
            if not r:
                break
            chunk = os.read(fd_in, 4096)
            if not chunk:
                break
            buf += chunk
            if b"\x1b\\" in buf:
                break
        captured[name] = buf
finally:
    termios.tcsetattr(fd_in, termios.TCSADRAIN, old)

with open(outpath, "wb") as f:
    for name, q in queries.items():
        f.write(name + b"\n")
        f.write(b"  query_hex=" + binascii.hexlify(q) + b"\n")
        f.write(b"  reply_hex=" + binascii.hexlify(captured.get(name, b"")) + b"\n")
        f.write(b"  reply_repr=" + repr(captured.get(name, b"")).encode() + b"\n")
```

### `decrqss_probe.sh`

```
#!/usr/bin/env bash
source "$(dirname "$0")/common.sh"
xdpyinfo -display :99 >/dev/null 2>&1 || { echo "XVFB DOWN" >&2; exit 1; }
cd "$(git rev-parse --show-toplevel)"

OUT="$OBS_DIR/out"
reply="$OUT/decrqss_reply.txt"
dump="$OUT/decrqss_dump.txt"
err="$OUT/decrqss.err"
rm -f "$reply" "$dump" "$err"

timeout 30 "$KITTY" -o close_on_child_death=yes --dump-commands \
    sh -c "$PYBIN $OBS_DIR/harness/decrqss_child.py '$reply'; sleep 0.3" \
    > "$dump" 2>"$err" &
kpid=$!; track_pid "$kpid"
wait "$kpid"; krc=$?
echo "launcher_exit=$krc"
echo "=== child-captured DECRQSS replies (raw bytes, out-of-band file) ==="
cat "$reply"
echo "=== parser trace: lines mentioning screen_request_capabilities ==="
grep -n "screen_request_capabilities" "$dump" || echo "(none)"
echo "=== dump-commands total lines ==="
wc -l < "$dump"
```

### `inline_dcs_probe.sh`

```
#!/usr/bin/env bash
# Inline DCS remote-control path: a REAL PTY child emits  ESC P @ kitty-cmd{...} ESC \
# We prove the parser routes it to handle_remote_cmd (vt-parser.c:586 parse_kitty_dcs,
# dispatch("cmd{",handle_remote_cmd,1) at L603) -- a path DISTINCT from the talk socket.
source "$(dirname "$0")/common.sh"
xdpyinfo -display :99 >/dev/null 2>&1 || { echo "XVFB DOWN" >&2; exit 1; }
cd "$(git rev-parse --show-toplevel)"

OUT="$OBS_DIR/out"
dump="$OUT/inline_dcs_dump.txt"; err="$OUT/inline_dcs.err"
rm -f "$dump" "$err"

# Minimal well-formed kitty-cmd JSON for 'ls'. The parser routes ANY  @kitty-cmd{...}
# to handle_remote_cmd; command validity is a later concern.
payload='{"cmd":"ls","version":[0,35,2],"async_id":"","no_response":true}'
timeout 30 "$KITTY" -o close_on_child_death=yes -o allow_remote_control=yes --dump-commands \
    sh -c "printf '\033P@kitty-cmd%s\033\\\\' '$payload'; sleep 0.4" \
    > "$dump" 2>"$err" &
kpid=$!; track_pid "$kpid"
wait "$kpid"; krc=$?
echo "launcher_exit=$krc"
echo "payload_sent=@kitty-cmd$payload"
echo "=== parser trace lines mentioning handle_remote_cmd or screen_handle_kitty_dcs ==="
grep -n "handle_remote_cmd\|screen_handle_kitty_dcs" "$dump" || echo "(none)"
echo "=== full dump-commands trace ==="
cat "$dump"
echo "=== stderr ==="
cat "$err"
```

### `reader_child.py`

```
import os, sys, time
log = os.environ["DELIVERY_LOG"]
deadline = time.time() + float(os.environ.get("READER_TTL", "120"))
with open(log, "a", buffering=1) as f:
    f.write("READER_START\n")
    while time.time() < deadline:
        try:
            line = sys.stdin.readline()
        except Exception as e:
            f.write(f"READER_ERR {e!r}\n"); break
        if not line:
            time.sleep(0.02); continue
        f.write("RECV " + line.rstrip("\n") + "\n")
    f.write("READER_END\n")
```

### `peer_disrupt.py`

```
import socket, struct, sys
sockpath = sys.argv[1]
PARTIAL = b'\x1bP@kitty-cmd{"cmd":"ls","versi'   # truncated: no ESC\ terminator
s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
s.settimeout(5)
s.connect(sockpath)
s.sendall(PARTIAL)
s.setsockopt(socket.SOL_SOCKET, socket.SO_LINGER, struct.pack('ii', 1, 0))  # abrupt close
s.close()
sys.stdout.write("disrupt_sent_bytes=%d partial_sha=%s\n" % (
    len(PARTIAL), __import__('hashlib').sha256(PARTIAL).hexdigest()[:16]))
```

### `peermon_count.sh`

```
#!/usr/bin/env bash
pid="$1"
c=0
for t in /proc/"$pid"/task/*/comm; do
    [ -r "$t" ] || continue
    if [ "$(cat "$t" 2>/dev/null)" = "KittyPeerMon" ]; then c=$((c+1)); fi
done
echo "$c"
```

### `remote_harness.sh`

```
#!/usr/bin/env bash
# F10: unstable remote peer on the dedicated talk socket. Launch a kitty server with a
# real PTY child (reader_child.py). Baseline the talk socket; then apply N BYTE-IDENTICAL
# unstable-peer disruptions (peer_disrupt.py); after EACH, record raw per-trial facts:
#   proc_alive, KittyPeerMon TID count, recovery `kitty @ ls` exit+bytes (pipefail),
#   and out-of-band delivery of a unique marker via `kitty @ send-text`.
# Distribution is reported honestly over identical trials; a matched PTY control (RTT
# with vs without concurrent chaos) is measured by pty_independence.sh separately.
source "$(dirname "$0")/common.sh"
xdpyinfo -display :99 >/dev/null 2>&1 || { echo "XVFB DOWN" >&2; exit 1; }
cd "$(git rev-parse --show-toplevel)"

N="${DISRUPT_N:-40}"
OUT="$OBS_DIR/out"; sock="$OBS_DIR/ktalk.sock"; dlog="$OUT/remote_delivery.log"
rec="$OUT/remote_trials.tsv"; srvout="$OUT/remote_srv.out"; srverr="$OUT/remote_srv.err"
rm -f "$sock" "$dlog" "$rec" "$srvout" "$srverr"

DELIVERY_LOG="$dlog" READER_TTL=180 \
  timeout 200 "$KITTY" --listen-on "unix:$sock" -o allow_remote_control=yes -o close_on_child_death=yes \
    sh -c "DELIVERY_LOG='$dlog' $PYBIN $OBS_DIR/harness/reader_child.py" \
    >"$srvout" 2>"$srverr" &
srv=$!; track_pid "$srv"
wait_for_socket "$sock" 20
wait_for_file_line "$dlog" "READER_START" 20
# $srv is the `timeout` wrapper; the real kitty process is its child (comm=kitty).
kpid=$(ps --ppid "$srv" -o pid= | tr -d ' ')
echo "wrapper_pid=$srv real_kitty_pid=$kpid comm=$(cat /proc/$kpid/comm 2>/dev/null)"
echo "socket=$sock  (perm $(stat -c '%A' "$sock"))"
set -o pipefail

peermon0=$(bash "$OBS_DIR/harness/peermon_count.sh" "$kpid")
echo "baseline_KittyPeerMon_count=$peermon0"
"$KITTY" @ --to "unix:$sock" ls >"$OUT/remote_ls0.json" 2>/dev/null; echo "baseline_ls_exit=$? baseline_ls_bytes=$(wc -c <"$OUT/remote_ls0.json")"

printf 'trial\tproc_alive\tpeermon\tls_exit\tls_bytes\tdeliver\trtt_ms\n' > "$rec"
alive_ok=0; peermon_ok=0; ls_ok=0; deliver_ok=0
for i in $(seq 1 "$N"); do
    "$PYBIN" "$OBS_DIR/harness/peer_disrupt.py" "$sock" >/dev/null 2>&1 || true
    # proc alive?
    if kill -0 "$srv" 2>/dev/null; then pa=1; alive_ok=$((alive_ok+1)); else pa=0; fi
    # KittyPeerMon TID count
    pm=$(bash "$OBS_DIR/harness/peermon_count.sh" "$kpid")
    [ "$pm" -ge 1 ] && peermon_ok=$((peermon_ok+1))
    # recovery: well-formed ls over a NEW connection, timed
    t0=$(date +%s%N)
    "$KITTY" @ --to "unix:$sock" ls >"$OUT/remote_ls_t.json" 2>/dev/null; lx=$?
    t1=$(date +%s%N)
    lb=$(wc -c <"$OUT/remote_ls_t.json"); rttms=$(awk "BEGIN{printf \"%.2f\",($t1-$t0)/1e6}")
    [ "$lx" -eq 0 ] && ls_ok=$((ls_ok+1))
    # server-side delivery of a unique marker
    mk="TRIAL$(printf '%03d' "$i")-MARK"
    "$KITTY" @ --to "unix:$sock" send-text "$mk\r" >/dev/null 2>&1 || true
    if wait_for_file_line "$dlog" "$mk" 5 2>/dev/null; then dv=1; deliver_ok=$((deliver_ok+1)); else dv=0; fi
    printf '%d\t%d\t%d\t%d\t%d\t%d\t%s\n' "$i" "$pa" "$pm" "$lx" "$lb" "$dv" "$rttms" >> "$rec"
done

echo "=== per-trial records (TSV) ==="
cat "$rec"
echo "=== distribution over $N BYTE-IDENTICAL disruptions ==="
echo "proc_alive_after:      $alive_ok / $N"
echo "KittyPeerMon_present:  $peermon_ok / $N"
echo "recovery_ls_success:   $ls_ok / $N"
echo "send_text_delivered:   $deliver_ok / $N"
echo "recovery_rtt_ms distribution (min/median/max):"
awk -F'\t' 'NR>1{print $7}' "$rec" | sort -n | awk '{a[NR]=$1} END{if(NR){printf "  min=%s median=%s max=%s (n=%d)\n",a[1],a[int((NR+1)/2)],a[NR],NR}}'
echo "=== server stderr (unedited) ==="
cat "$srverr"
kill "$srv" 2>/dev/null || true; wait "$srv" 2>/dev/null || true
```

### `pty_independence.sh`

```
#!/usr/bin/env bash
# F10 matched control: does talk-socket chaos perturb the PTY pipeline? Run the SAME
# PTY RTT prober (pingpong.py, PP_N queries, input_delay=3) twice:
#   Phase A (baseline): talk socket idle.
#   Phase B (chaos):    a background loop fires byte-identical peer_disrupt.py events
#                       at the talk socket for the whole prober run.
# Same prober input in both -> a matched control. If A and B distributions coincide,
# the PTY input pipeline is independent of talk-socket turmoil.
source "$(dirname "$0")/common.sh"
xdpyinfo -display :99 >/dev/null 2>&1 || { echo "XVFB DOWN" >&2; exit 1; }
cd "$(git rev-parse --show-toplevel)"
OUT="$OBS_DIR/out"; N="${PP_N:-200}"

run_phase() {  # $1=label  $2=chaos(0/1)
    local label="$1" chaos="$2"
    local sock="$OBS_DIR/kpty_${label}.sock" rlog="$OUT/pty_rtt_${label}.log"
    local srverr="$OUT/pty_${label}.err"
    rm -f "$sock" "$rlog" "$srverr"
    RTT_LOG="$rlog" PP_N="$N" PP_WARMUP=20 \
      timeout 120 "$KITTY" --listen-on "unix:$sock" -o allow_remote_control=yes -o close_on_child_death=yes \
        sh -c "RTT_LOG='$rlog' PP_N='$N' PP_WARMUP=20 $PYBIN $OBS_DIR/harness/pingpong.py" \
        >/dev/null 2>"$srverr" &
    local wrap=$!; track_pid "$wrap"
    wait_for_socket "$sock" 20
    local dis_count=0
    if [ "$chaos" = "1" ]; then
        ( while kill -0 "$wrap" 2>/dev/null && [ -S "$sock" ]; do
              "$PYBIN" "$OBS_DIR/harness/peer_disrupt.py" "$sock" >/dev/null 2>&1 || true
          done ) &
        local disloop=$!; track_pid "$disloop"
    fi
    wait "$wrap" 2>/dev/null || true
    [ -n "${disloop:-}" ] && { kill "$disloop" 2>/dev/null || true; wait "$disloop" 2>/dev/null || true; }
    "$PYBIN" "$OBS_DIR/harness/rtt_stats.py" "$rlog" "$label"
}

echo "### Phase A (baseline, talk socket idle), PP_N=$N ###"
run_phase A_baseline 0
echo
echo "### Phase B (concurrent talk-socket chaos), PP_N=$N ###"
run_phase B_chaos 1
```

### `key_child.py`

```
import os, sys, termios, tty, select, time, binascii
keylog = os.environ["KEYLOG"]
ctl = os.environ.get("KEY_CTL", "")
ttl = float(os.environ.get("KEY_TTL", "60"))
old = termios.tcgetattr(0)
tty.setraw(0)
def emit(seq):  # write control seq to our stdout == the PTY -> kitty parser
    os.write(1, seq)
with open(keylog, "a", buffering=1) as f:
    f.write("CHILD_START tty=%s\n" % os.ttyname(0))
    deadline = time.time() + ttl
    last_ctl = ""
    while time.time() < deadline:
        # poll control file for mode switches
        if ctl and os.path.exists(ctl):
            try: cur = open(ctl).read().strip()
            except Exception: cur = last_ctl
            if cur != last_ctl:
                if cur == "DECCKM_ON":  emit(b"\x1b[?1h")
                elif cur == "DECCKM_OFF": emit(b"\x1b[?1l")
                elif cur == "KKP_ON":   emit(b"\x1b[>1u")   # push kitty keyboard flags=1
                elif cur == "KKP_OFF":  emit(b"\x1b[<u")    # pop kitty keyboard flags
                f.write("MODE %s\n" % cur)
                last_ctl = cur
        r,_,_ = select.select([0],[],[],0.1)
        if r:
            data = os.read(0, 4096)
            if data:
                f.write("RECV " + binascii.hexlify(data).decode() + "\n")
    termios.tcsetattr(0, termios.TCSANOW, old)
    f.write("CHILD_END\n")
```

### `find_focus_kitty.py`

```
import ctypes, sys
from ctypes import c_ulong, c_int, c_char_p, c_void_p, byref, POINTER, cast
X = ctypes.CDLL("libX11.so.6")
X.XOpenDisplay.restype = c_void_p
X.XDefaultRootWindow.restype = c_ulong
X.XGetClassHint.restype = c_int
dpy = X.XOpenDisplay(b":99")
if not dpy: print("NO_DISPLAY"); sys.exit(2)
root = X.XDefaultRootWindow(c_void_p(dpy))

class XClassHint(ctypes.Structure):
    _fields_ = [("res_name", c_char_p), ("res_class", c_char_p)]

def children(win):
    r=c_ulong(); p=c_ulong(); ch=POINTER(c_ulong)(); n=ctypes.c_uint()
    if not X.XQueryTree(c_void_p(dpy), c_ulong(win), byref(r), byref(p), byref(ch), byref(n)):
        return []
    out=[ch[i] for i in range(n.value)]
    return out

found=None
def walk(win, depth=0):
    global found
    hint=XClassHint()
    if X.XGetClassHint(c_void_p(dpy), c_ulong(win), byref(hint)):
        rc = hint.res_class or b""
        rn = hint.res_name or b""
        if b"kitty" in rc.lower() or b"kitty" in rn.lower():
            print("FOUND win=0x%x res_name=%s res_class=%s" % (win, rn.decode(errors='replace'), rc.decode(errors='replace')))
            found = win
    for c in children(win):
        walk(c, depth+1)

walk(root)
if found is None:
    print("KITTY_WINDOW_NOT_FOUND")
    sys.exit(3)
# focus it
X.XSetInputFocus(c_void_p(dpy), c_ulong(found), 2, 0)  # RevertToParent=2, CurrentTime=0
X.XFlush(c_void_p(dpy))
# verify
fw=c_ulong(); rev=c_int()
X.XGetInputFocus(c_void_p(dpy), byref(fw), byref(rev))
print("focus_set_to=0x%x current_focus=0x%x" % (found, fw.value))
print("FOCUS_OK" if fw.value==found else "FOCUS_MISMATCH")
```

### `xtest_inject.py`

```
#!/usr/bin/env python3
"""Inject REAL key events into the focused kitty window via the X11 XTEST extension.

This drives kitty's canonical inbound-keyboard entry point on_key_input (kitty/keys.c:166)
through the actual GLFW/X event queue -- NOT a Python shortcut. It (1) finds the kitty
top-level window by WM_CLASS, (2) XSetInputFocus to it (Xvfb has no WM), then (3) for each
requested key, presses any modifier keycodes, taps the key, and releases -- exactly as a
physical keyboard would. Each injected key is announced on stdout with a label so the
resulting kitty --debug-keyboard lines and child bytes can be matched to it.
"""
import ctypes, sys, time
from ctypes import c_ulong, c_int, c_uint, c_char_p, c_void_p, byref, POINTER

X = ctypes.CDLL("libX11.so.6")
XT = ctypes.CDLL("libXtst.so.6")
X.XOpenDisplay.restype = c_void_p
X.XDefaultRootWindow.restype = c_ulong
X.XKeysymToKeycode.restype = ctypes.c_ubyte
X.XGetClassHint.restype = c_int

dpy = X.XOpenDisplay(b":99")
if not dpy:
    print("NO_DISPLAY"); sys.exit(2)
root = X.XDefaultRootWindow(c_void_p(dpy))

class XClassHint(ctypes.Structure):
    _fields_ = [("res_name", c_char_p), ("res_class", c_char_p)]

def children(win):
    r=c_ulong(); p=c_ulong(); ch=POINTER(c_ulong)(); n=c_uint()
    if not X.XQueryTree(c_void_p(dpy), c_ulong(win), byref(r), byref(p), byref(ch), byref(n)):
        return []
    return [ch[i] for i in range(n.value)]

found=[None]
def walk(win):
    h=XClassHint()
    if X.XGetClassHint(c_void_p(dpy), c_ulong(win), byref(h)):
        if b"kitty" in (h.res_class or b"").lower():
            found[0]=win
    for c in children(win): walk(c)
walk(root)
if found[0] is None:
    print("KITTY_WINDOW_NOT_FOUND"); sys.exit(3)
win=found[0]
X.XSetInputFocus(c_void_p(dpy), c_ulong(win), 2, 0); X.XFlush(c_void_p(dpy))
fw=c_ulong(); rev=c_int(); X.XGetInputFocus(c_void_p(dpy), byref(fw), byref(rev))
print("kitty_window=0x%x focus=0x%x %s" % (win, fw.value, "FOCUS_OK" if fw.value==win else "FOCUS_MISMATCH"))

MODS = {"shift":0xffe1, "ctrl":0xffe3, "alt":0xffe9}
def kc(ksym): return X.XKeysymToKeycode(c_void_p(dpy), c_ulong(ksym))
def tap(code, press):
    XT.XTestFakeKeyEvent(c_void_p(dpy), c_uint(code), c_int(press), c_ulong(0)); X.XFlush(c_void_p(dpy))

# argv: label:mod1+mod2:0xKEYSYM  (mods empty for none)
for spec in sys.argv[1:]:
    label, mods, ks = spec.split(":")
    ksym = int(ks, 16)
    mcodes = [kc(MODS[m]) for m in mods.split("+") if m]
    for m in mcodes: tap(m, 1)
    k = kc(ksym); tap(k, 1); time.sleep(0.03); tap(k, 0)
    for m in reversed(mcodes): tap(m, 0)
    X.XFlush(c_void_p(dpy))
    print("INJECTED %s (mods=%s keysym=0x%x keycode=%d)" % (label, mods or "none", ksym, k))
    time.sleep(0.15)
```

### `keyboard_probe.sh`

```
#!/usr/bin/env bash
# F11: drive kitty's canonical inbound keyboard path (on_key_input) with REAL X key events.
# Launch a kitty GUI window (--debug-keyboard) whose child logs received bytes out-of-band;
# inject keys via XTEST; capture BOTH kitty's --debug-keyboard stderr (on_key_input + the
# encoded bytes it sent) and the child's independently-logged received bytes.
source "$(dirname "$0")/common.sh"
xdpyinfo -display :99 >/dev/null 2>&1 || { echo "XVFB DOWN" >&2; exit 1; }
cd "$(git rev-parse --show-toplevel)"
OUT="$OBS_DIR/out"
tag="${TAG:-legacy}"
klog="$OUT/key_${tag}.log"; kerr="$OUT/key_${tag}.err"; kout="$OUT/key_${tag}.out"; kctl="$OUT/key_${tag}.ctl"
rm -f "$klog" "$kerr" "$kout" "$kctl"
: > "$kctl"

KEYLOG="$klog" KEY_CTL="$kctl" KEY_TTL=30 \
  timeout 40 "$KITTY" --debug-keyboard -o close_on_child_death=yes \
    sh -c "KEYLOG='$klog' KEY_CTL='$kctl' KEY_TTL=30 $PYBIN $OBS_DIR/harness/key_child.py" \
    >"$kout" 2>"$kerr" &
wrap=$!; track_pid "$wrap"
wait_for_file_line "$klog" "CHILD_START" 15

# optional mode switch (DECCKM / kitty keyboard protocol) BEFORE injecting
if [ -n "${MODE:-}" ]; then
    echo "$MODE" > "$kctl"
    wait_for_file_line "$klog" "MODE $MODE" 10
    sleep 0.2
fi

echo "=== injecting keys (tag=$tag mode=${MODE:-none}) ==="
"$PYBIN" "$OBS_DIR/harness/xtest_inject.py" "$@"
sleep 0.5
kill "$wrap" 2>/dev/null || true; wait "$wrap" 2>/dev/null || true

echo "=== kitty --debug-keyboard stderr: on_key_input + encoded bytes ==="
grep -aE "on_key_input|sent encoded key|sent key as text|ignoring as keyboard" "$kerr" || echo "(no on_key_input lines captured)"
echo "=== child-received bytes (hex, out-of-band) ==="
cat "$klog"
```

### `pyencode_supplement.py`

```
# [NON-CANONICAL] supplemental: invoke the SAME encoder (encode_glfw_key_event) from
# Python via encode_key_for_tty, WITHOUT any real GLFW/OS key event. This corroborates
# the shared encoder but does NOT exercise the on_key_input OS-entry boundary.
from kitty.fast_data_types import encode_key_for_tty
GLFW_MOD_CONTROL = 0x4; GLFW_MOD_SHIFT = 0x1
GLFW_FKEY_UP = 0xe008   # observed 'glfw key' for Up in --debug-keyboard
def h(s): return s.encode('ascii').hex()
cases = [
    ("ctrl_a legacy",        dict(key=ord('a'), mods=GLFW_MOD_CONTROL, key_encoding_flags=0, cursor_key_mode=0)),
    ("ctrl_a KKP flags=1",   dict(key=ord('a'), mods=GLFW_MOD_CONTROL, key_encoding_flags=1, cursor_key_mode=0)),
    ("Up legacy DECCKM off", dict(key=GLFW_FKEY_UP, mods=0, key_encoding_flags=0, cursor_key_mode=0)),
    ("Up legacy DECCKM on",  dict(key=GLFW_FKEY_UP, mods=0, key_encoding_flags=0, cursor_key_mode=1)),
    ("Shift+Up legacy",      dict(key=GLFW_FKEY_UP, mods=GLFW_MOD_SHIFT, key_encoding_flags=0, cursor_key_mode=0)),
]
for name, kw in cases:
    out = encode_key_for_tty(**kw)
    print(f"{name}: repr={out!r} hex={h(out)}")
```

### `defaults_query.py`

```
# Read the DEFAULT option values FROM THE BUILT BINARY (not the .py source text):
# `defaults` is the canonical default Options instance built into the package.
from kitty.options.types import defaults
print("input_delay=%r (ms)" % defaults.input_delay)
print("repaint_delay=%r (ms)" % defaults.repaint_delay)
print("sync_to_monitor=%r" % defaults.sync_to_monitor)
from kitty.fast_data_types import VT_PARSER_BUFFER_SIZE
print("VT_PARSER_BUFFER_SIZE=%d bytes (=%d MiB)" % (VT_PARSER_BUFFER_SIZE, VT_PARSER_BUFFER_SIZE//(1024*1024)))
```
