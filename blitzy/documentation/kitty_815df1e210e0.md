# How Keyboard Input Flows Through Kitty — A Runtime-Observation-Driven Investigation

This document answers, **from captured runtime observation**, how a keypress made inside
Kitty's default shell travels through Kitty's core components until the screen updates. It
was produced by building Kitty from this repository in its default configuration, launching
the built binary under its own tracing flags (`--debug-input` and `--debug-rendering`),
pressing a small set of simple keys inside a real `bash` child, and attributing every trace
line that consistently appeared to the exact source file and line that emits it.

- **Source branch:** `kitty_815df1e210e0` (git `HEAD` `815df1e21`).
- **Built binary:** `kitty 0.35.2 created by Kovid Goyal`.
- **Method:** real X11 key events injected into the real Kitty window (**not** remote control,
  **not** any debug/injection hook), captured twice and confirmed byte-for-byte identical.
- **Evidence discipline:** every behavioral claim below sits next to the observed output that
  supports it and carries an exact `file:line` locator. Anything that comes from reading the
  code rather than from a captured trace is explicitly marked **(inferred from code)**.

> **A note on the embedded evidence.** All trace blocks in this document are the *actual,
> unedited* output captured from the running binary in this environment (ANSI color codes
> stripped for readability; the `[N.NNN]` timestamps are wall-clock-relative and vary
> run-to-run — only the line *content* is deterministic). The investigation was reproduced
> independently and the key-event content matched byte-for-byte across two runs (see
> **Stability / Determinism**).

---

## Short Answer

Pressing a key in Kitty produces a consistent **two-line-per-key signature** in the
`--debug-input` trace: a *reception* line, immediately followed by a *processing* line. That
ordering is the empirical spine of the whole answer.

- **RECEIVE FIRST:** the vendored **GLFW X11 backend** sees the key first. Its XKB translation
  routine `glfw_xkb_handle_key_event` (`glfw/xkb_glfw.c:864`) converts the raw X11 hardware
  keycode into a keysym/text via `libxkbcommon` and emits the `Press`/`Release … xkb_keycode …`
  line (`glfw/xkb_glfw.c:875`). Kitty's GLFW keyboard callback `key_callback`
  (`kitty/glfw.c:430`) then forwards the translated event into Kitty core via `on_key_input`
  (`kitty/glfw.c:439`).
- **INTERMEDIATE PROCESSING:** `on_key_input` (`kitty/keys.c:166`) emits the `on_key_input: …`
  line (`kitty/keys.c:176`), runs a shortcut check through the Python `Boss`, encodes the event
  with `encode_glfw_key_event` (`kitty/key_encoding.c:414`), and either **queues** the bytes
  for the child via `schedule_write_to_child` (`kitty/child-monitor.c:372`) or ignores the
  event. That call copies the bytes into the screen's `write_buf` and wakes the I/O loop; the
  actual `write(2)` to the child PTY happens on the **I/O thread** (`write_to_child`,
  `kitty/child-monitor.c:1540`, whose body drains `write_buf` at `kitty/child-monitor.c:1443`).
  The shell echoes those bytes; the same **I/O thread** reads them (`io_loop`
  `kitty/child-monitor.c:1481` → `read_bytes` `kitty/child-monitor.c:1337`) and the VT parser
  (`kitty/vt-parser.c:1496`) mutates the in-memory screen grid (`kitty/screen.c`). *(This
  read → parse → mutate segment is **source-inferred** — those functions emit no trace line;
  `--debug-input` observes only reception and the `on_key_input` decision.)*
- **DISPLAY UPDATE:** on the **main thread**, the loop tick `process_global_state`
  (`kitty/child-monitor.c:1224`) calls `parse_input` (`kitty/child-monitor.c:1236`)
  **immediately before** `render` (`kitty/child-monitor.c:1237`); rendering issues OpenGL draw
  calls via `draw_cells` (`kitty/shaders.c:1009`) and presents the frame by swapping buffers,
  `swap_window_buffers` (`kitty/glfw.c:1802`) → `glfwSwapBuffers` (`kitty/glfw.c:1803`).
  *(This per-frame parse → render → draw → swap mechanism is **source-inferred**; what is
  directly **observed** under `--debug-rendering` is the **startup** GL/window/child lifecycle —
  including the GL context line at `kitty/gl.c:72` — which corroborates that the render path is
  active but **does not log a frame for each key**.)*

Concrete observed values (default legacy keyboard mode): `a`/`b`/`c` are sent as their literal
UTF-8 text; `Shift+b` is sent as `B`; `Enter` is sent as the carriage-return byte `0x0d`;
`Ctrl+a` is sent as the C0 control byte `0x1`; every key *release* and every standalone
*modifier* press is ignored. Of the 16 `on_key_input` events captured, **6 were sent** to the
child and **10 were ignored**.

---

## How This Was Observed (Observation Setup)

### Environment and branch

| Item | Value (as observed in this environment) |
|------|------------------------------------------|
| Source branch / HEAD | `kitty_815df1e210e0` / `815df1e21` |
| Built version banner | `kitty 0.35.2 created by Kovid Goyal` |
| C compiler | gcc 13.3.0 |
| Python (build/runtime interpreter) | 3.12.3 |
| Go | 1.22.2 (satisfies `go 1.22` in `go.mod:3`) |
| Display | Xvfb `:99`, 1280x800x24 (headless) |
| OpenGL | `4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.2`, renderer `llvmpipe (LLVM 20.1.2, 256 bits)` |

> **Toolchain provenance.** The versions above are the exact toolchain of the canonical
> build/observation environment — the reference Ubuntu 24.04.2 container — captured with the
> commands shown immediately below. Two points of note: (1) Kitty documents a Python floor of
> `>= 3.8` (`docs/build.rst:83`) and its CI tests up to Python 3.11 (`.github/workflows/ci.yml:85`),
> yet **Python 3.12.3 built and ran Kitty 0.35.2
> successfully here** — `setup.py build` completed with no errors and produced a working
> launcher, so the newer interpreter is reported as an actual, verified success rather than a
> documented-supported version. (2) Go builds only the standalone `kitten` CLI and is **not on
> the observed input→display path**; `1.22.2` satisfies the `go 1.22` directive in `go.mod:3`.
> The `renderer llvmpipe (LLVM 20.1.2, …)` string is reported by `glxinfo` (a separate X11/GL
> probe), **not** by Kitty's own trace, which prints only the GL *version* line (see §3.1). The
> `[N.NNN]` timestamps in every trace block are wall-clock offsets from Kitty's monotonic clock
> (`monotonic_t_to_s_double(monotonic())`, `kitty/gl.c:72`) and therefore vary run-to-run;
> determinism is verified on the timestamp-stripped, normalized output (see **Stability /
> Determinism**).

