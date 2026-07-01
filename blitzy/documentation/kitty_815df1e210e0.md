# Kitty Terminal Emulator — Startup Sequence Investigation (v0.35.2)

> A **run-first, verbatim-grounded** walkthrough of what happens when the Kitty
> terminal emulator boots from commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`,
> from the native process launch up to the point the terminal is ready to host a
> shell. Every claim below is backed by either an exact `file:line` citation into
> the repository at this commit **or** by real output captured from a headless run
> under Xvfb + Mesa llvmpipe software OpenGL. No values are paraphrased or invented;
> where a command genuinely could not run headless, that is stated explicitly.

## Overview

This document answers four questions about Kitty's startup. Kitty is a hybrid
application — a **Python** frontend/orchestration layer, a **C** terminal core and
GPU renderer (compiled into `kitty/fast_data_types.so`), and a small **Go** CLI
(`kitten`) — all launched by a tiny **native C** executable that embeds CPython.
The version under investigation is **`kitty 0.35.2 created by Kovid Goyal`** (proof
in §4.1). Because the container has no GPU and no physical display, the terminal is
exercised **headlessly** with a virtual X11 framebuffer (`xvfb-run`) plus Mesa's
software OpenGL implementation (llvmpipe); the resolved OpenGL context is
`4.5 (Core Profile) Mesa 25.2.8` (§4.3, §5, §8).

The document is organized as: a build & run preamble (§4, done first, per the
run-first rule), then one section per question (§5 Q1 startup systems, §6 Q2
configuration, §7 Q3 terminal↔shell readiness, §8 Q4 display-system evidence), and
finally a coverage-pass checklist (§9).

> **A note on secrets.** Kitty's `kitten @ ls` remote-control command emits the full
> child-process environment. In this container that environment contains real API
> keys and tokens, so **every `ls` excerpt below has had its `env` block removed**;
> only geometry/layout/title/command fields are shown. No secret value appears in
> this document.

---

## 4. Build & Run preamble (run-first)

### 4.1 Version (grounded)

The version is assembled from three literals in `kitty/constants.py` and formatted
by `kitty/cli.py`:

- `kitty/constants.py:23` → `appname: str = 'kitty'`
- `kitty/constants.py:25` → `version: Version = Version(0, 35, 2)`
- `kitty/constants.py:26` → `str_version: str = '.'.join(map(str, version))`
- `kitty/cli.py:486` → `def version(add_rev: bool = False) -> str:`
- `kitty/cli.py:492` → `return '{} {}{} created by {}'.format(italic(appname), green(str_version), rev, title('Kovid Goyal'))`

Running the built launcher confirms the resulting banner verbatim:

```
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

So `str_version` materializes to **`0.35.2`**, exactly matching `Version(0, 35, 2)`.

### 4.2 Build (real exit status)

The `Makefile` `all:` target builds via `setup.py`:

- `Makefile:12` → `all:`
- `Makefile:13` → `python3 setup.py $(VVAL)`

The AAP mandates the CI form. The real build was run and its exit status captured:

```
$ CI=true python3 setup.py > /tmp/kitty_build.log 2>&1
$ echo "BUILD_EXIT=$?"
BUILD_EXIT=0
```

`BUILD_EXIT=0` — the build succeeded. It produces three artifacts, all of which are
**gitignored** (`.gitignore` contains `*.so` and `/kitty/launcher/kitt*`), so they do
not alter the tracked tree: `kitty/launcher/kitty` (the native launcher),
`kitty/launcher/kitten` (the Go CLI), and `kitty/fast_data_types.so` (the C
extension). The build log's only notable line is a benign capability notice:

```
Package 'wayland-protocols', required by 'virtual:world', not found
... Disabling building of wayland backend
```

This is expected — the container targets X11/Xvfb, not Wayland — and does not affect
the X11 headless run used throughout this document.

Dependency floors, for context (from `docs/build.rst`, `pyproject.toml`, `go.mod`):

- `docs/build.rst:83` → `python >= 3.8`
- `docs/build.rst:84` → `harfbuzz >= 2.2.0`
- `docs/build.rst:90` → freetype; `docs/build.rst:91` → fontconfig
- `pyproject.toml:2` → `requires-python = ">=3.8"`
- `go.mod:3` → `go 1.22`

(The environment actually used exceeds all floors: Python 3.13.7, Go 1.24.4.)

### 4.3 Headless harness (verified pattern)

Kitty's renderer is **entirely GPU/OpenGL-based** — there is no CPU text-drawing
fallback. On a machine with no GPU and no physical display, the standard approach is
a virtual X11 display (`Xvfb`) paired with Mesa's software OpenGL (llvmpipe), which
supplies an OpenGL core profile well above Kitty's minimum. The `xvfb-run` wrapper
starts Xvfb, runs the program against it, and tears Xvfb down afterward. The verified
smoke run:

```
$ xvfb-run -a --server-args="-screen 0 1024x768x24" \
    ./kitty/launcher/kitty --config NONE -o font_size=12 sh -c 'echo SMOKE_OK; sleep 1'
$ echo "RUN_EXIT=$?"
RUN_EXIT=0
```

