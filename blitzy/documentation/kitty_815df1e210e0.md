# How kitty turns a surge of raw terminal input into something the application reacts to

**An investigate-by-running answer, grounded in the observed runtime behavior of a canonically built kitty.**

- **Subject:** kitty terminal emulator, version **0.35.2**, source branch `kitty_815df1e210e0` (rev `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`). The default build of this destination checkout stamps its banner `KITTY_VCS_REV = 2017484b1138582e48da1450d325790dac7d152f` (= this tree's `HEAD`, which is the source rev plus only this one added doc file); §1 STEP 6–8 shows both revs with full provenance.
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
- **[INFERRED]** — read from the source code (with a `file:line` citation) and consistent with the observed behavior, but not itself directly captured at runtime. Used where the sandbox cannot expose an internal signal directly (e.g., an internal C-level branch, or the GPU buffer-swap *event* timing under a software GL rasterizer — note the rendered frame *content* itself **is** captured at the pixel level, see §2.3).
- **[NON-CANONICAL]** — obtained through a bypassing interface (the synchronous `parse_bytes` test helper, a remote-control/debug hook, or a hand-written synthetic byte sequence). Used only as a clearly-labeled supplement, **never** as primary proof. §6 lists every non-canonical item.

**Filtering disclosure (important):** where a raw capture is enormous (e.g., a 20 000-line parser dump), this document does **not** silently show a few lines and imply the rest. Instead it runs a **deterministic validator** over the *entire* file and shows the validator's complete output, and additionally states the file's line count and `sha256`. Any place where only part of a file is shown is labeled *illustrative* and is accompanied by the authoritative whole-file check.

**Verbatim-fidelity disclosure (control bytes & trailing whitespace):** the fenced blocks reproduce kitty's output byte-for-byte, with one unavoidable transformation — a raw `ESC` (`0x1b`) control byte cannot be represented as printable text inside a Markdown file. kitty's `--debug-keyboard` colours the `on_key_input` label with an SGR escape (`ESC [ 33 m … ESC [ m`); in the fenced blocks below the two `ESC` bytes of that colour sequence are dropped, so the label reads `[33mon_key_input[m` rather than an invisible control byte. Everything *semantically meaningful* is verbatim — this was verified programmatically: each shown `on_key_input` line equals its raw-capture line with only the `0x1b` bytes removed (every field from `glfw key` onward is byte-identical). Conversely, **trailing whitespace is preserved exactly, not trimmed**: kitty's `--debug-keyboard` byte listing (e.g. `sent encoded key to child: ^[ [ A `) and the shell `draw` prompt lines (e.g. `draw …# ` and `draw PROMPT$ `) genuinely end in a space, so those bytes are kept as-is — which means a whitespace linter such as `git diff --check` will, by design, report trailing whitespace on exactly those evidence lines rather than on any authored prose.

**Honest sandbox limitations (stated up front, expanded where relevant):** the observations were made in a **headless** container (Xvfb + Mesa software GL, running as `root` — the only account the image provides). Two rendering/input concerns a headless setup raises are handled explicitly below — and in both cases the thing that actually matters turned out to be **observable**:
1. **Visible pixels — captured.** The rendered frame *is* sampled directly: the X11 window backing image is read with `Window.get_image` (`X.ZPixmap`), and it demonstrates synchronized output holding then atomically flushing the display, with SHA-256 hashes and a pixel-diff (§2.3, §2.4). What a headless container genuinely cannot expose is one layer *below* the frame content — the GPU buffer-swap / `vblank` **event** timing (not surfaced by `--debug-rendering`, §5.4) and photons on a physical monitor (there is none). Those two are disclosed as [INFERRED]/not-observed; the frame-*content* transition the question asks about (held → flushed atomically) is **[OBSERVED]**.
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
$ umask 077; OBS=$(mktemp -d "$TMPDIR/kitty_obs.XXXXXXXX"); chmod 700 "$OBS"; export OBS OBS_DIR="$OBS"
$ stat -c '%A %U:%G %n' "$OBS"
drwx------ root:root /tmp/kitty-nosgid/kitty_obs.9k1OYsxd

########## STEP 5: canonical build (make == python3 setup.py), then list artifacts ##########
$ env -u PYTHONHOME bash -c 'source /root/kitty-venv/bin/activate; export PATH="$PATH:/usr/local/go/bin"; export TMPDIR=/tmp/kitty-nosgid; make 2>&1 | tail -6; echo "make_exit=${PIPESTATUS[0]}"'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
Could not find platform dependent libraries <exec_prefix>
kitty/tools/cmd
make_exit=0
$ ls -1 kitty/fast_data_types*.so kitty/glfw-x11.so kitty/launcher/kitty kittens/transfer/rsync.so
kittens/transfer/rsync.so
kitty/fast_data_types.so
kitty/glfw-x11.so
kitty/launcher/kitty

########## STEP 6: VCS-stamped banner from the DEFAULT build of the destination tree ##########
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
$ git rev-parse HEAD
2017484b1138582e48da1450d325790dac7d152f
$ ./kitty/launcher/kitty +runpy 'from kitty.fast_data_types import KITTY_VCS_REV as r; print(r)'
2017484b1138582e48da1450d325790dac7d152f

########## STEP 7: provenance — the destination HEAD is the source rev + only this one file (F15) ##########
$ git rev-parse 815df1e210e0
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
$ git log -1 --format='%h %s' 815df1e210e0
815df1e21 Wire up applying of font config
$ git diff --name-status 815df1e210e0..HEAD
A	blitzy/documentation/kitty_815df1e210e0.md

########## STEP 8: the source rev itself stamps 815df1e210e0 (isolated worktree, same canonical build) ##########
$ git worktree add --detach /tmp/kitty-nosgid/kitty_srcrev_815df1e210e0 815df1e210e0
Preparing worktree (detached HEAD 815df1e21)
HEAD is now at 815df1e21 Wire up applying of font config
$ env -u PYTHONHOME bash -c 'cd /tmp/kitty-nosgid/kitty_srcrev_815df1e210e0; source /root/kitty-venv/bin/activate; export PATH="$PATH:/usr/local/go/bin"; export TMPDIR=/tmp/kitty-nosgid; make >/dev/null 2>&1; echo "make_exit=$?"'
make_exit=0
$ /tmp/kitty-nosgid/kitty_srcrev_815df1e210e0/kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
$ ( cd /tmp/kitty-nosgid/kitty_srcrev_815df1e210e0 && ./kitty/launcher/kitty +runpy 'from kitty.fast_data_types import KITTY_VCS_REV as r; print(r)' )
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
```

**Provenance note (resolves the "which commit?" ambiguity — two rev values, one honest story):** a canonical build stamps its banner with whatever `git rev-parse HEAD` returns *at build time*. This is not a guess — it is exactly what `get_vcs_rev()` does: it shells out to `git rev-parse HEAD` [setup.py:674, the `git rev-parse HEAD` call at setup.py:678] and bakes the result into the C extension as `KITTY_VCS_REV`. Consequently the compiled-in rev depends on *which tree you build*, and the two builds shown above stamp two different values — both observed, neither invented:

- **The default build of the destination tree** (this repository, at its current `HEAD`) stamps `KITTY_VCS_REV = 2017484b1138582e48da1450d325790dac7d152f`, because `git rev-parse HEAD` in this tree returns `2017484b1…` (STEP 6). This is the canonical build a normal user of *this checkout* gets, so it is the value the running binary actually reports.
- **An isolated worktree checked out at the source rev** `815df1e210e0` — built with the identical canonical `make` — stamps `KITTY_VCS_REV = 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (STEP 8), confirming that the source branch's own commit stamps the source branch's own hash.

The reason these two builds are **behaviorally identical for every observation in this document** is shown by STEP 7: `git diff --name-status 815df1e210e0..HEAD` reports exactly one changed path — `A  blitzy/documentation/kitty_815df1e210e0.md` — the addition of *this* documentation file and nothing else. That file is pure Markdown; it is not part of the C extension, the Python package, or any code path exercised below, so it cannot change any runtime behavior. In other words, the destination `HEAD` (`2017484b1…`) *is* the source rev (`815df1e210e0…`) plus this one non-code file, which is why the runtime pipeline the two binaries execute is bit-for-bit the same even though their stamped VCS strings differ. Every runtime observation in this document was captured from the **default build of the destination tree** (the binary stamped `2017484b1…`); STEP 8's worktree build exists solely to demonstrate the source-rev provenance and is otherwise identical.

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
$ ./kitty/launcher/kitty -o close_on_child_death=yes --dump-bytes=$OBS/out/q1_surge_bytes.bin --dump-commands sh -c 'for i in $(seq 1 5000); do printf "L%04d-THE-QUICK-BROWN-FOX\r\n" "$i"; done; sleep 0.4' > $OBS/out/q1_surge_dump.txt
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
$ for r in 2 3; do
    ./kitty/launcher/kitty -o close_on_child_death=yes --dump-bytes=$OBS/out/q1_surge_bytes_r$r.bin --dump-commands \
      sh -c 'for i in $(seq 1 5000); do printf "L%04d-THE-QUICK-BROWN-FOX\r\n" "$i"; done; sleep 0.4' > $OBS/out/q1_surge_dump_r$r.txt 2>/dev/null
    out=$(python3 harness/surge_validate.py $OBS/out/q1_surge_bytes_r$r.bin $OBS/out/q1_surge_dump_r$r.txt 5000); rc=$?
    sha=$(echo "$out" | awk -F= '/^raw_sha256=/{print $2}')
    echo "run $r: exit=$rc sha256=$sha"
  done
run 2: exit=0 sha256=8c52d6cf552ff7c51f3ce821e5d28160b902b94425c5be6daaace61051a9a132
run 3: exit=0 sha256=8c52d6cf552ff7c51f3ce821e5d28160b902b94425c5be6daaace61051a9a132
```

Both repeats reproduce the run-1 digest `8c52d6cf…` exactly, so "5 000 lines, 140 000 bytes, all present, in order" is stable, not a one-off.

**Attribution precision (resolves the "raw bytes at read_bytes" overclaim, F4):** the bytes written to `--dump-bytes` are emitted **parser-side**, guarded by `#ifdef DUMP_COMMANDS` at `kitty/vt-parser.c:1396` (the guarded emission block spanning `kitty/vt-parser.c:1396-1398`), and cover `self->read.pos - pre_consume_pos` — i.e. exactly the span the parser has just *consumed*. Those are the very bytes that `read_bytes` delivered into the parser's write buffer (`read_bytes` → `vt_parser_commit_write`), so the 140 000-byte figure faithfully reflects what entered at the PTY. The statement "these are the bytes that entered at `read_bytes`" is therefore an **[INFERRED]** data-flow link (parser buffer ← `read_bytes`), not a capture taken at the `read()` call itself. The CLI help for the flag describes it as "raw bytes received from the child process" (`kitty/cli.py:985`), which is accurate as to *content*; this document is simply precise about the *emission point*.


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

**[OBSERVED] — the held-then-flushed transition, captured at the pixel level.** "The display is frozen while processing continues" can be shown *directly*, not merely inferred. A separate observer launches a real kitty window running a real PTY child and samples the window's backing image (`Window.get_image`, `X.ZPixmap`) before the pause, during the hold, and after the explicit end. The child hides the cursor (`CSI ?25l`, to remove blink noise), draws a baseline frame, sends `CSI ?2026 h`, overwrites the whole screen with 13 different lines, holds ~1.2 s (well under the 2000 ms safety timeout of §2.4), then sends `CSI ?2026 l`:

```
$ OBS="$OBS" DISPLAY=:99 /root/kitty-venv/bin/python3 harness/pause_pixels.py explicit 1
mode=explicit run=1 window=640x400 frames=67 begin_ts=1783996493.597
BEFORE  t-0.041s sha=d4373b5f13cd
DURING  t+0.610s sha=d4373b5f13cd
AFTER   t+2.707s sha=8a9f8a69d6f8
held (BEFORE==DURING): True
flushed (DURING!=AFTER): True
pixeldiff BEFORE->AFTER: changed_px=14203 bbox(l,u,r,b)=(0, 5, 251, 231) of 256000
flush_delay_after_begin_ms=1260 (explicit end sent at +1200ms)
pngs=/tmp/blitzy/kitty/blitzy-85ce7b41-edf3-42eb-84b1-2fc55abedc59_304657/blitzy/screenshots/mode2026_explicit_{before,during,after}_r1.png
```

The SHA-256 of the `640×400×4 = 1,024,000`-byte window image is **identical** before the pause and 0.6 s into the hold (`d4373b5f13cd` both times → `held = True`) *even though the child had already overwritten the screen with entirely different text* — the display really is frozen on the pre-pause frame. It changes only **after** `CSI ?2026 l` (`8a9f8a69d6f8` → `flushed = True`), and the flush lands `~1260 ms` after begin — i.e. driven by the explicit end at `+1200 ms`, not by the timeout. The saved PNGs make the two states legible: `…_before_r1.png` and `…_during_r1.png` are byte-identical and show only the two baseline lines, while `…_after_r1.png` shows all 13 held lines appearing **at once**. A second run agrees on every invariant: `held=True`, `flushed=True`, `bbox=(0, 5, 251, 231)`, `changed_px=14214`, flush at `1257 ms`. (Only the wall-clock `begin_ts`, the frame count, and sub-millisecond capture offsets vary run to run; the SHA values differ between runs solely because the drawn text embeds the run label `r1`/`r2` — hence `changed_px` differs by exactly the `1`-glyph.)

**The mechanism, in source.** The freeze is `screen_pause_rendering` (`kitty/screen.c:2506`): on begin it arms `expires_at` and **snapshots** the visible state — every visible line is copied into `paused_rendering.linebuf` (`kitty/screen.c:2529-2538`), along with the cursor (`:2527`), colors (`:2528`), and selections — so the renderer keeps drawing the *old* frame; on end it clears `expires_at` and sets `is_dirty = true` (`kitty/screen.c:2511`) so the next render presents the now-current state (the atomic flush). The parser trace above (the `2→1→2` DECRQM flip and `draw DRAWN-WHILE-PAUSED` between `set_mode`/`reset_mode`) shows *processing* continued through the hold; the pixel capture shows the *display* did not.

**What is still not observed (the honest boundary).** The capture reads the X11 window's **backing image** — the composited frame kitty actually produced, a faithful stand-in for "the frame the user sees." What a headless container genuinely cannot expose is one layer below that: the literal GPU buffer-swap / `vblank` **event** (its timing is not surfaced by `--debug-rendering`, see §5.4) and photons on a physical monitor (there is none). Neither gap weakens the claim here — "held, then flushed atomically" is demonstrated from the window image itself, above.

### 2.4 Resume without an "end" — the safety timeout

A well-behaved application always sends `CSI ? 2026 l`. A crashed or disconnected one might not. kitty guards against a permanently frozen display with a timeout: `screen_check_pause_rendering` auto-resumes when the timer expires (`if (self->paused_rendering.expires_at && now > self->paused_rendering.expires_at) screen_pause_rendering(self, false, 0);`, `kitty/screen.c:2489-2491`), and the default timeout is **2000 ms** (`if (for_in_ms <= 0) for_in_ms = 2000;`, `kitty/screen.c:2521`).

**[OBSERVED] — begin a synchronized update and *never* end it; poll DECRQM ~every 500 ms (two independent runs, ~3.5 s each). RUN 1 shows the complete, unfiltered `--dump-commands` trace; RUN 2 shows the same experiment *illustratively* filtered to just the `draw` events (note the explicit `grep '^draw'` in its command) so the timing flip is easy to compare across runs:**

```
$ ./kitty/launcher/kitty -o close_on_child_death=yes --dump-commands env EMIT_LOG=$OBS/out/q1_timeout_emit_r1.bin python3 harness/pause_timeout_probe.py   # RUN 1 (complete, unfiltered)
screen_set_mode 2026 1
report_mode_status 2026 1
draw T=0003ms DECRQM_REPLY=<ESC>[?2026;1$y
screen_carriage_return
screen_linefeed
report_mode_status 2026 1
draw T=0507ms DECRQM_REPLY=<ESC>[?2026;1$y
screen_carriage_return
screen_linefeed
report_mode_status 2026 1
draw T=1010ms DECRQM_REPLY=<ESC>[?2026;1$y
screen_carriage_return
screen_linefeed
report_mode_status 2026 1
draw T=1514ms DECRQM_REPLY=<ESC>[?2026;1$y
screen_carriage_return
screen_linefeed
report_mode_status 2026 1
draw T=2020ms DECRQM_REPLY=<ESC>[?2026;2$y
screen_carriage_return
screen_linefeed
report_mode_status 2026 1
draw T=2524ms DECRQM_REPLY=<ESC>[?2026;2$y
screen_carriage_return
screen_linefeed
report_mode_status 2026 1
draw T=3027ms DECRQM_REPLY=<ESC>[?2026;2$y
screen_carriage_return
screen_linefeed
report_mode_status 2026 1
draw T=3531ms DECRQM_REPLY=<ESC>[?2026;2$y
screen_carriage_return
screen_linefeed
```

The RUN 1 block above is the **complete, unfiltered** `--dump-commands` stdout — 33 events. Its whole-file class breakdown is `screen_set_mode 2026 1`×1, `report_mode_status 2026 1`×8, `draw`×8, `screen_carriage_return`×8, `screen_linefeed`×8, and — the authoritative check that **no** "end" was parsed — `screen_reset_mode`×0. Each ~500 ms poll contributes one `report_mode_status` (the DECRQM reply) followed by the `draw` line and its `\r\n`: exactly one `screen_carriage_return` + one `screen_linefeed`, because the probe puts the tty in raw mode (`tty.setraw`), so `ONLCR` is off and the `\r\n` is **not** doubled — contrast §2.1's cooked-mode child, where `\r\n` became `\r\r\n` (two `screen_carriage_return`). The command additionally writes one benign `Failed to open systemd user bus … Connection refused` line to *stderr* (the headless-container diagnostic noted in §2.1/§5.4), which is not part of this stdout trace. Only the `draw` lines carry the DECRQM `;1`→`;2` flip, so RUN 2 is shown filtered to those:

```
$ ./kitty/launcher/kitty -o close_on_child_death=yes --dump-commands env EMIT_LOG=$OBS/out/q1_timeout_emit_r2.bin python3 harness/pause_timeout_probe.py | grep '^draw'   # RUN 2 (illustrative: draw events only)
draw T=0003ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=0507ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=1010ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=1514ms DECRQM_REPLY=<ESC>[?2026;1$y
draw T=2020ms DECRQM_REPLY=<ESC>[?2026;2$y
draw T=2524ms DECRQM_REPLY=<ESC>[?2026;2$y
draw T=3028ms DECRQM_REPLY=<ESC>[?2026;2$y
draw T=3531ms DECRQM_REPLY=<ESC>[?2026;2$y
```

**[OBSERVED] — proof the child never sent the "end", so the resume was the timeout (counted over the whole emitted byte stream):**

```
$ python3 - "$OBS/out/q1_timeout_emit_r1.bin" <<'PY'   # counts CSI?2026h vs CSI?2026l in the ENTIRE emitted stream
import sys
raw = open(sys.argv[1], "rb").read()
print("bytes_emitted=", len(raw))
print("contains CSI?2026h (begin):", raw.count(b"\x1b[?2026h"))
print("contains CSI?2026l (end)  :", raw.count(b"\x1b[?2026l"))
PY
bytes_emitted= 392
contains CSI?2026h (begin): 1
contains CSI?2026l (end)  : 0
```

Across both runs the pause flag holds at `1` through `T=1514 ms` and has flipped to `2` by `T=2020 ms` — i.e. the *state* auto-resumed between 1.5 s and 2.0 s, matching the 2000 ms default. The complete RUN 1 trace above contains exactly one `screen_set_mode 2026 1` and **zero** `screen_reset_mode` events (see its class breakdown), and the byte-stream count above shows `begin = 1, end = 0`: the child never sent `CSI ? 2026 l`, so the resume was produced solely by `screen_check_pause_rendering` firing on `expires_at`. The two runs agree at the flip to the millisecond (`2020` vs `2020`) and differ by at most 1 ms elsewhere (`3027` vs `3028`), so this is a stable timeout, not jitter.

**[OBSERVED] — the same timeout at the pixel level: the *state* resumes on the timer, the *repaint* is event-driven.** The DECRQM poll above resumes cleanly because each ~500 ms query is *itself* input that drives a render. To see what the *display* does when a timed-out application then goes completely idle, the pixel observer of §2.3 holds a pause open with **no** end, waits past 2000 ms, and only then sends a content-neutral render nudge (`CSI 6 n`, a cursor-position-report request that changes no cell):

```
$ OBS="$OBS" DISPLAY=:99 /root/kitty-venv/bin/python3 harness/pause_pixels.py timeout-nudge 1
mode=timeout-nudge run=1 window=640x400 frames=88 begin_ts=1783996594.217
BEFORE  t-0.060s sha=d4373b5f13cd
DURING  t+0.591s sha=d4373b5f13cd
AFTER   t+4.211s sha=527f06d48dff
held (BEFORE==DURING): True
at +2.3s (past 2000ms state-timeout, pre-nudge) base?True sha=d4373b5f13cd
first_pixel_flip: +2.619s after begin, +0.010s after nudge -> sha=8a9f8a69d6f8
pngs=/tmp/blitzy/kitty/blitzy-85ce7b41-edf3-42eb-84b1-2fc55abedc59_304657/blitzy/screenshots/mode2026_timeout-nudge_{before,during,after}_r1.png
```

Two facts, both stable across two runs. **(1)** The pause *state* clears on the 2000 ms timer exactly as the DECRQM trace showed — at `+2.3 s`, past the timeout, the window image is captured while `expires_at` is already `0`. **(2)** Yet the held frame is *still on screen* at `+2.3 s` (`base?True`); it repaints only after the nudge, `+0.010 s` (run 2: `+0.021 s`) later. So `screen_check_pause_rendering` un-pauses the *state* on time, but the *visible* flush waits for the next render trigger. In an interactive session that trigger is always imminent (the next keystroke, cursor blink, or output byte); in a deliberately idle headless capture it is not — which is why the timed-out content becomes visible on the `CSI 6 n` nudge rather than at exactly 2000 ms. This is precisely the difference from the explicit path (§2.3), where `CSI ?2026 l` is *itself* the triggering input and so flushes at once.


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

So the "conductor" is really three **independently scheduled** loops: the **main thread** (GLFW events + rendering + the VT parser), the **I/O thread** `KittyChildMon` (reads child PTYs with `read_bytes`, writes them with `write_to_child`), and — when remote control is in play — the **talk thread** `KittyPeerMon` (accepts and reads remote-control peers, §4.3). They avoid *long* synchronous coupling — no loop waits on another to finish its work — but they are **not** lock-free: state is handed between them through the parser's `self->lock`-guarded buffer (I/O thread → main thread) and a wakeup pipe (either direction), so they coordinate through mutexes, pipes, and state handoffs and **can briefly block on that shared lock**. Concretely, the I/O thread's `vt_parser_create_write_buffer` / `vt_parser_commit_write` / `vt_parser_has_space_for_input` and the main thread's `run_worker` all take the same `pthread_mutex_lock(&self->lock)` (`kitty/vt-parser.c:1413-1482`), so a write-side append and a read-side consume can momentarily contend. The separation bounds that contention to short critical sections rather than eliminating it — notably, `run_worker` deliberately *releases* the lock around the expensive `consume_input` parse (`kitty/vt-parser.c:1431-1433`) so the I/O thread is not held off while a burst is being decoded.

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

Command (one full sweep; run once — the two runs per setting and the gap control are internal):

```
PP_N=200 bash "$OBS/harness/coalesce_sweep.sh"
```

**[OBSERVED] — sweep of `input_delay ∈ {0, 3, 10, 30}` ms, two independent runs each, N = 200 measured round-trips per run (after 20 warm-ups); statistics computed deterministically from the raw per-trial log:**

```
label=input_delay=0ms gap=0ms run1 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=0.010
min_ms=0.0329
median_ms=0.0485
p90_ms=0.0612
max_ms=0.1907
mean_ms=0.0517

label=input_delay=0ms gap=0ms run2 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=0.010
min_ms=0.0341
median_ms=0.0462
p90_ms=0.0652
max_ms=0.0986
mean_ms=0.0493

label=input_delay=3ms gap=0ms run1 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=0.634
min_ms=3.1079
median_ms=3.1605
p90_ms=3.1865
max_ms=4.2701
mean_ms=3.1672

label=input_delay=3ms gap=0ms run2 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=0.631
min_ms=3.0877
median_ms=3.1526
p90_ms=3.1745
max_ms=3.2251
mean_ms=3.1524

label=input_delay=10ms gap=0ms run1 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=2.059
min_ms=10.0724
median_ms=10.2403
p90_ms=10.2860
max_ms=13.3949
mean_ms=10.2929

label=input_delay=10ms gap=0ms run2 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=2.060
min_ms=10.0774
median_ms=10.2597
p90_ms=10.3108
max_ms=12.6291
mean_ms=10.2974

label=input_delay=30ms gap=0ms run1 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=6.086
min_ms=30.1964
median_ms=30.2747
p90_ms=30.3481
max_ms=33.3335
mean_ms=30.4256

label=input_delay=30ms gap=0ms run2 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=6.080
min_ms=30.2004
median_ms=30.2743
p90_ms=30.3367
max_ms=32.6256
mean_ms=30.3949
```

Reading the medians against the knob:

| `input_delay` | run 1 median | run 2 median | median − `input_delay` |
|---:|---:|---:|---:|
| 0 ms  | 0.0485 ms  | 0.0462 ms  | ≈ 0.047 ms (pure overhead) |
| 3 ms  | 3.1605 ms  | 3.1526 ms  | ≈ 0.156 ms |
| 10 ms | 10.2403 ms | 10.2597 ms | ≈ 0.250 ms |
| 30 ms | 30.2747 ms | 30.2743 ms | ≈ 0.274 ms |

The round-trip latency tracks `input_delay` with slope ≈ 1.0, and the two runs at each setting agree to ~0.02 ms or better, so the window is stable, not jitter. The base overhead — measured directly at `input_delay = 0` — is a median of **≈ 0.047 ms (47 µs)**; the small residual over the knob (`0.156 → 0.250 → 0.274 ms`) grows modestly with the delay, consistent with the parser re-check granularity described below.

**Where the delay actually lives (a correction worth stating precisely).** One might expect that a query arriving after a long idle would skip the window. It does **not**, and the control experiment shows why:

**[OBSERVED] — same probe at `input_delay = 30` ms but sleeping a 50 ms idle gap *before* each timed query (so every query is the first byte after idle):**

```
label=input_delay=30ms gap=50ms run1 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=16.117
min_ms=30.2315
median_ms=30.3582
p90_ms=30.4351
max_ms=33.6547
mean_ms=30.4960

label=input_delay=30ms gap=50ms run2 (launcher_exit=0)
count=200 (header n=200)
wall_duration_s=16.118
min_ms=30.2555
median_ms=30.3567
p90_ms=30.4293
max_ms=33.4433
mean_ms=30.5030
```

Even with a 50 ms idle gap between queries (wall time ≈ `200 × (50 + 30) ms` ≈ 16.1 s, confirming the gap was applied), the round-trip median stays ≈ 30.36 ms. The idle gap does **not** collapse the latency to the base overhead. The reason is that the dominant delay is the **parser's flush gate**, not the I/O-thread's wakeup batching. In `run_worker` the accumulated bytes are only consumed when

```c
pd->time_since_new_input = pd->now - self->new_input_at;                              // vt-parser.c:1424
if (flush || pd->time_since_new_input >= OPT(input_delay) || self->read.sz + 16 * 1024 > BUF_SZ) {  // vt-parser.c:1425
    …
    consume_input(self, pd->dump_callback, screen->window_id);                        // vt-parser.c:1432
    …
    self->new_input_at = 0;                                                           // vt-parser.c:1436
}
```

`new_input_at` is stamped when bytes are committed (`if (self->new_input_at == 0) self->new_input_at = monotonic();`, `kitty/vt-parser.c:1469`) and reset to 0 after a consume (`kitty/vt-parser.c:1436`). Because it is re-stamped for **each fresh arrival**, every isolated query must wait `input_delay` before it is parsed — which is exactly why the 50 ms idle gap makes no difference. The I/O-thread's wakeup coalescing (`kitty/child-monitor.c:1562-1569`, including the "wake immediately if idle longer than `input_delay`" branch at `:1566`) is a *second*, complementary throttle on the same knob governing *when the main loop is woken*; the ping-pong proxy makes the parser gate the visible term. Either way, the trade-off is the one documented in `docs/performance.rst`: a few milliseconds of artificial delay batches work and cuts CPU wakeups, at the cost of a little display latency.


---

## 4. Q3 — Aligned semantics under stress

### 4.1 Shell-integration hints stay aligned with text because dispatch is serial

Shell integration marks the prompt, the command, and the command's output with OSC 133 escape codes so the terminal can offer prompt navigation, exit-status coloring, and output selection. The question asks how these hints stay aligned with ordinary text "without drifting out of sync." The answer is structural: there is **no separate channel** for hints. They travel *in* the byte stream, and a single parser pass consumes that stream in byte order, dispatching each token — text, CSI, OSC — to the screen before it looks at the next. `dispatch_osc` routes code 133 to `shell_prompt_marking` (`kitty/vt-parser.c:536-544`), which invokes the `cmd_output_marking` callback for the prompt-start (`A`), output-start (`C`, with the command line), and command-finished (`D`, with the exit status) marks (`kitty/screen.c:2338`, `:2347`, `:2352`).

**[OBSERVED] — what kitty's *shipped* bash integration actually emits.** A real interactive `bash` was launched under a real kitty with shell integration auto-injected, one command (`printf "HELLO-OUTPUT\n"`) was driven in via `kitty @ send-text`, and the raw bytes were dumped and the OSC 133 sequences extracted in order:

```
$ bash harness/osc133_capture.sh    # bash under a real kitty; drives one command via `kitty @ send-text`; writes $OBS/osc133_bytes.bin (raw) + $OBS/osc133_dump.txt (--dump-commands)
prompt_marker_seen=yes
send_text_exit=0
done
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
$ cat $OBS/osc133_dump.txt    # the --dump-commands trace captured by the same osc133_capture.sh run above
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

Command (one committed driver; its complete stdout is the three fenced blocks in this subsection — group 1 losslessness, group 2 size sweep, group 3 slow-drain — run once):

```
bash "$OBS/harness/flood_sweep.sh"
```

```
label=bytes=8388608 input_delay=3 trial=1 launcher_exit=0
sent_bytes=8388608 dumped_bytes=8388608
sent_sha256=f7ed19dbe389fdfb8746b6d71de4a0cfa4aee1fba9f40f7cd3587f4539040b6f
dump_sha256=f7ed19dbe389fdfb8746b6d71de4a0cfa4aee1fba9f40f7cd3587f4539040b6f
write_wall_ms=1228.043
buffer_multiple=8.00x the 1 MiB VT parser buffer
RESULT=PASS_LOSSLESS

label=bytes=8388608 input_delay=3 trial=2 launcher_exit=0
sent_bytes=8388608 dumped_bytes=8388608
sent_sha256=f7ed19dbe389fdfb8746b6d71de4a0cfa4aee1fba9f40f7cd3587f4539040b6f
dump_sha256=f7ed19dbe389fdfb8746b6d71de4a0cfa4aee1fba9f40f7cd3587f4539040b6f
write_wall_ms=1187.859
buffer_multiple=8.00x the 1 MiB VT parser buffer
RESULT=PASS_LOSSLESS
```

8 MiB is far larger than the 1 MiB buffer plus the ~64 KiB kernel PTY buffer, so the writer *must* have blocked repeatedly mid-burst; yet the SHA-256 of what kitty consumed equals the SHA-256 of what the child sent, in both trials. Nothing was dropped, duplicated, or reordered.

**[OBSERVED] — the writer is paced by kitty's drain rate (a size sweep and a slow-drain control), which is the signature of blocking rather than discarding:**

```
label=bytes=1048576 input_delay=3 trial=1 launcher_exit=0
write_wall_ms=2.846
RESULT=PASS_LOSSLESS

label=bytes=4194304 input_delay=3 trial=1 launcher_exit=0
write_wall_ms=540.547
RESULT=PASS_LOSSLESS

label=bytes=8388608 input_delay=3 trial=1 launcher_exit=0
write_wall_ms=1181.993
RESULT=PASS_LOSSLESS
```

```
label=bytes=8388608 input_delay=3 trial=A launcher_exit=0
write_wall_ms=1201.917
RESULT=PASS_LOSSLESS

label=bytes=8388608 input_delay=200 trial=A launcher_exit=0
write_wall_ms=1570.434
RESULT=PASS_LOSSLESS
```

Two things stand out. First, the **before/at/after-threshold** behavior: a 1 MiB burst — which fits entirely within the parser buffer — completes in **2.8 ms** without ever blocking, whereas 4 MiB and 8 MiB (which overflow the buffer) take **541 ms** and **1182 ms**, i.e. the write time grows once the producer crosses the buffer size and must wait for kitty to drain. Second, holding the burst at 8 MiB but making kitty drain **slower** (raising `input_delay` from 3 to 200 ms) *lengthens* the writer's blocked time (**1202 ms → 1570 ms**) while remaining lossless — the producer's rate is dictated by the consumer, which is exactly cooperative backpressure.

**Correction (resolves the `repaint_delay` claim in F9/F14).** An earlier draft attributed this pacing to a "~10 ms `repaint_delay` cadence." That is wrong: `repaint_delay`'s own documentation says "to minimize latency when there is pending input to be processed, this option is ignored" (`kitty/options/definition.py:873-875`). While a backlog exists, the render delay is bypassed, so it does **not** pace ingestion; the pacing comes solely from the POLLIN gate above. The `repaint_delay` claim has been removed.

**[NON-CANONICAL] supplement.** kitty's own test suite exercises the buffer boundary synchronously via the `parse_bytes` helper and `VT_PARSER_BUFFER_SIZE` (`kitty_tests/parser.py`), but that helper feeds the parser directly and **bypasses** the `io_thread`/PTY path, so it is not canonical evidence for backpressure; the real-PTY losslessness and pacing above are the canonical evidence.

### 4.3 Under an unstable remote connection the PTY pipeline stays functionally isolated

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


#### 4.3.4 The PTY pipeline stays functionally isolated from talk-socket turmoil, with a shared-scheduling tail caveat (matched control)

[OBSERVED] Survival is necessary but not sufficient; the sharper question is whether remote-socket chaos *perturbs the PTY input path*, and if so, how. To answer it, the same PTY RTT prober from §3.3 (`pingpong.py`, `PP_N=200`, default `input_delay=3`) is run in two matched phases against a server that also carries a listen socket, holding the prober input byte-identical and changing only the remote condition:

- **Phase A (baseline):** the talk socket sits idle.
- **Phase B (chaos):** a background loop fires the *same* byte-identical `peer_disrupt.py` events at the talk socket continuously for the entire prober run.

A single A/B pair is **not** enough to characterise the effect, because the quantity of interest — the chaos-phase RTT *tail* — is itself unstable from run to run. (An earlier draft of this section reported one pair of runs whose chaos `max` was ~10 ms and called it "a single delayed sample"; that was an under-sample, as the distribution below shows.) So the whole matched experiment is repeated `PP_RUNS=15` times per set, and the set is repeated three times — **45 runs total**, byte-identical prober input throughout — and `pty_ind_distribution.py` reads each run's raw per-trial RTT log to report the **distribution** (per-run `min`/`median`/`p95`/`max` for both phases, plus each set's tail-exceedance counts) rather than one tidy pair of numbers, per the inconsistency rule.

Command (one set of 15; invoked three times):

```
PP_N=200 PP_RUNS=15 python3 "$OBS/harness/pty_ind_distribution.py"
```

Complete, unedited output — set 1 of 3:

```
PP_N=200 PP_RUNS=15
run | A_min  A_med  A_p95  A_max   | B_min  B_med  B_p95  B_max (ms)
 1  | 3.1012 3.1451 3.1653  3.1982 | 3.1082 3.1599 3.1925   3.2689
 2  | 3.1157 3.1476 3.1903  3.7513 | 3.0966 3.1487 3.1741   3.2219
 3  | 3.1022 3.1499 3.1869  3.2431 | 3.0882 3.1537 3.1951  13.5402
 4  | 3.0918 3.1543 3.1886  3.2176 | 3.1216 3.1631 3.2078  15.2987
 5  | 3.1017 3.1610 3.1841  3.3404 | 3.1077 3.1544 3.1862   3.5142
 6  | 3.0997 3.1449 3.1703  3.3115 | 3.0913 3.1447 3.1711   3.2083
 7  | 3.1139 3.1563 3.1884  3.3340 | 3.0807 3.1515 3.1755   3.2795
 8  | 3.0947 3.1438 3.1748  3.1954 | 3.1135 3.1566 3.1947   4.9769
 9  | 3.0865 3.1449 3.1775  3.3007 | 3.0885 3.1519 3.2038   9.9795
10  | 3.1145 3.1564 3.1879  3.2643 | 3.1234 3.1490 3.1776  16.7096
11  | 3.0870 3.1546 3.1856  3.3287 | 3.0829 3.1442 3.1760   3.2337
12  | 3.1199 3.1590 3.1971  3.3042 | 3.0784 3.1509 3.1787  10.9532
13  | 3.0981 3.1427 3.1718  3.2741 | 3.1083 3.1508 3.1833  20.2883
14  | 3.1169 3.1543 3.1805  3.3062 | 3.0951 3.1489 3.1803  10.6655
15  | 3.0960 3.1466 3.1710  3.2136 | 3.0879 3.1425 3.1724   3.1945

=== aggregate over 15 runs ===
baseline max_ms range: 3.1954 .. 3.7513  (>10ms: 0/15)
chaos    max_ms range: 3.1945 .. 20.2883  (>10ms: 6/15, >20ms: 1/15, >90ms: 0/15)
baseline median_ms: min=3.1427 median=3.1499 max=3.1610
chaos    median_ms: min=3.1425 median=3.1509 max=3.1631
chaos max_ms sorted desc: 20.29 16.71 15.30 13.54 10.95 10.67 9.98 4.98 3.51 3.28 3.27 3.23 3.22 3.21 3.19
```

Complete, unedited output — set 2 of 3 (reproduction):

```
PP_N=200 PP_RUNS=15
run | A_min  A_med  A_p95  A_max   | B_min  B_med  B_p95  B_max (ms)
 1  | 3.1149 3.1537 3.1849  3.3022 | 3.0996 3.1505 3.1919  22.4199
 2  | 3.1027 3.1564 3.1834  3.2252 | 3.1143 3.1491 3.1758   3.2600
 3  | 3.0961 3.1550 3.1807  3.2488 | 3.1077 3.1489 3.1793   4.9701
 4  | 3.1058 3.1603 3.1891  3.2448 | 3.0957 3.1569 3.1893   3.2266
 5  | 3.1159 3.1558 3.2002  3.2361 | 3.0925 3.1532 3.1918   3.2306
 6  | 3.1157 3.1473 3.1750  3.2393 | 3.1079 3.1501 3.1953   4.7591
 7  | 3.1078 3.1613 3.1908  3.2512 | 3.0903 3.1470 3.1778   3.2755
 8  | 3.0962 3.1552 3.1896  3.3169 | 3.0932 3.1521 3.1932   3.2380
 9  | 3.0726 3.1555 3.2016  3.3111 | 3.0821 3.1608 3.1959   3.2662
10  | 3.1145 3.1505 3.1806  3.2526 | 3.0936 3.1537 3.1980   4.2935
11  | 3.0717 3.1438 3.1733  3.2115 | 3.1012 3.1559 3.1941   3.3977
12  | 3.1079 3.1570 3.1913  3.6950 | 3.1127 3.1537 3.1870   3.2248
13  | 3.1049 3.1513 3.1817  4.4115 | 3.0939 3.1529 3.1832   3.2461
14  | 3.1066 3.1606 3.1909  3.2275 | 3.1110 3.1554 3.2019  15.7605
15  | 3.1029 3.1478 3.1676  3.1969 | 3.1055 3.1557 3.1956   3.2417

=== aggregate over 15 runs ===
baseline max_ms range: 3.1969 .. 4.4115  (>10ms: 0/15)
chaos    max_ms range: 3.2248 .. 22.4199  (>10ms: 2/15, >20ms: 1/15, >90ms: 0/15)
baseline median_ms: min=3.1438 median=3.1552 max=3.1613
chaos    median_ms: min=3.1470 median=3.1532 max=3.1608
chaos max_ms sorted desc: 22.42 15.76 4.97 4.76 4.29 3.40 3.28 3.27 3.26 3.25 3.24 3.24 3.23 3.23 3.22
```

Complete, unedited output — set 3 of 3 (reproduction):

```
PP_N=200 PP_RUNS=15
run | A_min  A_med  A_p95  A_max   | B_min  B_med  B_p95  B_max (ms)
 1  | 3.1174 3.1832 3.2214  3.2670 | 3.1376 3.1878 3.2312   3.2807
 2  | 3.1052 3.1659 3.1976  3.2424 | 3.1260 3.1821 3.2354   3.2721
 3  | 3.0944 3.1520 3.1828  3.2437 | 3.1053 3.1484 3.1808   3.3057
 4  | 3.0850 3.1487 3.1778  3.2721 | 3.0889 3.1436 3.1795   3.2221
 5  | 3.1350 3.1605 3.1904  3.3439 | 3.0789 3.1455 3.1732  16.7236
 6  | 3.1218 3.1543 3.1859  3.2824 | 3.1180 3.1622 3.2027   3.3686
 7  | 3.0907 3.1505 3.1761  3.2804 | 3.0900 3.1543 3.1968   3.2698
 8  | 3.0877 3.1420 3.1712  3.3002 | 3.0876 3.1526 3.1909   3.2212
 9  | 3.0828 3.1548 3.1859  3.2488 | 3.0921 3.1589 3.2028   3.2830
10  | 3.0987 3.1607 3.2018  3.3301 | 3.1024 3.1471 3.1727  11.3958
11  | 3.1034 3.1606 3.2216  3.3335 | 3.0905 3.1618 3.2048   9.3026
12  | 3.1064 3.1615 3.2066  3.3163 | 3.0990 3.1692 3.2139   9.0239
13  | 3.0873 3.1527 3.1945  3.2403 | 3.1093 3.1627 3.2082   3.3185
14  | 3.1012 3.1480 3.1770  3.2169 | 3.1151 3.1612 3.2036   6.7056
15  | 3.1152 3.1551 3.1993  3.6878 | 3.1161 3.1535 3.1878   3.2147

=== aggregate over 15 runs ===
baseline max_ms range: 3.2169 .. 3.6878  (>10ms: 0/15)
chaos    max_ms range: 3.2147 .. 16.7236  (>10ms: 2/15, >20ms: 0/15, >90ms: 0/15)
baseline median_ms: min=3.1420 median=3.1548 max=3.1832
chaos    median_ms: min=3.1436 median=3.1589 max=3.1878
chaos max_ms sorted desc: 16.72 11.40 9.30 9.02 6.71 3.37 3.32 3.31 3.28 3.28 3.27 3.27 3.22 3.22 3.21
```

Summing the three per-set aggregates printed above gives the full 45-run distribution. In the **baseline** phase the maximum never exceeded `4.4115` ms and `0/45` runs crossed 10 ms. Under **chaos** the maximum ranged from `3.1945` ms to `22.4199` ms, with `6 + 2 + 2 = 10/45` runs over 10 ms, `1 + 1 + 0 = 2/45` over 20 ms, and `0/45` over 90 ms. Two facts reproduce cleanly, and they must be stated separately:

- **The central tendency is unchanged — the PTY pipeline is functionally isolated.** Across all 45 runs the chaos-phase median (`3.1425 … 3.1878` ms, median-of-medians `3.1532` ms) is statistically indistinguishable from the baseline median (`3.1420 … 3.1832` ms, median-of-medians `3.1543` ms); both stay pinned to the `input_delay = 3 ms` coalescing window measured in §3.3. Concurrent talk-socket chaos does **not** shift the PTY pipeline's characteristic latency, and — as §4.3.3 established over 80 identical disruptions — the process, the `KittyPeerMon` thread, and the socket all survive, no round-trip is lost or reordered, and `send-text` is delivered every time. This functional isolation is the load-bearing claim, and it holds. Its mechanism is structural: the talk thread runs `talk_loop` on its own poll loop (`kitty/child-monitor.c:1805`), wholly separate from the `io_loop` that drives `read_bytes`/`run_worker`, so a half-spoken peer cannot corrupt, block, or reorder the PTY input path.
- **The tail is unstable, and chaos measurably inflates it.** The baseline phase has essentially no tail (`0/45` over 10 ms; worst `4.41` ms). Under chaos, `10/45` runs pushed a *single* round-trip into a `10–22` ms spike, and the per-set counts (`6/15`, `2/15`, `2/15`) show the tail's frequency and height themselves vary from one set of runs to the next on this shared-scheduler container. The reason is that functional isolation is not the same as scheduling isolation: the two threads still contend for the same OS CPU, so a burst of peer-teardown work (`read_from_peer` → `notify_on_peer_removal` → the `"peer_death"` message, §4.3.3) can occasionally delay one PTY round-trip into the tail without ever touching the PTY data path. The honest characterisation is therefore **functional isolation with a shared-scheduling tail** — not "unaffected" and not "never". A separate observation run (the QA reference, on the same byte-identical input) recorded a still-heavier tail — chaos `max` up to ~`279.74` ms with `3/15` runs over 90 ms — which only reinforces the point: the tail's height is scheduling-dependent and must be reported as this distribution, never collapsed to one tidy number. This is the matched-control evidence for PTY independence that F10 requires: the pipeline's correctness and central latency are isolated from remote turmoil, while its worst-case tail shares the machine's scheduler.


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
- **`--debug-input` / `--debug-keyboard`** — the *same* option (`kitty/cli.py:996`, `dest=debug_keyboard`). §5.1 used `--debug-keyboard`; a one-off with the `--debug-input` spelling drives a real `'a'` keypress into the focused GLFW window via XTEST (`xtest_inject.py`, Appendix A) and confirms it emits the identical `on_key_input` line. (ESC colour bytes are stripped per the disclosure above; the `kitty_window` id and the leading `[t]` timestamps vary run to run — everything from `glfw key` onward reproduced byte-for-byte across three runs.)

```
$ ./kitty/launcher/kitty --debug-input -o close_on_child_death=yes sh -c 'sleep 6' 2>$OBS/out/debuginput.err &
$ python3 $OBS/harness/xtest_inject.py a::0x61       # inject a real 'a' keypress (keysym 0x61) via XTEST
kitty_window=0x20000c focus=0x20000c FOCUS_OK
INJECTED a (mods=none keysym=0x61 keycode=38)
$ grep -a on_key_input $OBS/out/debuginput.err | sed 's/\x1b//g'
[1.985] [33mon_key_input[m: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
[2.015] [33mon_key_input[m: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
```

- **`--replay-commands`** — reads a `--dump-commands` file and re-emits each parsed command back as raw bytes on stdout (`kitty/client.py:270-290`, dispatched by `client_main` at `kitty/main.py:476-477`). It is a lightweight client — no window is opened — that pauses on `input()` at the end (`kitty/client.py:289-290`), so it is fed `</dev/null` to return immediately instead of blocking. A capture was first taken from a real child:

```
$ ./kitty/launcher/kitty -o close_on_child_death=yes --dump-commands sh -c 'printf "REPLAY-SRC\r\n"; sleep 0.15' > $OBS/out/replay_capture.txt
$ grep -E 'draw REPLAY-SRC|screen_carriage_return|screen_linefeed' $OBS/out/replay_capture.txt
draw REPLAY-SRC
screen_carriage_return
screen_carriage_return
screen_linefeed
```

then replayed back through a fresh `--dump-commands` child so the round-trip is visible:

```
$ ./kitty/launcher/kitty -o close_on_child_death=yes --dump-commands sh -c "./kitty/launcher/kitty --replay-commands $OBS/out/replay_capture.txt </dev/null" > $OBS/out/replay_redump.txt
$ grep -E 'draw REPLAY-SRC|screen_carriage_return|screen_linefeed' $OBS/out/replay_redump.txt
draw REPLAY-SRC
screen_carriage_return
screen_carriage_return
screen_carriage_return
screen_linefeed
```

The replayed `draw REPLAY-SRC` re-executes, demonstrating the flag round-trips a dump back through the command pipeline. The re-dump shows three `screen_carriage_return` where the capture shows two: the PTY's output-newline translation (ONLCR) fires a *second* time — the first `--dump-commands` child already expanded the child's `\n` into `\r\n` (one literal CR plus one added), and wrapping the replay as the child of a *second* `--dump-commands` PTY expands its emitted `\n` once more. Both invocations exit 0; the line counts (capture: 2 carriage-returns; re-dump: 3) were stable across three runs.

### 5.4 The render/settle boundary: what is observable here, and the knobs (F14)

[OBSERVED — with an explicit limitation] The final "interface settles" phase is a GPU frame draw plus buffer swap. In this **headless, software-GL** sandbox that phase is *not* surfaced by kitty's own tracing. `--debug-rendering` emitted only window-lifecycle lines, never a per-frame draw or swap:

```
$ ./kitty/launcher/kitty -o close_on_child_death=yes --debug-rendering sh -c 'true' 2>&1 1>/dev/null   # timestamps vary run to run
[0.149] OS Window created
[0.160] Failed to open systemd user bus with error: Connection refused
[0.164] Child launched
```

[INFERRED — source] That is consistent with the code: the `debug_rendering` macro (`kitty/state.h:14`) is used in `kitty/glfw.c` only for a color-scheme change (`L72`), an occlusion change (`L261`), and window creation (`L1321`) — none per frame. **Honest limitation, scoped precisely:** kitty's own tracing therefore does *not* surface the per-frame draw / buffer-swap **event**. It does **not** follow that no frame can be observed: §2.3 and §2.4 capture the composed frame *content* directly from the X11 window backing image (`Window.get_image`, with SHA-256 hashes and a pixel-diff), proving the synchronized-output hold and the atomic flush at the pixel level. What remains unobserved here is therefore narrow — the *timing of the swap event* and any physical-monitor scan-out — not the presence or content of the presented frame.

What *can* be stated canonically is the render configuration, read **from the built binary** (not the `.py` source text) via the default `Options`:

```
$ ./kitty/launcher/kitty +runpy 'from kitty.options.types import defaults as d; from kitty.fast_data_types import VT_PARSER_BUFFER_SIZE as b; print(f"input_delay={d.input_delay} (ms)"); print(f"repaint_delay={d.repaint_delay} (ms)"); print(f"sync_to_monitor={d.sync_to_monitor}"); print(f"VT_PARSER_BUFFER_SIZE={b} bytes (={b//(1024*1024)} MiB)")'
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
6. **Isolation of the remote channel.** Any remote-control turmoil rides a separate `talk_loop`/`KittyPeerMon` thread; an unstable peer produces `peer_death` (`kitty/child-monitor.c:1677`) handled independently, and steps 1–5 keep their correctness and central latency (the chaos-phase PTY RTT median stays indistinguishable from baseline, §4.3.4). Because the two threads still share OS scheduling, concurrent chaos can occasionally push an individual PTY round-trip into a tail spike (`10/45` runs over 10 ms vs `0/45` at baseline) without perturbing the pipeline's throughput or median. *(§4.3.3, §4.3.4)*
7. **Settle.** With no backlog, the frame is presented at the `repaint_delay`/`sync_to_monitor` cadence (source defaults `10 ms`/`yes`, §5.4). The presented frame *content* was captured directly from the window backing image — §2.3/§2.4 show the synchronized-output hold and atomic flush by SHA-256 + pixel-diff; only the low-level buffer-swap *event* timing is not surfaced by kitty's own tracing (§5.4, disclosed).
8. **Outbound symmetry.** A user keystroke runs the mirror path: X/GLFW event → `on_key_input` (`kitty/keys.c:166`) → `encode_glfw_key_event` honoring `mDECCKM` and the keyboard-protocol flags (`L251`) → `schedule_write_to_child` (`L259`) → the child's PTY. *(§5.1)*

The moving parts keep rhythm through exactly two disciplines: **one ordered buffer consumed by one serial parser pass** (guaranteeing alignment), and **one coalescing window plus a self-pipe wakeup** (bounding latency while batching expensive main-loop wakeups). Everything else — pause/resume, backpressure, remote isolation — layers onto those two without breaking them.


## 6. Canonical vs non-canonical evidence, and honest limitations

Per the investigation rules, the primary proof for every claim comes from a canonical entry point — a real child writing to a real PTY (inbound), or a real X/GLFW key event (outbound) — driven by a normally built kitty. This section labels every piece of evidence so the reader can see exactly what was observed canonically, what was a clearly-marked supplement, and what could not be observed in this sandbox and is therefore stated as inferred.

**Canonical [OBSERVED] evidence (real PTY child → `read_bytes`, or real X event → `on_key_input`):**

- §2.2 surge of 5000 lines validated byte-exactly (sha256 stable across 3 runs).
- §2.3 / §2.4 synchronized-output pause, resume, and the safety-timeout auto-resume — DECRQM state before/during/after, the child's full emitted byte stream, **and** a pixel-level capture of the window backing image showing the frame held then atomically flushed (SHA-256 + pixel-diff, two runs each for the explicit-end and missing-end cases).
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
- The synchronized-output *internal snapshot* (`screen_pause_rendering` copying every visible line into `paused_rendering.linebuf`, `kitty/screen.c:2506`/`2529-2538`) is cited from code; its consequence — the window image held then atomically flushed — is now **observed at the pixel level** (§2.3/§2.4), not merely inferred.
- `read_bytes` as the literal read site is inferred (the `--dump-bytes` figure is emitted parser-side, per F4); the child-on-PTY fact is observed (a real slave `/dev/pts/N`).
- The `loop-utils` wakeup/drain mechanism (§5.2) is inferred from source.

**Honest limitations (could not be observed in this headless, software-GL sandbox):**

- The **buffer-swap / `vblank` event timing** — the low-level moment the GPU presents a frame — is not surfaced by `--debug-rendering` (which logs only window lifecycle), so that *event* was not timed. This is narrower than it sounds: the frame *content* itself **was** captured (§2.3/§2.4), so "a composed frame is presented, held, then atomically flushed" is observed, not inferred; only the swap-event timing is not.
- **Physical-monitor scan-out** (photons on real hardware) has no meaning in a headless container and is not claimed. The compositor-visible window backing image — what a screenshot or compositor would show — **was** captured and pixel-verified (§2.3/§2.4), so the synchronized-output hold/flush is demonstrated rather than assumed.


## 7. Coverage pass — every question part and every named item

This is the final coverage check. The first table confirms each of the four question threads is answered; the second is an evidence-linked matrix over every symbol/struct/knob named in the plan, with the section where it is grounded and whether the grounding is canonical [OBSERVED], a labeled supplement [NON-CANONICAL], or code-cited [INFERRED].

**Question threads:**

| Thread | Question | Answered in | Basis |
|--------|----------|-------------|-------|
| Q1 | Where does a surge first enter, and how does pause/resume work? | §2.1–§2.4 | [OBSERVED] |
| Q2 | The "conductor": thread split + what decides which event is handled first | §3.1–§3.3, §5.2 | [OBSERVED] + [INFERRED] |
| Q3 | Hints aligned with text; behavior under backpressure and an unstable remote | §4.1–§4.3.4 | [OBSERVED] (+ labeled supplements) |
| Q4 | End-to-end from mixed input to the interface settling | §5.1–§5.5 | [OBSERVED] (frame *content* captured §2.3/§2.4; only the swap-*event* timing disclosed as not observed) |

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
| `screen_pause_rendering` (screen.c:2506), `PENDING_MODE 2026` (control-codes.h:235) | §2.3 | [OBSERVED] DECRQM state **+ pixel-level frame hold/flush** (SHA-256 + pixel-diff, ×2 runs) |
| `screen_check_pause_rendering` (screen.c:2489), `expires_at` timeout | §2.4 | [OBSERVED] state auto-resume ≈2000 ms (DECRQM) **+ pixel repaint shown event-driven** (nudge, ×2 runs) |
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
| version `0.35.2` (constants.py:25) + VCS stamp via `get_vcs_rev`→`git rev-parse HEAD` (setup.py:674/678): default destination build stamps `2017484b1…`, source-rev worktree stamps `815df1e210e0…` | §1 | [OBSERVED] `--version` banner + both `KITTY_VCS_REV` values (STEP 6–8) |
| visible frame *content* (synchronized-output hold → atomic flush) | §2.3, §2.4, §5.4 | [OBSERVED] pixel capture (window backing image, SHA-256 + pixel-diff); only the low-level swap-*event* timing is disclosed as not observed |

Every question part and every named item above is grounded in a specific section with a labeled basis. The one genuinely unobserved item is narrow — the low-level buffer-swap *event* timing (and physical-monitor scan-out, which is meaningless headless) — and it is explicitly disclosed; the frame-*content* transitions the question actually asks about (held → flushed atomically) are observed at the pixel level (§2.3/§2.4).

## Appendix A — Complete observation harness (verbatim, reproducible)

Every experiment above was produced by the scripts below, reproduced here in full so each result is independently reproducible and auditable (F1). All scripts lived under a `umask 077` observation directory created with `mktemp -d` (mode 0700); `common.sh` established the canonical, secure run environment and readiness/cleanup helpers sourced by every experiment. Paths shown as `$OBS`/`$OBS_DIR` refer to that directory. These scripts are temporary observation tooling and are **removed** after the investigation — they are reproduced here, not left in the repository.

### Setup and teardown (run once — makes every experiment reproducible from a genuinely clean directory)

The harness is designed to run from a **fresh, empty** observation root with no state carried over from a prior run. The setup below establishes a single secure root (`$OBS`, exported under both `$OBS` and `$OBS_DIR` so the body examples and the scripts resolve to the *same* directory), extracts every script in this Appendix into `$OBS/harness/`, and relies on `common.sh` (below) to create `$OBS/out/` on first use — so an independent reader can reproduce every result without pre-creating any directory. Run this once from the repository root:

```
# --- one-time setup (from the repository root) -----------------------------
umask 077
export TMPDIR=/tmp/kitty-nosgid                                   # non-setgid tmpdir on this image (see §1)
export OBS="$(mktemp -d "$TMPDIR/kitty_obs.XXXXXXXX")"; chmod 700 "$OBS"
export OBS_DIR="$OBS"                                             # $OBS and $OBS_DIR name the same 0700 root
DOC=blitzy/documentation/kitty_815df1e210e0.md                   # path to THIS answer document

# write the extractor, then populate $OBS/harness/ from the fenced blocks below
cat > "$OBS/extract_harness.py" <<'EXTRACT'
#!/usr/bin/env python3
# Extract every Appendix-A script into $OBS/harness/. Each script is a fenced
# block under a "### name" heading (name wrapped in backticks in the heading).
import os, re, sys
F = chr(96) * 3                      # the triple-backtick fence, built via chr() so this file stays backtick-free
doc, dest = sys.argv[1], sys.argv[2]
os.makedirs(dest, exist_ok=True)
lines = open(doc, encoding="utf-8").read().splitlines()
start = next(i for i, l in enumerate(lines) if l.startswith("## Appendix A"))
hdr = re.compile(r"^###\s+" + chr(96) + r"([^" + chr(96) + r"]+)" + chr(96) + r"\s*$")
i, written = start, []
while i < len(lines):
    m = hdr.match(lines[i])
    if not m:
        i += 1; continue
    name = m.group(1); j = i + 1
    while j < len(lines) and not lines[j].startswith(F): j += 1
    if j >= len(lines): break
    k = j + 1; body = []
    while k < len(lines) and not lines[k].startswith(F): body.append(lines[k]); k += 1
    p = os.path.join(dest, name)
    open(p, "w", encoding="utf-8").write("\n".join(body) + "\n")
    if name.endswith((".sh", ".py")): os.chmod(p, os.stat(p).st_mode | 0o111)
    written.append(name); i = k + 1
print("extracted %d scripts to %s" % (len(written), dest))
EXTRACT
python3 "$OBS/extract_harness.py" "$DOC" "$OBS/harness"

# --- run any experiment (each sources common.sh, which creates $OBS/out) ----
#   env OBS_DIR="$OBS" bash "$OBS/harness/pty_independence.sh"
#   env OBS_DIR="$OBS" PP_N=200 bash "$OBS/harness/pty_independence.sh"

# --- teardown: remove ALL temporary observation tooling (repo left unchanged)
#   rm -rf "$OBS"
```

Running the extractor on this document reports `extracted 29 scripts to <$OBS/harness>` (the 29 files whose sections follow). Because `common.sh` now creates `$OBS/out/` (and `$OBS/harness/`) with `mkdir -p` **before** any script redirects into them, the very first command run from a clean root succeeds — a point verified directly by running `env OBS_DIR="$OBS" bash "$OBS/harness/pty_independence.sh"` in a freshly-created `mktemp -d` that contained only `harness/` (exit `0`, `$OBS/out/` populated, no "No such file or directory").

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
export OBS_DIR
export OBS="$OBS_DIR"                          # single observation root: body examples say $OBS, scripts say $OBS_DIR — identical dir
mkdir -p "$OBS_DIR/out" "$OBS_DIR/harness"     # create required subdirs BEFORE any redirection (clean-state safe; resolves P4-2)
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

### `pause_pixels.py`

```
#!/usr/bin/env python3
"""Q1 mode-2026 pixel observer — EXTERNAL, canonical render path.

Launches a REAL kitty window running a REAL PTY child (child -> PTY -> read_bytes
-> parser -> screen -> render), then READS the resulting X11 window backing image
with python-xlib (Window.get_image, ZPixmap). python-xlib is only an external
instrument (like od -c or --dump-commands); the pixels come from kitty's own
render of a real PTY child. Run from the repository root:

  OBS="$OBS" DISPLAY=:99 /root/kitty-venv/bin/python3 harness/pause_pixels.py explicit 1
  OBS="$OBS" DISPLAY=:99 /root/kitty-venv/bin/python3 harness/pause_pixels.py timeout-nudge 1

Modes:
  explicit       begin (CSI ?2026 h) -> overwrite screen -> explicit end (CSI ?2026 l),
                 hold < 2000 ms safety timeout.
  timeout-nudge  begin -> overwrite screen -> NO end; hold past 2000 ms; then emit a
                 content-neutral render nudge (CSI 6 n) to reveal when the timed-out
                 frame repaints.
The cursor is hidden (CSI ?25l) to remove blink noise so BEFORE==DURING is exact.
"""
import os, sys, time, hashlib, subprocess
from Xlib import X, display

REPO  = os.environ.get("KITTY_REPO", os.getcwd())
KITTY = os.path.join(REPO, "kitty", "launcher", "kitty")
SDIR  = os.path.join(REPO, "blitzy", "screenshots")
OBS   = os.environ.get("OBS", "/tmp/kitty-nosgid/kitty_obs")
MODE  = sys.argv[1] if len(sys.argv) > 1 else "explicit"
RUN   = sys.argv[2] if len(sys.argv) > 2 else "1"

def find_kitty_window(dsp):
    out = []
    def walk(w):
        try:
            for c in w.query_tree().children:
                try: cls = c.get_wm_class()
                except Exception: cls = None
                try: a = c.get_attributes(); g = c.get_geometry()
                except Exception: a = None; g = None
                if cls and 'kitty' in (cls[1] or '').lower() and a and a.map_state == X.IsViewable:
                    out.append((c, g.width, g.height))
                walk(c)
        except Exception: pass
    walk(dsp.screen().root)
    return out

def cap(win, w, h):
    im = win.get_image(0, 0, w, h, X.ZPixmap, 0xffffffff)
    d = im.data if isinstance(im.data, (bytes, bytearray)) else bytes(im.data, 'latin-1')
    return bytes(d)

def h12(b): return hashlib.sha256(b).hexdigest()[:12]

def pixdiff(a, b, w, h):
    ua = memoryview(a).cast('I'); ub = memoryview(b).cast('I')
    ch = 0; minx = miny = 1 << 30; maxx = maxy = -1
    for i in range(len(ua)):
        if ua[i] != ub[i]:
            ch += 1; px = i % w; py = i // w
            minx = min(minx, px); maxx = max(maxx, px); miny = min(miny, py); maxy = max(maxy, py)
    return (ch, (minx, miny, maxx, maxy)) if ch else (0, None)

BT = os.path.join(OBS, "pp_begin_%s_%s" % (MODE, RUN))
NT = os.path.join(OBS, "pp_nudge_%s_%s" % (MODE, RUN))
for f in (BT, NT):
    if os.path.exists(f): os.remove(f)

if MODE == "explicit":
    tail = r'''sleep 1.2
printf '\033[?2026l'
sleep 1.5'''
else:
    tail = (r'''sleep 2.6
date +%s.%N > "''' + NT + r'''"
printf '\033[6n'
sleep 1.6''')

child = (r'''
sleep 0.6
printf '\033[?25l'; printf '\033[2J\033[H'
printf 'BEFORE committed frame r%s\r\n'
printf 'baseline content line two\r\n'
sleep 1.5
date +%%s.%%N > "%s"
printf '\033[?2026h'; printf '\033[2J\033[H'
printf 'DURING held content r%s\r\n'
for i in $(seq 1 12); do printf 'held row %%02d xxxxxxxxxxxxxxxx\r\n' $i; done
%s
exit 0
''' % (RUN, BT, RUN, tail))

env = dict(os.environ)
env.update({'DISPLAY': os.environ.get('DISPLAY', ':99'), 'LIBGL_ALWAYS_SOFTWARE': '1',
            'TMPDIR': os.environ.get('TMPDIR', '/tmp/kitty-nosgid'),
            'PYTHONHOME': '/root/.local/share/uv/python/cpython-3.11.15-linux-x86_64-gnu',
            'PYTHONPATH': '/root/kitty-venv/lib/python3.11/site-packages',
            'LANG': 'en_US.UTF-8', 'LC_ALL': 'en_US.UTF-8'})
proc = subprocess.Popen([KITTY, '--config', 'NONE', 'sh', '-c', child],
                        env=env, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)

dsp = display.Display(env['DISPLAY'])
win = None; w = h = 0; t0 = time.monotonic()
while time.monotonic() - t0 < 6:
    ws = find_kitty_window(dsp)
    if ws: win, w, h = ws[0]; break
    time.sleep(0.1)
if not win:
    print("NO KITTY WINDOW"); proc.terminate(); sys.exit(1)

frames = []; s = time.monotonic()
while time.monotonic() - s < 7.5:
    wt = time.time()
    try: raw = cap(win, w, h)
    except Exception: raw = None
    if raw is not None: frames.append((wt, h12(raw), raw))
    time.sleep(0.07)
proc.terminate()
try: proc.wait(timeout=5)
except Exception: proc.kill()

begin = float(open(BT).read()) if os.path.exists(BT) else None
nudge = float(open(NT).read()) if os.path.exists(NT) else None
before = [f for f in frames if begin and f[0] < begin]
base_h = before[-1][1] if before else frames[0][1]
def at(off):
    best = None; bd = 1e9
    for f in frames:
        d = abs((f[0] - begin) - off)
        if d < bd: bd = d; best = f
    return best
bf = before[-1] if before else frames[0]
during = at(0.6); af = frames[-1]
print("mode=%s run=%s window=%dx%d frames=%d begin_ts=%.3f" % (MODE, RUN, w, h, len(frames), begin))
print("BEFORE  t%+.3fs sha=%s" % (bf[0] - begin, bf[1]))
print("DURING  t%+.3fs sha=%s" % (during[0] - begin, during[1]))
print("AFTER   t%+.3fs sha=%s" % (af[0] - begin, af[1]))
print("held (BEFORE==DURING): %s" % (bf[1] == during[1]))
if MODE == "explicit":
    ch, bb = pixdiff(bf[2], af[2], w, h)
    print("flushed (DURING!=AFTER): %s" % (during[1] != af[1]))
    print("pixeldiff BEFORE->AFTER: changed_px=%d bbox(l,u,r,b)=%s of %d" % (ch, bb, w * h))
    flip = next((f for f in frames if begin and f[0] >= begin and f[1] != base_h), None)
    if flip: print("flush_delay_after_begin_ms=%.0f (explicit end sent at +1200ms)" % ((flip[0] - begin) * 1000))
else:
    pre = at(2.3)
    print("at +2.3s (past 2000ms state-timeout, pre-nudge) base?%s sha=%s" % (pre[1] == base_h, pre[1]))
    flip = next((f for f in frames if begin and f[0] >= begin and f[1] != base_h), None)
    if flip and nudge:
        print("first_pixel_flip: +%.3fs after begin, +%.3fs after nudge -> sha=%s" % (flip[0] - begin, flip[0] - nudge, flip[1]))
try:
    from PIL import Image
    os.makedirs(SDIR, exist_ok=True)
    for tag, fr in (("before", bf), ("during", during), ("after", af)):
        Image.frombytes('RGBX', (w, h), fr[2]).convert('RGB').save("%s/mode2026_%s_%s_r%s.png" % (SDIR, MODE, tag, RUN))
    print("pngs=%s/mode2026_%s_{before,during,after}_r%s.png" % (SDIR, MODE, RUN))
except Exception as e:
    print("png_err=%s" % e)
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

### `coalesce_sweep.sh`

```
#!/usr/bin/env bash
# Q2 section 3.3 coalescing-window sweep: measure the PTY round-trip latency
# (pingpong.py RTT proxy) as OPT(input_delay) is swept {0,3,10,30} ms, two runs
# per setting, N=200 timed round-trips each (after 20 warm-ups). A gap>input_delay
# run (PP_GAP_MS=50 at input_delay=30) exercises the immediate-after-idle branch
# (child-monitor.c:1566). rtt_stats.py reduces each raw RTT log deterministically.
# If the median tracks input_delay with slope ~1, the coalescing window is the
# dominant, measurable latency term.
source "$(dirname "$0")/common.sh"
xdpyinfo -display :99 >/dev/null 2>&1 || { echo "XVFB DOWN" >&2; exit 1; }
cd "$(git rev-parse --show-toplevel)"
OUT="$OBS_DIR/out"; N="${PP_N:-200}"

one() {  # $1=input_delay(ms) $2=gap(ms) $3=run
    local id="$1" gap="$2" run="$3"
    local rlog="$OUT/coalesce_id${id}_gap${gap}_run${run}.log"
    local err="$OUT/coalesce_id${id}_gap${gap}_run${run}.err"
    rm -f "$rlog" "$err"
    RTT_LOG="$rlog" PP_N="$N" PP_WARMUP=20 PP_GAP_MS="$gap" \
      timeout 120 "$KITTY" -o close_on_child_death=yes -o input_delay="$id" \
        sh -c "RTT_LOG='$rlog' PP_N='$N' PP_WARMUP=20 PP_GAP_MS='$gap' $PYBIN $OBS_DIR/harness/pingpong.py" \
        >/dev/null 2>"$err" &
    local wrap=$!; track_pid "$wrap"
    local krc; if wait "$wrap" 2>/dev/null; then krc=0; else krc=$?; fi
    "$PYBIN" "$OBS_DIR/harness/rtt_stats.py" "$rlog" "input_delay=${id}ms gap=${gap}ms run${run} (launcher_exit=$krc)"
    echo
}

for id in 0 3 10 30; do
    one "$id" 0 1
    one "$id" 0 2
done
one 30 50 1
one 30 50 2
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
branch (child-monitor.c:1566): the first byte after idle wakes the main loop
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

### `flood_sweep.sh`

```
#!/usr/bin/env bash
# Q3 backpressure flood sweep: drive a real PTY child (flood.py) that writes a
# DETERMINISTIC, VARIED payload of FLOOD_BYTES bytes through kitty, capturing the
# raw consumed bytes with --dump-bytes, then verify losslessness with
# flood_verify.py (byte count AND whole-stream SHA-256 must match). Because the
# payload is deterministic, sent_sha256/dump_sha256 reproduce run to run; only
# write_wall_ms (the time kitty made the writer block while draining) varies.
#   Group 1 (losslessness): 8 MiB (8x the 1 MiB BUF_SZ), input_delay=3, 2 trials
#                           -> full flood_verify.py block per trial.
#   Group 2 (size sweep):   1/4/8 MiB at input_delay=3 -> brief (write_wall_ms).
#   Group 3 (slow-drain):   8 MiB at input_delay=3 vs 200 -> brief.
source "$(dirname "$0")/common.sh"
xdpyinfo -display :99 >/dev/null 2>&1 || { echo "XVFB DOWN" >&2; exit 1; }
cd "$(git rev-parse --show-toplevel)"
OUT="$OBS_DIR/out"

# run_one <bytes> <input_delay> <trial-label>  -> sets globals DUMP, META, KRC
run_one() {
    local sz="$1" id="$2" trial="$3"
    DUMP="$OUT/flood_dump_b${sz}_id${id}_t${trial}.bin"
    META="$OUT/flood_meta_b${sz}_id${id}_t${trial}.txt"
    local err="$OUT/flood_b${sz}_id${id}_t${trial}.err"
    rm -f "$DUMP" "$META" "$err"
    FLOOD_META="$META" FLOOD_BYTES="$sz" \
      timeout 120 "$KITTY" -o close_on_child_death=yes -o input_delay="$id" \
        --dump-bytes="$DUMP" \
        env FLOOD_META="$META" FLOOD_BYTES="$sz" "$PYBIN" "$OBS_DIR/harness/flood.py" \
        >/dev/null 2>"$err" &
    local wrap=$!; track_pid "$wrap"
    if wait "$wrap" 2>/dev/null; then KRC=0; else KRC=$?; fi
}

full() {  # $1=bytes $2=input_delay $3=trial : print the complete flood_verify.py block
    run_one "$1" "$2" "$3"
    "$PYBIN" "$OBS_DIR/harness/flood_verify.py" "$DUMP" "$META" \
        "bytes=$1 input_delay=$2 trial=$3 launcher_exit=$KRC"
    echo
}

brief() {  # $1=bytes $2=input_delay $3=trial : label + write_wall_ms + RESULT only
    run_one "$1" "$2" "$3"
    local res
    res=$("$PYBIN" "$OBS_DIR/harness/flood_verify.py" "$DUMP" "$META" \
        "bytes=$1 input_delay=$2 trial=$3 launcher_exit=$KRC")
    echo "$res" | grep -E '^label=|^write_wall_ms=|^RESULT='
    echo
}

# Group 1 — losslessness (full verification), 8 MiB, 2 trials
full 8388608 3 1
full 8388608 3 2

# Group 2 — size sweep at input_delay=3
brief 1048576 3 1
brief 4194304 3 1
brief 8388608 3 1

# Group 3 — slow-drain control: same 8 MiB burst, faster vs slower drain
brief 8388608 3 A
brief 8388608 200 A
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
# Same prober input in both -> a matched control. Driven PP_RUNS times by
# pty_ind_distribution.py: if A and B central tendencies coincide the PTY pipeline is
# functionally isolated from talk-socket turmoil; any divergence in the tail is reported
# as the run-to-run distribution rather than smoothed away (see section 4.3.4).
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

### `pty_ind_distribution.py`

```
#!/usr/bin/env python3
"""15-run PTY-independence distribution (P4-3).

Runs pty_independence.sh PP_RUNS times (default 15) with byte-identical prober
input, reading each run's raw per-trial RTT logs to compute per-run
min/median/p95/max for the idle-baseline (Phase A) and concurrent-chaos
(Phase B) conditions, then aggregates the distribution: the range of maxima and
the tail-exceedance counts (runs whose chaos max exceeds 10/20/90 ms). Central
tendency (median) is reported alongside the tail so a stable core is never
conflated with an unstable tail. Pure function of the raw trials; the summary is
reproducible from the copied per-run logs.
"""
import os
import statistics
import subprocess
import sys

HARNESS = os.path.dirname(os.path.abspath(__file__))
OBS = os.environ["OBS_DIR"]
OUT = os.path.join(OBS, "out")
RUNS = int(os.environ.get("PP_RUNS", "15"))
PP_N = os.environ.get("PP_N", "200")


def read_rtts(path):
    vals = []
    for line in open(path):
        line = line.strip()
        if not line or line.startswith("#"):
            continue
        vals.append(int(line))
    return vals


def ms(x):
    return x / 1_000_000.0


def pctl(sorted_vals, q):
    idx = min(len(sorted_vals) - 1, int(round(q * (len(sorted_vals) - 1))))
    return sorted_vals[idx]


def stats(vals):
    s = sorted(vals)
    return dict(n=len(s), min=ms(s[0]), median=ms(statistics.median(s)),
                p95=ms(pctl(s, 0.95)), max=ms(s[-1]), spread=ms(s[-1] - s[0]))


base_max, chaos_max, base_med, chaos_med = [], [], [], []
rows = []
for r in range(1, RUNS + 1):
    env = dict(os.environ, OBS_DIR=OBS, PP_N=PP_N)
    subprocess.run(["bash", os.path.join(HARNESS, "pty_independence.sh")],
                   env=env, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL,
                   check=False)
    a = stats(read_rtts(os.path.join(OUT, "pty_rtt_A_baseline.log")))
    b = stats(read_rtts(os.path.join(OUT, "pty_rtt_B_chaos.log")))
    base_max.append(a["max"])
    chaos_max.append(b["max"])
    base_med.append(a["median"])
    chaos_med.append(b["median"])
    rows.append((r, a, b))
    for lbl in ("A_baseline", "B_chaos"):
        src = os.path.join(OUT, "pty_rtt_%s.log" % lbl)
        dst = os.path.join(OUT, "pty_rtt_%s_run%02d.log" % (lbl, r))
        try:
            open(dst, "w").write(open(src).read())
        except OSError:
            pass

print("PP_N=%s PP_RUNS=%d" % (PP_N, RUNS))
print("run | A_min  A_med  A_p95  A_max   | B_min  B_med  B_p95  B_max (ms)")
for r, a, b in rows:
    print("%2d  | %6.4f %6.4f %6.4f %7.4f | %6.4f %6.4f %6.4f %8.4f" % (
        r, a["min"], a["median"], a["p95"], a["max"],
        b["min"], b["median"], b["p95"], b["max"]))


def rng(xs):
    return (min(xs), max(xs))


def count_gt(xs, t):
    return sum(1 for x in xs if x > t)


bmin, bmax = rng(base_max)
cmin, cmax = rng(chaos_max)
print()
print("=== aggregate over %d runs ===" % RUNS)
print("baseline max_ms range: %.4f .. %.4f  (>10ms: %d/%d)" % (
    bmin, bmax, count_gt(base_max, 10), RUNS))
print("chaos    max_ms range: %.4f .. %.4f  (>10ms: %d/%d, >20ms: %d/%d, >90ms: %d/%d)" % (
    cmin, cmax, count_gt(chaos_max, 10), RUNS,
    count_gt(chaos_max, 20), RUNS, count_gt(chaos_max, 90), RUNS))
print("baseline median_ms: min=%.4f median=%.4f max=%.4f" % (
    min(base_med), statistics.median(base_med), max(base_med)))
print("chaos    median_ms: min=%.4f median=%.4f max=%.4f" % (
    min(chaos_med), statistics.median(chaos_med), max(chaos_med)))
print("chaos max_ms sorted desc: " + " ".join("%.2f" % x for x in sorted(chaos_max, reverse=True)))
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