The exact versions in the table were produced by these commands (run inside the canonical
environment), which identify the precise interpreter and toolchain rather than relying on a
bare `python3`:

```console
$ . /etc/os-release; echo "$PRETTY_NAME"
Ubuntu 24.04.2 LTS
$ gcc --version | head -1
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
$ command -v python3; python3 --version
/usr/bin/python3
Python 3.12.3
$ command -v go; go version
/usr/lib/go-1.22/bin/go
go version go1.22.2 linux/amd64
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

The OpenGL row is produced by Kitty's own `--debug-rendering` output (see §3.1):

```text
[0.122] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.2' Detected version: 4.5
```

### Build (default configuration)

Kitty ships source only; the runnable launcher must be built from source. The build is run in
its default, canonical configuration:

```sh
# Toolchain: gcc 13.3.0; python3 = /usr/bin/python3 (CPython 3.12.3);
# go = /usr/lib/go-1.22/bin/go (1.22.2, builds the kitten CLI only, off the input→display path)
export PATH=/usr/lib/go-1.22/bin:$PATH
CI=true python3 setup.py build          # -> kitty/launcher/kitty  (kitty 0.35.2)
```

This produces the launcher documented in `docs/build.rst:22` (`:file:` `kitty/launcher/kitty`).
The launcher starts the Python runtime, which loads the compiled C core as
`kitty/fast_data_types.so` (the child-monitor, VT parser, screen model, and shaders are all
compiled into that extension module). The version banner:

```text
kitty 0.35.2 created by Kovid Goyal
```

### Headless GPU context

Kitty is GPU-first and has **no CPU text-drawing fallback**, so it needs a real window and an
OpenGL context to run at all. A headless context is provided with Xvfb plus Mesa software
OpenGL (llvmpipe):

```sh
# Fresh PRIVATE authenticated display: access control stays ON (no -ac), and the
# server resets/exits normally on teardown (no -noreset). This line is excerpted
# from the complete fail-fast, trap-guarded, two-run workflow in Appendix B.
export XAUTHORITY="$(mktemp /tmp/obs_kitty.xauth.XXXXXX)"
xauth -f "$XAUTHORITY" add :99 . "$(mcookie)"
Xvfb :99 -screen 0 1280x800x24 -auth "$XAUTHORITY" +extension GLX +render &
export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe
```

### Launch commands (canonical, legacy default keyboard mode)

```sh
# input tracing:
./kitty/launcher/kitty --debug-input --config NONE -o confirm_os_window_close=0 bash --norc --noprofile
# separately, for the display lifecycle:
./kitty/launcher/kitty --debug-rendering --config NONE -o confirm_os_window_close=0 bash --norc --noprofile
```

`--config NONE` means no `kitty.conf` is read, so the observed behavior is canonical (the
default *legacy* keyboard mode: `mDECCKM` and the key-encoding flags are unset). The child is a
real `bash --norc --noprofile`, so the typed bytes are consumed and echoed by an actual shell.

`--debug-input` is aliased `--debug-keyboard`; `--debug-rendering` is aliased `--debug-gl`
(confirmed from `kitty --help`). The trace lines are emitted by `debug()` / `printf` calls
gated on `OPT(debug_keyboard)` / `global_state.debug_rendering`, so **no source change is
needed** to observe them.

### Injecting real X11 key events (the genuine reception path)

```sh
# Launch Kitty in the BACKGROUND so injection can proceed, and capture its PID.
./kitty/launcher/kitty --debug-input --config NONE -o confirm_os_window_close=0 \
  bash --norc --noprofile >/tmp/obs_kitty.input.log 2>&1 &
KITTY_PID=$!
# Bounded discovery CORRELATED to that PID (never a stale/unrelated window): accept
# only a window whose _NET_WM_PID equals the launched Kitty PID.
WID=""
for i in $(seq 1 150); do
  for w in $(xdotool search --class kitty 2>/dev/null || true); do
    [ "$(xdotool getwindowpid "$w" 2>/dev/null)" = "$KITTY_PID" ] && { WID="$w"; break; }
  done
  [ -n "$WID" ] && break
  kill -0 "$KITTY_PID" 2>/dev/null || break     # launched Kitty died -> stop
  sleep 0.1