`RUN_EXIT=0` — the full pipeline (Xvfb → llvmpipe → GLFW window → OpenGL context →
child spawn → clean exit) works end to end. The observed GL context string, captured
under `--debug-rendering` in §5, is:

```
GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5
```

> **Headless nuance (important for Q3/Q4).** The shell's `SMOKE_OK` text does **not**
> appear on the host stdout — it is drawn onto Kitty's GPU surface inside Xvfb. Proof
> that child output was parsed and drawn is therefore obtained by reading the rendered
> grid back with remote control (`kitten @ get-text`), not by watching host stdout.
> This is demonstrated in §7.

### 4.4 Log-format primer (how to read every `[%.3f]` line)

Every debug/log line below is prefixed with an elapsed-seconds timestamp. There are
three producers of that `[%.3f]` prefix, all writing the same format:

- `kitty/logging.c:22` → `log_error(const char *fmt, ...) {` … `kitty/logging.c:56`
  → `fprintf(stderr, "[%.3f] ", monotonic_t_to_s_double(monotonic()));` — used by C
  `log_error()` calls (e.g. the systemd line in §5).
- The C `debug()`/`debug_rendering()` macros ultimately call a `timed_debug_print`
  helper that emits the same `[%.3f] ` prefix to **stderr** (this backs
  `OS Window created` in §5).
- `kitty/window.py:871` → `print(f'[{now:.3f}] Child launched', file=sys.stderr)` —
  Python formats its own inline `[%.3f]` prefix.

So a line like `[0.150] OS Window created` reads as *"0.150 seconds after the
monotonic clock reference, the OS window was created."* The one apparent exception is
the GL version line, which uses `printf` and therefore goes to **stdout** rather than
stderr (`kitty/gl.c:72`); when stdout and stderr are merged its timestamp can appear
out of order relative to the stderr lines (demonstrated in §5).

---

## 5. Q1 — Which systems start up, and what you see confirming them

**Question:** *"Set up a virtual framebuffer (Xvfb) to attempt headless execution and
when Kitty launches, which systems start up on the way to a working terminal, and what
do you actually see on screen or in logs that shows them coming online?"*

**Method.** Run under the Xvfb harness (§4.3) with `--debug-rendering`, capturing both
streams. The relevant flag definitions are grounded at `kitty/cli.py:989` →
`--debug-rendering --debug-gl`, `kitty/cli.py:996` → `--debug-input --debug-keyboard`,
`kitty/cli.py:1002` → `--debug-font-fallback`, and `kitty/cli.py:985` → `--dump-bytes`.

The capture command and its **real** output (streams separated to show which log goes
where):

```
$ xvfb-run -a --server-args="-screen 0 1024x768x24" \
    ./kitty/launcher/kitty --debug-rendering --config NONE \
    sh -c 'echo BOOT_OK; sleep 1' > /tmp/kitty_boot.out 2> /tmp/kitty_boot.err
$ echo "RUN_EXIT=$?"
RUN_EXIT=0

$ cat /tmp/kitty_boot.out          # stdout (printf, gl.c:72)
[0.127] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5

$ cat /tmp/kitty_boot.err          # stderr (log_error / debug() / print(...,file=sys.stderr))
[0.150] OS Window created
[0.159] Failed to open systemd user bus with error: Connection refused
[0.162] Child launched
```

### 5.1 The boot chain (which systems start up)

The startup is a deterministic chain. Each stage is anchored to its exact source
location; stages that print during boot are tied to the verbatim line above.

1. **Native launcher** — `kitty/launcher/main.c:439` → `int main(int argc, char *argv[], char* envp[]) {`.
   The tiny C executable validates descriptors, resolves paths, and decides how to
   dispatch. *(Silent by default; inferred from process start.)*
2. **Single-instance dispatch** — `kitty/launcher/main.c:436` →
   `if (opts.single_instance) single_instance_main(argc, argv, &opts);`, whose contract
   is `kitty/launcher/launcher.h:20` → `single_instance_main(int argc, char *argv[], const CLIOptions *opts);`.
   With no `--single-instance` flag this branch is skipped. *(Silent.)*
3. **CPython bootstrap** — `kitty/launcher/main.c:211` → `status = Py_InitializeFromConfig(&config);`
   then `kitty/launcher/main.c:216` → `return Py_RunMain();`. The launcher embeds and
   starts CPython, handing control to Python. *(Silent.)*
4. **Entry-point dispatch** — `kitty/entry_points.py:151` → `entry_points = {`. Python
   inspects `argv` and routes to the GUI (vs. `+kitten`/`+runpy`/`+launch`). *(Silent.)*
5. **GUI orchestration** — `kitty/main.py:524` → `def main() -> None:`; the app is run via
   `kitty/main.py:202` → `def _run_app(opts: Options, args: CLIOptions, ...)` and
   `kitty/main.py:239` → `class AppRunner:`. This parses CLI, sets up env/locale/signals,
   and drives the rest. *(Silent.)*
