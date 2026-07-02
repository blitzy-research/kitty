# Kitty Terminal Emulator — Startup Investigation (commit 815df1e21)

## Investigation summary

This document answers four questions about what happens inside the [Kitty](https://sw.kovidgoyal.net/kitty/) terminal emulator during startup — from process launch up to the point where the shell is ready — by **building and running** Kitty **headlessly** and quoting the **real observed output** verbatim. Nothing was read-only-guessed: every behavioral claim below is paired with exactly one captured evidence line, the command that produced it, and an exact `file:line` citation into the source tree.

- **Branch:** `kitty_815df1e210e0`
- **HEAD commit:** `815df1e21` — *"Wire up applying of font config"*
- **Built binary version:** `kitty 0.35.2 created by Kovid Goyal`
- **Exact-commit confirmation:** the Go build embedded `VCSRevision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`, which matches the branch/commit under investigation.
- **Environment:** Ubuntu 24.04, Python 3.12.3, gcc 13.3.0, Go 1.22.2; headless via **Xvfb** display `:99` backed by **Mesa llvmpipe** software OpenGL.

Kitty is a GPU-accelerated terminal: it *requires* an OpenGL context to start. Since a headless host has no GPU, the entire investigation hinges on giving Kitty a **software** OpenGL stack (Mesa's `llvmpipe`) behind a virtual framebuffer. The recipe below records exactly how that context was provided and how each subsystem was made observable through Kitty's debug flags.

### Method / build-run-capture recipe

**1. Build command (authoritative).** The project's own CI defines the build command at `.github/workflows/ci.py:L104` (`cmd = f'{python} setup.py build --verbose'`):

```
python3 setup.py build --verbose
```

**Report reality — the first build FAILED.** On a stock Ubuntu 24.04 host (not the CI base image), the build aborted with this captured line:

```
The package libcrypto was not found on your system
```

This is because `setup.py` probes `libcrypto` through `pkg-config` (`setup.py:L253 def libcrypto_flags()` → `setup.py:L275 ldflags = pkg_config('libcrypto', '--libs', ...)`); the returned flags are consumed by the build at `setup.py:L616-L617` (the `libcrypto_flags()` call and `cflags.extend(libcrypto_cflags)`) and, for the link step, at `setup.py:L642` (`ans.ldpaths += ... + libcrypto_ldflags`). Installing `libssl-dev` (which provides `libcrypto.pc`) resolved it, after which the build completed (`BUILD_DONE rc=0`). **This `libcrypto`/`libssl-dev` prerequisite is required in addition to the apt install command at `.github/workflows/ci.py:L85-L88`** — the CI base image already ships it, so it is not repeated in that list.

**2. Native dependencies installed.** The authoritative apt command is at `.github/workflows/ci.py:L85-L88` (four implicitly-concatenated string literals that form one `apt-get install` invocation). Quoted **exactly** as implemented, in full:

```
sudo apt-get install -y libgl1-mesa-dev libxi-dev libxrandr-dev libxinerama-dev ca-certificates libxcursor-dev libxcb-xkb-dev libdbus-1-dev libxkbcommon-dev libharfbuzz-dev libx11-xcb-dev zsh libpng-dev liblcms2-dev libfontconfig-dev libxkbcommon-x11-dev libcanberra-dev libxxhash-dev uuid-dev libsimde-dev libsystemd-dev zsh bash dash systemd-coredump gdb
```

Note this list is quoted verbatim, so it includes `ca-certificates` (`.github/workflows/ci.py:L85`), `libsystemd-dev`, `bash`, `dash`, `systemd-coredump`, `gdb` (`.github/workflows/ci.py:L88`), and `zsh` **twice** — once at `.github/workflows/ci.py:L86` and again at `.github/workflows/ci.py:L88`. Of these, the subset actually consumed by Kitty's native C build is the graphics/text/X/desktop `-dev` libraries (`libgl1-mesa-dev`, `libxkbcommon-dev`, `libxkbcommon-x11-dev`, `libharfbuzz-dev`, `libpng-dev`, `liblcms2-dev`, `libfontconfig-dev`, `libxxhash-dev`, `uuid-dev`, `libsimde-dev`, `libsystemd-dev`, `libcanberra-dev`, and the `libx*`/`libxcb*`/`libxi`/`libxrandr`/`libxinerama`/`libxcursor`/`libdbus-1` packages); the remaining entries (`ca-certificates`, `zsh`, `bash`, `dash`, `systemd-coredump`, `gdb`) are CI shells, TLS roots, and debugging tooling rather than build inputs.

Beyond that CI list, the headless run and the failed-probe fix additionally require `xvfb mesa-utils libgl1-mesa-dri fonts-dejavu-core golang-go libssl-dev`. Language toolchains: Python `>=3.8` (`pyproject.toml:L2`), Go `1.22` (`go.mod:L3`).

**3. Headless OpenGL bring-up.** Start a virtual framebuffer, point `DISPLAY` at it, and force Mesa's software rasterizer:

```
Xvfb :99 -screen 0 1920x1080x24 &
export DISPLAY=:99
export LIBGL_ALWAYS_SOFTWARE=true
export GALLIUM_DRIVER=llvmpipe
```

**4. Verify the GL stack** with `glxinfo` before launching Kitty (captured verbatim; used again as corroborating evidence in Q1):

```
direct rendering: Yes
OpenGL vendor string: Mesa
OpenGL renderer string: llvmpipe (LLVM 20.1.2, 256 bits)
OpenGL core profile version string: 4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.2
OpenGL version string: 4.5 (Compatibility Profile) Mesa 25.2.8-0ubuntu0.24.04.2
```

**5. Main run command.** Launch the freshly built launcher with the three real debug flags, remote control enabled (so the drawn screen and effective config can be read back), and a scripted `bash` child:

```
./kitty/launcher/kitty --debug-rendering --debug-input --debug-font-fallback \
  -o allow_remote_control=yes --listen-on unix:/tmp/kitty_test.sock \
  bash --noprofile --norc /tmp/child.sh   # child script shown verbatim in Q3 item 4
```

---

## ⚠ Report-reality corrections

This investigation surfaced **four** report-reality corrections in total. **Two** of them correct statements in the governing plan that do not hold for this exact commit on the Linux/Xvfb target and are detailed in this section — Correction #1 (the Linux OpenGL minimum) and Correction #2 (`--debug-config` is not a CLI flag). The other **two** are reported inline where they arise: the `libcrypto`/`libssl-dev` build prerequisite (the failed first build documented above) and the non-fatal systemd user-bus line in Question 1 (the `[0.213] Failed to open systemd user bus with error: Connection refused` entry). All four are reported with their evidence, because the investigation is grounded in *observed* reality rather than an idealized description.

### Correction #1 — the Linux OpenGL minimum is **3.1**, not 3.3

The required minimum version is defined in `kitty/data-types.h:L19-L26`:

```
// Required minimum OpenGL version
#define OPENGL_REQUIRED_VERSION_MAJOR 3
#ifdef __APPLE__
#define OPENGL_REQUIRED_VERSION_MINOR 3
#else
#define OPENGL_REQUIRED_VERSION_MINOR 1
#endif
#define GLSL_VERSION 140
```

On the Linux/Xvfb target the `#else` branch is compiled, so `OPENGL_REQUIRED_VERSION_MINOR` = **1** (`kitty/data-types.h:L24`) and the hard minimum is **OpenGL 3.1**. The `3.3` value (`kitty/data-types.h:L22`) applies *only* to the `__APPLE__`/macOS branch. This is corroborated by `#define GLSL_VERSION 140` (`kitty/data-types.h:L26`) — GLSL 1.40 corresponds to OpenGL 3.1. The major version is `3` at `kitty/data-types.h:L20`. Mesa llvmpipe advertises OpenGL **4.5** (see the `glxinfo` block above), which satisfies either the 3.1 or the 3.3 bound, so startup proceeds on this host.

### Correction #2 — `--debug-config` is **not** a CLI flag at this commit

The only debug-related CLI flags defined at this commit are `--debug-rendering`/`--debug-gl` (`kitty/cli.py:L989`), `--debug-input`/`--debug-keyboard` (`kitty/cli.py:L996`), and `--debug-font-fallback` (`kitty/cli.py:L1002`). There is **no** `--debug-config` flag. Empirically confirmed with `./kitty/launcher/kitty --debug-config` (process exit code 1):

```
Unknown option: --debug-config
```

The effective-configuration dump is instead produced by the **`debug_config` action**, bound by default to `kitty_mod+f6` (`kitty/options/definition.py:L4256` → `'debug_config kitty_mod+f6 debug_config'`), implemented at `kitty/boss.py:L3059-L3060` (`@ac('debug', 'Show the effective configuration kitty is running with')` / `def debug_config(self)`), which renders the output built by `kitty/debug_config.py:L231 def debug_config(...)`. In this investigation it was captured through remote control (`kitten @ action debug_config` then `kitten @ get-text`). The plan's prose that calls `--debug-config` a CLI flag is therefore inaccurate for commit `815df1e21`.

---

## The four questions (verbatim)

1. "Set up a virtual framebuffer (Xvfb) to attempt headless execution. When Kitty launches, WHICH systems start up on the way to a working terminal, and what do you actually SEE on screen or in logs that shows them coming online?"
2. "How does Kitty decide on its initial configuration at startup? Which configuration SOURCES or DEFAULT settings does it use for the first launch, how do they affect what you see when the window appears, and what OUTPUT during a real launch shows those settings were applied?"
3. "How does Kitty get the terminal ready to communicate with the shell that will run inside it? When the shell prints its first output, what concrete BEHAVIOR shows the data was understood correctly and drawn in the terminal?"
4. "As first characters appear, what visible evidence about FONTS, LAYOUT, SCROLLING, or how the SCREEN UPDATES tells you the display system is active and working properly? Include any log or console messages that confirm this."

---

## Q1 — Startup subsystems and the evidence each is online

Kitty comes up as an ordered chain of subsystems: a native C launcher boots an embedded Python interpreter, Python's `main()` initializes windowing and the GPU, compiles shaders, initializes fonts, constructs the `Boss` controller and its child-monitor threads, forks a child on a freshly created PTY, and finally releases a terminal-ready handshake so the shell can run. Each stage below lists **(a)** its role, **(b)** exactly one verbatim evidence line, **(c)** the command/stream it came from, and **(d)** the `file:line`.

A note on the timestamps: every debug line carries a `[<time>]` prefix because the C debug macros funnel through a single timed printer — `debug_rendering(...)` (`kitty/state.h:L14`), `debug_input(...)` (`kitty/state.h:L15`), and `debug_fonts(...)` (`kitty/state.h:L16`) each expand to `timed_debug_print(__VA_ARGS__)`, which prepends the monotonic time. That is why observed evidence lines look like `[0.137] ...`.

### 1. Native launcher (process entry) — *first runtime evidence: the `--version` fast path*

**Role:** The OS executes a small native C binary that validates descriptors, resolves paths, and (for a normal launch) boots an embedded CPython interpreter that runs the rest of Kitty. The entry point is `kitty/launcher/main.c:L439` (`int main(int argc, char *argv[], char* envp[])`); for a normal launch it configures the interpreter via `PyConfig_InitPythonConfig` (`kitty/launcher/main.c:L193`), starts it with `Py_InitializeFromConfig` (`kitty/launcher/main.c:L211`), and hands control to Python with `Py_RunMain()` (`kitty/launcher/main.c:L216`), reached through `run_embedded(&run_data)` (`kitty/launcher/main.c:L464`).

**Runtime evidence (native — this proves the launcher, not Python):** the launcher parses the command line in native C *first*, via `handle_fast_commandline(argc, argv)` (`kitty/launcher/main.c:L453`). `--version` is a fast path served entirely there: `if (opts.version_requested)` (`kitty/launcher/main.c:L428`) prints the banner and calls `exit(0)` (`kitty/launcher/main.c:L434`) **before** `run_embedded()` (`kitty/launcher/main.c:L464`) is ever invoked — so no Python code runs for `--version`. The banner therefore demonstrates that the native launcher started and parsed arguments:

```
kitty 0.35.2 created by Kovid Goyal
```

**Command/stream:** `./kitty/launcher/kitty --version` (stdout, non-TTY). **Citation:** `kitty/launcher/main.c:L428` (the `opts.version_requested` fast path), `kitty/launcher/main.c:L432` (the non-TTY branch `printf("kitty %s created by Kovid Goyal\n", KITTY_VERSION)`), `kitty/launcher/main.c:L434` (`exit(0)`, before `run_embedded()` at `kitty/launcher/main.c:L464`).

### 2. Python entry-point dispatch

**Role:** For a normal launch (i.e. *not* the `--version`/`-v` fast path of item 1), the launcher boots Python and hands off to the entry-point module, which inspects `sys.argv` and routes the invocation. The dispatcher is `kitty/entry_points.py:L183 def main()`: it computes `first_arg` (`kitty/entry_points.py:L188`), looks it up (`kitty/entry_points.py:L189`), routes a `+`-prefixed subcommand to `namespaced(...)` (`kitty/entry_points.py:L192`), and otherwise — a normal GUI launch — does `from kitty.main import main as kitty_main` (`kitty/entry_points.py:L194`) then `kitty_main()` (`kitty/entry_points.py:L195`).

**Runtime evidence the Python layer booted and this dispatcher ran:** the `+runpy` command used in Q2 returns a value that only Python can compute, which requires `main()` to have dispatched the `+runpy` subcommand through `namespaced(...)` (`kitty/entry_points.py:L192`). One line of its captured output (full command and output in Q2) is:

```
term= 'xterm-kitty'
```

**Command/stream:** `./kitty/launcher/kitty +runpy '...print("term=",repr(d.term))...'` (stdout). **Citation:** `kitty/entry_points.py:L183`, `kitty/entry_points.py:L188-L192`.

The **normal GUI launch** takes the other branch — `kitty_main()` (`kitty/entry_points.py:L194-L195`). That this branch executed is proven downstream by the whole subsystem cascade in items 4–10 (for example `[0.200] OS Window created`), each line of which is emitted only by `kitty.main.main()` and its callees and is therefore reachable only through this dispatch. **Citation:** `kitty/entry_points.py:L194-L195`.

### 3. `main()` orchestration — *source-grounded*

**Role:** `kitty.main.main()` is the conductor: it initializes GLFW, creates the OS window (passing the shader-loader callback), constructs the `Boss`, starts it, and enters the child-monitor main loop. The ordered call sites are `init_glfw(...)` (`kitty/main.py:L514`), `create_os_window(...)` (`kitty/main.py:L221-L224`), `Boss(...)` (`kitty/main.py:L226`), `boss.start(...)` (`kitty/main.py:L227`), and finally `boss.child_monitor.main_loop()` (`kitty/main.py:L234`). These are structural call sites with no dedicated debug line; their effects are proven by the subsystem evidence that follows.

### 4. GLFW / windowing + keyboard (XKB) — *first observable logs*

**Role:** GLFW brings up the display backend, loads the keyboard (XKB) keymap, requests an OpenGL context of the required version, and creates the OS window. This is the first subsystem that emits observable log lines. The XKB keymap load:

```
[0.079] Loading new XKB keymaps
```

**Command/stream:** the main run with `--debug-rendering` (stderr). **Citation:** `glfw/xkb_glfw.c:L672`.

The XKB modifier-index resolution (proves the keymap was parsed, not just loaded):

```
[0.083] Modifier indices alt: 0x3 super: 0x6 hyper: 0xffffffff meta: 0xffffffff numlock: 0x4 shift: 0x0 capslock: 0x1
```

**Command/stream:** same run, stderr. **Citation:** `glfw/xkb_glfw.c:L376`.

Before creating the window, GLFW requests the *required* GL context version by feeding the constants from `kitty/data-types.h` into window hints: `glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, OPENGL_REQUIRED_VERSION_MAJOR)` (`kitty/glfw.c:L1127`) and `glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, OPENGL_REQUIRED_VERSION_MINOR)` (`kitty/glfw.c:L1128`). The OS window is then created:

```
[0.200] OS Window created
```

**Command/stream:** same run, stderr (via the `debug(...)` macro). **Citation:** `kitty/glfw.c:L1321`.

If the software GL stack had been unavailable, GLFW would have aborted earlier at the temp-window creation with `fatal("Failed to create GLFW temp window! ...")` (`kitty/glfw.c:L1199`). **Report reality: this path was NOT hit** — llvmpipe provided a valid context, so the window came up normally.

### 5. OpenGL context + version check (GPU online) — *the key line*

**Role:** With a context current, Kitty loads the GL function pointers, checks the version against the hard minimum, and (under `--debug-rendering`) prints the detected version. This is the single most important "the GPU is online" line:

```
[0.137] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.2' Detected version: 4.5
```

**Command/stream:** the main run with `--debug-rendering` — note this is emitted by a bare `printf` (so it appears on **stdout**, not stderr). **Citation:** `kitty/gl.c:L72` (`if (global_state.debug_rendering) printf("[%.3f] GL version string: %s\n", monotonic_t_to_s_double(monotonic()), gl_version_string());`). The `'%s' Detected version: %d.%d` format is built by the `gl_version_string()` helper at `kitty/gl.c:L47` (inside the function spanning `kitty/gl.c:L41-L48`).

The version gate immediately follows: `fatal("OpenGL version is %d.%d, version >= %d.%d required for kitty", ...)` at `kitty/gl.c:L74`. It was **NOT** triggered because the detected `4.5` is ≥ the Linux minimum `3.1` (Correction #1). This is corroborated by the earlier `glxinfo` output showing `llvmpipe (LLVM 20.1.2, 256 bits)` at OpenGL `4.5`.

### 6. Shaders (GLSL compile/link) — *source-grounded*

**Role:** Kitty compiles and links its GLSL programs (including the cell shaders that draw glyphs, cursor, and selection) before it can draw anything. The shader-loader callback `load_all_shaders` is defined at `kitty/main.py:L82` and is passed into `create_os_window(...)` at `kitty/main.py:L225`; it calls `load_shader_programs(...)` at `kitty/main.py:L84`. `load_shader_programs` is the `LoadShaderPrograms()` instance at `kitty/shaders.py:L204`; the actual GLSL compile/link happens in `LoadShaderPrograms.__call__` (`kitty/shaders.py:L147`) via `Program.compile(...)` (`kitty/shaders.py:L87`). The cell stage uses `kitty/cell_vertex.glsl` and `kitty/cell_fragment.glsl`, targeting `#define GLSL_VERSION 140` (`kitty/data-types.h:L26`). There is no dedicated shader-success log line; reaching *"OS Window created"* and then drawing implies the shaders compiled and linked, because Kitty aborts on a compile/link error. Note that `kitty/shaders.c:L1255-L1258` register the GL enum constants `GL_VERSION`, `GL_VENDOR`, `GL_SHADING_LANGUAGE_VERSION`, and `GL_RENDERER` into the module — these are **constants, not log lines**; the authoritative "GL is online" proof remains the `kitty/gl.c:L72` line above.

### 7. Fonts initialization

**Role:** Kitty resolves the four base text faces (regular/bold/italic/bold-italic) from the configured family and reports them under `--debug-font-fallback`:

```
[0.217] Text fonts:
[0.217]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.217]   Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
[0.217]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.217]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

**Command/stream:** the main run with `--debug-font-fallback` (stderr). **Citation:** `kitty/fonts/render.py:L161 def dump_font_debug()` → `kitty/fonts/render.py:L163 log_error('Text fonts:')` → the label mapping loop `{'medium': 'Normal', 'bold': 'Bold', 'italic': 'Italic', 'bi': 'Bold-Italic'}` at `kitty/fonts/render.py:L164`. This dump is emitted from `kitty/main.py:L228-L229` (`if args.debug_font_fallback: dump_font_debug()`).

### 8. Boss controller + child-monitor threads — *source-grounded*

**Role:** The `Boss` is the central controller that owns windows and the child-monitoring engine. It constructs `self.child_monitor = ChildMonitor(...)` (`kitty/boss.py:L370`), and `boss.start()` starts it via `self.child_monitor.start()` (`kitty/boss.py:L1183`). The three-thread engine (main/IO/talk) and its per-frame loop live in C: `main_loop(...)` (`kitty/child-monitor.c:L1259`) calls `run_main_loop(...)` (`kitty/child-monitor.c:L1262`), and each frame runs `render(...)` (`kitty/child-monitor.c:L871`). There is no single "child monitor online" log line; its being online is proven by the subsequent *"Child launched"* line and by the fact that the shell's output is later drawn to the screen (Q3).

### 9. Child fork + PTY setup

**Role:** Kitty allocates a pseudo-terminal and forks the child that will run the shell. The PTY is created at `kitty/child.py:L171` (`master, slave = os.openpty()  # Note that master and slave are in blocking mode`). Runtime proof that the fork + setup completed:

```
[0.216] Child launched
```

**Command/stream:** the main run (stderr). **Citation:** `kitty/window.py:L871` — this line is printed only after the child is forked and the terminal has been marked ready.

### 10. Terminal-ready handshake

**Role:** The child, after forking, blocks until Kitty signals that the screen object is set up and the PTY sized. The child waits in `wait_for_terminal_ready(int fd)` (`kitty/child.c:L71`), called at `kitty/child.c:L152` right after the comment `// Wait for READY_SIGNAL which indicates kitty has setup the screen object` (`kitty/child.c:L150`). The parent releases it by closing the signal fd in `mark_terminal_ready` (`kitty/child.py:L362-L363`, `def mark_terminal_ready(self): os.close(self.terminal_ready_fd)`), which makes the child's blocking `read` return EOF. The observable marker that the handshake completed is the same *"Child launched"* line shown above (`kitty/window.py:L871`); the correctness-critical ordering is detailed in Q3.

### Ordered timeline (both streams merged by timestamp)

Combining the stdout GL line with the stderr debug lines by their `[<time>]` prefix gives the real startup order observed on this host:

| Time | Event | Stream | Citation |
|------|-------|--------|----------|
| `0.079` | Loading new XKB keymaps | stderr | `glfw/xkb_glfw.c:L672` |
| `0.083` | Modifier indices resolved | stderr | `glfw/xkb_glfw.c:L376` |
| `0.137` | GL version string 4.5 detected | **stdout** | `kitty/gl.c:L72` |
| `0.200` | OS Window created | stderr | `kitty/glfw.c:L1321` |
| `0.213` | systemd user bus unavailable (reality) | stderr | `kitty/systemd.c:L87` |
| `0.216` | Child launched | stderr | `kitty/window.py:L871` |
| `0.217` | Text fonts resolved | stderr | `kitty/fonts/render.py:L163` |

The `0.213` entry is an honest "reality" observation: the sandbox has no systemd user bus, so Kitty logs a **non-fatal** warning and continues:

```
[0.213] Failed to open systemd user bus with error: Connection refused
```

**Command/stream:** the main run (stderr). **Citation:** `kitty/systemd.c:L87` (`log_error("Failed to open systemd user bus with error: %s", strerror(-ret)); return;`). Because the function simply `return`s after logging, this does not abort startup.

---

## Q2 — How Kitty decides its initial configuration

Kitty's initial configuration comes from two named inputs: configuration **SOURCES** (a `kitty.conf` file located via the config directory, plus `--config`/`--override` on the command line) and, when no file overrides a setting, the built-in **DEFAULT settings**. This section covers both, explains how the defaults shape the first window, and shows the real output proving which settings were applied.

### 1. Sources / resolution order

**Role:** Kitty looks for `kitty.conf` in a config directory and merges any CLI overrides on top. The file location is computed at `kitty/constants.py:L133` (`defconf = os.path.join(config_dir, 'kitty.conf')`), where `config_dir` comes from `_get_config_dir()` (`kitty/constants.py:L87`). That resolver honors `KITTY_CONFIG_DIRECTORY` first (`kitty/constants.py:L88-L89`) and otherwise falls back through `XDG_CONFIG_HOME` (`kitty/constants.py:L92-L93`). Command-line overrides are `--config`/`-c` (`kitty/cli.py:L870`) and `--override`/`-o` (`kitty/cli.py:L876`). The merge of parsed file values over defaults — including conflict detection — happens in `load_config()` (defined at `kitty/config.py:L163`, body `kitty/config.py:L164-L186`).

Because this run used an **empty** `KITTY_CONFIG_DIRECTORY` with **no** `kitty.conf` present, Kitty falls back entirely to the built-in defaults — i.e., the genuine "first launch" scenario.

### 2. Built-in DEFAULT settings

**Role:** Every option's default value is declared in `kitty/options/definition.py`. Rather than read them, the exact defaults were captured from the built binary by materializing the generated `defaults` object (this uses `+runpy`, so it needs no GL and runs headlessly). Command:

```
./kitty/launcher/kitty +runpy 'from kitty.options.types import defaults as d; print("term=",repr(d.term)); print("font_family=", d.font_family); print("font_size=",d.font_size); print("scrollback_lines=",d.scrollback_lines); print("repaint_delay=",d.repaint_delay); print("input_delay=",d.input_delay); print("sync_to_monitor=",d.sync_to_monitor)'
```

Captured verbatim:

```
term= 'xterm-kitty'
font_family= FontSpec(family='', style='', postscript_name='', full_name='', system='monospace', axes=(), variable_name='', created_from_string='')
font_size= 11.0
scrollback_lines= 2000
repaint_delay= 10
input_delay= 3
sync_to_monitor= True
```

Each captured value is paired below with the exact default that produced it:

- `term= 'xterm-kitty'` ← `kitty/options/definition.py:L3242` (`opt('term', 'xterm-kitty', ...)`).
- `font_family= ... system='monospace' ...` ← `kitty/options/definition.py:L35` (`opt('font_family', 'monospace', option_type='parse_font_spec', ...)`). The default family is the abstract `monospace`, which the font subsystem later resolves to a concrete face.
- `font_size= 11.0` ← `kitty/options/definition.py:L59` (`opt('font_size', '11.0', ...)`).
- `scrollback_lines= 2000` ← `kitty/options/definition.py:L372` (`opt('scrollback_lines', '2000', ...)`).
- `repaint_delay= 10` (milliseconds) ← `kitty/options/definition.py:L866` (`opt('repaint_delay', '10', ...)`).
- `input_delay= 3` (milliseconds) ← `kitty/options/definition.py:L878` (`opt('input_delay', '3', ...)`).
- `sync_to_monitor= True` ← `kitty/options/definition.py:L889` (`opt('sync_to_monitor', 'yes', ...)`).

### 3. How the defaults affect the first window

Each default has a direct, visible consequence on the window that appears:

- `term='xterm-kitty'` becomes the child's `TERM` environment variable, defining how programs address the terminal (proven in Q3).
- `font_family=monospace` resolves to a real face — the `DejaVuSansMono` faces seen in Q1 item 7 (and again in the dump below) — which determines the glyph shapes drawn.
- `font_size=11.0` fixes the pixel cell size and therefore the initial number of columns × lines for a given window size.
- `scrollback_lines=2000` sizes the history buffer, so 2000 lines of scrollback are available from first launch (see Q4).
- `repaint_delay=10` (ms) together with `sync_to_monitor=yes` govern the render cadence — how often the screen is repainted.

### 4. OUTPUT proving the settings were applied

The proof that these settings were actually applied is the effective-configuration dump. First, the empirical confirmation from Correction #2 that there is no CLI flag for it — command `./kitty/launcher/kitty --debug-config`:

```
Unknown option: --debug-config
```

The real dump was then obtained through the `debug_config` action over remote control — `kitten @ --to <sock> action debug_config` (the action is `Boss.debug_config()` at `kitty/boss.py:L3060`, which calls `debug_config(get_options())`), then read back with `kitten @ --to <sock> get-text --extent all`. To keep the long font/OpenGL/PATH lines from wrapping, the window was first widened at runtime with `kitten @ set-font-size 8` (yielding a 133-column grid; this is a live font-size change, **not** a config override). Captured verbatim (ANSI-stripped) through the config-comparison section:

```
kitty 0.35.2 (815df1e210) created by Kovid Goyal
Linux 341b68ee4f91 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64
Ubuntu 24.04.2 LTS 341b68ee4f91 /dev/tty

DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=24.04
DISTRIB_CODENAME=noble
DISTRIB_DESCRIPTION="Ubuntu 24.04.2 LTS"
Running under: X11
OpenGL: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.2' Detected version: 4.5
Frozen: False
Fonts:
  medium: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
  bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
  italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
  bi: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
Paths:
  kitty: /tmp/kitty_build/kitty/launcher/kitty
  base dir: /tmp/kitty_build
  extensions dir: /tmp/kitty_build/kitty
  system shell: /bin/bash
Loaded config overrides:
  allow_remote_control yes

Config options different from defaults:
allow_remote_control yes

Important environment variables seen by the kitty process:
```

The dump then continues with the kitty process's own environment variables; its first entry, `PATH`, is longer than even the 133-column window and appears wrapped by the scrollback pager (ending `…:/usr/local/>` with a trailing `:` pager prompt), so that trailing section is elided here. It is the *parent kitty process's* environment — the *child shell's* environment is the subject of Q3 and is shown there.

**Citation:** the dump is produced by `kitty/debug_config.py:L231 def debug_config(opts)`. In print order: the version line is `kitty/debug_config.py:L235` (`p(version(add_rev=True))`); the `os.uname()` line is `kitty/debug_config.py:L236` (`p(' '.join(os.uname()))`); the `/etc/issue` line is `kitty/debug_config.py:L240-L252`; the `/etc/lsb-release` block is `kitty/debug_config.py:L253-L255`; `Running under:` is `kitty/debug_config.py:L257`; `OpenGL:` is `kitty/debug_config.py:L258` (`p(green('OpenGL:'), opengl_version_string())`); `Frozen:` is `kitty/debug_config.py:L259`; `Fonts:` is `kitty/debug_config.py:L260-L263`; the `Paths:` block is `kitty/debug_config.py:L264-L268` — including `extensions dir:` at `kitty/debug_config.py:L267`; `Loaded config overrides:` is `kitty/debug_config.py:L272-L274`; the `Config options different from defaults:` heading and its list come from `compare_opts(opts, p)` at `kitty/debug_config.py:L275`; and the `Important environment variables seen by the kitty process:` heading is `kitty/debug_config.py:L277`.

Three things this dump proves:

- **The effective config is pure defaults except for the single override deliberately added for capture.** The `Config options different from defaults:` heading is **unconditional** — `compare_opts()` prints it every time at `kitty/debug_config.py:L75` (`print('Config options different from defaults:')`), and `debug_config()` always calls `compare_opts()` at `kitty/debug_config.py:L275`, so its presence or absence proves nothing on its own. What is meaningful is the *list beneath it*: it contains exactly one line, `allow_remote_control yes`, which is precisely the `-o allow_remote_control=yes` override this investigation passed to enable remote-control capture. Every other option therefore equals its built-in default. (The same single override is echoed under `Loaded config overrides:`, populated from `opts.config_overrides` at `kitty/debug_config.py:L272-L274`.)
- **No user `kitty.conf` was loaded — true first-launch defaults.** The dump has **no** `Loaded config files:` section. That section is printed *only* `if opts.config_paths:` (`kitty/debug_config.py:L269-L271`), so its absence means `opts.config_paths` is empty — i.e. no `kitty.conf` was found or loaded. This — not the always-present "different from defaults" heading — is the correct evidence that the settings are untouched defaults, and it is corroborated by the independently-captured `+runpy` defaults in item 2.
- **Font config was applied (HEAD-commit relevance).** The HEAD commit `815df1e21` is *"Wire up applying of font config"*. The `Fonts:` section shows the abstract `monospace` default was actually applied and resolved to concrete faces (`DejaVuSansMono` and its bold/italic/bi variants) at startup — the same faces reported by the `--debug-font-fallback` dump in Q1, confirming the font configuration path is live at this commit.


---

## Q3 — Getting the terminal ready to talk to the shell

Getting the terminal ready to communicate with the shell has three parts: create a pseudo-terminal (PTY), populate the child's environment so programs know what terminal they are talking to, and perform a **terminal-ready handshake** that guarantees the shell sees the correct terminal size before it runs. Then, when the shell prints, the bytes must be *parsed* and *drawn*.

### 1. PTY creation — *source-grounded*

**Role:** A pseudo-terminal is the bidirectional channel between Kitty (master) and the shell (slave). Kitty creates it at `kitty/child.py:L171`:

```
master, slave = os.openpty()  # Note that master and slave are in blocking mode
```

This is the literal source line; `os.openpty()` returns the master/slave fd pair, after which Kitty marks the descriptors inheritable and sets UTF-8 mode on the slave. **Citation:** `kitty/child.py:L171`.

### 2. Child environment population

**Role:** Before `exec`-ing the shell, Kitty injects environment variables that tell programs what terminal they are running in and how to talk back to Kitty. These were read back live from the running child with `kitten @ --to <sock> ls`, which emits pretty-printed JSON. The terminal-related keys of the window's `env` object are quoted **verbatim** below, exactly as `kitten @ ls` prints them (alphabetically ordered, 14-space indent); the object's other 17 entries (`HOME`, `PATH`, `DISPLAY`, `SHELL`, and so on) are omitted:

```
              "COLORTERM": "truecolor",
              "KITTY_INSTALLATION_DIR": "/tmp/kitty_build",
              "KITTY_LISTEN_ON": "unix:/tmp/kitty_test.sock",
              "KITTY_PID": "624",
              "KITTY_PUBLIC_KEY": "1:IXg7()8DI`^it|01{aQ|_NENNljMuX?C6I<M9w8g",
              "KITTY_WINDOW_ID": "1",
              "TERM": "xterm-kitty",
              "TERMINFO": "/tmp/kitty_build/terminfo",
```

Each displayed variable is grounded in the exact assignment that produced it:

- `TERM` ← `kitty/child.py:L242` (`env['TERM'] = opts.term`); the value comes from the `term` option whose default is `xterm-kitty` (Q2).
- `COLORTERM` ← `kitty/child.py:L243` (`env['COLORTERM'] = 'truecolor'`), advertising 24-bit color.
- `KITTY_PID` ← `kitty/child.py:L244` (`env['KITTY_PID'] = getpid()`); its per-session value in this run is `624`.
- `KITTY_PUBLIC_KEY` ← `kitty/child.py:L245` (`env['KITTY_PUBLIC_KEY'] = boss.encryption_public_key`); a per-session public key (the exact value appears in the JSON above).
- `KITTY_LISTEN_ON` ← `kitty/child.py:L246-L247` (`if self.add_listen_on_env_var and boss.listening_on:` → `env['KITTY_LISTEN_ON'] = boss.listening_on`); its value is the `--listen-on` socket this run passed.
- `TERMINFO` ← `kitty/child.py:L258` (`env['TERMINFO'] = tdir`) in path mode; the alternative direct mode uses `base64_terminfo_data()` at `kitty/child.py:L260` (defined at `kitty/child.py:L184`).
- `KITTY_INSTALLATION_DIR` ← `kitty/child.py:L261` (`env['KITTY_INSTALLATION_DIR'] = kitty_base_dir`).
- `KITTY_WINDOW_ID` ← `kitty/tabs.py:L491` (`fenv['KITTY_WINDOW_ID'] = str(next_window_id())`); this one is assigned by the tab when the window is created, not in `child.py`.

The `TERM` value `xterm-kitty` is defined by `terminfo/kitty.terminfo`, whose first line is `xterm-kitty|KovIdTTY,` (`terminfo/kitty.terminfo:L1`) — this is the terminfo entry exported to the child so that `tput`/`ncurses` programs know Kitty's capabilities. Shell integration is layered on top by `kitty/shell_integration.py` (`def modify_shell_environ` at `kitty/shell_integration.py:L218`), which sources the per-shell scripts under `shell-integration/` (directories `bash`, `fish`, `ssh`, and `zsh`).

### 3. Terminal-ready handshake + correctness-critical ordering

**Role:** The handshake guarantees the shell observes the correct terminal size *before* it starts producing output. After forking, the child **blocks** at `wait_for_terminal_ready(ready_read_fd)` (`kitty/child.c:L152`; function defined at `kitty/child.c:L71`), immediately after the comment `// Wait for READY_SIGNAL which indicates kitty has setup the screen object` (`kitty/child.c:L150`).

The parent only releases that block **after** the first layout has sized the PTY. The ordering is visible in `kitty/window.py`'s geometry setup: `boss.child_monitor.resize_pty(self.id, *current_pty_size)` runs **first** (`kitty/window.py:L863`), and only **then** `self.child.mark_terminal_ready()` (`kitty/window.py:L866`) — which is `def mark_terminal_ready(self): os.close(self.terminal_ready_fd)` (`kitty/child.py:L362-L363`). Closing that fd makes the child's blocking `read` return EOF and unblocks it. **Why this ordering matters:** if `mark_terminal_ready()` ran before `resize_pty()`, the shell could start and query its size before the PTY was resized, seeing a wrong (default) geometry. Sizing first, signalling second, closes that race. Runtime proof that the handshake completed:

```
[0.216] Child launched
```

**Command/stream:** the main run (stderr). **Citation:** `kitty/window.py:L871` (printed right after `mark_terminal_ready()`).

### 4. Concrete behavior showing the first output was understood and drawn

**Role:** When the shell prints, the bytes travel PTY → parser → screen cells. Concretely: the child-monitor reads the bytes in `read_bytes(...)` (`kitty/child-monitor.c:L1337`) into the VT parser's write buffer; `kitty/vt-parser.c` classifies and dispatches them, routing printable text to `screen_draw_text(...)` at `kitty/vt-parser.c:L236`; and `screen_draw_text(Screen*, const uint32_t*, size_t)` at `kitty/screen.c:L866` writes those characters into the on-screen cells. Because `kitten @ get-text` reads back the **cell buffer** (what is actually drawn), its returning our text is direct proof that parsing and drawing succeeded.

**The producer.** The child was the small `bash` script from the Method-section run command (`bash --noprofile --norc /tmp/child.sh`), shown here **verbatim** so the exact statements that produced each output line are visible:

```
#!/bin/bash
echo "HELLO_FROM_SHELL_12345"
echo "TERM=[$TERM] COLORTERM=[$COLORTERM] KITTY_PID=[$KITTY_PID] KITTY_WINDOW_ID=[$KITTY_WINDOW_ID] TERMINFO=[$TERMINFO]"
echo "cols=$(tput cols) lines=$(tput lines)"
sleep 300
```

**The composite observation.** Reading the drawn screen with `kitten @ --to <sock> get-text` returned exactly the three lines below. This is **one composite capture**; each line is used as evidence for a *distinct* claim, split out beneath it:

```
HELLO_FROM_SHELL_12345
TERM=[xterm-kitty] COLORTERM=[truecolor] KITTY_PID=[624] KITTY_WINDOW_ID=[1] TERMINFO=[/tmp/kitty_build/terminfo]
cols=71 lines=22
```

- **Claim — the shell's first output was parsed and drawn into cells.** Produced by the script's `echo "HELLO_FROM_SHELL_12345"`; evidence is line 1 of the capture:

  ```
  HELLO_FROM_SHELL_12345
  ```

  `kitten @ get-text` reads this back from the drawn **cell buffer**, so the bytes were classified/dispatched by the VT parser (`kitty/vt-parser.c:L236`) and written to cells by `screen_draw_text(...)` (`kitty/screen.c:L866`).

- **Claim — the `TERM` environment variable propagated into the child.** Produced by the script's `echo "TERM=[$TERM] ..."`; evidence is line 2 of the capture:

  ```
  TERM=[xterm-kitty] COLORTERM=[truecolor] KITTY_PID=[624] KITTY_WINDOW_ID=[1] TERMINFO=[/tmp/kitty_build/terminfo]
  ```

  The child expanded `$TERM` to `xterm-kitty`, independently confirming the `env['TERM'] = opts.term` assignment (`kitty/child.py:L242`) took effect inside the child. (The same line echoes `KITTY_PID=[624]`, `KITTY_WINDOW_ID=[1]`, and `TERMINFO=[/tmp/kitty_build/terminfo]`, matching the item-2 JSON.)

- **Claim — the PTY size negotiated by the handshake reached the shell.** Produced by the script's `echo "cols=$(tput cols) lines=$(tput lines)"`; evidence is line 3 of the capture:

  ```
  cols=71 lines=22
  ```

  `tput cols`/`tput lines` query the `xterm-kitty` terminfo through the PTY; the result `71 × 22` **matches** the `kitten @ ls` JSON geometry (`"columns": 71`, `"lines": 22`), proving the size set by `resize_pty(...)` (`kitty/window.py:L863`) *before* the terminal-ready release (item 3) propagated all the way to the shell.


---

## Q4 — Evidence the display system is active

As the first characters appear, four distinct facets of the display system show it is working: **FONTS**, **LAYOUT**, **SCROLLING**, and **how the SCREEN UPDATES**. Each named item is addressed explicitly below with its evidence.

### FONTS

**Role/evidence:** The primary evidence that font handling is active is the `Text fonts:` block from Q1 item 7 — the abstract `monospace` family was resolved to four concrete `DejaVuSansMono` faces (Normal/Bold/Italic/Bold-Italic). The line that anchors it:

```
[0.217] Text fonts:
```

**Command/stream:** the main run with `--debug-font-fallback` (stderr). **Citation:** `kitty/fonts/render.py:L163` (the loop that prints each face is at `kitty/fonts/render.py:L164`). The `--debug-font-fallback` flag itself (`kitty/cli.py:L1002`) is what makes the selection observable.

The underlying font machinery (source-grounded — these are the code paths, not separate log lines):

- **Discovery** — FontConfig selects the face for a family at `kitty/fontconfig.c:L276` (`match = FcFontMatch(NULL, pat, &result)`).
- **Rasterization** — FreeType renders glyphs into cell bitmaps at `kitty/freetype.c:L675` (`render_glyphs_in_cells(...)`), which calls `FT_Render_Glyph(...)` at `kitty/freetype.c:L904`.
- **GPU glyph cache/atlas** — rendered glyphs are cached to a texture atlas in `kitty/glyph-cache.c`: the `SpritePosItem` struct that maps a glyph to its atlas position is defined at `kitty/glyph-cache.c:L12-L16`, and cache lookups/insertions go through `find_or_create_sprite_position(...)` at `kitty/glyph-cache.c:L33-L54`.

The `Fonts:` section of the `debug_config` dump (Q2) independently confirms the same four faces were applied.

### LAYOUT

**Role/evidence:** Layout is the cell grid — how many columns × lines fit — and the matching PTY size. The evidence is the agreement between what Kitty reports and what the shell sees. From `kitten @ ls` the window JSON reports `columns=71 lines=22`, and the shell's own `tput` output (Q3) reported:

```
cols=71 lines=22
```

**Command/stream:** the scripted `bash` child running `tput cols`/`tput lines`, read back via `kitten @ get-text` (see Q3). **Citation:** the PTY size tuple `current_pty_size` is computed as `(lines, columns, width_px, height_px)` at `kitty/window.py:L857-L859`. The fact that Kitty's grid and the shell's terminfo-derived size agree at `71 × 22` proves the layout was computed and propagated correctly.

### SCREEN UPDATES

**Role/evidence:** Screen updates are driven by the render cycle, and a PTY-size change (from recomputing the cell grid) is signalled to the child with `SIGWINCH`. To capture real update activity, the font size was changed at runtime (`kitten @ --to <sock> set-font-size 24` then `kitten @ --to <sock> set-font-size 8`), which recomputes the cell grid and resizes the PTY. Captured verbatim:

```
[5.998] SIGWINCH sent to child in window: 1 with size: (13, 42, 798, 494)
[7.023] SIGWINCH sent to child in window: 1 with size: (38, 133, 798, 494)
```

**Command/stream:** the `set-font-size` remote-control commands (stderr). **Citation:** `kitty/window.py:L873`. The tuple is `(lines, columns, width_px, height_px)`: font-size 24 yields a `13 × 42` grid, font-size 8 yields a `38 × 133` grid, while the window stays `798 × 494` px — proving that recomputing the cell grid drives layout and that the PTY is resized on screen updates. The per-frame GPU render cycle that pushes cell data to the GPU lives in `kitty/child-monitor.c`: `send_cell_data_to_gpu(...)` for the tab bar at `kitty/child-monitor.c:L714` and for window cells at `kitty/child-monitor.c:L766`, invoked from `render(...)` (`kitty/child-monitor.c:L871`). The default cadence is `repaint_delay=10` ms (`kitty/options/definition.py:L866`) with `sync_to_monitor=yes` (`kitty/options/definition.py:L889`). That `get-text` returns fully drawn cells (Q3) *and* GL 4.5 is online (Q1) is the combined proof that the render path actually executes.

### SCROLLING

**Role/evidence:** Scrolling is backed by the scrollback/history buffer. The captured default (Q2) shows how much history is available on first launch:

```
scrollback_lines= 2000
```

**Command/stream:** the `+runpy` defaults capture (stdout) from Q2. **Citation:** `kitty/options/definition.py:L372` (`opt('scrollback_lines', '2000', ...)`). The screen model maintains this history in `kitty/screen.c`: the `Screen` object's history buffer is allocated at `kitty/screen.c:L130` (`self->historybuf = alloc_historybuf(MAX(scrollback, lines), columns, OPT(scrollback_pager_history_size))`), and the render path consults the scroll position `scrolled_by` at `kitty/child-monitor.c:L673` when deciding what to draw. So on first launch, **2000 lines of scrollback are available by default**.


---

## Coverage pass — every named item addressed

Re-reading each question and confirming every distinct thing it names is answered above, with the evidence used:

**Q1 — WHICH systems start up + what you SEE:**

- [x] Xvfb virtual framebuffer set up — recipe in *Method* (`Xvfb :99 -screen 0 1920x1080x24`), corroborated by the `glxinfo` block.
- [x] Native launcher (process entry) — `kitty/launcher/main.c:L439`; native fast-path runtime evidence `kitty 0.35.2 created by Kovid Goyal` from `--version` (`kitty/launcher/main.c:L428`/`L432`/`L434`, which `exit(0)`s before `run_embedded()` at `kitty/launcher/main.c:L464`).
- [x] Python entry-point dispatch — `kitty/entry_points.py:L183`/`L188-L195`; runtime evidence the Python layer ran via `+runpy` (`term= 'xterm-kitty'`), with the GUI branch (`kitty/entry_points.py:L194-L195`) proven by the downstream item 4–10 cascade. The `--version` banner is native-launcher evidence (item 1), **not** proof of Python dispatch.
- [x] `main()` orchestration — source-grounded, `kitty/main.py:L221-L234`, `kitty/main.py:L514`.
- [x] GLFW / windowing — evidence `[0.200] OS Window created` (`kitty/glfw.c:L1321`).
- [x] XKB keyboard — evidence `[0.079] Loading new XKB keymaps` (`glfw/xkb_glfw.c:L672`) and `[0.083] Modifier indices ...` (`glfw/xkb_glfw.c:L376`).
- [x] OpenGL context + version check — evidence `[0.137] GL version string: '4.5 ...' Detected version: 4.5` (`kitty/gl.c:L72`).
- [x] Shaders (GLSL compile/link) — source-grounded, `kitty/main.py:L82`/`L84`, `kitty/shaders.py:L204`/`L147`/`L87`.
- [x] Fonts — evidence `[0.217] Text fonts:` (`kitty/fonts/render.py:L163`).
- [x] Boss controller — source-grounded, `kitty/boss.py:L370`.
- [x] Child-monitor threads — source-grounded, `kitty/boss.py:L1183`, `kitty/child-monitor.c:L1259`/`L1262`/`L871`.
- [x] Child fork + PTY — evidence `[0.216] Child launched` (`kitty/window.py:L871`); PTY at `kitty/child.py:L171`.
- [x] Terminal-ready handshake — `kitty/child.c:L71`/`L150`/`L152`, released by `kitty/child.py:L362-L363`.
- [x] Ordered timeline + honest systemd reality line `[0.213] Failed to open systemd user bus with error: Connection refused` (`kitty/systemd.c:L87`).

**Q2 — SOURCES / DEFAULT settings / how they affect the window / OUTPUT proving applied:**

- [x] SOURCES — `kitty.conf` via config dir (`kitty/constants.py:L133`/`L87`), `--config` (`kitty/cli.py:L870`), `--override` (`kitty/cli.py:L876`), merge in `kitty/config.py:L163`.
- [x] DEFAULT settings — `term`, `font_family`, `font_size`, `scrollback_lines`, `repaint_delay`, `input_delay`, `sync_to_monitor`, each captured via `+runpy` and cited to `kitty/options/definition.py`.
- [x] How they affect the first window — `TERM`, resolved face, cell size/columns×lines, history size, render cadence.
- [x] OUTPUT proving applied — the `debug_config` dump incl. the `OpenGL:` line (`kitty/debug_config.py:L258`); the always-printed `Config options different from defaults:` heading (`kitty/debug_config.py:L75`, called unconditionally at `kitty/debug_config.py:L275`) lists only the deliberate `allow_remote_control yes` override, and the **absence of a `Loaded config files:` section** (printed only `if opts.config_paths:` at `kitty/debug_config.py:L269-L271`) proves no `kitty.conf` was loaded — untouched defaults, corroborated by the `+runpy` capture.
- [x] `--debug-config` not-a-flag reality — evidence `Unknown option: --debug-config`.

**Q3 — PTY / child env / handshake / first output parsed + drawn:**

- [x] PTY `os.openpty()` — `kitty/child.py:L171`.
- [x] Child env `TERM` / `COLORTERM` / `TERMINFO` / `KITTY_*` — captured via `kitten @ ls`, cited to `kitty/child.py:L242-L261`.
- [x] Handshake + correctness-critical ordering (`resize_pty` before `mark_terminal_ready`) — `kitty/window.py:L863`/`L866`, `kitty/child.c:L71-L152`.
- [x] First output parsed (`kitty/vt-parser.c:L236`) + drawn (`kitty/screen.c:L866`) — proven by `get-text` returning `HELLO_FROM_SHELL_12345` and matching `cols=71 lines=22`.

**Q4 — FONTS / LAYOUT / SCROLLING / SCREEN UPDATES + confirming messages:**

- [x] FONTS — `Text fonts:` block; FontConfig `kitty/fontconfig.c:L276`, FreeType `kitty/freetype.c:L675`/`L904`, atlas `kitty/glyph-cache.c:L12-L16` (`SpritePosItem`) / `kitty/glyph-cache.c:L33-L54` (`find_or_create_sprite_position`).
- [x] LAYOUT — `columns=71 lines=22` grid; `kitty/window.py:L857-L859`.
- [x] SCROLLING — `scrollback_lines= 2000` (`kitty/options/definition.py:L372`); history buffer allocated at `kitty/screen.c:L130` (`self->historybuf = alloc_historybuf(...)`), `scrolled_by` at `kitty/child-monitor.c:L673`.
- [x] SCREEN UPDATES — `SIGWINCH sent to child ...` lines (`kitty/window.py:L873`); render cycle `send_cell_data_to_gpu` (`kitty/child-monitor.c:L714`/`L766`).
- [x] Confirming log/console messages — every item above is paired with a captured line or an explicit source-grounded note.

### Reproducibility & environment notes

- **Exact commit:** `815df1e21` (*"Wire up applying of font config"*), branch `kitty_815df1e210e0`; Go build embedded `VCSRevision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.
- **Kitty version:** `kitty 0.35.2 created by Kovid Goyal`.
- **Graphics:** headless via Xvfb `:99` with Mesa **llvmpipe** advertising OpenGL **4.5**, which satisfies Kitty's Linux minimum of **3.1** (`kitty/data-types.h:L24`).
- **Build prerequisite reality:** the first `python3 setup.py build --verbose` failed with `The package libcrypto was not found on your system` and succeeded after installing `libssl-dev` (which provides `libcrypto.pc`); `setup.py:L253`/`L275` probe `libcrypto` via `pkg-config`. This is an additional prerequisite beyond the apt list at `.github/workflows/ci.py:L84-L91`.
- **Read-only investigation:** this document (`blitzy/documentation/kitty_815df1e210e0.md`) is the only file added to the repository. All observation scaffolding — scratch scripts and captured logs — lived under `/tmp` and were removed; no existing repository file was modified.