done
[ -n "$WID" ] || { echo "no correlated kitty window"; kill "$KITTY_PID" 2>/dev/null; exit 1; }
xdotool windowactivate --sync "$WID"; xdotool windowfocus --sync "$WID"
for k in a b c Return ctrl+a shift+b; do xdotool key --window "$WID" "$k"; sleep 0.5; done
# (Complete fail-fast / trap-teardown / two-run form: Appendix B.)
```

Keys exercised: `a`, `b`, `c` (unmodified printable keys); `Enter` (a control key that produces
no text); and `Ctrl+a` and `Shift+b` (modifier combinations). Injecting via `xdotool` delivers
genuine X11 key events into the window, so they travel Kitty's **real reception path**
(X11 → XKB → GLFW → `keys.c`) rather than any bypass.

### The two-line-per-key trace signature (the empirical spine)

Every key press consistently produces two lines, in this order — a *reception* line from the
XKB/GLFW layer followed by a *processing* line from `kitty/keys.c`. For the `a` press:

```text
[0.808] Press xkb_keycode: 0x26 clean_sym: a composed_sym: a text: a mods: none glfw_key: 97 (a) xkb_key: 97 (a)
[0.808] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
```

The first line is emitted by `glfw/xkb_glfw.c:875`; the second by `kitty/keys.c:176`. This
ordering — reception first, processing second — is exactly what the runtime reveals about
"which parts receive input first" versus "which parts handle intermediate processing." The
complete 32-line trace for all six keys (press + release) is in **Appendix A**.

---

## 1. RECEIVE FIRST — which parts of the system receive the input first

**Observed answer:** the first component to see each key is the **vendored GLFW X11 backend's
XKB translation**, and the second is **Kitty's GLFW keyboard callback**. This is proven by the
fact that the *reception* line always appears before the `on_key_input` processing line for
every key (see the two-line signature above and the full trace in Appendix A).

### 1.1 XKB translation in the GLFW backend — `glfw/xkb_glfw.c`

The routine `glfw_xkb_handle_key_event` (`glfw/xkb_glfw.c:864`) receives the raw X11 hardware
keycode and translates it — via `libxkbcommon` — into a keysym and, where applicable, text. It
emits the reception line at `glfw/xkb_glfw.c:875`:

```c
debug("%s xkb_keycode: 0x%x ", action == GLFW_RELEASE ? "\x1b[32mRelease\x1b[m" : "\x1b[31mPress\x1b[m", xkb_keycode);
```

(Quoted verbatim from source. The `\x1b[32m…\x1b[m` / `\x1b[31m…\x1b[m` wrappers are the
green/red **ANSI color codes** around the literal words `Release`/`Press`; those color codes are
exactly what was stripped from every trace block in this document, which is why the traces read
plain `Press`/`Release`.)

The observed reception lines for the four representative keys (from the same run; full set in
Appendix A):

```text
[0.808] Press xkb_keycode: 0x26 clean_sym: a composed_sym: a text: a mods: none glfw_key: 97 (a) xkb_key: 97 (a)
[2.350] Press xkb_keycode: 0x24 clean_sym: Return composed_sym: Return mods: none glfw_key: 57345 (ENTER) xkb_key: 65293 (Return)
[2.873] Press xkb_keycode: 0x26 clean_sym: a composed_sym: a mods: ctrl glfw_key: 97 (a) xkb_key: 97 (a)
[3.401] Press xkb_keycode: 0x38 clean_sym: b composed_sym: B text: B mods: shift glfw_key: 98 (b) xkb_key: 98 (b) shifted_key: 66 (B)
```

Cause → effect that this layer is demonstrably responsible for:

- **Hardware keycode → keysym/text happens here.** The line reports the raw `xkb_keycode`
  (e.g. `0x26` for the physical `a` key) alongside the resolved `clean_sym`, `composed_sym`,
  `text`, and the GLFW key number. That resolution is XKB's job, done before Kitty core is
  ever called.
- **Shift resolution happens here, at reception.** For `Shift+b` the reception line already
  shows `composed_sym: B … text: B … shifted_key: 66 (B)`. The uppercase `B` is produced by the
  XKB layer, not by later processing — the evidence is that `text: B` is present on the
  *reception* line itself.
- **Modifier state is attached here.** For the `a` press that follows `Ctrl` being held, the
  reception line reports `mods: ctrl`; the plain `a` reception line reports `mods: none`.

### 1.2 Handoff into Kitty core — `kitty/glfw.c`

Kitty registers `key_callback` (`kitty/glfw.c:430`) as GLFW's keyboard callback. Once the
window is ready, it forwards the translated event into Kitty's core at `kitty/glfw.c:439`:

```c
if (is_window_ready_for_callbacks() && !ev->fake_event_on_focus_change) on_key_input(ev);
```

The observable proof of this handoff is temporal: the `on_key_input:` line (emitted inside
`on_key_input`) appears on the very next line after each reception line, with matching key data
(e.g. reception `glfw_key: 97 (a)` → processing `glfw key: 0x61`, which is `97`).

### 1.3 IME / IBus — present in code but **not** on this path (inferred from code)

`glfw/ibus_glfw.c` provides the IME (IBus) hook on the Linux input path. In this default run no
input method is active, so it never fires — there is no IME line in the captured trace. Its
existence and role are noted for completeness and are **(inferred from code, not observed)**.

---

## 2. INTERMEDIATE PROCESSING — which parts handle the intermediate processing

**Observed answer:** the second trace line of every key comes from `on_key_input` in
`kitty/keys.c`, which is the decision point. It logs the event, checks for a configured
shortcut through the Python `Boss`, encodes the event according to the current keyboard mode
via `kitty/key_encoding.c`, and then either **queues** the resulting bytes for the child (a
main-thread enqueue that is actually written to the PTY on the I/O thread) or ignores the
event. The child echoes the bytes back; Kitty's I/O thread reads them and the VT parser turns
them into screen-model mutations.

### 2.1 The dispatch decision — `kitty/keys.c`

`on_key_input` (`kitty/keys.c:166`) emits the `on_key_input:` line at `kitty/keys.c:176`, then
proceeds through a fixed sequence:

1. **No-active-window guard** (`kitty/keys.c:182`) — skipped here (a window is active).
2. **IME state switch** (`kitty/keys.c:187`–`215`). In this run the state is `GLFW_IME_NONE`
   (visible as `state: 0` in every `on_key_input` line), so the switch simply breaks through to
   normal processing.
3. **Shortcut check** — on `PRESS`/`REPEAT` the macro `dispatch_key_event(dispatch_possible_special_key)`
   calls into the Python `Boss` (`kitty/boss.py` → `dispatch_possible_special_key`; the key-map
   and action definitions live in `kitty/keys.py`). If a shortcut matches, `kitty/keys.c:231`
   prints `handled as shortcut` and returns. **None of the six simple keys matched**, so every
   one fell through — there is no `handled as shortcut` line anywhere in the capture.
4. **Encoding** (`kitty/keys.c:251`):

   ```c
   int size = encode_glfw_key_event(ev, screen->modes.mDECCKM, screen_current_key_encoding_flags(screen), encoded_key);
   ```

5. **Delivery branch**, one of three:
   - `kitty/keys.c:253` — `schedule_write_to_child(w->id, 1, text, strlen(text));` then
     `debug("sent key as text to child: %s\n", text);` (`kitty/keys.c:254`);
   - `kitty/keys.c:259` — `schedule_write_to_child(w->id, 1, encoded_key, size);` then, under the
     `if (OPT(debug_keyboard))` gate (`kitty/keys.c:260`), the label print
     `debug("sent encoded key to child: ");` (`kitty/keys.c:261`) followed by a per-byte loop
     (`kitty/keys.c:262`–`267`) that prints each non-printable byte as `debug("0x%x ", …)`
     (`kitty/keys.c:266`) — that per-byte `0x%x ` format is exactly why the encoded trace lines
     end with a trailing space;
   - `kitty/keys.c:271` — otherwise `debug("ignoring as keyboard mode does not support encoding this event")`.

### 2.2 Encoding: legacy vs. Kitty protocol — `kitty/key_encoding.c`

`encode_glfw_key_event` (`kitty/key_encoding.c:414`) performs the legacy-vs-Kitty-protocol
encoding. In the default *legacy* mode the progressive-enhancement flags are all `0`
(`disambiguate=1, report_all_event_types=2, report_alternate_key=4, report_text=8,
embed_text=16` — **none set**). The following branches, read from the source, explain every
observed value **(cause → effect, corroborated by the observed decision lines below)**:

- `if (!ev.report_text && is_modifier_key(e->key)) return 0;` (`kitty/key_encoding.c:425`)
  ⇒ standalone `Control_L` / `Shift_L` presses **encode to nothing** and are ignored.
- `send_text_standalone = !ev.report_text` is `true` (`kitty/key_encoding.c:428`), so
  `if (send_text_standalone && ev.has_text && (PRESS||REPEAT)) return SEND_TEXT_TO_CHILD;`
  (`kitty/key_encoding.c:437`) ⇒ printable text keys (`a`, `b`, `c`, and the shifted `B`) are
  sent as **literal UTF-8 text**. (`SEND_TEXT_TO_CHILD` is `INT_MIN`, `kitty/keys.h:15`.)
- Otherwise `return encode_key(&ev, output);` (`kitty/key_encoding.c:439`) ⇒ `Enter` →
  `0x0d`, `Ctrl+a` → `0x1` (a C0 control byte).
- For `RELEASE`, `encode_key` short-circuits: `if (!ev->report_all_event_types && ev->action == RELEASE) return 0;`
  (`kitty/key_encoding.c:368`) ⇒ every release **encodes to nothing** and is ignored.
- The structured `CSI u` "Kitty keyboard protocol" path is `serialize()`
  (`kitty/key_encoding.c:65`) — the opt-in enhanced encoding, **not exercised** by this default
  run.

### 2.3 The observed encoding outcomes (the six events that produced child output)

These are the six `PRESS` events whose decision was to send bytes to the child. This block is
**verbatim** from the run (note that the two *encoded* lines end with a trailing space, exactly
as the source's per-byte `0x%x ` formatting loop produces it):

```text
[0.808] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
[1.318] on_key_input: glfw key: 0x62 native_code: 0x62 action: PRESS mods: none text: 'b' state: 0 sent key as text to child: b
[1.834] on_key_input: glfw key: 0x63 native_code: 0x63 action: PRESS mods: none text: 'c' state: 0 sent key as text to child: c
[2.350] on_key_input: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd 
[2.873] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: 0x1 
[3.401] on_key_input: glfw key: 0x62 native_code: 0x62 action: PRESS mods: shift text: 'B' state: 0 sent key as text to child: B
```

| Key pressed | What Kitty sent to the child | Why (cause → effect) |
|-------------|------------------------------|----------------------|
| `a` | UTF-8 text `a` | printable text key; `send_text_standalone` path (`kitty/key_encoding.c:437`) |
| `b` | UTF-8 text `b` | same as `a` |
| `c` | UTF-8 text `c` | same as `a` |
| `Shift+b` | UTF-8 text `B` | XKB already produced `text: B`; sent as text (`kitty/key_encoding.c:437`) |
| `Enter` | byte `0x0d` (carriage return) | no text; legacy `encode_key` (`kitty/key_encoding.c:439`) |
| `Ctrl+a` | byte `0x1` (C0 control) | no text; legacy `encode_key` maps Ctrl+letter to its control code (`kitty/key_encoding.c:439`) |

This matches Kitty's documented legacy behavior. `docs/keyboard-protocol.rst:110` states that
in **"legacy compatibility mode (the default)"** non-text key events use legacy escape codes,
while text-producing keys are "sent directly as UTF-8 encoded bytes" (`docs/keyboard-protocol.rst:107`).
The structured `CSI u` protocol is described there as an opt-in progressive enhancement — i.e.
what we observed is the canonical default, not an environment artifact.

### 2.4 Delivery to the child PTY — the main-thread queue and the I/O-thread `write(2)`

Both send branches call `schedule_write_to_child` (`kitty/child-monitor.c:372`). On the
**main thread** this does *not* write to the PTY directly — it copies the bytes into the target
screen's `write_buf` and wakes the I/O loop (`wakeup_io_loop`, `kitty/child-monitor.c:363`). The
actual `write(2)` to the child's pseudo-terminal happens on the **I/O thread**: when the child
fd becomes writable, `io_loop` calls `write_to_child` (`kitty/child-monitor.c:1540`), whose body
(`kitty/child-monitor.c:1443`) drains `write_buf` via
`write(fd, screen->write_buf + written, …)` (`kitty/child-monitor.c:1448`). So the handoff is:
**queue on the main thread → `write(2)` on the I/O thread.**

The child's pseudo-terminal is **allocated in Python**, not in `child.c`: `os.openpty()` at
`kitty/child.py:171` returns the `master, slave` pair (`kitty/child.py:281`), and the master is
retained as `self.child_fd = master` (`kitty/child.py:338`). `kitty/child.c` then **consumes the
slave** inside the forked child to establish the session and exec the shell — `fork`
(`kitty/child.c:97`), `setsid` (`kitty/child.c:123`), `TIOCSCTTY` (`kitty/child.c:129`), `dup2`
of the slave onto stdin/stdout/stderr (`kitty/child.c:138`–`145`), and `execvp`
(`kitty/child.c:159`). The Python `Window` object (`kitty/window.py`) logs `Child launched` at
`kitty/window.py:871` (gated on `debug_rendering`; see the display section). **(The
queue→I-O-write handoff and the PTY setup are inferred from code; `--debug-input` proves the
bytes were scheduled, and the child's echo — read back in §2.5 — proves they reached the
shell.)**

### 2.5 Reading the echo and parsing into the screen model

The shell writes the typed/echoed bytes back through the PTY. Kitty's **I/O thread** reads them:
`io_loop` (`kitty/child-monitor.c:1481`, whose source comment is literally `// The I/O thread
loop`) calls `read_bytes` (`kitty/child-monitor.c:1337`). The bytes are then parsed by the VT
parser — `parse_worker` (`kitty/vt-parser.c:1496`) → `run_worker` (`kitty/vt-parser.c:1417`) —
which turns them into terminal operations that mutate the in-memory grid in `kitty/screen.c`
(backed by `kitty/line.c` / `kitty/line-buf.c`). **(The I/O-thread read ownership and the
parser/screen-model internals here are all source-inferred — `read_bytes` and the parser emit no
trace line. `--debug-input` observes only that the bytes were scheduled to the child, and
`--debug-rendering` confirms the render path is active; neither logs the per-key read/parse nor a
per-key frame.)**

---

## 3. DISPLAY UPDATE — how the updated display is ultimately produced

**Observed answer:** the display is produced by the GPU render path. Under `--debug-rendering`
we directly **observe** the **startup** window/GL/child lifecycle (the GL context line,
`OS Window created`, `Child launched`); this corroborates that the GPU render path is active but
**does not log a frame for each key**. The per-frame mechanism itself — the main-loop tick
parsing any pending child output and then rendering each OS window, issuing OpenGL draw calls and
swapping the window's buffers to present the frame — is **source-inferred** (attributed with
`file:line` in §3.2).

### 3.1 Observed lifecycle evidence

Running the binary with `--debug-rendering` produced these lines (verbatim, ANSI stripped,
shown in timestamp order):

```text
[0.122] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.2' Detected version: 4.5
[0.148] OS Window created
[0.161] Child launched
```

The command that produced it (the render leg of the complete Appendix B workflow, which
launches Kitty in the background, captures its PID, correlates the window to that PID, and tears
it down via a trap):

```sh
./kitty/launcher/kitty --debug-rendering --config NONE -o confirm_os_window_close=0 bash --norc --noprofile
```

(The run also emitted `Failed to open systemd user bus with error: No such file or directory` — this
is unrelated **environment noise** from the headless container's missing systemd user bus, not
part of Kitty's render pipeline.)

### 3.2 Attribution (with `file:line`)

- **GL context established and validated — `kitty/gl.c`.** The `GL version string …` line is
  printed at `kitty/gl.c:72`:

  ```c
  if (global_state.debug_rendering) printf("[%.3f] GL version string: %s\n", monotonic_t_to_s_double(monotonic()), gl_version_string());
  ```

  and the `'…' Detected version: X.Y` portion is formatted by `gl_version_string()`
  (`kitty/gl.c:47`). The observed `4.5 (Core Profile)` is comfortably above Kitty's required
  minimum, which is **platform-specific**: OpenGL **3.1 on Linux/non-Apple** and **3.3 on
  Apple** (`kitty/data-types.h:19`–`25`, `OPENGL_REQUIRED_VERSION_MAJOR 3` with `MINOR 1`
  non-Apple / `MINOR 3` on `__APPLE__`). This is a Linux/X11 run, so the applicable floor is
  3.1; the observed 4.5 exceeds both, so the render path exercised is **representative**, not a
  degraded fallback.
- **Window created — `kitty/glfw.c`.** `OS Window created` is printed at `kitty/glfw.c:1321`.
- **Child launched — `kitty/window.py`.** The Python `Window` logs `Child launched` at
  `kitty/window.py:871` (gated on `boss.args.debug_rendering`).
- **The render tick — `kitty/child-monitor.c`.** The main-loop tick `process_global_state`
  (`kitty/child-monitor.c:1224`) calls `parse_input` (`kitty/child-monitor.c:1236`)
  **immediately before** `render` (`kitty/child-monitor.c:1237`); `render` in turn calls
  `render_os_window` (`kitty/child-monitor.c:833`) for each OS window. **(This internal call
  chain is inferred from code, corroborated by the observed lifecycle above — `--debug-rendering`
  reports the lifecycle, not each per-frame draw.)**
- **GPU draw — `kitty/shaders.c`.** `draw_cells` (`kitty/shaders.c:1009`) issues the OpenGL
  draw calls that rasterize the cell grid. The GPU **sprite/texture atlas** itself is
  allocated and bound by the shader/font-rendering code in `kitty/shaders.c`; `kitty/glyph-cache.c`
  is a CPU-side **glyph-property and sprite-position cache** (its `SpritePosItem` hash and
  `find_or_create_sprite_position` map a glyph to its atlas position) consulted during
  rasterization — not the GPU texture atlas itself. **(inferred from code.)**
- **Present the frame — `kitty/glfw.c`.** The per-frame wrapper `swap_window_buffers`
  (`kitty/glfw.c:1802`) calls `glfwSwapBuffers` at `kitty/glfw.c:1803`, putting the rendered
  frame on screen. (The separate `glfwSwapBuffers` at `kitty/glfw.c:1221` is a one-time
  **startup** blank-canvas swap, done once before the window is first shown — not the per-frame
  present.) **(inferred from code.)**

### 3.3 The "before the screen updates" boundary + threading note

The precise moment at which processed input becomes a new frame is the adjacency of
`parse_input` (`kitty/child-monitor.c:1236`) and `render` (`kitty/child-monitor.c:1237`) inside
`process_global_state`:

```c
if (parse_input(self)) input_read = true;   // kitty/child-monitor.c:1236
render(now, input_read);                     // kitty/child-monitor.c:1237
```

This matters because of Kitty's three-thread Child-Monitor architecture (Main, I/O, Talk): the
child's echoed bytes are **read on the I/O thread** (`io_loop` / `read_bytes`, section 2.5;
source-inferred), but they are **parsed and rendered on the main thread** here. So "before the
screen updates" is concretely this `parse_input` → `render` step on the main thread. **(This
thread split, the `read_bytes` I/O-read ownership, and the `parse_input` → `render` adjacency are
all source-inferred — `read_bytes` emits no trace. What is directly observed is the
`on_key_input` dispatch under `--debug-input` and the startup GL/window/child lifecycle under
`--debug-rendering`; neither logs a per-key frame.)**

---

## Release and modifier-only events (the edge / ignored path)

Not every observed event produces child output. Every key **release**, and every standalone
**modifier** press (`Control_L`, `Shift_L`), printed:

```text
ignoring as keyboard mode does not support encoding this event
```

which is emitted at `kitty/keys.c:271`. The cause is exactly the encoding logic from section
2.2: in default legacy mode, `encode_glfw_key_event` returns `0` for releases
(`kitty/key_encoding.c:368`) and for standalone modifier keys
(`kitty/key_encoding.c:425`), so `on_key_input` takes the "ignore" branch.

**Quantified from the captured run:** of the **16** `on_key_input` events, **6 were sent** to
the child and **10 were ignored**. The 10 ignored break down as **8 releases + 2 modifier-only
presses**. The verbatim ignored lines:

```text
[0.810] on_key_input: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.324] on_key_input: glfw key: 0x62 native_code: 0x62 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.841] on_key_input: glfw key: 0x63 native_code: 0x63 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[2.357] on_key_input: glfw key: 0xe001 native_code: 0xff0d action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[2.867] on_key_input: glfw key: 0xe062 native_code: 0xffe3 action: PRESS mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[2.879] on_key_input: glfw key: 0xe062 native_code: 0xffe3 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[2.885] on_key_input: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[3.395] on_key_input: glfw key: 0xe061 native_code: 0xffe1 action: PRESS mods: shift text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[3.408] on_key_input: glfw key: 0xe061 native_code: 0xffe1 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[3.414] on_key_input: glfw key: 0x62 native_code: 0x62 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
```

Note the two modifier-only presses (`[2.867]` `Control_L`, `native_code: 0xffe3`, and
`[3.395]` `Shift_L`, `native_code: 0xffe1`) are `PRESS` actions yet still ignored — proving
the ignore rule is not merely "ignore releases" but "the legacy keyboard mode has no encoding
for this event," which for modifier keys is enforced by the `is_modifier_key` guard at
`kitty/key_encoding.c:425`.

---

## Stability / Determinism

The identical, unchanged input (`a b c Return ctrl+a shift+b`) was run **twice** by the
Appendix B workflow. After stripping the `[N.NNN]` timestamps and ANSI color codes (the
`normalize` step), the **32-line key-event sequence was byte-for-byte identical** across both
runs. The comparison is auditable: the workflow emits the `diff` result and the two MD5 sums of
the normalized captures, which were:

```text
$ diff -u norm1.txt norm2.txt && echo IDENTICAL
IDENTICAL
$ md5sum norm1.txt norm2.txt
dec3329116425f7feb6d7ba26ce86592  norm1.txt
dec3329116425f7feb6d7ba26ce86592  norm2.txt
```

The reported behavior is therefore **deterministic**, not incidental: only the wall-clock
`[N.NNN]` timestamps differ between runs (they are read from a monotonic clock), which is
exactly why they are stripped before comparison. `norm1.txt` / `norm2.txt` are the normalized
captures produced from the two `--debug-input` logs by the Appendix B `normalize` function.

---

## Not Observed / Inferred

The following were **not exercised** by this default X11 run and any mention of them is
**inferred from code, not observed**:

- **Wayland input backend** — this run used the X11 backend (keys arrived as X11 events and
  were translated by XKB). The Wayland path was not exercised.
- **macOS / Cocoa input backend** — this is a Linux run; the Cocoa path was not exercised.
- **IME composition via `glfw/ibus_glfw.c`** — no input method was active; every
  `on_key_input` line reports `state: 0` (`GLFW_IME_NONE`), so no IME composition occurred.

Additionally, the following pipeline internals are **inferred from code** (their *upstream* and
*downstream* effects are observed, but the specific functions do not each emit a trace line in
this run): the VT parser turning bytes into screen mutations (`kitty/vt-parser.c`,
`kitty/screen.c`); the per-frame `parse_input` → `render` → `render_os_window` call chain
(`kitty/child-monitor.c:1236`/`1237`/`833`); the GPU draw in `kitty/shaders.c:1009`; and the
buffer swap in `kitty/glfw.c:1802`/`1803` (the per-frame `swap_window_buffers` → `glfwSwapBuffers`; `1221` is the one-time startup swap). Each is labeled inline where it appears above.

---

## End-to-End Pipeline Summary

The full path reconstructed from the observations, from an X11 key event to a frame on screen:

```mermaid
flowchart TD
    K["X11 key event (injected via xdotool)"]
    XKB["glfw/xkb_glfw.c:864/875<br/>glfw_xkb_handle_key_event (XKB)<br/>emits reception line (Press/Release xkb_keycode)"]
    CB["kitty/glfw.c:430/439<br/>key_callback → on_key_input(ev)"]
    OKI["kitty/keys.c:166/176<br/>on_key_input — emits on_key_input decision line"]
    SC{"Configured shortcut?<br/>boss.py dispatch_possible_special_key<br/>(keys.py defs)"}
    SH["kitty/keys.c:231<br/>handled as shortcut (none matched)"]
    ENC["kitty/key_encoding.c:414<br/>encode_glfw_key_event (legacy vs Kitty)"]
    WR["kitty/keys.c:253 / 259 / 271<br/>write text / write encoded / ignore"]
    PTY["kitty/child-monitor.c:372<br/>schedule_write_to_child: queue to write_buf (MAIN)"]
    IOW["kitty/child-monitor.c:1540/1443<br/>write_to_child: write(2) to PTY (I/O thread)"]
    SHELL["child bash echoes bytes"]
    IO["kitty/child-monitor.c:1481/1337<br/>io_loop → read_bytes (I/O thread)"]
    VP["kitty/vt-parser.c:1496/1417<br/>parse → mutates kitty/screen.c grid"]
    TICK["kitty/child-monitor.c:1224/1236/1237<br/>process_global_state: parse_input → render (MAIN)"]
    DRAW["kitty/shaders.c:1009<br/>draw_cells (OpenGL draw, GPU)"]
    SWAP["kitty/glfw.c:1802/1803<br/>swap_window_buffers → glfwSwapBuffers → frame on screen<br/>(1221 = one-time startup swap)"]

    K --> XKB --> CB --> OKI --> SC
    SC -->|yes| SH
    SC -->|no| ENC --> WR --> PTY --> IOW --> SHELL --> IO --> VP --> TICK --> DRAW --> SWAP
```

As an ASCII trace of the same path:

```text
X11 key
  → glfw/xkb_glfw.c:864/875   (XKB translate; emits reception line)          [OBSERVED]
  → kitty/glfw.c:430/439      (key_callback → on_key_input)                  [OBSERVED handoff]
  → kitty/keys.c:166/176      (on_key_input; emits processing line)          [OBSERVED]
  → boss.py / keys.py         (shortcut check — none matched)                [OBSERVED: no "handled as shortcut"]
  → kitty/key_encoding.c:414  (encode; legacy mode)                          [OBSERVED via decision values]
  → kitty/keys.c:253/259/271  (send text / send encoded / ignore)           [OBSERVED]
  → kitty/child-monitor.c:372 (schedule_write_to_child: queue to write_buf, MAIN) [OBSERVED: bytes scheduled]
  → kitty/child-monitor.c:1540/1443 (write_to_child: write(2) to PTY, I/O thread)  [inferred from code]
  → bash echoes bytes
  → kitty/child-monitor.c:1481/1337 (io_loop / read_bytes, I/O thread)       [inferred from code]
  → kitty/vt-parser.c:1496/1417 (parse) → kitty/screen.c (grid)             [inferred from code]
  → kitty/child-monitor.c:1224/1236/1237 (parse_input → render, MAIN)        [inferred; startup lifecycle OBSERVED, not per-frame]
  → kitty/shaders.c:1009      (draw_cells, GPU)                              [inferred from code]
  → kitty/glfw.c:1802/1803    (swap buffers → frame on screen; 1221=startup)  [inferred from code]
```

---

## Coverage Checklist

- **All three named sub-parts answered by name:** RECEIVE FIRST (§1), INTERMEDIATE PROCESSING
  (§2), DISPLAY UPDATE (§3). ✔
- **All six keys covered:** `a`, `b`, `c` (UTF-8 text), `Enter` (`0x0d`), `Ctrl+a` (`0x1`),
  `Shift+b` (`B`). ✔
- **Press + release + ignored paths shown:** 16 `on_key_input` events = 6 sent + 10 ignored
  (8 releases + 2 modifier-only presses). ✔
- **Exact values reported with reasons:** `Enter` → `0x0d` and `Ctrl+a` → `0x1` via legacy
  `encode_key` (`kitty/key_encoding.c:439`); `Shift+b` → `B` via the send-text path
  (`kitty/key_encoding.c:437`), with `B` resolved by XKB at reception. ✔
- **Determinism stated:** identical 32-line sequence across two runs (`diff` clean, equal MD5). ✔
- **Inferred items labeled:** Wayland, macOS/Cocoa, and IME/IBus are labeled inferred/not
  observed, as are the VT-parser/screen internals, the per-frame render call chain, the GPU
  draw, and the buffer swap. ✔
- **Every behavioral claim carries adjacent observed output and a `file:line` locator, or is
  explicitly marked (inferred from code).** ✔

---

## Appendix A — Complete captured `--debug-input` trace (32 lines)

Produced by the input leg of the complete **Appendix B** workflow (background launch, PID
capture, bounded PID-correlated window discovery, then fixed-key injection):

```sh
./kitty/launcher/kitty --debug-input --config NONE -o confirm_os_window_close=0 bash --norc --noprofile
# with: xdotool key --window <wid> a b c Return ctrl+a shift+b   (full runnable script: Appendix B)
```

The complete, unedited, ordered key-event sequence — 16 reception lines interleaved with 16
`on_key_input` lines, so that both `PRESS` and `RELEASE` transitions and the ignored
modifier-only presses are all present. ANSI color codes were stripped for readability;
timestamps are preserved as captured (the two encoded `on_key_input` lines end with a trailing
space, exactly as the binary emits them):

```text
[0.808] Press xkb_keycode: 0x26 clean_sym: a composed_sym: a text: a mods: none glfw_key: 97 (a) xkb_key: 97 (a)
[0.808] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: none text: 'a' state: 0 sent key as text to child: a
[0.810] Release xkb_keycode: 0x26 clean_sym: a mods: none glfw_key: 97 (a) xkb_key: 97 (a)
[0.810] on_key_input: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.318] Press xkb_keycode: 0x38 clean_sym: b composed_sym: b text: b mods: none glfw_key: 98 (b) xkb_key: 98 (b)
[1.318] on_key_input: glfw key: 0x62 native_code: 0x62 action: PRESS mods: none text: 'b' state: 0 sent key as text to child: b
[1.324] Release xkb_keycode: 0x38 clean_sym: b mods: none glfw_key: 98 (b) xkb_key: 98 (b)
[1.324] on_key_input: glfw key: 0x62 native_code: 0x62 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[1.834] Press xkb_keycode: 0x36 clean_sym: c composed_sym: c text: c mods: none glfw_key: 99 (c) xkb_key: 99 (c)
[1.834] on_key_input: glfw key: 0x63 native_code: 0x63 action: PRESS mods: none text: 'c' state: 0 sent key as text to child: c
[1.841] Release xkb_keycode: 0x36 clean_sym: c mods: none glfw_key: 99 (c) xkb_key: 99 (c)
[1.841] on_key_input: glfw key: 0x63 native_code: 0x63 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[2.350] Press xkb_keycode: 0x24 clean_sym: Return composed_sym: Return mods: none glfw_key: 57345 (ENTER) xkb_key: 65293 (Return)
[2.350] on_key_input: glfw key: 0xe001 native_code: 0xff0d action: PRESS mods: none text: '' state: 0 sent encoded key to child: 0xd 
[2.357] Release xkb_keycode: 0x24 clean_sym: Return mods: none glfw_key: 57345 (ENTER) xkb_key: 65293 (Return)
[2.357] on_key_input: glfw key: 0xe001 native_code: 0xff0d action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[2.867] Press xkb_keycode: 0x25 clean_sym: Control_L composed_sym: Control_L mods: none glfw_key: 57442 (LEFT_CONTROL) xkb_key: 65507 (Control_L)
[2.867] on_key_input: glfw key: 0xe062 native_code: 0xffe3 action: PRESS mods: ctrl text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[2.873] Press xkb_keycode: 0x26 clean_sym: a composed_sym: a mods: ctrl glfw_key: 97 (a) xkb_key: 97 (a)
[2.873] on_key_input: glfw key: 0x61 native_code: 0x61 action: PRESS mods: ctrl text: '' state: 0 sent encoded key to child: 0x1 
[2.879] Release xkb_keycode: 0x25 clean_sym: Control_L mods: ctrl glfw_key: 57442 (LEFT_CONTROL) xkb_key: 65507 (Control_L)
[2.879] on_key_input: glfw key: 0xe062 native_code: 0xffe3 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[2.885] Release xkb_keycode: 0x26 clean_sym: a mods: none glfw_key: 97 (a) xkb_key: 97 (a)
[2.885] on_key_input: glfw key: 0x61 native_code: 0x61 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[3.395] Press xkb_keycode: 0x32 clean_sym: Shift_L composed_sym: Shift_L mods: none glfw_key: 57441 (LEFT_SHIFT) xkb_key: 65505 (Shift_L)
[3.395] on_key_input: glfw key: 0xe061 native_code: 0xffe1 action: PRESS mods: shift text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[3.401] Press xkb_keycode: 0x38 clean_sym: b composed_sym: B text: B mods: shift glfw_key: 98 (b) xkb_key: 98 (b) shifted_key: 66 (B)
[3.401] on_key_input: glfw key: 0x62 native_code: 0x62 action: PRESS mods: shift text: 'B' state: 0 sent key as text to child: B
[3.408] Release xkb_keycode: 0x32 clean_sym: Shift_L mods: shift glfw_key: 57441 (LEFT_SHIFT) xkb_key: 65505 (Shift_L)
[3.408] on_key_input: glfw key: 0xe061 native_code: 0xffe1 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
[3.414] Release xkb_keycode: 0x38 clean_sym: b mods: none glfw_key: 98 (b) xkb_key: 98 (b)
[3.414] on_key_input: glfw key: 0x62 native_code: 0x62 action: RELEASE mods: none text: '' state: 0 ignoring as keyboard mode does not support encoding this event
```

---

## Appendix B — Exact commands used

```sh
#!/usr/bin/env bash
# Complete, noninteractive, fail-fast, auditable capture of Kitty's input->display
# debug traces. Run from the repository root. It builds the launcher, starts a
# fresh PRIVATE authenticated display, captures every spawned PID, redirects all
# trace output to named logs, discovers the Kitty window bounded AND correlated to
# the launched PID, injects a fixed key set, runs the input capture TWICE, then
# normalizes and compares with diff + md5sum. A trap tears down only the spawned
# Xvfb/Kitty PIDs; cleanup is limited to one named temp dir.
set -uo pipefail

DISPLAY_NUM=":99"
KITTY="$PWD/kitty/launcher/kitty"            # produced by the build step below
KEYS=(a b c Return ctrl+a shift+b)           # fixed keys: unmodified + control + modifier
WORK="$(mktemp -d /tmp/obs_kitty.XXXXXX)"    # named private scratch (removed by trap)
STATUS="$WORK/status"; : > "$STATUS"
log(){ echo "$*" | tee -a "$STATUS"; }

# 0) Build the launcher from source in default configuration (canonical toolchain).
#    go builds only the kitten CLI (off the input->display path); python3 is the
#    canonical CPython 3.12.3 interpreter.
export PATH=/usr/lib/go-1.22/bin:$PATH
CI=true python3 setup.py build               # -> kitty/launcher/kitty  (kitty 0.35.2)

# 1) Prerequisite checks (fail fast if any tool or the launcher is missing).
for t in Xvfb xdotool xdpyinfo mcookie xauth; do
  command -v "$t" >/dev/null 2>&1 || { log "PREREQ MISSING: $t"; exit 1; }
done
[ -x "$KITTY" ] || { log "PREREQ MISSING: $KITTY (build did not produce launcher)"; exit 1; }

# 2) Fresh, PRIVATE, authenticated display. Access control stays ON (NO -ac): a
#    per-run MIT-MAGIC-COOKIE restricts the display to this workflow. NO -noreset,
#    so the server resets and exits normally when torn down.
export XAUTHORITY="$WORK/xauth"; : > "$XAUTHORITY"
xauth -f "$XAUTHORITY" add "$DISPLAY_NUM" . "$(mcookie)" >/dev/null 2>&1

# 3) Trap-based teardown: kill ONLY the PIDs we spawn and remove ONLY our temp dir.
XVFB_PID=""; KITTY_PID=""
cleanup() {
  [ -n "$KITTY_PID" ] && kill "$KITTY_PID" 2>/dev/null || true
  [ -n "$XVFB_PID" ]  && kill "$XVFB_PID"  2>/dev/null || true
  rm -rf "$WORK"
}
trap cleanup EXIT INT TERM

Xvfb "$DISPLAY_NUM" -screen 0 1280x800x24 -auth "$XAUTHORITY" +extension GLX +render \
  >"$WORK/xvfb.log" 2>&1 &
XVFB_PID=$!
export DISPLAY="$DISPLAY_NUM" LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe
for i in $(seq 1 50); do xdpyinfo >/dev/null 2>&1 && break; sleep 0.1; done
xdpyinfo >/dev/null 2>&1 || { log "Xvfb did not come up"; exit 1; }
log "Xvfb up (pid $XVFB_PID)"

# 4) Bounded window discovery CORRELATED to the launched Kitty PID: accept only a
#    window whose _NET_WM_PID (via getwindowpid) equals our launched PID, so a
#    stale/unrelated window on a shared display can never be selected. Bails out if
#    the launched Kitty dies (failure is never masked) and is time-bounded.
find_window() {                               # $1 = launched Kitty PID
  local pid="$1" wid wpid
  for i in $(seq 1 150); do                   # <= ~15s
    for wid in $(xdotool search --class kitty 2>/dev/null || true); do
      wpid=$(xdotool getwindowpid "$wid" 2>/dev/null || echo "")
      [ "$wpid" = "$pid" ] && { echo "$wid"; return 0; }
    done
    kill -0 "$pid" 2>/dev/null || return 1    # launched Kitty died -> fail fast
    sleep 0.1
  done
  return 1                                    # bounded timeout -> fail (do not mask)
}

run_input() {                                 # $1 = log file
  "$KITTY" --debug-input --config NONE -o confirm_os_window_close=0 \
    bash --norc --noprofile >"$1" 2>&1 &
  KITTY_PID=$!
  local WID; WID=$(find_window "$KITTY_PID") \
    || { log "no correlated kitty window (input)"; kill "$KITTY_PID" 2>/dev/null; return 1; }
  log "input  window=$WID correlated_to kitty_pid=$KITTY_PID"
  xdotool windowactivate --sync "$WID" >/dev/null 2>&1 || true
  xdotool windowfocus  --sync "$WID"   >/dev/null 2>&1 || true
  sleep 0.6
  for k in "${KEYS[@]}"; do xdotool key --window "$WID" "$k"; sleep 0.5; done
  sleep 0.6
  kill "$KITTY_PID" 2>/dev/null || true; wait "$KITTY_PID" 2>/dev/null || true; KITTY_PID=""
}

run_render() {                                # $1 = log file
  "$KITTY" --debug-rendering --config NONE -o confirm_os_window_close=0 \
    bash --norc --noprofile >"$1" 2>&1 &
  KITTY_PID=$!
  local WID; WID=$(find_window "$KITTY_PID") \
    || { log "no correlated kitty window (render)"; kill "$KITTY_PID" 2>/dev/null; return 1; }
  log "render window=$WID correlated_to kitty_pid=$KITTY_PID"
  sleep 1.2
  kill "$KITTY_PID" 2>/dev/null || true; wait "$KITTY_PID" 2>/dev/null || true; KITTY_PID=""
}

# strip ANSI colours + [N.NNN] timestamps; keep only the two key-event line types
normalize(){ sed -E 's/\x1b\[[0-9;]*m//g; s/^\[[0-9]+\.[0-9]+\] //' "$1" \
             | grep -E '(xkb_keycode:|on_key_input:)'; }

# 5) Two identical input runs + one render run, each into a named log.
log "== RUN 1 (input) =="; run_input  "$WORK/input1.log" || { log "RUN1 FAILED";   exit 1; }
log "== RUN 2 (input) =="; run_input  "$WORK/input2.log" || { log "RUN2 FAILED";   exit 1; }
log "== RENDER =="       ; run_render "$WORK/render.log" || { log "RENDER FAILED"; exit 1; }

# 6) Normalize and compare (auditable): diff must be empty and both MD5s equal.
normalize "$WORK/input1.log" > "$WORK/norm1.txt"
normalize "$WORK/input2.log" > "$WORK/norm2.txt"
log "trace1 key-event lines: $(wc -l < "$WORK/norm1.txt")"   # expect 32
log "trace2 key-event lines: $(wc -l < "$WORK/norm2.txt")"   # expect 32
diff -u "$WORK/norm1.txt" "$WORK/norm2.txt" && log "DIFF: identical"
md5sum "$WORK/norm1.txt" "$WORK/norm2.txt"
sed -E 's/\x1b\[[0-9;]*m//g' "$WORK/render.log" \
  | grep -E 'GL version string:|OS Window created|Child launched'
log "sent=$(grep -c 'to child:' "$WORK/norm1.txt")  ignored=$(grep -c 'ignoring as keyboard' "$WORK/norm1.txt")"
log "CAPTURE_DONE"
# Trap now fires: only the spawned Xvfb/Kitty PIDs are killed and only "$WORK" is removed.
```

All observation scripts and logs were temporary and were removed after the investigation; the
only change this task makes to the repository is the addition of this one Markdown document.