6. **GLFW windowing + OpenGL context** — `kitty/main.py:514` →
   `init_glfw(opts, cli_opts.debug_keyboard, cli_opts.debug_rendering)`. The OpenGL
   version is detected and printed at `kitty/gl.c:72` →
   `if (global_state.debug_rendering) printf("[%.3f] GL version string: %s\n", ...)`,
   with the string built by `kitty/gl.c:42` → `gl_version_string(void) {` and
   `kitty/gl.c:47` → `snprintf(buf, sizeof(buf), "'%s' Detected version: %d.%d", ...)`.
   **Observed (stdout):** `[0.127] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5`
   — this is the concrete proof the software-GL context came online.
7. **Font initialization** — the resolved-font block is emitted by
   `kitty/fonts/render.py:163` → `log_error('Text fonts:')` (covered in detail in §8;
   it prints only under `--debug-font-fallback`).
8. **Boss controller** — `kitty/boss.py:323` → `class Boss:`; it registers itself as the
   singleton at `kitty/boss.py:375` → `set_boss(self)`, and brings up sessions/windows at
   `kitty/boss.py:1181` → `def start(self, first_os_window_id: int, startup_sessions: Iterable[Session]) -> None:`.
   *(Silent by default; its effect is the OS window + child appearing next.)*
9. **Child-monitor threads** — the C core runs a three-thread engine declared at
   `kitty/child-monitor.c:229` → `static void* io_loop(void *data);` and
   `kitty/child-monitor.c:230` → `static void* talk_loop(void *data);` (plus the main
   thread). The IO thread pumps PTY bytes (see §7). *(Silent.)*
10. **OS window created** — `kitty/glfw.c:1321` → `debug("OS Window created\n");`.
    **Observed (stderr):** `[0.150] OS Window created` — proof the GLFW/X11 window exists.
11. **Child + PTY spawn** — the shell is forked onto a PTY (see §7 for `kitty/child.py`),
    and a marker is printed by `kitty/window.py:871` →
    `print(f'[{now:.3f}] Child launched', file=sys.stderr)` (gated on
    `boss.args.debug_rendering`). **Observed (stderr):** `[0.162] Child launched` — proof
    the child process was launched onto its pseudo-terminal.

### 5.2 What you actually see (log evidence, summarized)

Reading the timestamps in order tells the boot story directly:

| Elapsed | Stream | Line (verbatim) | Subsystem proven |
|--------:|--------|-----------------|------------------|
| `[0.127]` | stdout | `GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5` | OpenGL context (llvmpipe) |
| `[0.150]` | stderr | `OS Window created` | GLFW/X11 window |
| `[0.159]` | stderr | `Failed to open systemd user bus with error: Connection refused` | (benign — see below) |
| `[0.162]` | stderr | `Child launched` | child process on PTY |

The `[%.3f]` prefix is elapsed seconds since the monotonic reference (§4.4). Because
the GL line uses `printf` → **stdout** while the others use stderr, merging the two
streams can reorder the GL line relative to the rest (its `0.127` timestamp shows it
was actually emitted first); separating the streams, as above, restores the true order.

### 5.3 The benign systemd line (not a defect)

`[0.159] Failed to open systemd user bus with error: Connection refused` originates at
`kitty/systemd.c:87` →
`if (ret < 0) { log_error("Failed to open systemd user bus with error: %s", strerror(-ret)); return; }`.
There is **no systemd user bus** inside the container, so the connect attempt is
refused and Kitty logs it and moves on. This is an expected environment artifact, not a
bug and not a failure of any startup subsystem — the terminal still reaches a working
state (`RUN_EXIT=0`).


---

## 6. Q2 — How Kitty decides its initial configuration

**Question:** *"How does Kitty decide on its initial configuration when it starts? Which
configuration sources or default settings does it use for the first launch, how do they
affect what you see when the window appears, and what output during a real launch shows
that those settings were applied?"*

### 6.1 The two configuration sources exercised at first launch

Kitty resolves options through a single pipeline that merges, in order: **built-in
defaults** → an optional user `kitty.conf` → **`-o key=value` command-line overrides**.
The entry point is `kitty/config.py:163` →
`def load_config(*paths: str, overrides: Optional[Iterable[str]] = None, accumulate_bad_lines: Optional[List[BadLine]] = None) -> Options:`,
which parses config text via `kitty/config.py:151` →
`def parse_config(lines: Iterable[str], accumulate_bad_lines: Optional[List[BadLine]] = None) -> Dict[str, Any]:`
and returns a typed object `kitty/options/types.py:471` → `class Options:` (the
process-wide instance is `kitty/options/types.py:752` → `defaults = Options()`). Saved
config is written atomically by `kitty/config.py:32` → `def atomic_save(data: bytes, path: str) -> None:`.

The two sources this investigation exercises deliberately isolate the layers:

- **`--config NONE`** → **no** user `kitty.conf` is read, so Kitty runs on **pure
  built-in defaults**. This is the "first launch with no user config" case the question
  asks about.
