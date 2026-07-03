# Kitty terminal-interaction pipeline — a runtime-observed answer

> **Scope note.** This document answers four questions about how the [Kitty](https://github.com/kovidgoyal/kitty) terminal emulator moves input through its pipeline. Every behavioral claim below was produced by **building and running** Kitty at branch `kitty_815df1e210e0` / HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`, then driving the *genuine* PTY / VT‑parser / screen code paths in‑process and pasting the verbatim output next to the claim. This was a strictly **read‑only** investigation: no repository source file was modified; the only artifact added is this document. Temporary observation scripts lived under `/tmp` (outside the repository tree) and were removed afterward.
>
> **How to read the evidence.** Each behavioral statement carries one adjacent fenced code block showing the command/script and its observed output, plus the exact `file:line` literal that implements it. Values that could only be *read from source* (not exercised at runtime) are labeled **[inferred]**. Any value taken through a bypassing interface, a fallback, or a synthetic stand‑in is labeled **[non‑canonical]**; the *only* such value in this document is the **supplemental** synthetic OSC 133 stream in §4.3(a) (hand‑crafted *input* fed through the real parser), and the very same alignment claim is proven **canonically** by the real‑`bash` PTY run in §4.3(b). Every other runtime value is canonical. Where a `file:line` in the planning notes had drifted by a few lines from the real source, the **verified** line is cited and the drift is called out.

---

## Table of contents

1. [Build & version baseline](#1-build--version-baseline)
2. [Q1 — Where raw input first enters, and pause/resume](#2-q1--where-raw-input-first-enters-and-pauseresume)
3. [Q2 — The "unseen conductor": timing, ordering, hand‑offs](#3-q2--the-unseen-conductor-timing-ordering-hand-offs)
4. [Q3 — Staying in sync: OSC 133, backpressure, unstable remote](#4-q3--staying-in-sync-osc-133-backpressure-unstable-remote)
5. [Q4 — End‑to‑end rhythm: from arrival to "settling again"](#5-q4--end-to-end-rhythm-from-arrival-to-settling-again)
6. [Evidence & coverage appendix](#6-evidence--coverage-appendix)
7. [Appendix A — Full observation scripts & build log](#7-appendix-a--full-observation-scripts--build-log)

---

## 1. Build & version baseline

Kitty is a three‑language monorepo: a C core (compiled into the `fast_data_types` extension module), a Python orchestration layer, and a Go tools layer. The runtime evidence below all comes from the C core exercised *in‑process* through the Python test harness, which is exactly the code that the live terminal runs.

### 1.1 Build (default, canonical configuration)

The canonical build action is `build` (this is `setup.py`'s default `action`). It was run as a normal user with **no flag overrides**. To capture the full compiler output verbatim (rather than an incremental no‑op), this run was performed after removing the *git‑ignored* build artifacts (`build/` and `kitty/fast_data_types.so`) to force a from‑scratch C recompile. The block below shows the **verbatim head** (backend selection) and **verbatim tail** (last compile step + the four link steps + exit status); the per‑file `[N/85] Compiling …` progress lines in between are elided as explicitly marked, and the **complete 98‑line log is reproduced in Appendix A.1**:

```console
$ cd /tmp/blitzy/kitty/blitzy-cd587963-f961-4938-8656-dd2a48df9768_65ebf2
$ python3 setup.py build ; echo "BUILD EXIT=$?"
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
[1/85] Compiling kitty/screen.c ...
[6/85] Compiling kitty/child-monitor.c ...
[excerpt] per-file lines [2/85] through [84/85] omitted here; the complete 98-line build log is in Appendix A.1
[85/85] Compiling kitty/gl-wrapper.c ...
 done
[1/4] Linking kitty/fast_data_types ...
[2/4] Linking [x11] kitty/glfw-x11 ...
[3/4] Linking kittens/transfer/rsync ...
[4/4] Linking launcher ...
 done
BUILD EXIT=0
```

`BUILD EXIT=0` confirms a clean build: all **85** C translation units compiled and the four link steps (`fast_data_types`, the X11 GLFW backend, the `rsync` transfer helper, and the `launcher`) succeeded. The two documented core files are visible in the log — `[1/85] Compiling kitty/screen.c` and `[6/85] Compiling kitty/child-monitor.c`. The `Disabling building of wayland backend` line is the canonical behavior in this environment: `wayland-protocols` is absent, so `setup.py` auto‑selects the X11 backend. This is **not** an accommodation and needs **no** `--ignore-compiler-warnings` flag — the strict default flags (`-pedantic-errors -Werror -Wall -Wextra`) compiled cleanly (no compiler diagnostic appears anywhere in the 98‑line log; only the `[N/85] Compiling …` progress and the link steps). The Wayland/X11 windowing backend is orthogonal to the VT‑parser / screen / keys code documented here, so all evidence remains canonical.

### 1.2 Version banner (authoritative, run twice)

The version banner is a VCS‑stamped identifier, so it must come from the built binary in its default configuration:

```console
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

The banner is **`kitty 0.35.2 created by Kovid Goyal`**, identical across two runs. The binary output is authoritative; it matches the source of truth `version: Version = Version(0, 35, 2)` at `kitty/constants.py:25`.

### 1.3 Runtime constants (the "knobs" that set the rhythm)

Three constants govern buffer size and cross‑thread cadence. They were read from the *compiled* extension via the launcher (`./kitty/launcher/kitty +launch /tmp/kobs_constants.py`), twice:

```python
# /tmp/kobs_constants.py
from kitty.fast_data_types import VT_PARSER_BUFFER_SIZE, get_options, set_options
from kitty.options.types import defaults
set_options(defaults)
o = get_options()
print("VT_PARSER_BUFFER_SIZE =", VT_PARSER_BUFFER_SIZE)
print("input_delay =", o.input_delay)
print("repaint_delay =", o.repaint_delay)
```

```console
$ ./kitty/launcher/kitty +launch /tmp/kobs_constants.py   # run 1
VT_PARSER_BUFFER_SIZE = 1048576
input_delay = 3
repaint_delay = 10
$ ./kitty/launcher/kitty +launch /tmp/kobs_constants.py   # run 2
VT_PARSER_BUFFER_SIZE = 1048576
input_delay = 3
repaint_delay = 10
```

| Constant | Observed value | Source of truth | Role |
|----------|---------------|-----------------|------|
| `VT_PARSER_BUFFER_SIZE` | `1048576` (= 1 MiB = `1024u*1024u`) | `#define BUF_SZ (1024u*1024u)` at `kitty/vt-parser.c:18`, exported at `kitty/vt-parser.c:1589` | Parser read/write buffer cap; the backpressure ceiling |
| `input_delay` | `3` (ms) | `input_delay: int = 3` at `kitty/options/types.py:536`; `opt('input_delay', '3', …)` at `kitty/options/definition.py:878` | How long the I/O thread batches child output before waking the main loop |
| `repaint_delay` | `10` (ms) | `repaint_delay: int = 10` at `kitty/options/types.py:567`; `opt('repaint_delay', '10', …)` at `kitty/options/definition.py:866` | Minimum spacing between GPU repaints (~100 FPS) |

The two delays' own documentation states *why* they exist. `repaint_delay` (`kitty/options/definition.py:866`): <code>"The default value yields ~100 FPS … to minimize latency when there is pending input to be processed, this option is ignored."</code> `input_delay` (`kitty/options/definition.py:878`): <code>"Delay before input from the program running in the terminal is processed … This setting is ignored when the input buffer is almost full."</code> These two "ignored when …" clauses are exactly the self‑tuning behavior that makes the interface settle (see §5).

### 1.4 In‑process pipeline harnesses (what actually runs the real code)

The Python suite under `kitty_tests/` imports the compiled C extension and drives the real parser/screen/keys in‑process. Selecting a module uses `--module <name>`; a bare positional name is rejected — reported here exactly because it is a common trap:

```console
$ python3 test.py parser
No test named ['parser'] found
```

Running the six relevant modules (`CI=true LANG=C.UTF-8 python3 test.py --module <name>`) produced these verbatim end markers:

```console
$ python3 test.py --module parser
Ran 16 tests in 0.055s
OK
$ python3 test.py --module screen
Ran 36 tests in 0.066s
OK
$ python3 test.py --module keys
Ran 3 tests in 0.060s
OK
```

The three modules that exercise the compiled pipeline this document explains — the VT parser, the screen model, and key encoding — all pass (`OK`). The remaining three modules exercise code that needs the full launcher bootstrap and therefore surface a known invocation artifact, reported honestly:

```console
$ python3 test.py --module shell_integration
test_bash_integration (kitty_tests.shell_integration.ShellIntegrationWithKitten.test_bash_integration) ... ERROR
test_fish_integration (kitty_tests.shell_integration.ShellIntegrationWithKitten.test_fish_integration) ... ERROR
test_zsh_integration (kitty_tests.shell_integration.ShellIntegrationWithKitten.test_zsh_integration) ... ERROR
test_bash_integration (kitty_tests.shell_integration.ShellIntegration.test_bash_integration) ... ok
test_fish_integration (kitty_tests.shell_integration.ShellIntegration.test_fish_integration) ... ok
test_zsh_integration (kitty_tests.shell_integration.ShellIntegration.test_zsh_integration) ... ok
Ran 6 tests in 0.478s
FAILED (errors=3)
```

The three **plain** `ShellIntegration` tests (`bash`, `fish`, `zsh`) all pass `ok` — these fork a real shell over a real PTY and are the basis of the §4 evidence. (In this container `fish` and `zsh` *are* installed, so they run rather than skip.) The three `ShellIntegrationWithKitten` variants `ERROR` with a single root cause, `AttributeError: module 'sys' has no attribute 'kitty_run_data'` (raised at `kitty/constants.py:67` in `kitty_exe()`), because that path needs the full launcher's `sys.kitty_run_data`. The `ssh` and `check_build` modules fail for the identical reason:

```console
$ python3 test.py --module ssh
Ran 8 tests in 4.079s
FAILED (errors=57)
$ python3 test.py --module check_build
Ran 9 tests in 0.020s
FAILED (errors=3, skipped=1)
```

These are test‑invocation artifacts, not defects in the parser/screen/keys pipeline; every claim below is grounded in a directly exercised code path.

---

## 2. Q1 — Where raw input first enters, and pause/resume

**The question.** *Where does raw input first enter the system, and how does the raw stream become something the application can react to — for a surge of keystrokes, paste bursts, and resize signals, especially when a session is paused then resumed?*

**The short answer.** "Raw input" arrives at **two different entry points that must not be conflated**, because Kitty sits *between* the user and the child program:

- **Child → terminal (program output).** Bytes the running program writes first materialize in **`read_bytes()`** at `kitty/child-monitor.c:1337`, via a `read()` from the child PTY file descriptor into the VT parser's write buffer. This is the stream that "becomes something the application can react to" once the parser turns it into screen mutations.
- **User → child (keystrokes / paste / resize).** User actions first materialize on the main/GUI thread and are *queued toward the child*: keystrokes in **`on_key_input()`** at `kitty/keys.c:166`, paste in **`paste_with_actions()`** at `kitty/window.py:1643`, and resize as an **`ioctl(TIOCSWINSZ)`** in **`pty_resize()`** at `kitty/child-monitor.c:577`.

The rest of this section walks each named input kind and both pause/resume mechanisms.

### 2.1 Program output first enters at `read_bytes()`

`read_bytes(int fd, Screen *screen)` is the sole place child output enters:

```c
// kitty/child-monitor.c:1337
read_bytes(int fd, Screen *screen) {
```

It performs the `read()` into the parser‑owned write buffer; the returned boolean tells the I/O loop whether the child is still alive. From here the bytes flow to the VT parser (§4) and become screen state (§5). The "how the raw stream becomes actionable" transformation *is* the VT parser demultiplexing bytes into text, control codes, and escape sequences — demonstrated directly in §4.

### 2.2 Keystrokes first enter at `on_key_input()` → `schedule_write_to_child()`

The keyboard entry point is:

```c
// kitty/keys.c:166
on_key_input(GLFWkeyevent *ev) {
```

Inside, the key event is encoded to a byte sequence and handed to the child‑write queue. The encoding is performed by `encode_glfw_key_event(...)` at `kitty/keys.c:251`, and the encoded bytes are queued by `schedule_write_to_child(w->id, 1, encoded_key, size)` at `kitty/keys.c:259`. (An IME‑commit path also calls `schedule_write_to_child(...)` at `kitty/keys.c:202`.)

The *encoding step* — the moment a physical key becomes bytes the child can react to — was exercised directly. The Python‑exposed `encode_key_for_tty` calls the **same** C function, `encode_glfw_key_event`, that `on_key_input` uses (`kitty/keys.c:319` vs `:251`); it is therefore canonical for the transformation, not a bypass:

```python
# /tmp/kobs_keyenc.py
import kitty.fast_data_types as defines
enc = defines.encode_key_for_tty
ctrl = defines.GLFW_MOD_CONTROL
print("plain 'a'                         ->", repr(enc(key=ord('a'))))
print("Ctrl+a (GLFW_MOD_CONTROL)         ->", repr(enc(key=ord('a'), mods=ctrl)))
print("Ctrl+a kitty-proto flags=0b1111   ->", repr(enc(key=ord('a'), mods=ctrl, key_encoding_flags=0b1111)))
print("F1 (GLFW_FKEY_F1)                 ->", repr(enc(key=defines.GLFW_FKEY_F1)))
```

```console
$ ./kitty/launcher/kitty +launch /tmp/kobs_keyenc.py     # identical across 2 runs
plain 'a'                         -> 'a'
Ctrl+a (GLFW_MOD_CONTROL)         -> '\x01'
Ctrl+a kitty-proto flags=0b1111   -> '\x1b[97;5u'
F1 (GLFW_FKEY_F1)                 -> '\x1bOP'
```

Cause → effect, one line each:

- A bare letter passes through as itself — `'a'` → `'a'`.
- The Control modifier maps to the C0 control code — `Ctrl+a` → `'\x01'` (0x01), the classic "control character" transformation.
- With the **Kitty keyboard protocol** enabled (`key_encoding_flags=0b1111`), the same chord becomes an unambiguous **CSI‑u** sequence — `Ctrl+a` → `'\x1b[97;5u'` (97 = `'a'`, 5 = the ctrl modifier encoding, `u` = the CSI‑u trailer). This is the `kitty/key_encoding.c` CSI‑u encoding.
- A function key becomes its escape sequence — `F1` → `'\x1bOP'`.

Supporting modules named for completeness: `kitty/keys.py` resolves configured key *mappings*/actions before the C entry point; `kitty/key_encoding.c` implements the CSI‑u protocol shown above; and `kitty/mouse.c` is the sibling input source for pointer events. The high‑level Python dispatcher `dispatch_possible_special_key(self, ev)` lives at `kitty/boss.py:1408`.

### 2.3 Paste bursts enter at `paste_with_actions()` and are sanitized

A paste is not treated as keystrokes; it flows through `paste_with_actions(self, text)` at `kitty/window.py:1643`. When the screen is in bracketed‑paste mode, the payload is scrubbed by `sanitize_for_bracketed_paste(...)` (imported at `kitty/window.py:117`, called at `kitty/window.py:1718`, implemented at `kitty/utils.py:1139`). The purpose is **paste‑injection prevention**: an attacker‑supplied payload must not be able to smuggle the bracketed‑paste *end* marker and thereby "break out" into executed input. Exercised directly against the real sanitizer:

```python
# /tmp/kobs_paste.py
from kitty.utils import sanitize_for_bracketed_paste
payload = b"safe text \x1b[201~ echo pwned"
print("input  =", repr(payload))
print("output =", repr(sanitize_for_bracketed_paste(payload)))
```

```console
$ ./kitty/launcher/kitty +launch /tmp/kobs_paste.py      # identical across 2 runs
input  = b'safe text \x1b[201~ echo pwned'
output = b'safe text  echo pwned'
```

The embedded bracketed‑paste **END** marker `\x1b[201~` is stripped from the payload — `b'…\x1b[201~ echo…'` → `b'…  echo…'` — so the pasted text cannot terminate the paste early and inject a command. Paste filtering itself is configured via `load_paste_filter()` at `kitty/window.py:423`.

### 2.4 Resize signals enter as `ioctl(fd, TIOCSWINSZ, dim)` and are debounced

A window resize is delivered to the child by writing the new window dimensions into the kernel PTY with `TIOCSWINSZ`:

```c
// kitty/child-monitor.c:577-579
pty_resize(int fd, struct winsize *dim) {
    while(true) {
        if (ioctl(fd, TIOCSWINSZ, dim) == -1) {
```

Rapid resizes are **debounced** by `process_pending_resizes(monotonic_t now)` at `kitty/child-monitor.c:1043`, so a drag‑resize does not flood the child with `SIGWINCH`. The genuine `ioctl(TIOCSWINSZ)` path was exercised via the test PTY's `set_window_size`, which issues the real `fcntl.ioctl(master_fd, TIOCSWINSZ, …)` — mirroring `pty_resize`:

```python
# /tmp/kobs_resize.py  (excerpt — full script in Appendix A.2)
pty = t.create_pty(['cat'], cols=80, lines=25)
print("initial: screen.lines =", pty.screen.lines, "screen.columns =", pty.screen.columns)
pty.set_window_size(rows=20, columns=40)      # -> screen.resize + real ioctl(TIOCSWINSZ)
print("after set_window_size(rows=20, columns=40): screen.lines =", pty.screen.lines, "screen.columns =", pty.screen.columns)
r = fcntl.ioctl(pty.master_fd, termios.TIOCGWINSZ, struct.pack('HHHH', 0,0,0,0))
rows, cols, xpix, ypix = struct.unpack('HHHH', r)
print("kernel TIOCGWINSZ readback: rows =", rows, "cols =", cols, "xpix =", xpix, "ypix =", ypix)
```

```console
$ ./kitty/launcher/kitty +launch /tmp/kobs_resize.py     # identical across 2 runs
initial: screen.lines = 25 screen.columns = 80
after set_window_size(rows=20, columns=40): screen.lines = 20 screen.columns = 40
kernel TIOCGWINSZ readback: rows = 20 cols = 40 xpix = 400 ypix = 400
```

The screen model updates (`25×80` → `20×40`) *and* the kernel‑side window size, read back with `TIOCGWINSZ`, confirms the real `ioctl` propagated: `rows = 20 cols = 40` (with `400×400` pixels = 40·10 × 20·20 cell metrics). The child now sees the new geometry — the resize became actionable.

### 2.5 "Paused then resumed" — TWO distinct mechanisms

The user's "paused then resumed" maps onto two entirely separate mechanisms; both are documented so neither is missed.

**(a) The child *process* is suspended (Ctrl‑Z → `SIGTSTP`).** When the user presses the suspend key, the terminal's control‑character map turns it into a job‑control signal. The mapping is explicit:

```python
# kitty/child.py:492-493
        elif key_num == cc[termios.VSUSP]:
            s = signal.SIGTSTP
```

So the `VSUSP` control character (default Ctrl‑Z) is delivered to the child as `SIGTSTP`, suspending the *program*; resuming (e.g. `fg`) sends `SIGCONT`. **[inferred]** — this mapping is read from source; it is a property of the kernel line discipline and job control, not of the in‑process parser path, so it was not exercised at runtime. (Related PTY setup: `set_iutf8_fd(master, True)` at `kitty/child.py:174`.)

**(b) *Rendering* is paused while parsing continues (synchronized output, DEC mode 2026).** This is the mechanism that makes a burst "settle" without tearing: the application asks the terminal to hold the last displayed frame while it keeps sending bytes, then release atomically. This is observed and quantified in **§5**; the key runtime fact for Q1 is that **input is never paused — only presentation is** (text sent while paused still lands in the screen buffer, proven in §5.3).

---

## 3. Q2 — The "unseen conductor": timing, ordering, hand-offs

**The question.** *How are the responsibilities of timing, ordering, and state hand‑offs split among the moving parts — and critically, what decides which event gets handled first?*

**The short answer.** The "unseen conductor" is the **Child Monitor** in `kitty/child-monitor.c`, a **three‑thread** engine. "What gets handled first" is **not** a priority queue; it is a **fixed `if`‑branch order** executed after `poll()` returns in the I/O thread. Timing/cadence is governed by two knobs, `input_delay` (input batching) and `repaint_delay` (render batching); state hand‑offs cross threads through a mutex‑guarded parser buffer (§4) and lightweight cross‑thread wakeups — an `eventfd`/`signalfd` waking the `poll()`‑based I/O and talk loops, and `glfwPostEmptyEvent()` waking the GUI loop (the two are distinct mechanisms; see §3.3).

### 3.1 The three threads

Two threads are spawned; the third is the process's own main/GUI thread. The declaration and the `pthread_create`/`pthread_join` calls:

```c
// kitty/child-monitor.c:55
    pthread_t io_thread, talk_thread;
```

```console
$ grep -n "pthread_create\|pthread_join" kitty/child-monitor.c
256:        if ((ret = pthread_create(&self->talk_thread, NULL, talk_loop, self)) != 0) {
286:        if ((ret = pthread_create(&self->talk_thread, NULL, talk_loop, self)) != 0) {
291:    ret = pthread_create(&self->io_thread, NULL, io_loop, self);
427:    int ret = pthread_join(self->io_thread, NULL);
430:        ret = pthread_join(self->talk_thread, NULL);
1002:    int ret = pthread_create(&thread, NULL, thread_write, data);
```

- **I/O thread** — `io_loop(void *data)` at `kitty/child-monitor.c:1481`, created at `:291`. Polls child PTY fds, reads/writes, reaps dead children.
- **Main / GUI thread** — `main_loop(ChildMonitor *self, …)` at `kitty/child-monitor.c:1259`. Consumes the parser buffer, updates screen state, schedules GPU repaints.
- **Talk thread** — `talk_loop`, created at `:256`/`:286`. Serves peer sockets for remote control; its `poll()` handlers are the analogues `read_from_peer`/`write_to_peer`/`POLLNVAL`. (The `pthread_create` at `:1002` is a helper `thread_write`, not one of the three long‑lived loops.)

This three‑thread split *is* the "split of responsibilities": the I/O thread owns fd multiplexing and timing of wakeups; the main thread owns parsing/screen/render; the talk thread owns the remote‑control side channel. Because they are separate threads, *normal* I/O is decoupled from parsing and rendering: a momentarily slow frame does not block the I/O thread's reads, and routine child output does not block the GUI. This decoupling is **bounded by flow control**, however — it is deliberately *not* unlimited isolation. If the main thread stalls long enough that the mutex‑guarded parser buffer reaches its 1 MiB cap, the I/O thread **intentionally stops reading that child**: it stops requesting `POLLIN` for the child fd (`kitty/child-monitor.c:1501`, gated on `vt_parser_has_space_for_input`, `kitty/vt-parser.c:1481`), which lets the kernel PTY buffer fill and throttles the child's own `write()` rather than dropping or reordering bytes (detailed in §4.4). The hand‑off between the threads is that parser buffer, and the buffer's fullness is exactly what applies the brakes — so under sustained backpressure a stalled consumer *does* propagate back to the reader, by design.

### 3.2 What decides which event is handled first: the deterministic `poll()` branch order

Inside the I/O thread, once `poll()` returns, the ready file descriptors are serviced in a **fixed source‑order sequence of `if` statements** — this literal order is the answer to "what gets handled first":

```c
// kitty/child-monitor.c:1515-1545 (verbatim, abridged of inner bodies)
            if (children_fds[0].revents && POLLIN) drain_fd(children_fds[0].fd); // wakeup   [:1515]
            if (children_fds[1].revents && POLLIN) {                              // signals  [:1516]
                ...
                read_signals(children_fds[1].fd, handle_signal, &ss);
                ...
            }
            for (i = 0; i < self->count; i++) {
                if (children_fds[EXTRA_FDS + i].revents & (POLLIN | POLLHUP)) {   // child read [:1529]
                    ...
                    has_more = read_bytes(children_fds[EXTRA_FDS + i].fd, children[i].screen);
                    ...
                }
                if (children_fds[EXTRA_FDS + i].revents & POLLOUT) {              // child write [:1539]
                    write_to_child(children[i].fd, children[i].screen);
                }
                if (children_fds[EXTRA_FDS + i].revents & POLLNVAL) {             // fd closed   [:1542]
                    ...
                }
            }
```

The priority is therefore, in order:

1. **Wakeup fd** — `kitty/child-monitor.c:1515` `drain_fd(children_fds[0].fd)` — the cross‑thread "please wake up" nudge is drained first so subsequent iterations start clean.
2. **Signal fd** — `:1516` → `read_signals(...)` handles `SIGCHLD`/kill/reload (child death, config reload, termination) before touching data fds, so lifecycle events win over payload.
3. **Per‑child readable/hangup** — `:1529` `(POLLIN | POLLHUP)` → `read_bytes(...)` (§2.1): drain program output.
4. **Per‑child writable** — `:1539` `POLLOUT` → `write_to_child(...)`: flush queued user input toward the child.
5. **Per‑child invalid** — `:1542` `POLLNVAL`: a closed fd is marked for removal.

There is no runtime scheduler weighing events; the *code layout* is the schedule. This is why the behavior is deterministic and easy to reason about: for any `poll()` wakeup, wakeup‑drain precedes signals, signals precede reads, reads precede writes, writes precede cleanup.

### 3.3 Cross‑thread wakeups: two *distinct* mechanisms (`eventfd`/`signalfd` for the poll loops, `glfwPostEmptyEvent()` for the GUI)

There are **two** different wakeup mechanisms and it is important not to conflate them: one wakes the `poll()`‑based loops (the I/O and talk threads), the other wakes the GUI / main thread.

**(a) The poll‑based loops (I/O and talk) are woken by an `eventfd`; OS signals arrive via a `signalfd`.** These fds are set up in `kitty/loop-utils.c` (each with a portable self‑pipe fallback) and are exactly the fds serviced by branches (1) and (2) of §3.2:

```console
$ grep -n "eventfd\|signalfd\|self_pipe" kitty/loop-utils.c
42:        ld->signal_read_fd = signalfd(-1, &ld->signals, SFD_NONBLOCK | SFD_CLOEXEC);
48:        if (!self_pipe(ld->signal_fds, true)) return false;
70:    ld->wakeup_read_fd = eventfd(0, EFD_CLOEXEC | EFD_NONBLOCK);
73:    if (!self_pipe(ld->wakeup_fds, true)) return false;
133:    static struct signalfd_siginfo fdsi[32];
144:        size_t num_signals = s / sizeof(struct signalfd_siginfo);
145:        if (num_signals == 0 || num_signals * sizeof(struct signalfd_siginfo) != (size_t)s) {
146:            log_error("Incomplete signal read from signalfd");
```

The fd‑setup matches are the first four lines — `:42`/`:48`/`:70`/`:73`; the trailing four (`:133`/`:144`/`:145`/`:146`) are incidental substring matches of the `signalfd_siginfo` struct type and a log string in the signal‑*reading* code (`read_signals`), not fd creation.

- `signalfd(...)` at `kitty/loop-utils.c:42` turns asynchronous OS signals into a pollable fd (fallback self‑pipe at `:48`) — this feeds branch (2) above.
- `eventfd(...)` at `kitty/loop-utils.c:70` (fallback self‑pipe at `:73`) is the I/O thread's `LoopData.wakeup_read_fd`. It is wired into the poll set as `children_fds[0]` (`kitty/child-monitor.c:183`) and drained by branch (1). Other threads ring this doorbell to wake the **I/O** thread by calling `wakeup_io_loop()` → `wakeup_loop(&self->io_loop_data, …)` (`kitty/child-monitor.c:225-226`); the **talk** thread's loop is woken the same way through its own `LoopData` (`kitty/child-monitor.c:1755`). This `eventfd` therefore wakes **only the poll‑based loops — never the GUI thread.**

**(b) The GUI / main thread is woken by `glfwPostEmptyEvent()`, not by the `eventfd`.** The main thread does not `poll()` on the `eventfd`; it runs a GLFW event loop (`main_loop` → `run_main_loop`, `kitty/child-monitor.c:1259`/`:1262`). When the I/O thread has batched input ready to hand over, its `WAKEUP` macro (`kitty/child-monitor.c:1562`, see §3.4) calls `wakeup_main_loop()`, which is defined as a single `glfwPostEmptyEvent()`:

```console
$ grep -n "define WAKEUP" kitty/child-monitor.c
1562:#define WAKEUP { wakeup_main_loop(); last_main_loop_wakeup_at = now; has_pending_wakeups = false; }
$ grep -n -A2 "^wakeup_main_loop" kitty/glfw.c
1807:wakeup_main_loop(void) {
1808-    glfwPostEmptyEvent();
1809-}
```

So the direction matters: the **main → I/O** nudge is the `eventfd` (`kitty/loop-utils.c:70`); the **I/O → GUI** nudge is `glfwPostEmptyEvent()` (`kitty/glfw.c:1807-1808`). The GUI is never woken by the `eventfd` — an important correction to the intuition that a single doorbell serves both directions.

### 3.4 The cadence: batched wakeups every `input_delay`

The I/O thread deliberately does **not** wake the GUI thread on every byte. It coalesces input and only fires its `WAKEUP` — i.e. the `glfwPostEmptyEvent()` GUI nudge of §3.3(b), **not** the `eventfd` — once per `input_delay` window:

```c
// kitty/child-monitor.c:1562-1566
#define WAKEUP { wakeup_main_loop(); last_main_loop_wakeup_at = now; has_pending_wakeups = false; }
        // we only wakeup the main loop after input_delay as wakeup is an expensive operation
        // on some platforms, such as cocoa
        if (data_received) {
            if ((now = monotonic()) - last_main_loop_wakeup_at > OPT(input_delay)) WAKEUP
```

The comment at `kitty/child-monitor.c:1563` states the rationale verbatim — *"we only wakeup the main loop after input_delay as wakeup is an expensive operation"* — and the condition at `:1566` fires `WAKEUP` only when more than `OPT(input_delay)` (observed `= 3` ms, §1.3) has elapsed since the last wakeup. On the render side, `repaint_delay` (observed `= 10` ms) bounds repaints to ~100 FPS. Together these two knobs are the metronome that keeps the "conductor" from thrashing; the full settle cycle is narrated in §5. The parse dispatch entry points the main thread ultimately calls are `parse_input(...)` at `kitty/child-monitor.c:451` and `do_parse(...)` at `:438`.

---

## 4. Q3 — Staying in sync: OSC 133, backpressure, unstable remote

**The question.** *When shell‑integration hints arrive mixed in with ordinary text, how does the system keep screen state, command context, and input meaning aligned without drifting out of sync — and does that differ under heavy backpressure or an unstable remote connection?*

**The short answer.** Alignment is guaranteed by **construction**: the hints (OSC 133 sequences) and the ordinary text ride the **same byte stream** and are demultiplexed by the **single VT parser**. Because there is exactly one ordered consumer of that stream, marker order and text order cannot diverge. Under heavy backpressure the alignment is *preserved* (the parser stops reading rather than dropping/reordering bytes). The unstable‑remote case is handled by the SSH kitten deploying integration over the controlling TTY — documented from source because a live server is unavailable here (stated explicitly in §4.5).

### 4.1 One stream, one demux point

Child output enters once (`read_bytes`, §2.1) and is parsed by one state machine. OSC sequences are dispatched by `dispatch_osc(...)`; OSC code 133 (shell integration) routes to `shell_prompt_marking`:

```c
// kitty/vt-parser.c:457
dispatch_osc(PS *self, uint8_t *buf, size_t limit, bool is_extended_osc) {
// kitty/vt-parser.c:536
        case 133:
// kitty/vt-parser.c:544
                shell_prompt_marking(self->screen, (char*)buf + i);
```

The handler in the screen model classifies the marker letter and updates command context:

```c
// kitty/screen.c:2328
shell_prompt_marking(Screen *self, char *buf) {
```

- `'A'` → **prompt start** (`PromptKind pk = PROMPT_START;` at `kitty/screen.c:2333`); sub‑tokens are parsed by `parse_prompt_mark`, where `k=s` selects `SECONDARY_PROMPT` (`kitty/screen.c:2321`), `redraw=0` and `special_key=1` set prompt settings.
- `'C'` → **command/output start** (`prompt_kind = OUTPUT_START;` at `kitty/screen.c:2341`); a `;cmdline…` suffix is captured as the command line (`cmdline = buf + 2` at `kitty/screen.c:2344`) and dispatched to the `cmd_output_marking` Python callback.
- `'D'` → **command end + exit status** (`const char *exit_status = buf[1] == ';' ? buf + 2 : "";` at `kitty/screen.c:2351`).

Because these markers are consumed inline at their exact byte position, the prompt/command/output boundaries land on exactly the screen rows where the bytes arrived — the "command context" stays glued to the text.

### 4.2 Every OSC 133 marker variant (exhaustive, per shell, with literal + reason)

The four marker letters (`A`, `B`, `C`, `D`) plus the kitty‑internal `k` region marker appear in per‑shell variants. All were verified in the shell‑integration sources at HEAD:

| Marker | Meaning | Variant (verbatim literal) | Shell — `file:line` | Why it exists (cause → effect) |
|--------|---------|----------------------------|---------------------|-------------------------------|
| **A** | Prompt start | `\e]133;A` | zsh `shell-integration/zsh/kitty-integration:153`; (bash emits `…D;$?…A` combined, see below) | Marks the row where a fresh prompt begins, so jump‑to‑prompt / prompt reflow know the boundary |
| **A** | Secondary‑prompt start | `\e]133;A;k=s` | bash `shell-integration/bash/kitty.bash:137` & `:240`; zsh `shell-integration/zsh/kitty-integration:163` | `k=s` tells kitty this is a continuation (PS2) prompt → parsed to `SECONDARY_PROMPT` at `kitty/screen.c:2321` |
| **A** | Prompt start, special keys | `\e]133;A;special_key=1` | fish `shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish:85` | `special_key=1` sets `uses_special_keys_for_cursor_movement` so cursor‑key edits are handled correctly |
| **B** | Prompt end / command typed | `\e]133;B` | zsh `shell-integration/zsh/kitty-integration:226` (commented‑out) | The prompt‑end marker is optional; zsh ships it disabled, so it is mostly implicit — reported as present‑but‑commented |
| **C** | Command start (pre‑exec) + cmdline | `\e]133;C;cmdline=%q` | bash `shell-integration/bash/kitty.bash:208`; zsh `shell-integration/zsh/kitty-integration:218` | `%q`‑quoted command line captured at pre‑exec → populates command context (`cmdline = buf+2`, `kitty/screen.c:2344`) |
| **C** | Command start, URL‑escaped cmdline | `\e]133;C;cmdline_url=%s` | fish `shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish:91` | fish URL‑escapes the command line instead of shell‑quoting → same effect, different encoding |
| **D** | Command end (no status) | `\e]133;D` | zsh `shell-integration/zsh/kitty-integration:149`; fish `shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish:83` | Marks output end when the exit status is not (yet) known |
| **D** | Command end + status | `\e]133;D;<status>` | zsh `shell-integration/zsh/kitty-integration:145` | Records the process exit status for the just‑finished command |
| **D** | Command end + `$?`, then next A | `\e]133;D;$?` `\e]133;A` | bash `shell-integration/bash/kitty.bash:239` | bash appends the exit code `$?` and immediately starts the next prompt in one PS1 fragment |
| **D** | Command end + `$status` | `\e]133;D;$status` | fish `shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish:96` | fish's equivalent of `$?` |
| **k** | kitty‑internal region marker | `\e]133;k;<name>_kitty` | bash `shell-integration/bash/kitty.bash:127` | Delimits kitty‑inserted regions so they can be stripped from `PS0`/`PS1`/`PS2` and not double‑counted |

Orchestration that turns these on lives in `kitty/shell_integration.py:218` `def modify_shell_environ(opts, env, argv)`, which sets `KITTY_SHELL_INTEGRATION` and wires the per‑shell rc files.

### 4.3 Proof of alignment: markers are split out of the visible text

**(a) Synthetic mixed stream through the real parser [non‑canonical, supplemental].** The *input* here is a hand‑crafted byte stream — a synthetic stand‑in for what a shell actually emits — so this illustration is labeled **[non‑canonical]** and is **supplemental** to the canonical real‑`bash` proof in (b). It is still instructive because those bytes pass through the *genuine* parser (via `parse_bytes`, which is exactly the `read_bytes` → `test_create_write_buffer` → `test_commit_write_buffer` → `test_parse_written_data` sequence). The run shows the markers vanish from the screen while text order is preserved and the exit status is captured:

```python
# /tmp/kobs_osc133.py  (excerpt — full script in Appendix A.3)
run('\x1b]133;A\x07host$ \x1b]133;C;cmdline=ls -la /etc\x07out1  out2\r\n\x1b]133;D;0\x07', "exit 0")
run('\x1b]133;A\x07host$ \x1b]133;C;cmdline=false\x07out1  out2\r\n\x1b]133;D;1\x07', "exit 1")
```

```console
$ ./kitty/launcher/kitty +launch /tmp/kobs_osc133.py     # identical across 2 runs
[exit 0]
  input        = '\x1b]133;A\x07host$ \x1b]133;C;cmdline=ls -la /etc\x07out1  out2\r\n\x1b]133;D;0\x07'
  screen.line0 = 'host$ out1  out2'
  last_cmdline = 'ls'
  exit_status  = 0
[exit 1]
  input        = '\x1b]133;A\x07host$ \x1b]133;C;cmdline=false\x07out1  out2\r\n\x1b]133;D;1\x07'
  screen.line0 = 'host$ out1  out2'
  last_cmdline = 'false'
  exit_status  = 1
```

The rendered row is `screen.line0 = 'host$ out1  out2'` — **all** three OSC 133 markers (`A`, `C;cmdline=…`, `D;<status>`) were demultiplexed *out* of the visible text, leaving prompt text and command output in their original order. The command context is captured alongside: `exit_status = 0` for the first run and `exit_status = 1` for the second, tracking the `D;0`/`D;1` markers exactly. (`last_cmdline` reflects the raw `;cmdline` field parsed from this synthetic stream; the real shell path in (b) yields the full command line.)

**(b) Real bash over a genuine PTY.** Forking real `bash` with shell integration enabled and reading its output through the genuine `os.read(master_fd)` → parser path confirms the same alignment end‑to‑end with real OSC 133 emission:

```python
# /tmp/kobs_realbash.py  (excerpt — full script in Appendix A.4)
env = safe_env_for_running_shell(['bash'], home_dir, rc='PS1="PROMPT> "', shell='bash', with_kitten=False)
pty = t.create_pty(['bash'], cwd=home_dir, env=env)     # forks real bash
... pump until 'PROMPT> ' ...
pty.send_cmd_to_child('echo hello-kitty'); pty.wait_till(lambda: exit_status set)
pty.send_cmd_to_child('false');           pty.wait_till(lambda: exit_status set)
```

```console
$ ./kitty/launcher/kitty +launch /tmp/kobs_realbash.py   # identical across 2 runs
startup screen.line(0)          = 'PROMPT> '
after 'echo hello-kitty': cmdline = 'echo hello-kitty' exit_status = 0
after 'false':            cmdline = 'false' exit_status = 1
```

The prompt row renders as `'PROMPT> '` (the `A` marker consumed, not shown); running `echo hello-kitty` yields `cmdline = 'echo hello-kitty' exit_status = 0` and running `false` yields `cmdline = 'false' exit_status = 1`. Screen text, command context (cmdline), and exit status are all mutually consistent — no drift.

For scrollback, the same marker discipline lets kitty locate a command's output later: `reverse_find(buf, sz, (const uint8_t*)"\x1b]133;C\x1b\\")` at `kitty/history.c:475` searches for the `C` marker to bound command output.

### 4.4 Under heavy backpressure — alignment is preserved by *stopping*, not by dropping

The parser buffer is a producer/consumer region guarded by a mutex (`pthread_mutex_t lock;` at `kitty/vt-parser.c:206`) with a hard cap `BUF_SZ` = 1 MiB (`kitty/vt-parser.c:18`); a single escape code may be at most `BUF_SZ / 4u` = 256 KiB (`MAX_ESCAPE_CODE_LENGTH`, `kitty/vt-parser.c:21`). The I/O thread only asks `poll()` for `POLLIN` on a child while the parser reports free space:

```c
// kitty/child-monitor.c:1501  (POLLIN gate; POLLOUT gate is :1503)
            children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;
```

> **Anchor note.** The AAP text placed this gate at `:1502`; the verified line is **`kitty/child-monitor.c:1501`** (the `POLLOUT` gate is `:1503`). Cited as verified.

`vt_parser_has_space_for_input` returns `read.sz + write.pending < BUF_SZ` (`kitty/vt-parser.c:1481`). When the buffer is full there is no space → no `POLLIN` requested → the kernel PTY buffer fills → the child's own `write()` blocks. That is classic flow control: kitty throttles the child rather than dropping or reordering bytes, so alignment cannot break. Measured at scale by committing a **4 MiB** surge into the write buffer *without parsing* (simulating a stalled main thread):

```python
# /tmp/kobs_backpressure.py  (excerpt — full script in Appendix A.5)
chunk = b'x' * (64*1024); attempted = 4*1024*1024   # 4 MiB
while sent < attempted:
    dest = s.test_create_write_buffer()
    if len(dest) == 0: break                          # no space -> backpressure
    total_committed += s.test_commit_write_buffer(chunk, dest)  # DO NOT parse
```

```console
$ ./kitty/launcher/kitty +launch /tmp/kobs_backpressure.py   # RUN 1 and RUN 2 — identical
VT_PARSER_BUFFER_SIZE           = 1048576
attempted surge                 = 4194304 bytes (4 MiB)
total committed before FULL     = 1048576 bytes
available space now (len buf)   = 0
```

Against a 4 MiB attempted surge, exactly **`1048576`** bytes (1 MiB, = `BUF_SZ`) were accepted before the buffer filled, after which `test_create_write_buffer()` returned **0** bytes of space. Zero available space means `vt_parser_has_space_for_input` is false, so the I/O thread drops `POLLIN` and the child is flow‑controlled. The measurement was **stable across two runs at the 4 MiB scale**. The consumer‑side flush that keeps this moving is gated at `kitty/vt-parser.c:1425`: `if (flush || pd->time_since_new_input >= OPT(input_delay) || self->read.sz + 16 * 1024 > BUF_SZ)` — i.e. force a flush when within 16 KiB of full — which dovetails with the `input_delay` batching of §3.4. So *under heavy backpressure the behavior differs only in throughput, not in ordering*: the same single‑stream, single‑parser discipline holds, which is precisely why sync is not lost.

### 4.5 Under an unstable remote connection — the SSH kitten (path unavailability stated)

For remote sessions the SSH kitten deploys terminfo and the shell‑integration scripts to the remote host over the controlling TTY, then a bootstrap script stages them. The bootstrap is guarded so a partial/interrupted deployment cleans up its *temporary* extraction directory (and restores the TTY) after itself — see the exact, limited scope below:

- `cleanup_on_bootstrap_exit()` trap definition — `shell-integration/ssh/bootstrap.sh:10`, invoked at `:156`.
- `request_data="REQUEST_DATA"` placeholder, substituted by the kitten at transmit time — `shell-integration/ssh/bootstrap.sh:90`.
- `data_dir` resolved from `$KITTY_SSH_KITTEN_DATA_DIR` (absolute `/*` vs `$HOME/`‑relative) — `shell-integration/ssh/bootstrap.sh:119-120` (`case` block `:118-121`); staging dir `shell_integration_dir="$data_dir/shell-integration"` — `:122`.
- Driver: `kittens/ssh/main.go` (and `kittens/ssh/main.py`), which transmit terminfo + integration over the TTY.

The cleanup trap narrows the blast radius of an *interrupted* bootstrap, but its scope is **limited** — and because there is no `sshd` in this container the following is **[inferred]** from source, not observed on a live link. `cleanup_on_bootstrap_exit()` (`shell-integration/ssh/bootstrap.sh:10-15`) does exactly two things: it restores terminal echo (`command stty echo`, gated on `$echo_on`, `:11`) and removes the **temporary extraction directory** `$tdir` (`command rm -rf "$tdir"`, `:13`; then `tdir=""`, `:14`). It does **not** roll back the *final* staged files: once `untar_and_read_env` has run `mv_files_and_dirs "$tdir/home" "$HOME"` (`:131`; plus the root tree at `:132`) — helper defined at `shell-integration/ssh/bootstrap-utils.sh:9-15` — the terminfo and integration files already live under `$HOME` (in `data_dir`/`shell_integration_dir`, resolved at `:119-120`/`:122`) and **persist**. So the trap keeps a *dropped mid‑transfer* attempt from leaving a stray temp dir or an echo‑off TTY; it does **not** erase already‑staged data, so it would be inaccurate to claim a reconnect necessarily "starts clean." **The live end‑to‑end remote path was NOT exercised, and this is stated explicitly rather than synthesized:**

```console
$ command -v ssh || echo '(not found)'
/usr/bin/ssh
$ command -v sshd || echo '(not found)'
(not found)
$ ./kitty/launcher/kitty +kitten ssh --help ; echo EXIT=$?
Usage: kitten ssh arguments for the ssh command

The ssh kitten is a thin wrapper around the ssh command. It automatically
enables shell integration on the remote host, re-uses existing connections to
reduce latency, makes the kitty terminfo database available, etc. Its invocation
is identical to the ssh command. For details on its usage, see Truly convenient
SSH.

Options:
  --help, -h
    Show help for this command

kitten ssh 0.35.2 created by Kovid Goyal
EXIT=0
```

The `ssh` **client** is present — `command -v ssh` prints `/usr/bin/ssh` — but the `sshd` **server** is absent: `command -v sshd` finds nothing, so its `|| echo '(not found)'` fallback prints `(not found)`. The SSH kitten *driver* is itself functional (`+kitten ssh --help` exits `EXIT=0`), yet with **no `sshd` server** in this container a genuine remote bootstrap over a live connection cannot be performed here. Per the real‑entry‑point rule this is reported as **unavailable**; the bootstrap mechanism above is therefore documented **[inferred]** from source, not from a live run. (The in‑process `ssh` test module likewise cannot run its launcher‑dependent cases — it errors on `AttributeError: module 'sys' has no attribute 'kitty_run_data'`, §1.4.)

---

## 5. Q4 — End-to-end rhythm: from arrival to "settling again"

**The question.** *Narrate the full journey from the moment mixed input arrives to the moment the interface "settles again," and how the moving parts keep their rhythm.*

### 5.1 The settle cycle, step by step

Combining §2–§4, the journey of a surge of mixed input is:

1. **Arrival (I/O thread).** The child's bytes become readable; `poll()` returns and — per the fixed branch order (§3.2) — the wakeup fd is drained (`:1515`), signals handled (`:1516`), then `read_bytes(...)` (`kitty/child-monitor.c:1529`→`:1337`) copies bytes into the parser's write buffer under the mutex (`kitty/vt-parser.c:206`). Queued user input is flushed out on `POLLOUT` (`:1539`).
2. **Batch (I/O thread).** Rather than waking the GUI per byte, the I/O thread coalesces and rings the doorbell only once per `input_delay` (observed `3` ms): `if ((now = monotonic()) - last_main_loop_wakeup_at > OPT(input_delay)) WAKEUP` at `kitty/child-monitor.c:1566`, with the rationale comment at `:1563`. This is the first half of the "rhythm."
3. **Hand‑off + parse (main thread).** That `WAKEUP` (step 2) calls `wakeup_main_loop()` → `glfwPostEmptyEvent()` (`kitty/glfw.c:1807-1808`), waking the GUI `main_loop` (`kitty/child-monitor.c:1259`) — the GUI is nudged through GLFW, **not** the `eventfd` (§3.3(b)). `main_loop` then parses the buffered bytes under the same mutex via `parse_input`/`do_parse` (`:451`/`:438`). The single parser demultiplexes text, control codes, and OSC 133 markers (§4.1) into screen state — prompt/command/output boundaries included.
4. **Render (main thread).** The GPU repaint is scheduled subject to `repaint_delay` (observed `10` ms, ~100 FPS). Per its own docs, `repaint_delay` *"is ignored"* while input is pending (§1.3), so a burst is drained quickly; once input stops, the repaint cadence takes over. This is the second half of the "rhythm."
5. **Settle.** When the surge ends, no more wakeups fire, the buffer drains below the flush threshold, and the display reaches a steady frame — the interface has "settled."

The two knobs `input_delay` (3 ms) and `repaint_delay` (10 ms) are the metronome: `input_delay` decouples read frequency from wake frequency; `repaint_delay` decouples screen mutation from frame presentation. Each is *self‑tuning* — both are explicitly ignored under load (pending input / near‑full buffer) so latency stays low exactly when it matters.

### 5.2 Synchronized output (DEC mode 2026): pausing presentation, not parsing

An application that redraws a whole frame can ask the terminal to **hold the currently displayed frame** while it streams the update, then reveal it atomically — eliminating tearing/flicker. This is DEC private mode **2026**:

```c
// kitty/control-codes.h:235
#define PENDING_MODE 2026
// kitty/screen.c:1174-1175  (enable/disable handler)
        case PENDING_MODE << 5:
            if (!screen_pause_rendering(self, val, 0)) {
```

`CSI ? 2026 h` enables it (pause), `CSI ? 2026 l` disables it (resume). This is a **standards‑aligned, best‑effort hint**, not a bespoke kitty behavior: while paused, the terminal keeps *processing* incoming text and sequences into its off‑screen state but keeps *displaying* the last rendered frame; on disable it re‑reads the latest grid and presents one atomic frame. Because a hint could otherwise wedge the display, implementations apply a safety timeout — kitty's is §5.4.

### 5.3 Observed: the DECRQM state cycles ;2 → ;1 → ;2, and text lands while paused

The terminal reports mode 2026's state on a DECRQM query (`CSI ? 2026 $ p`), replying `?2026;1$y` when *set/paused* and `?2026;2$y` when *reset/live*. Driving the real parser and capturing the bytes written back to the child (`Callbacks.wtcbuf`):

```python
# /tmp/kobs_mode2026.py  (excerpt — full script in Appendix A.6)
decrqm("initial")
parse_bytes(s, b'\x1b[?2026h')              # enable  -> pause
decrqm("after CSI ?2026h (enable/pause)")
parse_bytes(s, b'while-paused-text')        # text arrives WHILE paused
print("screen.line(0) while paused =", repr(str(s.line(0))))
parse_bytes(s, b'\x1b[?2026l')              # disable -> resume
decrqm("after CSI ?2026l (disable/resume)")
print("screen.pause_rendering(100) =", s.pause_rendering(100))
decrqm("after pause_rendering(100)")
```

```console
$ ./kitty/launcher/kitty +launch /tmp/kobs_mode2026.py   # identical across 2 runs
  initial                                DECRQM reply = b'\x1b[?2026;2$y'
  after CSI ?2026h (enable/pause)        DECRQM reply = b'\x1b[?2026;1$y'
  screen.line(0) while paused          = 'while-paused-text'
  after CSI ?2026l (disable/resume)      DECRQM reply = b'\x1b[?2026;2$y'
  screen.pause_rendering(100)          = True
  after pause_rendering(100)             DECRQM reply = b'\x1b[?2026;1$y'
```

Cause → effect, one line each:

- Before any toggle, the mode is **live**: DECRQM reply `b'\x1b[?2026;2$y'` (the `;2` = reset).
- `CSI ? 2026 h` **pauses presentation**: reply flips to `b'\x1b[?2026;1$y'` (the `;1` = set/paused).
- Crucially, **text sent while paused still lands in the buffer**: `screen.line(0) while paused = 'while-paused-text'` — parsing continues; only the *frame* is held. This is the runtime proof that mode 2026 pauses rendering, not input.
- `CSI ? 2026 l` **resumes**: reply returns to `b'\x1b[?2026;2$y'` (live) — and on resume the screen is marked dirty so the live grid is re‑read into one atomic frame.
- The direct C API agrees: `screen.pause_rendering(100)` returns `True` and the subsequent DECRQM reply is `b'\x1b[?2026;1$y'` (paused).

The `1`/`2` code is produced by `kitty/screen.c:2238` `ans = self->paused_rendering.expires_at ? 1 : 2;` (in the `case PENDING_UPDATE:` at `:2237`), written back as `?2026;<1|2>$y` via `snprintf(..., "%s%u;%u$y", ...)` at `:2240`.

> **Anchor note.** The AAP/agent‑prompt cited the DECRQM `ans` line at `:2242`; the verified line is **`kitty/screen.c:2238`** (with `case PENDING_UPDATE:` at `:2237`). Cited as verified.

### 5.4 The self‑healing safety timeout (2000 ms)

So that a misbehaving or disconnected application cannot freeze the display forever, a pause carries an expiry. When `screen_pause_rendering` is called with no explicit duration it defaults to **2000 ms**:

```c
// kitty/screen.c:2506  entry
screen_pause_rendering(Screen *self, bool pause, int for_in_ms) {
// kitty/screen.c:2521-2522  default timeout + expiry
    if (for_in_ms <= 0) for_in_ms = 2000;
    self->paused_rendering.expires_at = monotonic() + ms_to_monotonic_t(for_in_ms);
```

The render loop force‑resumes once the deadline passes:

```c
// kitty/screen.c:2489-2490
screen_check_pause_rendering(Screen *self, monotonic_t now) {
    if (self->paused_rendering.expires_at && now > self->paused_rendering.expires_at) screen_pause_rendering(self, false, 0);
```

So if a frame never gets its `CSI ? 2026 l`, kitty auto‑resumes after **2000 ms** (field `monotonic_t expires_at;` at `kitty/screen.h:160`) — the interface *always* settles. **[inferred]** — the 2000 ms default and the auto‑resume comparison are read from source; the pause/resume *toggle* itself is observed in §5.3, but firing the render‑loop timeout was not driven in‑process.

> **Anchor note.** The AAP/agent‑prompt placed the `2000` default at `:2523` and the expiry assignment at `:2524`; the verified lines are **`kitty/screen.c:2521`** and **`:2522`**. Cited as verified.

On pause, `screen_pause_rendering` snapshots the line buffer, cursor, and color profile (holding the last rendered state); on resume (`!pause`) it sets `self->is_dirty = true` so the live grid is re‑read — the atomic‑frame reveal. The screen model these mutate is spread across `kitty/line.c`, `kitty/line-buf.c`, `kitty/cursor.c`, `kitty/charsets.c`, `kitty/history.c`, and the mode flags in `kitty/modes.h`.

---

## 6. Evidence & coverage appendix

Every named item from the four questions — each mechanism, function, condition, file, flag, and every "keystrokes / paste bursts / resize signals" style example — mapped to the section that answers it and its observed evidence line and/or `file:line`.

### 6.1 Coverage matrix

| # | Named item (from the questions) | Answered in | Verified `file:line` | Observed evidence line |
|---|--------------------------------|-------------|----------------------|------------------------|
| **Baseline** |  |  |  |  |
| B1 | Build (default canonical) | §1.1 | `setup.py` default `action='build'` | `BUILD EXIT=0`; `Disabling building of wayland backend` |
| B2 | Version banner (VCS‑stamped) | §1.2 | `kitty/constants.py:25` | `kitty 0.35.2 created by Kovid Goyal` (×2) |
| B3 | `VT_PARSER_BUFFER_SIZE` | §1.3 | `kitty/vt-parser.c:18`, `:1589` | `VT_PARSER_BUFFER_SIZE = 1048576` |
| B4 | `input_delay` | §1.3,§3.4 | `kitty/options/types.py:536`; `kitty/options/definition.py:878` | `input_delay = 3` |
| B5 | `repaint_delay` | §1.3,§5.1 | `kitty/options/types.py:567`; `kitty/options/definition.py:866` | `repaint_delay = 10` |
| B6 | Harness markers | §1.4 | `kitty_tests/` | `Ran 16 tests`/`OK`; `Ran 36 tests`/`OK`; `Ran 3 tests`/`OK` |
| **Q1 — input entry** |  |  |  |  |
| 1a | Child output first enters | §2.1 | `kitty/child-monitor.c:1337` `read_bytes` | (path feeds §4.3 parse evidence) |
| 1b | User input queue → flush | §2.2 | `kitty/child-monitor.c:372` `schedule_write_to_child`, `:1443` `write_to_child` | (poll `POLLOUT` branch, §3.2) |
| 1c | **Keystrokes** entry | §2.2 | `kitty/keys.c:166` `on_key_input` → `:251` encode → `:259`/`:202` `schedule_write_to_child` | `'a'`→`'a'`; `Ctrl+a`→`'\x01'`; flags=0b1111→`'\x1b[97;5u'`; `F1`→`'\x1bOP'` |
| 1d | Kitty keyboard protocol CSI‑u | §2.2 | `kitty/key_encoding.c` (via `kitty/keys.c:319`) | `Ctrl+a` flags=0b1111 → `'\x1b[97;5u'` |
| 1e | Key mapping / mouse sibling | §2.2 | `kitty/keys.py`, `kitty/mouse.c`, `kitty/boss.py:1408` | (named; `dispatch_possible_special_key`) |
| 1f | **Paste bursts** entry + sanitize | §2.3 | `kitty/window.py:1643` `paste_with_actions`, `:117`/`:1718`, `kitty/utils.py:1139`; `:423` filter | `b'…\x1b[201~ echo…'` → `b'…  echo…'` |
| 1g | **Resize signals** → `ioctl(TIOCSWINSZ)` | §2.4 | `kitty/child-monitor.c:577-579` `pty_resize`; debounce `:1043` | screen `25×80`→`20×40`; kernel `TIOCGWINSZ rows=20 cols=40` |
| 1h | boss.py wiring | §2.2,§3.1 | `kitty/boss.py:370` `ChildMonitor(`, `:587` `add_child` | (named) |
| 1i | **Paused/resumed (a)** child suspend | §2.5 | `kitty/child.py:492-493` `VSUSP`→`SIGTSTP`; `:174` `set_iutf8_fd` | **[inferred]** (line‑discipline, not parser path) |
| 1j | **Paused/resumed (b)** render suspend | §2.5,§5 | `kitty/screen.c:2506` (see Q4) | DECRQM `;1`/`;2` toggle (§5.3) |
| **Q2 — the conductor** |  |  |  |  |
| 2a | Three threads | §3.1 | `kitty/child-monitor.c:55`, `:1481` `io_loop`, `:1259` `main_loop`, `talk_loop` | `grep pthread_create` → `:256/:286/:291`; joins `:427` (io_thread)/`:430` (talk_thread) |
| 2b | **What gets handled first** (poll order) | §3.2 | `kitty/child-monitor.c:1515`→`:1516`→`:1529`→`:1539`→`:1542` | verbatim branch block (wakeup→signal→read→write→NVAL) |
| 2c | Not a priority queue | §3.2 | (fixed `if`‑branch order) | (explicit statement) |
| 2d | `eventfd`/`signalfd` + self‑pipe (**poll‑based I/O & talk loops only**) | §3.3(a) | `kitty/loop-utils.c:42`,`:48`,`:70`,`:73`; wired `kitty/child-monitor.c:183`, rung `wakeup_io_loop` `:225-226`, talk `:1755` | `grep` → `signalfd`(:42)/`eventfd`(:70) lines |
| 2e | GUI‑loop wakeup (`glfwPostEmptyEvent`, **not** the `eventfd`) | §3.3(b),§3.4 | `kitty/child-monitor.c:1562` `WAKEUP`→`wakeup_main_loop()`; `kitty/glfw.c:1807-1808` `glfwPostEmptyEvent()`; loop `:1259`/`:1262` | `grep` → `1808-    glfwPostEmptyEvent();` |
| 2f | Batched wakeup / `input_delay` | §3.4 | `kitty/child-monitor.c:1562` `WAKEUP`, `:1563` comment, `:1566` condition | verbatim macro+comment+condition |
| 2g | Parse dispatch | §3.4,§5.1 | `kitty/child-monitor.c:451` `parse_input`, `:438` `do_parse` | (named) |
| **Q3 — staying in sync** |  |  |  |  |
| 3a | Single demux point | §4.1 | `kitty/vt-parser.c:457` `dispatch_osc`, `:536` `case 133:`, `:544` | verbatim lines |
| 3b | Handler A/C/D | §4.1 | `kitty/screen.c:2328` `shell_prompt_marking`; A `:2333`, `k=s`→SECONDARY `:2321`, C `:2341`/`:2344`, D `:2351` | verbatim handler |
| 3c | OSC 133 **A** (plain / `k=s` / `special_key=1`) | §4.2 | zsh `:153`; bash `:137`/`:240`, zsh `:163`; fish `:85` | marker table |
| 3d | OSC 133 **B** (commented) | §4.2 | zsh `shell-integration/zsh/kitty-integration:226` | marker table |
| 3e | OSC 133 **C** (`cmdline=%q` / `cmdline_url=%s`) | §4.2 | bash `:208`, zsh `:218`; fish `:91` | marker table |
| 3f | OSC 133 **D** (`D` / `D;<status>` / `D;$?` / `D;$status`) | §4.2 | zsh `:149`/`:145`, bash `:239`, fish `:83`/`:96` | marker table |
| 3g | OSC 133 **k** (internal region) | §4.2 | bash `shell-integration/bash/kitty.bash:127` | marker table |
| 3h | Integration orchestration | §4.2 | `kitty/shell_integration.py:218` `modify_shell_environ` | (named) |
| 3i | Alignment proof — synthetic *input*, real parser **[non‑canonical, supplemental]** | §4.3(a) | via `parse_bytes` | `screen.line0 = 'host$ out1  out2'`; `exit_status 0`/`1` |
| 3j | Alignment proof — real `bash` over a genuine PTY **[canonical]** | §4.3(b) | `kitty_tests/shell_integration.py` real PTY | `'PROMPT> '`; `echo hello-kitty`→`0`; `false`→`1` |
| 3k | Scrollback cmd‑output find | §4.3 | `kitty/history.c:475` `reverse_find "…133;C…"` | (named) |
| 3l | **Heavy backpressure**: `BUF_SZ` 1 MiB | §4.4 | `kitty/vt-parser.c:18` | `total committed before FULL = 1048576` |
| 3m | POLLIN gate (`has_space`) | §4.4 | `kitty/child-monitor.c:1501` (POLLOUT `:1503`); `kitty/vt-parser.c:1481` | `available space now = 0` |
| 3n | Mutex / flush gate / max esc len | §4.4 | `kitty/vt-parser.c:206`, `:1425`, `:21` | 4 MiB surge, stable ×2 |
| 3o | **Unstable remote**: bootstrap (trap scope **limited** — echo + temp `$tdir` only, no final‑file rollback) | §4.5 | `shell-integration/ssh/bootstrap.sh:10-15` (echo `:11`, `$tdir` `:13`), `:90`, `:119-120`, `:122`, `:131-133`, `:156`; `shell-integration/ssh/bootstrap-utils.sh:9-15`; `kittens/ssh/main.go` | **[inferred]**; unavailability stated |
| 3p | Remote path unavailability | §4.5 | (no `sshd`) | `command -v sshd` → not found; `+kitten ssh --help` `EXIT=0` |
| **Q4 — settling** |  |  |  |  |
| 4a | Arrival→batch→wake→parse→render→settle | §5.1 | `kitty/child-monitor.c:1566` (input_delay), `:1562` `WAKEUP`→`wakeup_main_loop()`, `:1259` `main_loop`; `kitty/glfw.c:1807-1808` `glfwPostEmptyEvent()` | (narrative built from observed §2–§4) |
| 4b | Mode 2026 enable/disable | §5.2 | `kitty/control-codes.h:235`; `kitty/screen.c:1174-1175` | verbatim handler |
| 4c | DECRQM `;2`→`;1`→`;2` | §5.3 | `kitty/screen.c:2238` (`case PENDING_UPDATE:` `:2237`, snprintf `:2240`) | `;2$y`→`;1$y`→`;2$y`; pause_rendering(100)→`;1$y` |
| 4d | Parsing continues while paused | §5.3 | (mode 2026 semantics) | `screen.line(0) while paused = 'while-paused-text'` |
| 4e | Safety timeout 2000 ms | §5.4 | `kitty/screen.c:2521`,`:2522`; field `kitty/screen.h:160` | **[inferred]** (read from source) |
| 4f | Auto‑resume | §5.4 | `kitty/screen.c:2489-2490` `screen_check_pause_rendering` | **[inferred]** |
| 4g | Screen model files | §5.4 | `line.c`,`line-buf.c`,`cursor.c`,`charsets.c`,`history.c`,`modes.h` | (named) |

### 6.2 Anchor drifts encountered (verified line cited throughout)

| Item | AAP/prompt line | **Verified line** |
|------|-----------------|-------------------|
| Backpressure `POLLIN` gate | `:1502` (AAP) | **`kitty/child-monitor.c:1501`** |
| DECRQM `ans = … ? 1 : 2` | `:2242` | **`kitty/screen.c:2238`** (`case PENDING_UPDATE:` `:2237`) |
| Mode‑2026 default timeout `2000` | `:2523` | **`kitty/screen.c:2521`** |
| `expires_at = monotonic() + …` | `:2524` | **`kitty/screen.c:2522`** |
| `'D'` case `exit_status` | `:2349` | **`kitty/screen.c:2351`** |

### 6.3 Evidence discipline notes

- **Stability.** Every measured magnitude was identical across two runs: version banner; the three runtime constants; keystroke encodings; paste sanitize; resize dims + kernel readback; OSC 133 demux; real‑bash cmdline/exit status; the 4 MiB backpressure measurement (`1048576` cap, `0` free); and the DECRQM `;2`/`;1`/`;2` cycle.
- **Scale.** Backpressure was driven at **4 MiB** (4× the 1 MiB cap) to force the buffer full and observe the `0`‑free / no‑`POLLIN` state.
- **[inferred] labels.** Read‑from‑source, not runtime‑exercised: the `VSUSP`→`SIGTSTP` mapping (§2.5a); the SSH bootstrap mechanism and the whole live remote path (§4.5, unavailable — no `sshd`); the 2000 ms safety‑timeout default and its render‑loop auto‑resume (§5.4). The mode‑2026 pause/resume *toggle* itself is observed (§5.3).
- **Canonicality of evidence.** Every runtime value in this document is canonical — taken from the genuine PTY/parser/screen path (the same code `read_bytes` feeds), the default canonical build, or the compiled extension's exported constants — **with one explicitly labeled exception**: the **supplemental** synthetic OSC 133 stream in §4.3(a), whose *input* is a hand‑crafted stand‑in and is therefore labeled **[non‑canonical]**. That single illustration is not load‑bearing — the same alignment claim is proven canonically by the real‑`bash` PTY run in §4.3(b). No value comes from a remote‑control socket or debug hook.
- **Environment deviations reported honestly.** `fish` and `zsh` are installed here (their plain integration tests pass rather than skip); the `ssh` module reported `errors=57`. These are reported as observed, independent of any prior expectation.


---

## 7. Appendix A — Full observation scripts & build log

> This appendix reproduces, **verbatim and in full**, the observation artifacts that §1–§5 quote as excerpts, so every quoted line is fully reproducible. Each script is exactly the file that was run via `./kitty/launcher/kitty +launch <path>`. (The key‑encoding, paste‑sanitize and runtime‑constants scripts are already shown in full inline at §2.2, §2.3 and §1.3 and are not repeated here.) All temporary scripts lived under `/tmp` — outside the repository tree — and were removed after the investigation, leaving the repo unchanged.

### A.1 — Build log (full)

The complete verbatim output of the default build, run after removing the git‑ignored `build/` and `kitty/fast_data_types.so` to force a from‑scratch C recompile (§1.1 shows the head+tail excerpt of this exact log):

```console
$ python3 setup.py build ; echo "BUILD EXIT=$?"
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
[1/85] Compiling kitty/screen.c ...
[2/85] Compiling kitty/unicode-data.c ...
[3/85] Compiling [x11] glfw/x11_window.c ...
[4/85] Compiling kitty/glfw.c ...
[5/85] Compiling kitty/graphics.c ...
[6/85] Compiling kitty/child-monitor.c ...
[7/85] Compiling kitty/fonts.c ...
[8/85] Compiling kitty/shaders.c ...
[9/85] Compiling kitty/vt-parser.c ...
[10/85] Compiling kitty/vt-parser.c ...
[11/85] Compiling kitty/state.c ...
[12/85] Compiling [x11] glfw/input.c ...
[13/85] Compiling kitty/mouse.c ...
[14/85] Compiling [x11] glfw/xkb_glfw.c ...
[15/85] Compiling kitty/freetype.c ...
[16/85] Compiling [x11] glfw/window.c ...
[17/85] Compiling kitty/line.c ...
[18/85] Compiling kitty/glfw-wrapper.c ...
[19/85] Compiling kittens/transfer/algorithm.c ...
[20/85] Compiling [x11] glfw/x11_init.c ...
[21/85] Compiling kitty/freetype_render_ui_text.c ...
[22/85] Compiling [x11] glfw/egl_context.c ...
[23/85] Compiling kitty/disk-cache.c ...
[24/85] Compiling [x11] glfw/glx_context.c ...
[25/85] Compiling kitty/line-buf.c ...
[26/85] Compiling kitty/data-types.c ...
[27/85] Compiling kitty/colors.c ...
[28/85] Compiling kitty/history.c ...
[29/85] Compiling kitty/keys.c ...
[30/85] Compiling [x11] glfw/x11_monitor.c ...
[31/85] Compiling kitty/fontconfig.c ...
[32/85] Compiling [x11] glfw/context.c ...
[33/85] Compiling kitty/crypto.c ...
[34/85] Compiling [x11] glfw/ibus_glfw.c ...
[35/85] Compiling kitty/key_encoding.c ...
[36/85] Compiling kitty/launcher/main.c ...
[37/85] Compiling [x11] glfw/monitor.c ...
[38/85] Compiling kitty/font-names.c ...
[39/85] Compiling [x11] glfw/backend_utils.c ...
[40/85] Compiling kitty/charsets.c ...
[41/85] Compiling [x11] glfw/linux_joystick.c ...
[42/85] Compiling [x11] glfw/init.c ...
[43/85] Compiling [x11] glfw/dbus_glfw.c ...
[44/85] Compiling kitty/gl.c ...
[45/85] Compiling [x11] glfw/vulkan.c ...
[46/85] Compiling [x11] glfw/osmesa_context.c ...
[47/85] Compiling kitty/cursor.c ...
[48/85] Compiling kitty/launcher/single-instance.c ...
[49/85] Compiling kitty/desktop.c ...
[50/85] Compiling kitty/loop-utils.c ...
[51/85] Compiling 3rdparty/ringbuf/ringbuf.c ...
[52/85] Compiling kitty/simd-string.c ...
[53/85] Compiling kitty/systemd.c ...
[54/85] Compiling kitty/shlex.c ...
[55/85] Compiling kitty/child.c ...
[56/85] Compiling kitty/kittens.c ...
[57/85] Compiling 3rdparty/base64/lib/codec_choose.c ...
[58/85] Compiling kitty/png-reader.c ...
[59/85] Compiling [x11] glfw/linux_notify.c ...
[60/85] Compiling kitty/rowcolumn-diacritics.c ...
[61/85] Compiling kitty/hyperlink.c ...
[62/85] Compiling kitty/wcswidth.c ...
[63/85] Compiling kitty/fast-file-copy.c ...
[64/85] Compiling 3rdparty/base64/lib/lib.c ...
[65/85] Compiling [x11] glfw/posix_thread.c ...
[66/85] Compiling kitty/window_logo.c ...
[67/85] Compiling kitty/glyph-cache.c ...
[68/85] Compiling kitty/logging.c ...
[69/85] Compiling 3rdparty/base64/lib/arch/neon64/codec.c ...
[70/85] Compiling 3rdparty/base64/lib/tables/tables.c ...
[71/85] Compiling 3rdparty/base64/lib/arch/neon32/codec.c ...
[72/85] Compiling 3rdparty/base64/lib/arch/avx/codec.c ...
[73/85] Compiling 3rdparty/base64/lib/arch/ssse3/codec.c ...
[74/85] Compiling 3rdparty/base64/lib/arch/sse42/codec.c ...
[75/85] Compiling 3rdparty/base64/lib/arch/sse41/codec.c ...
[76/85] Compiling 3rdparty/base64/lib/arch/avx2/codec.c ...
[77/85] Compiling kitty/utmp.c ...
[78/85] Compiling 3rdparty/base64/lib/arch/avx512/codec.c ...
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
BUILD EXIT=0
```

> **Note — from‑scratch capture vs. incremental re‑verification.** The 98‑line log above is the **from‑scratch** capture: it was produced after removing the git‑ignored `build/` directory and `kitty/fast_data_types.so`, which forces `setup.py` to compile all **85** C translation units. A *subsequent* `python3 setup.py build` run **without** first removing those artifacts is **incremental** and will not reproduce the full `[N/85] Compiling …` sequence — with the artifacts already present it recompiles **0** of the 85 units and still exits `0`. This was verified directly by observation. Once the Go build cache and the `kitty/launcher/kitten` binary are fully settled, the incremental re‑run emits **only** the `Disabling building of wayland backend` notice, then `BUILD EXIT=0`, with **zero** `[N/85] Compiling …` lines — a 6‑line log that was stable across four consecutive runs. Two secondary states also occur, and neither is a discrepancy: (i) the *first* incremental run taken after the `kitten` binary needs relinking additionally prints a single `kitty/tools/cmd` line — the Go **main‑package** relink emitted by `go build -v` — after which it settles back to the notice‑only 6‑line output; and (ii) a fully **cold** Go build cache makes `go build -v` print its entire transitive dependency set (287 import paths in one observed run, of which exactly one ends in `kitty/tools/cmd`, the main package). In every warm‑cache case **0** of the 85 C translation units recompile; the full `[N/85] Compiling …` C sequence appears only on a genuine from‑scratch build (after removing `build/` and `kitty/fast_data_types.so`). This is expected build‑system behavior, not a discrepancy; independently regenerating the from‑scratch log requires deleting the git‑ignored `build/` and `kitty/fast_data_types.so` again first. Either way the build exits `0` and yields the same `./kitty/launcher/kitty` and `kitty/fast_data_types.so` artifacts. (Only git‑ignored artifacts are affected by a rebuild — no tracked source file changes.)

### A.2 — `/tmp/kobs_resize.py` (full)

Exercises the genuine `ioctl(TIOCSWINSZ)` resize path via the test PTY (mirrors `pty_resize()` → `ioctl(fd, TIOCSWINSZ, dim)` at `kitty/child-monitor.c:577-579`). §2.4 shows the excerpt.

```python
# /tmp/kobs_resize.py
# Exercises the genuine ioctl(TIOCSWINSZ) path via the test PTY's
# set_window_size(), which issues the real fcntl.ioctl(master_fd, TIOCSWINSZ, ...)
# mirroring pty_resize() -> ioctl(fd, TIOCSWINSZ, dim) at kitty/child-monitor.c:577-579.
import fcntl, termios, struct
from kitty_tests import BaseTest

class T(BaseTest):
    def runTest(self):
        pass

t = T()
pty = t.create_pty(['cat'], cols=80, lines=25)
print("initial: screen.lines =", pty.screen.lines, "screen.columns =", pty.screen.columns)
pty.set_window_size(rows=20, columns=40)      # -> screen.resize + real ioctl(TIOCSWINSZ)
print("after set_window_size(rows=20, columns=40): screen.lines =", pty.screen.lines, "screen.columns =", pty.screen.columns)
r = fcntl.ioctl(pty.master_fd, termios.TIOCGWINSZ, struct.pack('HHHH', 0, 0, 0, 0))
rows, cols, xpix, ypix = struct.unpack('HHHH', r)
print("kernel TIOCGWINSZ readback: rows =", rows, "cols =", cols, "xpix =", xpix, "ypix =", ypix)
```

Observed output (identical across two runs):

```console
$ ./kitty/launcher/kitty +launch /tmp/kobs_resize.py
initial: screen.lines = 25 screen.columns = 80
after set_window_size(rows=20, columns=40): screen.lines = 20 screen.columns = 40
kernel TIOCGWINSZ readback: rows = 20 cols = 40 xpix = 400 ypix = 400
```

### A.3 — `/tmp/kobs_osc133.py` (full)

Feeds a stream interleaving OSC 133 shell‑integration markers with ordinary text through the **real** parser (`parse_bytes` = `test_create_write_buffer` → `test_commit_write_buffer` → `test_parse_written_data`, exactly what `read_bytes` feeds). §4.2 shows the excerpt.

```python
# /tmp/kobs_osc133.py
# Feeds a stream interleaving OSC 133 markers with ordinary text through the
# REAL parser (parse_bytes = test_create_write_buffer -> test_commit_write_buffer
# -> test_parse_written_data, exactly what read_bytes feeds).
from kitty.options.types import defaults
from kitty.fast_data_types import set_options, Screen
from kitty_tests import Callbacks, parse_bytes
set_options(defaults)

def run(stream, label):
    c = Callbacks()
    s = Screen(c, 25, 80, 100, 10, 20, 0, c)
    parse_bytes(s, stream.encode('utf-8'))
    print("[%s]" % label)
    print("  input        =", repr(stream))
    print("  screen.line0 =", repr(str(s.line(0))))
    print("  last_cmdline =", repr(c.last_cmd_cmdline))
    print("  exit_status  =", c.last_cmd_exit_status)

run('\x1b]133;A\x07host$ \x1b]133;C;cmdline=ls -la /etc\x07out1  out2\r\n\x1b]133;D;0\x07', "exit 0")
run('\x1b]133;A\x07host$ \x1b]133;C;cmdline=false\x07out1  out2\r\n\x1b]133;D;1\x07', "exit 1")
```

Observed output (identical across two runs):

```console
$ ./kitty/launcher/kitty +launch /tmp/kobs_osc133.py
[exit 0]
  input        = '\x1b]133;A\x07host$ \x1b]133;C;cmdline=ls -la /etc\x07out1  out2\r\n\x1b]133;D;0\x07'
  screen.line0 = 'host$ out1  out2'
  last_cmdline = 'ls'
  exit_status  = 0
[exit 1]
  input        = '\x1b]133;A\x07host$ \x1b]133;C;cmdline=false\x07out1  out2\r\n\x1b]133;D;1\x07'
  screen.line0 = 'host$ out1  out2'
  last_cmdline = 'false'
  exit_status  = 1
```

### A.4 — `/tmp/kobs_realbash.py` (full)

Forks **real** bash with shell integration enabled and reads its output through the genuine `os.read(master_fd)` → `parse_bytes` path (`process_input_from_child()`), mirroring `kitty_tests/shell_integration.py:test_bash_integration`. §4.3 shows the excerpt. Note the in‑place `argv` mutation documented in the script header. §4.3 also documents the initial failure this script header guards against.

```python
# /tmp/kobs_realbash.py
# Forks REAL bash with shell integration enabled and reads its output through the
# genuine os.read(master_fd) -> parse_bytes path (process_input_from_child()).
# Mirrors kitty_tests/shell_integration.py:test_bash_integration. NOTE:
# safe_env_for_running_shell() MUTATES the argv list in place (it appends
# "--posix" so bash sources $ENV=kitty.bash); the SAME argv object must then be
# handed to create_pty(). We wait for integration to finish loading (cursor
# becomes CURSOR_BEAM) before sending commands so the command-start (OSC 133 C)
# DEBUG-trap hook is installed and the command-end (OSC 133 D;$?) status is captured.
import os, sys, tempfile
from kitty_tests import BaseTest
from kitty_tests.shell_integration import safe_env_for_running_shell
from kitty.fast_data_types import CURSOR_BEAM

class T(BaseTest):
    def runTest(self):
        pass

t = T()
ps1 = 'PROMPT> '
home_dir = os.path.realpath(tempfile.mkdtemp())
argv = ['bash']                                                # ONE list, mutated below
env = safe_env_for_running_shell(argv, home_dir, rc='PS1="%s"' % ps1, shell='bash', with_kitten=False)
env['KITTY_RUNNING_SHELL_INTEGRATION_TEST'] = '1'
pty = t.create_pty(argv, cwd=home_dir, env=env)                # forks real bash (argv now ['bash','--posix'])
# Wait until shell integration has fully loaded (it switches the cursor to a beam).
pty.wait_till(lambda: pty.screen.cursor.shape == CURSOR_BEAM)
pty.wait_till(lambda: pty.screen_contents().count(ps1) == 1)
print("startup screen.line(0)          =", repr(str(pty.screen.line(0))))

for cmd in ('echo hello-kitty', 'false'):
    pty.callbacks.clear()
    pty.send_cmd_to_child(cmd)
    pty.wait_till(lambda: pty.callbacks.last_cmd_exit_status != sys.maxsize)
    print("after %-19s cmdline = %s exit_status = %s" % (
        repr(cmd) + ':', repr(pty.callbacks.last_cmd_cmdline), pty.callbacks.last_cmd_exit_status))
```

Observed output (identical across two runs):

```console
$ ./kitty/launcher/kitty +launch /tmp/kobs_realbash.py
startup screen.line(0)          = 'PROMPT> '
after 'echo hello-kitty': cmdline = 'echo hello-kitty' exit_status = 0
after 'false':            cmdline = 'false' exit_status = 1
```

### A.5 — `/tmp/kobs_backpressure.py` (full)

Commits a 4 MiB surge into the parser write buffer **without parsing** (simulating a stalled main thread) to force the 1 MiB `BUF_SZ` cap and observe backpressure. §4.4 shows the excerpt.

```python
# /tmp/kobs_backpressure.py
# Commits a 4 MiB surge into the parser write buffer WITHOUT parsing (simulating a
# stalled main thread) to force the 1 MiB BUF_SZ cap and observe backpressure.
from kitty.options.types import defaults
from kitty.fast_data_types import set_options, Screen, VT_PARSER_BUFFER_SIZE
from kitty_tests import Callbacks
set_options(defaults)

c = Callbacks()
s = Screen(c, 25, 80, 100, 10, 20, 0, c)
chunk = b'x' * (64 * 1024)
attempted = 4 * 1024 * 1024                       # 4 MiB
sent = 0
total_committed = 0
while sent < attempted:
    dest = s.test_create_write_buffer()
    if len(dest) == 0:                            # no space -> backpressure
        break
    n = s.test_commit_write_buffer(chunk, dest)   # DO NOT parse
    total_committed += n
    sent += len(chunk)

print("VT_PARSER_BUFFER_SIZE           =", VT_PARSER_BUFFER_SIZE)
print("attempted surge                 =", attempted, "bytes (4 MiB)")
print("total committed before FULL     =", total_committed, "bytes")
print("available space now (len buf)   =", len(s.test_create_write_buffer()))
```

Observed output (identical across two runs):

```console
$ ./kitty/launcher/kitty +launch /tmp/kobs_backpressure.py
VT_PARSER_BUFFER_SIZE           = 1048576
attempted surge                 = 4194304 bytes (4 MiB)
total committed before FULL     = 1048576 bytes
available space now (len buf)   = 0
```

### A.6 — `/tmp/kobs_mode2026.py` (full)

Drives the real parser and captures the bytes written back to the child (`Callbacks.wtcbuf`) in reply to a DECRQM query (`CSI ? 2026 $ p`) around enabling/disabling synchronized output (DEC private mode 2026). §5.2 shows the excerpt.

```python
# /tmp/kobs_mode2026.py
# Drives the real parser and captures the bytes written back to the child
# (Callbacks.wtcbuf) in reply to a DECRQM query (CSI ? 2026 $ p) around
# enabling/disabling synchronized output (DEC private mode 2026).
from kitty.options.types import defaults
from kitty.fast_data_types import set_options, Screen
from kitty_tests import Callbacks, parse_bytes
set_options(defaults)

c = Callbacks()
s = Screen(c, 25, 80, 100, 10, 20, 0, c)

def decrqm(label):
    c.wtcbuf = b''
    parse_bytes(s, b'\x1b[?2026$p')
    print("  %-38s DECRQM reply = %r" % (label, c.wtcbuf))

decrqm("initial")
parse_bytes(s, b'\x1b[?2026h')              # enable  -> pause
decrqm("after CSI ?2026h (enable/pause)")
parse_bytes(s, b'while-paused-text')        # text arrives WHILE paused
print("  screen.line(0) while paused          =", repr(str(s.line(0))))
parse_bytes(s, b'\x1b[?2026l')              # disable -> resume
decrqm("after CSI ?2026l (disable/resume)")
print("  screen.pause_rendering(100)          =", s.pause_rendering(100))
decrqm("after pause_rendering(100)")
```

Observed output (identical across two runs):

```console
$ ./kitty/launcher/kitty +launch /tmp/kobs_mode2026.py
  initial                                DECRQM reply = b'\x1b[?2026;2$y'
  after CSI ?2026h (enable/pause)        DECRQM reply = b'\x1b[?2026;1$y'
  screen.line(0) while paused          = 'while-paused-text'
  after CSI ?2026l (disable/resume)      DECRQM reply = b'\x1b[?2026;2$y'
  screen.pause_rendering(100)          = True
  after pause_rendering(100)             DECRQM reply = b'\x1b[?2026;1$y'
```