- **`-o key=value`** → a single option is overridden at runtime, layered on top of the
  defaults, letting us prove the merge actually takes effect.

The authoritative default values live in `kitty/options/definition.py` (4327 lines) as
`opt(...)` declarations. The specific defaults relevant to what you see when the window
appears:

| `definition.py` line | Declaration (verbatim head) | Effect on the visible window |
|---|---|---|
| `:59` | `opt('font_size', '11.0',` | Text scale (points) for the grid |
| `:320` | `opt('cursor_shape', 'block',` | Cursor drawn as a solid block |
| `:372` | `opt('scrollback_lines', '2000',` | How many lines of history are kept |
| `:866` | `opt('repaint_delay', '10',` | Frame cadence — ms between repaints |
| `:878` | `opt('input_delay', '3',` | ms to wait before processing input |
| `:994` | `opt('initial_window_width', '640',` | Window width (px) when it first appears |
| `:2896` | `opt('shell', '.',` | `'.'` means "use the user's login shell" |

Per-item parsing is generated in `kitty/options/parse.py` (1482 lines), and the resolved
options are pushed to the C core via `kitty/options/to-c-generated.h` (1346 lines). A
human-readable dump of the live configuration is available at
`kitty/debug_config.py:231` → `def debug_config(opts: KittyOpts) -> str:` (bound to the
`kitty_mod+f6` action inside a running terminal).

### 6.2 Real-launch output proving the defaults are in force

Using the built launcher's embedded CPython (`+runpy`), the **materialized** default
values were read back directly from the `defaults` object. This is real captured output:

```
$ ./kitty/launcher/kitty +runpy 'from kitty.options.types import defaults
print("font_size=", defaults.font_size)
print("cursor_shape=", defaults.cursor_shape)
print("scrollback_lines=", defaults.scrollback_lines)
print("repaint_delay=", defaults.repaint_delay)
print("input_delay=", defaults.input_delay)
print("initial_window_width=", defaults.initial_window_width)
print("shell=", repr(defaults.shell))'
font_size= 11.0
cursor_shape= 1
scrollback_lines= 2000
repaint_delay= 10
input_delay= 3
initial_window_width= (640, 'px')
shell= '.'
```

Each printed value matches its `definition.py` literal exactly:

- **`font_size = 11.0`** ← `definition.py:59` `'11.0'`. Governs text scale in the window.
- **`cursor_shape = 1`** ← `definition.py:320` `'block'`. The `'block'` string is parsed
  into an integer enum; the enum mapping was confirmed live:

  ```
  $ ./kitty/launcher/kitty +runpy 'from kitty.fast_data_types import CURSOR_BLOCK, CURSOR_BEAM, CURSOR_UNDERLINE
  print("CURSOR_BLOCK=", CURSOR_BLOCK, "CURSOR_BEAM=", CURSOR_BEAM, "CURSOR_UNDERLINE=", CURSOR_UNDERLINE)'
  CURSOR_BLOCK= 1 CURSOR_BEAM= 2 CURSOR_UNDERLINE= 3
  ```

  So `cursor_shape = 1` **is** `CURSOR_BLOCK` — the cursor is drawn as a solid block.
- **`scrollback_lines = 2000`** ← `definition.py:372` `'2000'`. 2000 lines of history.
- **`repaint_delay = 10`** ← `definition.py:866` `'10'`. ~10 ms between repaints.
- **`input_delay = 3`** ← `definition.py:878` `'3'`. 3 ms input coalescing.
- **`initial_window_width = (640, 'px')`** ← `definition.py:994` `'640'`. The window is
  640 px wide when it appears; the `'px'` unit is carried alongside the number.
- **`shell = '.'`** ← `definition.py:2896` `'.'`. The sentinel `'.'` tells Kitty to run
  the user's login shell rather than a hard-coded program.

### 6.3 Proving an override is merged on top

To show the `-o` source actually changes the materialized value, `load_config` was
invoked both with and without an override. Real captured output:

```
$ ./kitty/launcher/kitty +runpy 'from kitty.config import load_config
o = load_config(overrides=["font_size 12"])
print("overridden font_size=", o.font_size)
d = load_config()
print("default font_size=", d.font_size)
print("default cursor_shape=", d.cursor_shape)'
overridden font_size= 12.0
default font_size= 11.0
default cursor_shape= 1
```

With no override, `load_config()` yields `font_size = 11.0` (the `definition.py:59`
default); with `overrides=["font_size 12"]`, the same pipeline yields
`font_size = 12.0`. This is the same mechanism exercised by the smoke run's
`-o font_size=12` in §4.3. The override layer is therefore proven to merge on top of the
built-in defaults, exactly as the `load_config(..., overrides=...)` signature at
`kitty/config.py:163` implies.

**Summary for Q2.** (1) *Sources:* built-in defaults in `kitty/options/definition.py`
(used wholesale under `--config NONE`), optionally a user `kitty.conf`, and `-o`
runtime overrides — all merged by `load_config` (`config.py:163`) into an `Options`
object (`types.py:471`). (2) *Effect on the window:* the defaults set the font scale
(`font_size = 11.0`), the block cursor (`cursor_shape = 1`), the initial window width
(`initial_window_width = (640, 'px')`), the history depth (`scrollback_lines = 2000`),
and the login-shell choice (`shell = '.'`). (3) *Proof applied:* the `+runpy` readback
prints the exact literals, and the override readback shows `11.0 → 12.0`.


---

## 7. Q3 — Getting the terminal ready to talk to the shell, and proving first output

**Question:** *"How does Kitty get the terminal ready to communicate with the shell that
will run inside it? When the shell prints its first output, what concrete behavior shows
the data was understood correctly and drawn in the terminal?"*

### 7.1 How the terminal is made ready (PTY → fork/exec → env → VT parser → screen)

A terminal emulator talks to its child through a **pseudo-terminal (PTY)**: a
master/slave pair where the emulator holds the master and the child's stdin/stdout/stderr
are wired to the slave. Kitty sets this up as follows:

1. **PTY allocation** — `kitty/child.py:170` → `def openpty() -> Tuple[int, int]:`
   allocates the master/slave descriptors.
2. **Fork/exec the child** — `kitty/child.py:276` → `def fork(self) -> Optional[int]:`;
   inside it the PTY is opened at `kitty/child.py:281` → `master, slave = openpty()`, then
   the child is forked and `exec`'d with its stdio bound to the slave.
3. **Environment preparation (incl. shell integration).** Before the fork, Kitty builds
   the child's environment. `kitty/child.py:266-267` guard and call
   `modify_shell_environ(opts, env, self.argv)`. That function is
   `kitty/shell_integration.py:218` →
   `def modify_shell_environ(opts: Options, env: Dict[str, str], argv: List[str]) -> None:`,
   and when the launched program is a **supported** shell it sets
   `kitty/shell_integration.py:223` → `env['KITTY_SHELL_INTEGRATION'] = ksi`. Support is
   decided by `kitty/shell_integration.py:186` → `def get_supported_shell_name(path: str) -> Optional[str]:`,
   which returns a name only for shells in `ENV_MODIFIERS` (bash/zsh/fish) and `None`
   otherwise. The shell-side scripts that consume this — `shell-integration/bash/kitty.bash`
   (391 lines) and `shell-integration/zsh/kitty.zsh` (21 lines) — emit **OSC 133** prompt
   marks, **OSC 7** cwd reporting, and **DECSCUSR** cursor-shape sequences back to Kitty.
4. **Byte pump + VT parser.** The child-monitor IO thread (`kitty/child-monitor.c:229`
   `io_loop`) reads bytes off the PTY master and feeds them to the VT state machine, which
   classifies every byte: `kitty/vt-parser.c:230` → `consume_normal(PS *self) {` (printable
   text), `kitty/vt-parser.c:261` → `consume_esc(PS *self) {` (escape sequences), and
   `kitty/vt-parser.c:457` → `dispatch_osc(PS *self, uint8_t *buf, size_t limit, bool is_extended_osc) {`
   (OSC strings such as the OSC 133/OSC 7 above).
5. **Screen model update.** Printable runs are written into the grid by
   `kitty/screen.c:866` → `screen_draw_text(Screen *self, const uint32_t *chars, size_t num_chars) {`,
   one code point at a time via `kitty/screen.c:872` → `draw_codepoint(Screen *self, char_type ch) {`,
   backed by the line buffers `kitty/line.c` (1003 lines) and `kitty/line-buf.c` (641 lines).

### 7.2 Evidence the environment/terminal wiring is correct

Because the earlier boot run launched `sh -c ...`, and `sh` (dash) is **not** a supported
shell, `KITTY_SHELL_INTEGRATION` was correctly **absent**. Re-running with **bash** (a
supported shell) and reading two variables back off the rendered screen confirms both the
integration-env injection and the terminal-type wiring. Real captured output:

```
$ ./kitty/launcher/kitty --config NONE -o allow_remote_control=yes \
    --listen-on unix:/tmp/kitty_test_tmp/ksi.sock \
    bash --norc --noprofile -c 'printf "KSI=[%s]\n" "$KITTY_SHELL_INTEGRATION"; printf "TERM=[%s]\n" "$TERM"; sleep 8' &
# then, against the same socket:
$ ./kitty/launcher/kitty @ --to unix:/tmp/kitty_test_tmp/ksi.sock get-text
KSI=[enabled]
TERM=[xterm-kitty]
```

- `KSI=[enabled]` proves `modify_shell_environ` set `KITTY_SHELL_INTEGRATION=enabled`
  (`shell_integration.py:223`) for the bash child.
- `TERM=[xterm-kitty]` proves Kitty advertised its own terminfo-backed terminal type to
  the child, so the shell knows which capabilities it may use.

### 7.3 Concrete proof the first output was understood and drawn

**The headless subtlety:** what the shell prints is drawn onto Kitty's GPU surface inside
Xvfb, **not** echoed to the host stdout (§4.3). So the proof cannot be "we saw it on the
host terminal." Instead, Kitty's remote control reads the **rendered grid** back — if a
string the shell printed can be recovered from the screen model, then those child bytes
were received, classified by the VT parser, and drawn into the grid.

A temporary probe (created under `/tmp`, deleted afterward) launched Kitty with remote
control enabled, had the shell print a unique marker `READY_MARKER_42`, then read the
screen back with `get-text` and the geometry with `ls`. Real captured output:

```
$ ./kitty/launcher/kitty --config NONE -o allow_remote_control=yes \
    --listen-on unix:/tmp/kitty_test_tmp/krc.sock \
    sh -c "printf '%s\n' 'READY_MARKER_42'; sleep 8" &
# read the rendered screen text back:
$ ./kitty/launcher/kitty @ --to unix:/tmp/kitty_test_tmp/krc.sock get-text
READY_MARKER_42
```

The marker printed by the shell — `READY_MARKER_42` — was read back **verbatim** from the
rendered screen. That round trip is the concrete behavior proving the data was understood
correctly: the child's bytes traversed `io_loop` → `consume_normal` (`vt-parser.c:230`) →
`screen_draw_text`/`draw_codepoint` (`screen.c:866`/`:872`) and landed in the grid as the
exact characters the shell emitted.

The companion `ls` call reports the window/grid structure (its `env` block is **omitted**
here for the security reason stated in the Overview):

```
$ ./kitty/launcher/kitty @ --to unix:/tmp/kitty_test_tmp/krc.sock ls
[
  {
    "id": 1, "is_focused": true, "wm_class": "kitty", "wm_name": "kitty",
    "tabs": [ { "layout": "fat", "title": "sh",
      "windows": [ {
        "id": 1, "title": "sh", "pid": 57827,
        "cmdline": ["sh","-c","printf '%s\\n' 'READY_MARKER_42'; sleep 8"],
        "columns": 71, "lines": 22,
        "at_prompt": false, "last_cmd_exit_status": 0
        /* "env": { ...omitted: contains secrets... } */
      } ] } ]
  }
]
```

The reported `cmdline` matches exactly what the shell was told to run, and
`last_cmd_exit_status: 0` confirms clean execution. The grid geometry (`columns: 71`,
`lines: 22`) is analyzed further under Q4 (§8).

**Summary for Q3.** (1) *Readiness path:* allocate a PTY (`child.py:170`), fork/exec the
child onto it (`child.py:276`, `:281`), prepare its environment including
`KITTY_SHELL_INTEGRATION` (`shell_integration.py:218`, `:223`) and `TERM=xterm-kitty`,
then pump PTY bytes through the VT parser (`vt-parser.c:230/261/457`) into the screen model
(`screen.c:866/872`). (2) *First-output proof:* the shell's `READY_MARKER_42` round-tripped
verbatim through `kitten @ get-text`, and `KSI=[enabled]` / `TERM=[xterm-kitty]` confirm the
integration/terminal wiring — the data was received, parsed, and drawn exactly.


---

## 8. Q4 — Display-system evidence: fonts, layout, scrolling, screen updates

**Question:** *"As those first characters appear, what visible evidence about fonts,
layout, scrolling, or how the screen updates that tells you the display system is active
and working properly? Include any log or console messages that help confirm this?"*

This question names four facets — **fonts, layout, scrolling, screen updates** — each
answered below with a `file:line` anchor and, where available, verbatim captured output.

### 8.1 Fonts (discovery, resolution, rasterization)

On Linux, fonts are discovered via FontConfig in `kitty/fontconfig.c` (514 lines) and
rasterized into glyph bitmaps by FreeType in `kitty/freetype.c` (1037 lines). The
resolved set is logged by `dump_font_debug()` at `kitty/fonts/render.py:161` →
`def dump_font_debug() -> None:`, which prints `kitty/fonts/render.py:163` →
`log_error('Text fonts:')`, then one line per style using the mapping at
`kitty/fonts/render.py:164` → `{'medium': 'Normal', 'bold': 'Bold', 'italic': 'Italic', 'bi': 'Bold-Italic'}`
via `kitty/fonts/render.py:165` → `log_error(f'  {text}:', cf[key].identify_for_debug())`,
and (if any) a `kitty/fonts/render.py:168` → `log_error('Symbol map fonts:')` block.

Captured with `--debug-font-fallback` (real output):

```
$ xvfb-run -a --server-args="-screen 0 1024x768x24" \
    ./kitty/launcher/kitty --debug-font-fallback --config NONE \
    sh -c 'echo FONT_OK; sleep 1'
[0.158] Failed to open systemd user bus with error: Connection refused
[0.162] Text fonts:
[0.162]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.162]   Bold: DejaVuSansMono-Bold: /root/.local/share/fonts/DejaVuSansMono-Bold.ttf:0
[0.162]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.162]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

The default monospace family resolved to **DejaVu Sans Mono** for all four styles
(provided by `fonts-dejavu-core`). The `Text fonts:` block is the direct log evidence
that font discovery + face selection succeeded before any character was drawn. Note the
Bold face was picked up from a user font directory (`/root/.local/share/fonts`) while the
others came from the system DejaVu directory — exactly the kind of cross-directory
resolution FontConfig performs. Rasterized glyphs are then cached in the GPU texture
atlas managed by `kitty/glyph-cache.c` (91 lines).

**Headless caveat (grounded).** The TUI font browser cannot run without a controlling
terminal. Real captured failure:

```
$ xvfb-run -a --server-args="-screen 0 1024x768x24" ./kitty/launcher/kitty +list-fonts
Error: open /dev/tty: no such device or address        # exit status 1
```

`+list-fonts` needs `/dev/tty`, which does not exist headless, so it exits `1`. Font
evidence is therefore taken from `--debug-font-fallback` (above), **not** `+list-fonts`.

### 8.2 Layout (the computed grid: columns × lines)

The terminal grid dimensions are computed as *window pixel size ÷ cell size*, where the
cell size derives from the resolved font metrics (§8.1). The live geometry was read back
via remote control (from the §7 `ls`, `env` omitted):

```
"columns": 71, "lines": 22
```

So the 640-px-class window (default `initial_window_width = (640, 'px')`, §6.2) rendered a
**71 columns × 22 lines** grid with DejaVu Sans Mono at `font_size = 11.0`. That Kitty
reports a concrete, nonzero `columns × lines` is evidence the layout engine sized the grid
from real font metrics — a prerequisite for placing any glyph.

### 8.3 Scrolling (scrollback history)

Lines that scroll off the top are retained in the scrollback buffer by
`kitty/history.c:287` → `historybuf_add_line(HistoryBuf *self, const Line *line, ANSIBuf *as_ansi_buf) {`.
The depth of that buffer is governed by the `scrollback_lines = 2000` default proven in
§6.2 (`definition.py:372`). Together these show the display system preserves 2000 lines of
history as new output pushes old lines upward. The cursor position within the grid is
tracked by the model in `kitty/cursor.c` (338 lines).

### 8.4 Screen updates (shader pipeline + buffer swap)

Drawing a frame is a GPU operation. The cell-rendering GLSL program is initialized at
`kitty/shaders.c:217` → `init_cell_program(void) {` — a multi-stage pipeline using
`kitty/cell_vertex.glsl` (233 lines) and `kitty/cell_fragment.glsl` (204 lines), with color
management in `kitty/srgb_gamma.h` (20 lines) and `kitty/linear2srgb.glsl` (15 lines). Once
a frame is drawn, the back buffer is presented to the (virtual) display.

> **★ Citation correction (verified).** The AAP originally placed the buffer-swap in
> `kitty/shaders.c`; that is **wrong at this commit**. The present/swap functions live in
> **`kitty/glfw.c`**. Verified:
>
> ```
> $ grep -n "swap_window_buffers\|glfwSwapBuffers" kitty/glfw.c
> 283:        swap_window_buffers(window);
> 1221:    if (glfwAreSwapsAllowed(glfw_window)) glfwSwapBuffers(glfw_window);
> 1802:swap_window_buffers(OSWindow *os_window) {
> 1803:    if (glfwAreSwapsAllowed(os_window->handle)) glfwSwapBuffers(os_window->handle);
> $ grep -n "swap_window_buffers\|glfwSwapBuffers" kitty/shaders.c
> (no matches)
> ```
>
> So the correct anchors are `kitty/glfw.c:1802` → `swap_window_buffers(OSWindow *os_window) {`
> and `kitty/glfw.c:1221` → `if (glfwAreSwapsAllowed(glfw_window)) glfwSwapBuffers(glfw_window);`
> (called from `kitty/glfw.c:283`). `kitty/shaders.c` (1285 lines) contains neither symbol.

The frame cadence is paced by the `repaint_delay = 10` default (§6.2, `definition.py:866`)
— roughly one repaint every 10 ms when there is something to draw.

### 8.5 The log/console messages that confirm the display system is active

Pulling the confirming messages together (all captured verbatim above):

- **GPU/OpenGL context up:** `kitty/gl.c:72` →
  `[0.127] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5`.
  A live GL 4.5 core-profile context is exactly what the shader pipeline (§8.4) needs.
- **Window surface up:** `kitty/glfw.c:1321` → `[0.150] OS Window created`.
- **Fonts resolved:** `kitty/fonts/render.py:163` → the `[0.162] Text fonts:` block naming
  DejaVu Sans Mono (§8.1).
- **Grid sized:** `columns: 71`, `lines: 22` from `kitten @ ls` (§8.2).

**Summary for Q4.** *Fonts* — the `Text fonts:` block resolves DejaVu Sans Mono across
Normal/Bold/Italic/Bold-Italic (`render.py:163`), rasterized by FreeType (`freetype.c`) and
cached in the GPU atlas (`glyph-cache.c`). *Layout* — a computed `71 × 22` grid
(`kitten @ ls`) derived from font metrics and the 640-px window. *Scrolling* — 2000 lines
of scrollback (`history.c:287` + `definition.py:372`). *Screen updates* — the GLSL cell
program (`shaders.c:217`) drawing frames that are presented by
`swap_window_buffers`/`glfwSwapBuffers` in **`kitty/glfw.c:1802/:1221`** (corrected anchor),
paced by `repaint_delay = 10`. The `GL version string`, `OS Window created`, and
`Text fonts:` log lines are the console evidence the display system is live.


---

## 9. Coverage-pass checklist

Re-reading each question and confirming every sub-part is addressed by content above:

- [x] **Q1a — Systems that start up enumerated.** §5.1 lists the full chain: native
  launcher (`main.c:439`) → single-instance dispatch (`main.c:436`) → CPython bootstrap
  (`main.c:211/216`) → entry dispatch (`entry_points.py:151`) → GUI `main()`
  (`main.py:524`) → GLFW/OpenGL (`main.py:514`, `gl.c:72`) → fonts (`render.py:163`) →
  Boss (`boss.py:323/375/1181`) → child-monitor threads (`child-monitor.c:229/230`) → OS
  window (`glfw.c:1321`) → child/PTY (`child.py`, `window.py:871`).
- [x] **Q1b — On-screen/log evidence quoted verbatim.** §5 quotes `GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2' Detected version: 4.5`,
  `OS Window created`, `Child launched`; §4.4 explains the `[%.3f]` elapsed-seconds format
  (`logging.c:56`); §5.3 labels the systemd line (`systemd.c:87`) benign.
- [x] **Q2a — Configuration sources identified.** §6.1: built-in defaults
  (`definition.py`, used wholesale under `--config NONE`), optional user `kitty.conf`, and
  `-o` overrides, merged by `load_config` (`config.py:163`) into `Options` (`types.py:471`).
- [x] **Q2b — Default literals cited + on-screen effects.** §6.1/§6.2: `font_size = 11.0`
  (`:59`), `cursor_shape = block`→enum `1` (`:320`), `scrollback_lines = 2000` (`:372`),
  `repaint_delay = 10` (`:866`), `input_delay = 3` (`:878`),
  `initial_window_width = (640, 'px')` (`:994`), `shell = '.'` (`:2896`).
- [x] **Q2c — Real-launch output proving settings applied.** §6.2 `+runpy` readback prints
  each default; §6.3 shows `-o` override `font_size` `11.0 → 12.0`; cursor enum confirmed
  `CURSOR_BLOCK= 1`.
- [x] **Q3a — Terminal-readiness path documented.** §7.1: PTY alloc (`child.py:170`),
  fork/exec (`child.py:276/281`), env + shell integration (`shell_integration.py:218/223`),
  VT parser (`vt-parser.c:230/261/457`), screen model (`screen.c:866/872`).
- [x] **Q3b — First-output proof shown.** §7.3: shell marker `READY_MARKER_42`
  round-tripped verbatim via `kitten @ get-text`; §7.2: `KSI=[enabled]`,
  `TERM=[xterm-kitty]`; geometry via `ls`.
- [x] **Q4a — Fonts evidence.** §8.1: the `Text fonts:` block resolves **DejaVu Sans Mono**
  (Normal/Bold/Italic/Bold-Italic), `render.py:163`; `+list-fonts` headless failure quoted.
- [x] **Q4b — Layout evidence.** §8.2: grid `columns: 71`, `lines: 22` from `kitten @ ls`.
- [x] **Q4c — Scrolling evidence.** §8.3: scrollback `historybuf_add_line`
  (`history.c:287`) with `scrollback_lines = 2000` (`definition.py:372`).
- [x] **Q4d — Screen-update evidence.** §8.4: GLSL cell program (`shaders.c:217`) + buffer
  swap in **`kitty/glfw.c:1802/:1221`** (citation correction noted and verified).
- [x] **Build/run preamble present with real exit codes.** §4.2 `BUILD_EXIT=0`; §4.3
  `RUN_EXIT=0`; §5 `RUN_EXIT=0`.
- [x] **Read-only + cleanup.** All observation scripts/logs were created only under `/tmp`
  and deleted; the only tracked change introduced by this task is this document,
  `blitzy/documentation/kitty_815df1e210e0.md`. Build artifacts (`*.so`,
  `kitty/launcher/kitt*`) are gitignored and do not affect the tracked tree.

### 9.1 Method & grounding notes

- **Run-first:** every quoted value above is real output captured from the commands shown,
  or an exact literal read from the source at commit `815df1e210e0…`. Nothing is invented.
- **Exactness:** values are given as literals (e.g. `scrollback_lines = 2000`,
  `columns: 71`, `lines: 22`, `cursor_shape = 1`), never paraphrased.
- **Security:** `kitten @ ls` emits the child environment, which contains real secrets in
  this container; every `ls` excerpt here has its `env` block removed. No secret appears in
  this document.
- **Honest limits:** `+list-fonts` genuinely cannot run headless (`open /dev/tty` error,
  §8.1); font evidence was taken from `--debug-font-fallback` instead. The
  `Failed to open systemd user bus … Connection refused` line is an expected
  no-systemd-user-bus artifact (`systemd.c:87`), not a defect.

