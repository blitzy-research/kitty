# kitty Startup, Configuration, Terminal-Shell Setup and Display: Runtime Investigation

> **Subject:** `kitty` terminal emulator (kovidgoyal/kitty).
> **Version:** `kitty 0.35.2`, grounded at `kitty/constants.py:25` (`version: Version = Version(0, 35, 2)`) and confirmed at runtime by `./kitty/launcher/kitty --version` -> `kitty 0.35.2 created by Kovid Goyal`.
> **Commit:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (short `815df1e21`); source branch `kitty_815df1e210e0`.
> **Environment (mandated):** every build and every runtime observation below was produced **inside the mandated container** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0` (Ubuntu 24.04.2 LTS, Python 3.12.3, Go 1.23.4, gcc 13.3.0). The container ships kitty prebuilt at the exact commit and was also rebuilt from source (Section 1.3). Because the container has no X server, kitty was run against an **authenticated Xvfb** started on the host and reached through the shared X11 socket; the software OpenGL context is the container's own Mesa (Section 1.6).
> **Platform scope:** **Linux / X11 (Xvfb) headless path ONLY.** macOS (CoreText/Cocoa) and the native Wayland runtime path were **not exercised**; wherever they are mentioned they are labelled `(inferred)` / *not exercised*.
> **Method:** *Run-first.* kitty was launched through its **real canonical entry point** (`kitty/launcher/kitty` -> embedded CPython -> `kitty.main.main()`). No debug hooks, mocks, or synthetic bypasses were used. Every behavioural claim is presented as **command -> complete unedited output/status -> source authority (symbol + `file:line`) -> interpretation**. Anything deduced only from reading (not observed at runtime) is marked **`(inferred)`**.

---

## How to read this document

- **Direct answer first.** Each of the four questions leads with a concise answer, then decomposes every named sub-item, pairing each behavioural claim with the captured evidence.
- **Test format.** Behavioural evidence is shown as an explicit `$ command`, then the **complete** captured output (with exit status where relevant), then the source symbol that produces it, then the interpretation. Output blocks are reproduced verbatim; where a value is intentionally shortened it is called out as *excerpt* (never as "complete").
- **Grounding.** `[path:line]` after a claim points at the exact source location, and the specific function/struct/variable that performs the work is named.
- **Observed vs inferred.** Output blocks are *observed*. A statement without an adjacent output block is grounded in source reading and, when it describes runtime behaviour that was not directly captured, is tagged `(inferred)`.
- **Identifiers.** Runtime paths are the container's (`/app/...`); the container hostname was set to the fixed, non-identifying value `kitty-mandated-container` for reproducibility (Section 1.5). No host or workspace identifiers are reproduced.

### Table of contents

1. [Methodology, Environment, Build and Safe Harness](#1-methodology-environment-build-and-safe-harness)
2. [Q1: Startup Subsystems](#2-q1-startup-subsystems)
3. [Q2: Initial Configuration](#3-q2-initial-configuration)
4. [Q3: Terminal-Shell Communication](#4-q3-terminal-shell-communication)
5. [Q4: Display System Evidence](#5-q4-display-system-evidence)
6. [Edge and Error Conditions](#6-edge-and-error-conditions)
7. [Honesty and Compatibility Notes](#7-honesty-and-compatibility-notes)
8. [Cleanup and Final Repository Proof](#8-cleanup-and-final-repository-proof)
9. [Appendix A: Observed vs Inferred Ledger](#appendix-a-observed-vs-inferred-ledger)
10. [Appendix B: Citation Index](#appendix-b-citation-index)
11. [Appendix C: Complete Build Log](#appendix-c-complete-build-log)

---

## 1. Methodology, Environment, Build and Safe Harness

All evidence comes from a **default, canonical build** exercised through kitty's real entry point inside the mandated container. The only non-default inputs are command-line `-o` overrides used *purely as a capture harness* (documented at each use and visible in the Q2 dump).

### 1.1 Mandated environment and toolchain (observed)

**Command and complete output** (run inside the mandated container):

```text
$ cat /etc/os-release | grep -E "^PRETTY_NAME|^VERSION_ID"
PRETTY_NAME="Ubuntu 24.04.2 LTS"
VERSION_ID="24.04"
$ python3 --version
Python 3.12.3
$ go version
go version go1.23.4 linux/amd64
$ gcc --version | head -1
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
$ git -C /app rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
$ git -C /app rev-parse --short HEAD
815df1e21
```

**Interpretation.** The runtime matches the mandated image exactly. Python `3.12.3` satisfies the project floor `requires-python = ">=3.8"` `[pyproject.toml:2]`; Go `1.23.4` satisfies the module floor `go 1.22` `[go.mod:3]`; the checkout is the exact commit `815df1e210e0`.

### 1.2 Native dependencies (observed, with exact producing command)

**Command and complete output:**

```text
$ for p in harfbuzz fontconfig freetype2 libpng lcms2 libcrypto libxxhash gl; do printf "%-12s %s\n" "$p" "$(pkg-config --modversion $p 2>/dev/null || echo N/A)"; done
harfbuzz     8.3.0
fontconfig   2.15.0
freetype2    26.1.20
libpng       1.6.43
lcms2        2.14
libcrypto    3.0.13
libxxhash    0.8.2
gl           1.2
```

| Dependency | Required floor | Observed (container) | Role |
|---|---|---|---|
| HarfBuzz | `>=1.5` `[setup.py:609]` | 8.3.0 | complex-text shaping |
| fontconfig | `[setup.py:634]` | 2.15.0 | Linux font discovery |
| freetype2 | (freetype dep) | 26.1.20 (see note) | glyph rasterisation |
| libpng / lcms2 | `[setup.py:610-611]` | 1.6.43 / 2.14 | PNG decode / colour management |
| libcrypto / libxxhash | (pkg-config deps) | 3.0.13 / 0.8.2 | crypto / hashing |
| gl | `[setup.py:639]` | 1.2 (`gl.pc`) | GL headers link contract |

> **FreeType version terminology (precise).** `26.1.20` is the **`freetype2.pc` pkg-config / libtool-ABI version string**, not the upstream FreeType release number (e.g. the "2.13.x" marketing version). It is reported here exactly as `pkg-config --modversion freetype2` prints it; do not read it as an upstream release. The `gl` value `1.2` is likewise the `gl.pc` link-contract version, *not* the runtime OpenGL context version (that is 4.5 Mesa, Section 1.6).

### 1.3 Canonical build (run-first) in the mandated container

The canonical build entry point is `python3 setup.py` -- the `Makefile all:` target `[Makefile:12]` whose recipe is `python3 setup.py $(VVAL)` `[Makefile:13]`.

**Command and status** (a from-scratch build; the prebuilt artifacts were removed first). The
block below is an **excerpt** -- it shows the command, the first five compile steps, all five
link steps, and the final exit status, with the repetitive interior compile lines and the Go
package-build lines elided **as explicitly marked**. It is *not* labelled complete; the
**complete 332-line log is reproduced verbatim in [Appendix C](#appendix-c-complete-build-log)**.

```text
$ cd /app && rm -rf build kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so
$ python3 setup.py build ; echo "BUILD_EXIT=$?"
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
[4/122] Compiling [x11] glfw/x11_window.c ...
[5/122] Compiling kitty/glfw.c ...
      ... [6/122] .. [121/122] elided (122 C translation units total; see Appendix C) ...
[122/122] Compiling kitty/gl-wrapper.c ...
 done
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
      ... Go package compilation lines elided (see Appendix C) ...
BUILD_EXIT=0
```

**Key result (honesty).** In the **mandated Ubuntu 24.04 container the pure `python3 setup.py build` succeeds with exit code 0 and requires no deviation flag.** All 122 C translation units compile, including the vendored-GLFW Wayland unit `glfw/wl_window.c` (unit `[3/122]`), and a `grep -ic 'werror\|warnings being treated as errors'` over the full log returns `0`. `setup.py` compiles with `werror = '' if ignore_compiler_warnings else '-pedantic-errors -Werror'` `[setup.py:491; also :1231]`, so exit 0 here means the `-Werror` gate was satisfied without `--ignore-compiler-warnings`.

> **Disclosed environment difference (inferred for the host case).** On a *newer* host (Ubuntu 25.10) the pure build instead fails at `glfw/wl_window.c:668` because that release's `wayland-protocols` adds `XDG_TOPLEVEL_STATE_CONSTRAINED_{LEFT,RIGHT,TOP,BOTTOM}` enum values the bundled GLFW `switch (*state)` does not handle, tripping `-Werror=switch`; there the official `--ignore-compiler-warnings` flag `[setup.py:2003-2004]` is required. That failure is **not** reproduced in the mandated image and is noted only to explain why prior host-based capture differed. It is a build-strictness artifact of the Wayland backend, never entered on the observed X11 path, and does not affect runtime behaviour.

**Artifact inventory (command and complete output):**

```text
$ ls -l kitty/fast_data_types.so kitty/launcher/kitty kitty/launcher/kitten
-rwxr-xr-x 1 root 1001  1213072 kitty/fast_data_types.so
-rwxr-xr-x 1 root 1001 15945988 kitty/launcher/kitten
-rwxr-xr-x 1 root 1001    36224 kitty/launcher/kitty
$ sha256sum kitty/fast_data_types.so kitty/launcher/kitty kitty/launcher/kitten
582933cfd7b6cecb5ee60cfd20ef35a1f74acc2c6a905022180a60a76bf722e8  kitty/fast_data_types.so
8311daddf6bbccf949233c9fdd58fbbe46748dbfc957847b7e4228b4973fc24c  kitty/launcher/kitty
f4da89b44f075e53e8a5b51cc7efcefa43c0c4125fe169c1537925dc7d89141d  kitty/launcher/kitten
```

The build produces the C extension imported by the Python layer (`kitty/fast_data_types.so`), the native launcher / real entry point (`kitty/launcher/kitty`), and the Go `kitten` binary (`kitty/launcher/kitten`).

### 1.4 Version banner (observed)

```text
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

This grounds to `version: Version = Version(0, 35, 2)` `[kitty/constants.py:25]`.

### 1.5 Safe headless harness (disclosed)

Because the container has no display, kitty is run against an **Xvfb started on the host** and reached over the shared `/tmp/.X11-unix` socket. The harness is deliberately hardened (it replaces an earlier unsafe pattern):

- **Authenticated display, not `-ac`.** Xvfb is started **with** `-auth <cookie-file>` and **without** `-ac`, so X access control stays on. A per-run `MIT-MAGIC-COOKIE-1` is generated with `xauth`; a `FamilyWild` (`ffff`) copy of the entry is merged so the container (which uses the fixed hostname `kitty-mandated-container`) can authenticate without embedding any host hostname.
- **Dynamic, unique display.** The display number is chosen at run time from the first free `/tmp/.X11-unix/X<n>` slot (not a fixed `:99`), avoiding collisions on a shared host.
- **Private, non-predictable paths.** All transient files (Xauthority, logs, and the remote-control socket) live under a single `mktemp -d` directory created mode `0700`; nothing uses a predictable path such as `/tmp/ksock` or `/tmp/kitty_startup.log`.
- **Restricted remote control.** Where remote control is used as a read-back harness it listens on a socket **inside** that private `0700` directory, and kitty is started with `allow_remote_control=socket-only` so control is possible only via that socket, not from programs running inside the terminal.
- **Deterministic process lifecycle.** Scripts run with `set -o pipefail`. Every background process PID is captured (`$!`); a single `trap ... EXIT INT TERM` kills the exact Xvfb PID, `wait`s for it, force-removes any container it started, and deletes the private directory. Readiness is polled with bounded loops (`xdpyinfo` for the display) rather than fixed sleeps.

The canonical launch used throughout (a real shell child) is:

```text
$ docker run --rm --net=host --hostname kitty-mandated-container \
    -e DISPLAY=":<n>" -e XAUTHORITY=/xauth/cookie -e LIBGL_ALWAYS_SOFTWARE=1 \
    -v /tmp/.X11-unix:/tmp/.X11-unix -v "$XAUTHORITY":/xauth/cookie:ro \
    --entrypoint bash <mandated-image> -lc \
    'cd /app && ./kitty/launcher/kitty --debug-rendering -o confirm_os_window_close=0 \
       sh -c "echo HELLO_FROM_SHELL_\$\$; sleep 4"'
```

### 1.6 OpenGL context actually obtained (observed)

kitty requires an OpenGL **>= 3.3** context or `gl_init()` aborts with `fatal("OpenGL version is %d.%d, version >= %d.%d required for kitty")` `[kitty/gl.c:74]`. The container's Mesa supplies a software one. Two independent commands confirm it:

```text
$ glxinfo -B | grep -E "OpenGL (vendor|renderer|core profile version) string"
OpenGL vendor string: Mesa
OpenGL renderer string: llvmpipe (LLVM 19.1.1, 256 bits)
OpenGL core profile version string: 4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1
```

```text
$ ./kitty/launcher/kitty --debug-rendering -o confirm_os_window_close=0 sh -c "true" 2>&1 | grep "GL version string"
[0.185] GL version string: '4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1' Detected version: 4.5
```

**Source authority.** kitty's line is emitted by `if (global_state.debug_rendering) printf("[%.3f] GL version string: %s\n", ...)` `[kitty/gl.c:72]`, whose string is built by `gl_version_string()` `[kitty/gl.c:42-49]` wrapping `glGetString(GL_VERSION)`.

**Interpretation.** The context is Mesa's **`llvmpipe` software renderer**, `4.5 (Core Profile) Mesa 24.2.8` (forced software by `LIBGL_ALWAYS_SOFTWARE=1`), comfortably above kitty's 3.3 floor. This is the container's own Mesa 24.2.8 (the host's Mesa is a different version and is not what kitty links against here).

---

## 2. Q1: Startup Subsystems

**Direct answer.** From process launch to a terminal ready to talk to a shell, kitty brings subsystems online in this **observed** order (true order established by the monotonic timestamps, see the stream note below): **(1)** the native C launcher, which embeds CPython; **(2)** the Python bootstrap / mode dispatcher, which routes the default invocation into the GUI `main()`; **(3)** `kitty.main.main()`, which initialises **GLFW**, prepares the startup **sessions**, and then **creates the OS window** -- and *during* that window creation acquires the **OpenGL** context (the GL line is emitted first, at the earliest timestamp); **(4)** the `OS Window created` completion marker, printed at the *end* of window creation; **(5)** the **Boss** controller, constructed *after* the window already exists, whose `start()` begins monitoring and launches the first child into the existing window/session; **(6)** the **child monitor** I/O thread (always) plus a *conditional* remote-control ("talk") thread; **(7)** the fork of the shell child, followed by an *optional, post-fork* **systemd** attempt to move the child into its own scope (which fails gracefully here); **(8)** the `Child launched` line, printed only once the PTY geometry is set and the child is released (see Q3). Four of these emit a log line under `--debug-rendering`.

### 2.1 Streams and ordering (Finding-critical): buffering, not threads

The `--debug-rendering` lines are written to **two different streams**, and that -- not thread scheduling -- is why a naive merged capture looks out of order.

**Command and complete output (merged `2>&1`, i.e. what you literally see piped):**

```text
$ ./kitty/launcher/kitty --debug-rendering -o confirm_os_window_close=0 sh -c 'echo HELLO_FROM_SHELL_$$; sleep 3' 2>&1
[0.278] OS Window created
[0.292] Failed to open systemd user bus with error: No medium found
[0.296] Child launched
[0.203] GL version string: '4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1' Detected version: 4.5
```

**Command and complete output (the SAME run with streams separated):**

```text
$ ./kitty/launcher/kitty --debug-rendering ... 1>/tmp/k.out 2>/tmp/k.err
$ cat /tmp/k.out      # STDOUT
[0.183] GL version string: '4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1' Detected version: 4.5
$ cat /tmp/k.err      # STDERR
[0.251] OS Window created
[0.263] Failed to open systemd user bus with error: No medium found
[0.266] Child launched
```

**Source authority.** The GL line uses `printf(...)` -> **stdout** `[kitty/gl.c:72]`. The other three use stderr: `OS Window created` via the `debug` macro (`#define debug debug_rendering` `[kitty/glfw.c:34]`) which routes to `fprintf(stderr, ...)` `[kitty/logging.c:56,61]`; the systemd line via `log_error(...)` `[kitty/systemd.c:87]` (same stderr path); and `Child launched` via `print(..., file=sys.stderr)` `[kitty/window.py:871]`.

**Interpretation.** By **timestamp** (monotonic seconds since start, `monotonic_t_to_s_double(monotonic())` `[kitty/gl.c:72]`) the true event order is **GL context `0.183` -> OS Window created `0.251` -> systemd attempt `0.263` -> Child launched `0.266`**. In the merged pipe the GL line appears *last* only because **stdout is block-buffered when piped** (flushed at process exit) whereas **stderr is unbuffered**; the ordering is a buffering artifact, not evidence of thread ordering. (`--debug-gl` is a pure alias of `--debug-rendering`; both names are defined together with `type=bool-set` `[kitty/cli.py:989]`.)

### 2.2 Subsystem-by-subsystem, with the exact emitter and true position

**(1) Native C launcher -- process entry.** `int main(int argc, char *argv[], char* envp[])` `[kitty/launcher/main.c:439]` is the real OS entry. It embeds CPython via `Py_InitializeFromConfig(&config)` `[kitty/launcher/main.c:211]` and hands control to the interpreter with `return Py_RunMain()` `[kitty/launcher/main.c:216]`. It prints nothing under `--debug-rendering`; its execution is proven *transitively* -- the Python-emitted lines above only exist because `Py_RunMain()` reached the Python layer. *(That `main.c:439` is the OS entry is `(inferred)` from reading; that execution reached Python is observed.)*

**(2) Python bootstrap / mode dispatch.** `entry_points.main()` `[kitty/entry_points.py:183]` is the first Python function; it looks `sys.argv[1]` up in the `entry_points` table `[kitty/entry_points.py:151]` and, for the default GUI invocation, calls `from kitty.main import main as kitty_main; kitty_main()`. *(The exact branch is `(inferred)`; it must have been taken, because the GUI window and GL context below exist only on that path.)*

**(3) GUI init and OS-window creation.** `kitty.main.main()` `[kitty/main.py:524]` initialises GLFW through `init_glfw_module()` `[kitty/main.py:90]` -> `glfw_init(...)` `[kitty/main.py:91]` with the backend chosen by `init_glfw()` `[kitty/main.py:95]` (here **x11**), then runs `_run_app()` `[kitty/main.py:202]`. Inside `_run_app`, in source order: the startup **sessions are prepared** by `create_sessions(...)` `[kitty/main.py:214]`; then the **OS window is created** by `create_os_window(...)` `[kitty/main.py:220-224]` (passing the `load_all_shaders` callback, so shader programs are built here).

**(4) OpenGL context (emitted first) and the `OS Window created` completion marker.** Window creation runs `if (is_first_window) gl_init();` `[kitty/glfw.c:1212]`, and `gl_init()` is what prints the `GL version string` line `[kitty/gl.c:72]` -- hence its earliest timestamp. The literal string `OS Window created` is emitted by `debug("OS Window created\n")` `[kitty/glfw.c:1321]` at the **very end** of the same window-creation function, *after* `gl_init`. So the GL line genuinely precedes the `OS Window created` marker in both timestamp and source causality; the marker is a *completion* signal for window setup, not the first thing that happens.

**(5) Boss controller -- constructed after the window exists.** Only after the window is created does `_run_app` construct `boss = Boss(opts, args, cached_values, global_shortcuts, talk_fd)` `[kitty/main.py:225]` (`class Boss` `[kitty/boss.py:323]`) and call `boss.start(window_id, startup_sessions)` `[kitty/main.py:226]`. Boss therefore does **not** create the initial OS window or the sessions (both already exist); `Boss.start()` `[kitty/boss.py:1181]` **starts monitoring and launches the first child into the existing window/session**: it calls `self.child_monitor.start()` `[kitty/boss.py:1183]` and then `self.startup_first_child(first_os_window_id, startup_sessions=startup_sessions)` `[kitty/boss.py:1194]`. *(Boss emits no `--debug-rendering` line; its effect -- a forked child whose bytes are drawn -- is observed via `Child launched` and the rendered screen in Q3/Q4.)*

**(6) Child monitor -- I/O thread always, talk thread conditional.** The monitor does **not** fork the shell. Its `start()` unconditionally creates the I/O thread and creates the remote-control "talk" thread **only if** a talk/listen fd exists: `if (self->talk_fd > -1 || self->listen_fd > -1) { pthread_create(&self->talk_thread, ...); } ... pthread_create(&self->io_thread, ...)` `[kitty/child-monitor.c:281-289]`. So the thread set is: the main/render thread, the always-present I/O-poll thread, and a *conditional* talk thread. The parse function pointer is selected at construction (`self->parse_func = parse_worker` or `parse_worker_dump`) `[kitty/child-monitor.c:178-181]`, and `parse_worker` itself is defined in `kitty/vt-parser.c:1496` (not in `child-monitor.c`).

**(7) Fork of the shell child + optional post-fork systemd scope.** The fork happens in `Child.fork()` `[kitty/child.py:276]` (via `fast_data_types.spawn(...)`), reached from `Tab.launch_child()` under `startup_first_child`. **After** the fork returns, and only on Linux, kitty attempts to move the *child* PID into its own systemd scope: `fast_data_types.systemd_move_pid_into_new_scope(pid, f'kitty-{ppid}-{self.id}.scope', ...)` `[kitty/child.py:346-353]`. So systemd is a **post-fork, child-scope** operation, not an early startup step (the old placement before Boss/fork was wrong).

**(8) systemd graceful degradation -- the real mechanism (Finding-critical).** In this container there is no user bus, so the C helper logs `log_error("Failed to open systemd user bus with error: %s", strerror(-ret)); return;` `[kitty/systemd.c:87]` and leaves `systemd.ok == false`. The subsequent call then raises a Python exception: `PyErr_SetString(PyExc_NotImplementedError, "Could not connect to systemd user bus")` `[kitty/systemd.c:185-188]`. Continuation happens because **`Child.fork()` catches it**: `except NotImplementedError: pass` `[kitty/child.py:349-351]` (a separate `except OSError` logs a different message). Observed: `[0.263] Failed to open systemd user bus with error: No medium found`, after which kitty proceeds normally (the next line is `Child launched`). *(Before: attempt; after: caught + continue.)*

**(9) `Child launched` -- terminal-ready release, not fork.** By the time this line prints, the fork has already occurred; the native child is **blocked** in `wait_for_terminal_ready(ready_read_fd)` `[kitty/child.c:150-153]` until kitty finishes screen/PTY setup. In the window layout path, once the PTY size is computed and pushed via `resize_pty(...)`, kitty releases the child with `self.child.mark_terminal_ready()` (which closes the ready pipe, `[kitty/child.py:362-364]`) and only then prints `[t] Child launched` `[kitty/window.py:861-871]`. So this line is **child-release / terminal-ready** evidence, not fork-time evidence.

### 2.3 Observed startup ordering (assembled from timestamps + source)

```text
launcher main() [main.c:439]
  -> Py_InitializeFromConfig [main.c:211] -> Py_RunMain [main.c:216]
    -> entry_points.main() [entry_points.py:183] -> kitty.main.main() [main.py:524]
      -> init_glfw / glfw_init [main.py:90-95]
      -> _run_app [main.py:202]
        -> create_sessions (sessions prepared) [main.py:214]
        -> create_os_window [main.py:220-224]
             -> gl_init() [glfw.c:1212] -> GL version string (STDOUT, t=0.183) [gl.c:72]
             -> OS Window created marker (STDERR, t=0.251)               [glfw.c:1321]
        -> Boss(...) constructed in existing window [main.py:225]
        -> boss.start() [boss.py:1181]
             -> child_monitor.start(): io thread always, talk thread conditional [child-monitor.c:281-289]
             -> startup_first_child -> Child.fork()/spawn [child.py:276]
                  -> post-fork systemd scope attempt (STDERR, t=0.263; NotImplementedError caught) [child.py:346-353; systemd.c:87,185-188]
             -> PTY geometry set -> mark_terminal_ready -> "Child launched" (STDERR, t=0.266) [window.py:861-871; child.py:362-364]
        -> boss.child_monitor.main_loop() [main.py:232] -> reads child bytes (Q3)
```

---

## 3. Q2: Initial Configuration

**Direct answer.** At first launch kitty decides its configuration by resolving an
**ordered set of sources** and merging them, then materializing the result into a
native `Options` object that the C rendering layer reads. From lowest to highest
precedence the sources are:

1. **Built-in defaults** — every option's default value is baked into the generated
   options schema `kitty/options/definition.py` via `opt(...)` declarations (e.g.
   `opt('term', 'xterm-kitty', ...)` [kitty/options/definition.py:3242]), materialized
   as the dataclass `Options` in `kitty/options/types.py` (e.g. `term: str =
   'xterm-kitty'` [kitty/options/types.py:602]). This is the sole source on a truly
   first launch when no config file exists.
2. **System config** — `SYSTEM_CONF = /etc/xdg/kitty/kitty.conf` [kitty/cli.py:1064],
   always consulted first among files (absent in this environment).
3. **User config** — `defconf = <config_dir>/kitty.conf` [kitty/constants.py:133],
   i.e. `$KITTY_CONFIG_DIRECTORY` or `~/.config/kitty/kitty.conf`
   [kitty/constants.py:87-133]. A `--config/-c FILE` **replaces** `defconf` (the
   system file is still prepended); `-c NONE` **suppresses all files**
   [kitty/conf/utils.py:322-329; kitty/cli.py:1067-1068].
4. **Command-line overrides** — every `-o/--override NAME=VALUE`
   [kitty/cli.py:876] is applied **last** and therefore wins over any file
   [kitty/config.py:163 `load_config`].

The runtime proof that a given set of settings was applied is the built-in
`debug_config` action [kitty/debug_config.py:231], which dumps the version, OS,
OpenGL string, resolved fonts, paths, the loaded files/overrides, and — crucially —
the list of *options that differ from defaults*. On a pure-default launch that list
is **empty**, which is exactly what first-launch configuration should produce.

> Note on `--debug-config`: it is **not** a CLI flag at this commit. The
> configuration dump is reached only through the in-session `debug_config`
> keybinding/action. The invalid-flag case is shown in §3.6.

### 3.1 Config directory and default file path (observed)

**Command** (run inside the mandated container; `+runpy` executes a snippet through
kitty's real Python entry point):

```text
$ ./kitty/launcher/kitty +runpy "from kitty.constants import config_dir, defconf; import os; print(\"config_dir:\", config_dir); print(\"defconf:\", defconf); print(\"defconf exists:\", os.path.exists(defconf))"
config_dir: /root/.config/kitty
defconf: /root/.config/kitty/kitty.conf
defconf exists: False
```

**Source authority.** `config_dir` is computed by `_get_config_dir()`
[kitty/constants.py:87] which honours `$KITTY_CONFIG_DIRECTORY`
[kitty/constants.py:88-89] and otherwise falls back to the XDG config home; the
default file path is `defconf = os.path.join(config_dir, 'kitty.conf')`
[kitty/constants.py:133].

**Interpretation (observed).** The three labels `config_dir:`, `defconf:` and
`defconf exists:` are emitted **by this command** (they are the literal `print(...)`
prefixes in the snippet), so the block is genuine program output — not a
hand-written table. `defconf exists: False` confirms that in a pristine container
**no user `kitty.conf` is present**, so this launch is driven purely by built-in
defaults (+ the absent system file). This is the corrected, non-fabricated evidence
for the config-directory claim.

### 3.2 Source ordering / precedence (observed)

**Command.** The resolver that decides *which files are consulted, and in what
order* is `kitty.cli.default_config_paths()` (a thin wrapper over
`resolve_config`). Exercising all three branches directly:

```text
$ ./kitty/launcher/kitty +runpy "
import kitty.cli as cli
from kitty.constants import defconf
print('SYSTEM_CONF          :', cli.SYSTEM_CONF)
print('defconf              :', defconf)
print('no -c (default)      :', list(cli.default_config_paths(())))
print('-c a.conf b.conf     :', list(cli.default_config_paths(('a.conf','b.conf'))))
print('-c NONE              :', list(cli.default_config_paths(('NONE',))))
"
SYSTEM_CONF          : /etc/xdg/kitty/kitty.conf
defconf              : /root/.config/kitty/kitty.conf
no -c (default)      : ['/etc/xdg/kitty/kitty.conf', '/root/.config/kitty/kitty.conf']
-c a.conf b.conf     : ['/etc/xdg/kitty/kitty.conf', 'a.conf', 'b.conf']
-c NONE              : []
```

**Source authority.** `SYSTEM_CONF = '/etc/xdg/kitty/kitty.conf'`
[kitty/cli.py:1064]; `default_config_paths(conf_paths)` returns
`tuple(resolve_config(SYSTEM_CONF, defconf, conf_paths))` [kitty/cli.py:1067-1068];
`resolve_config` yields the system file, then either the `-c` files (when given and
`'NONE'` is not among them) or `defconf`, and yields **nothing** when any path is
`'NONE'` [kitty/conf/utils.py:322-329].

**Interpretation (observed).** The output demonstrates the precedence rule exactly:
the system file `/etc/xdg/kitty/kitty.conf` is **always prepended**; with no `-c`
the user `defconf` follows it; a `-c a.conf b.conf` **replaces** `defconf` with the
named files (system file still first); and `-c NONE` collapses the list to `[]`,
suppressing every file so only built-in defaults remain. This corrects the earlier
document, which described the ordering incorrectly.

### 3.3 First-launch defaults and how the dump is produced (observed)

`debug_config` is not a CLI flag; it is a **Boss action** bound to
`kitty_mod+f6`. `kitty_mod` defaults to `ctrl+shift` [kitty/options/definition.py:3474]
and the binding is `debug_config kitty_mod+f6 debug_config`
[kitty/options/definition.py:4256]. Pressing it calls `Boss.debug_config`
[kitty/boss.py:3060], which builds the report via `debug_config(get_options())`
[kitty/boss.py:3064; kitty/debug_config.py:231], strips the ANSI colour codes with
`re.sub(r'\x1b.+?m', '', output)` and copies the result to the clipboard via
`set_clipboard_string(...)` [kitty/boss.py:3065] (it also echoes the report on screen
through `display_scrollback` [kitty/boss.py:3067]). The harness therefore drives a
**real keypress** and reads the clipboard the running kitty populated (no debug hook,
no mock).

**Command** (pure default — **zero** `-c` and **zero** `-o`; the display number is
whatever free number the harness picked for this run):

```text
# host: focus the kitty window on the harness Xvfb, then deliver the real chord
$ xdotool windowfocus --sync "$WID"
$ xdotool keydown ctrl keydown shift; xdotool key F6; xdotool keyup shift keyup ctrl
# container log confirms the action fired:
#   KeyPress matched action: debug_config, handled as shortcut
$ xclip -selection clipboard -o        # read what kitty copied
```

**Complete captured dump (verbatim; the environment-variable lines are tab-indented
in the raw output):**

```text
kitty 0.35.2 (815df1e210) created by Kovid Goyal
Linux kitty-mandated-container 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64
Ubuntu 24.04.2 LTS kitty-mandated-container /dev/tty

DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=24.04
DISTRIB_CODENAME=noble
DISTRIB_DESCRIPTION="Ubuntu 24.04.2 LTS"
Running under: X11
OpenGL: '4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1' Detected version: 4.5
Frozen: False
Fonts:
  medium: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
  bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
  italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
  bi: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
Paths:
  kitty: /app/kitty/launcher/kitty
  base dir: /app
  extensions dir: /app/kitty
  system shell: /bin/bash

Config options different from defaults:

Important environment variables seen by the kitty process:
	PATH                                /app/kitty/launcher:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
	DISPLAY                             :71
	LC_CTYPE                            C.UTF-8
```

**Source authority.** The report layout (version line, uname, OS-release, `Running
under:`, `OpenGL:`, `Fonts:`, `Paths:`, the "Config options different from defaults"
diff, and the curated env-var list) is produced by `debug_config()`
[kitty/debug_config.py:231].

**Interpretation (observed).** The section **`Config options different from
defaults:` is empty** — there is no line between that header and the next section —
which is the definitive proof that this launch used **only built-in defaults**
(consistent with §3.1's `defconf exists: False`). The dump is **complete and
untruncated**: the `PATH` value runs to its end with no ellipsis, and the hostname
is the clean `kitty-mandated-container` with `/app`-based paths (no host machine
identifiers). The applied defaults are directly visible — e.g. the `DejaVuSansMono`
font family the container resolved, and the `xterm-kitty`-driving `system shell:
/bin/bash`.

### 3.4 Overrides and `-c` files change the dump (observed)

Two supplemental launches prove that higher-precedence sources both (a) appear in
their own dump section and (b) show up in the diff.

**(a) `-o` overrides** — launched with
`-o confirm_os_window_close=0 -o cursor_shape=beam -o scrollback_lines=5000`;
the launch used exactly those three overrides. Complete captured dump (verbatim;
the environment-variable lines are tab-indented in the raw output):

```text
kitty 0.35.2 (815df1e210) created by Kovid Goyal
Linux kitty-mandated-container 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64
Ubuntu 24.04.2 LTS kitty-mandated-container /dev/tty

DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=24.04
DISTRIB_CODENAME=noble
DISTRIB_DESCRIPTION="Ubuntu 24.04.2 LTS"
Running under: X11
OpenGL: '4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1' Detected version: 4.5
Frozen: False
Fonts:
  medium: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
  bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
  italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
  bi: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
Paths:
  kitty: /app/kitty/launcher/kitty
  base dir: /app
  extensions dir: /app/kitty
  system shell: /bin/bash
Loaded config overrides:
  confirm_os_window_close 0
  cursor_shape beam
  scrollback_lines 5000

Config options different from defaults:
confirm_os_window_close 0
cursor_shape            2
scrollback_lines        5000

Important environment variables seen by the kitty process:
	PATH                                /app/kitty/launcher:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
	DISPLAY                             :71
	LC_CTYPE                            C.UTF-8
```

**(b) `-c` config file** — a file was written in the container
(`confirm_os_window_close 0` / `scrollback_lines 12345` / `cursor_shape underline`)
and loaded with `-c /tmp/myconf.conf` (no `-o`). Complete captured dump (verbatim):

```text
kitty 0.35.2 (815df1e210) created by Kovid Goyal
Linux kitty-mandated-container 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64
Ubuntu 24.04.2 LTS kitty-mandated-container /dev/tty

DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=24.04
DISTRIB_CODENAME=noble
DISTRIB_DESCRIPTION="Ubuntu 24.04.2 LTS"
Running under: X11
OpenGL: '4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1' Detected version: 4.5
Frozen: False
Fonts:
  medium: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
  bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
  italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
  bi: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
Paths:
  kitty: /app/kitty/launcher/kitty
  base dir: /app
  extensions dir: /app/kitty
  system shell: /bin/bash
Loaded config files:
  /tmp/myconf.conf

Config options different from defaults:
confirm_os_window_close 0
cursor_shape            3
scrollback_lines        12345

Important environment variables seen by the kitty process:
	PATH                                /app/kitty/launcher:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
	DISPLAY                             :71
	LC_CTYPE                            C.UTF-8
```

**(c) `-c NONE`** — suppresses *all* config files; a single pair of `-o` options
proves overrides still apply even when every file is suppressed. Complete captured
dump (verbatim):

```text
kitty 0.35.2 (815df1e210) created by Kovid Goyal
Linux kitty-mandated-container 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64
Ubuntu 24.04.2 LTS kitty-mandated-container /dev/tty

DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=24.04
DISTRIB_CODENAME=noble
DISTRIB_DESCRIPTION="Ubuntu 24.04.2 LTS"
Running under: X11
OpenGL: '4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1' Detected version: 4.5
Frozen: False
Fonts:
  medium: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
  bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
  italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
  bi: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
Paths:
  kitty: /app/kitty/launcher/kitty
  base dir: /app
  extensions dir: /app/kitty
  system shell: /bin/bash
Loaded config overrides:
  confirm_os_window_close 0
  scrollback_lines 7777

Config options different from defaults:
confirm_os_window_close 0
scrollback_lines        7777

Important environment variables seen by the kitty process:
	PATH                                /app/kitty/launcher:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
	DISPLAY                             :71
	LC_CTYPE                            C.UTF-8
```

Note there is **no `Loaded config files:` section at all** — `NONE` removed both the
system file and the user file — yet the `Loaded config overrides:` section (and the
diff) still carry the two `-o` values, showing that suppression applies to files
only, not to command-line overrides.

**Source authority.** Overrides are captured into their own section and merged last
by `load_config` [kitty/config.py:163]; a `-c` file is recorded under `Loaded config
files:` because it is the file `resolve_config` yielded in place of `defconf`
[kitty/conf/utils.py:322-329]. The string→value conversion (`beam`, `underline`) is
performed by the per-option parsers in `kitty/options/parse.py` — e.g.
`Parser.cursor_shape()` [kitty/options/parse.py:919] calls `to_cursor_shape(val)`
[kitty/options/parse.py:920] to map `beam`→`2`/`underline`→`3`.

**Interpretation (observed).** In (a) the three `-o` values appear under **`Loaded
config overrides:`** *and* in the diff, proving overrides are applied last and win.
In (b) the **`Loaded config files:` lists `/tmp/myconf.conf`** and **not** `defconf`,
proving `-c` *replaces* the default user file exactly as §3.2 predicted. A subtle but
important detail visible across the two: the raw override text `cursor_shape beam`
materializes in the diff as `cursor_shape 2`, while the file's `cursor_shape
underline` materializes as `cursor_shape 3` — i.e. the parser converts the
human-readable name into the internal enum stored on the `Options` object.

### 3.5 Materialization into the native `Options` object (observed + inferred)

Once the sources are merged, the resulting Python `Options` is pushed into the C
layer so the renderer sees the same settings.

- The first-run scaffolding that would *write* a starter `kitty.conf` is
  `prepare_config_file_for_editing()` [kitty/config.py:78-86]; the helper that merely
  *generates* the fully-commented default text is
  `commented_out_default_config()` [kitty/config.py:74-76]. **(observed via source;
  neither ran here because no editing/first-run path was invoked — inferred that the
  absent `defconf` is why `defconf exists: False`.)**
- The native transfer occurs when the app runner applies options: `AppRunner.__call__`
  calls `set_options(opts, ...)` [kitty/main.py:247-252], which reaches the C function
  `PYWRAP1(set_options)` [kitty/state.c:726-744] and copies each field via the
  generated converter `kitty/options/to-c-generated.h`. **(observed via source; the effect is visible at
  runtime because the OpenGL/font settings reported by `debug_config` match what the
  renderer draws — inferred linkage between the transfer call and the drawn frame.)**

### 3.6 `--debug-config` is not a valid flag at this commit (observed edge case)

```text
$ ./kitty/launcher/kitty --debug-config; echo "EXIT=$?"
Unknown option: --debug-config
EXIT=1
```

**Interpretation (observed).** kitty rejects `--debug-config` with **exit status 1**
and the message `Unknown option: --debug-config`. This confirms the AAP's note and
justifies why the configuration dump in §3.3–§3.4 is obtained through the in-session
`debug_config` **action** rather than a command-line flag.

---

## 4. Q3: Terminal-Shell Communication

**Direct answer.** kitty makes the terminal ready to talk to the shell in a precise
ordered sequence, then routes the shell's bytes into the on-screen grid:

1. **Allocate a PTY** — `openpty()` [kitty/child.py:170-171] calls `os.openpty()`,
   returning a master/slave fd pair (in blocking mode).
2. **Fork the child** — `Child.fork()` [kitty/child.py:276] forks; the native side
   [kitty/child.c:97] runs `fork()`, and in the child branch calls `setsid()`
   [kitty/child.c:123] to start a new session and `ioctl(tfd, TIOCSCTTY, 0)`
   [kitty/child.c:129] to make the PTY slave its **controlling terminal**.
3. **Wait on a readiness pipe** — the child does **not** `execvp` immediately. It
   blocks in `wait_for_terminal_ready(ready_read_fd)` [kitty/child.c:150-153] until
   kitty has built the screen object and set the PTY geometry.
4. **Set geometry, then release** — when the window is first sized,
   `resize_pty(...)` is called and then `mark_terminal_ready()`
   [kitty/child.py:362-364] closes the ready fd, unblocking the child, which finally
   `execvp`s the shell [kitty/child.c:159]. Only now does kitty print `Child launched`
   [kitty/window.py:871].
5. **Advertise capabilities** — the child's environment carries `TERM=xterm-kitty`
   [kitty/child.py:242] and `TERMINFO` [kitty/child.py:255-260], pointing at the
   `xterm-kitty|KovIdTTY,` capability record [terminfo/kitty.terminfo:1].
6. **Parse and draw the shell's bytes** — the child-monitor I/O thread reads the
   shell's output and the VT parser walks it: ordinary text goes through
   `consume_normal()` [kitty/vt-parser.c:230] into `screen_draw_text()`
   [kitty/screen.c:866], which writes cells into the screen model.

The concrete behavior proving the shell's first output was understood correctly is a
**full round-trip**: a child process printed its environment, and that text was parsed
by the VT engine, written into the screen model, and read back verbatim with
`kitty @ get-text` — while the framebuffer visibly changed from blank to showing the
text (§4.6).

### 4.1 PTY allocation and controlling-terminal setup (observed via source)

```text
kitty/child.py:170  def openpty() -> Tuple[int, int]:
kitty/child.py:171      master, slave = os.openpty()  # Note that master and slave are in blocking mode
kitty/child.c:97       pid_t pid = fork();
kitty/child.c:123          if (setsid() == -1) exit_on_err("setsid() in child process failed");
kitty/child.c:129          if (ioctl(tfd, TIOCSCTTY, 0) == -1) exit_on_err("Failed to set controlling terminal with TIOCSCTTY");
kitty/child.c:159          execvp(exe, argv);
```

**Interpretation (observed via source; runtime effect shown in §4.3/§4.6).** kitty
allocates the PTY in Python (`os.openpty`), then the forked child makes the slave its
controlling terminal (`setsid` + `TIOCSCTTY`) so that job control and the terminal's
size/signals work — before it ever `execvp`s the shell.

### 4.2 The ready-pipe/geometry handshake — what "Child launched" really means (observed)

This is the central readiness sequence, and it corrects a common misreading: the
`Child launched` log line is **not** fork-time evidence — the fork already happened
in §4.1. The child is deliberately held back until kitty finishes screen setup:

```text
kitty/child.c:150      // Wait for READY_SIGNAL which indicates kitty has setup the screen object
kitty/child.c:151      safe_close(ready_write_fd, __FILE__, __LINE__);
kitty/child.c:152      wait_for_terminal_ready(ready_read_fd);
kitty/child.c:153      safe_close(ready_read_fd, __FILE__, __LINE__);

kitty/window.py:861        if current_pty_size != self.last_reported_pty_size:
kitty/window.py:862            boss = get_boss()
kitty/window.py:863            boss.child_monitor.resize_pty(self.id, *current_pty_size)   # (1) geometry
kitty/window.py:864            self.last_resized_at = monotonic()
kitty/window.py:865            if not self.child_is_launched:
kitty/window.py:866                self.child.mark_terminal_ready()                        # (2) release child
kitty/window.py:867                self.child_is_launched = True
kitty/window.py:868                update_ime_position = True
kitty/window.py:869                if boss.args.debug_rendering:
kitty/window.py:870                    now = monotonic()
kitty/window.py:871                    print(f'[{now:.3f}] Child launched', file=sys.stderr)   # (3) log

kitty/child.py:362     def mark_terminal_ready(self) -> None:
kitty/child.py:363         os.close(self.terminal_ready_fd)
kitty/child.py:364         self.terminal_ready_fd = -1
```

**Runtime evidence.** In the §2 startup capture the line appeared as
`[0.266] Child launched` on **stderr**, i.e. *after* the OS window and GL context and
*after* PTY geometry was computed — consistent with the code above.

**Interpretation (observed).** The order is unambiguous in the source: geometry first
(`resize_pty`, window.py:863), then the child is released (`mark_terminal_ready`,
window.py:866 → child.py:363 closes the pipe fd the child is blocked on), and only
then is `Child launched` printed (window.py:871). Therefore the line is
**terminal-ready child-release evidence**, marking the moment the shell is finally
allowed to `execvp` and run against a fully-sized screen — not the moment of `fork()`.

### 4.3 Environment contract: `TERM=xterm-kitty` is child-visible (observed)

A real child (`sh`) printed the environment kitty handed it; the text was read back
through the terminal model with `kitty @ get-text` (complete output; the RC public
key value is redacted — see note):

```text
### kitty @ get-text (model readback of the child's rendered stdout) ###
TERM=xterm-kitty
TERMINFO=/app/terminfo
KITTY_INSTALLATION_DIR=/app
KITTY_LISTEN_ON=unix:/tmp/krc.11755
KITTY_PID=1
KITTY_PUBLIC_KEY=<redacted: ephemeral per-session remote-control public key, regenerated every launch; value carries no cross-session meaning>
KITTY_WINDOW_ID=1
READY_MARKER_Q3ENV
```

**Source authority.** `env['TERM'] = opts.term` [kitty/child.py:242]; the default is
`opt('term', 'xterm-kitty', ...)` [kitty/options/definition.py:3242], materialized as
`term: str = 'xterm-kitty'` [kitty/options/types.py:602]; `TERMINFO` is set from the
resolved terminfo directory [kitty/child.py:255-260]; the capability record it points
to begins `xterm-kitty|KovIdTTY,` [terminfo/kitty.terminfo:1].

**Interpretation (observed).** The child genuinely sees `TERM=xterm-kitty` and
`TERMINFO=/app/terminfo` (plus kitty's `KITTY_*` variables). Because these values were
*printed by the child and read back through `get-text`*, this simultaneously proves
(a) the environment contract and (b) that the child's bytes traversed the PTY → VT
parser → screen model successfully (the byte path is detailed in §4.5).

### 4.4 Shell integration: `/bin/bash --posix` and the `kitty.bash` control flow (observed)

With no explicit command, kitty launches the default interactive shell with its
shell-integration wrapper. `kitty @ ls` reports the foreground process:

```text
### kitty @ ls -> the foreground process the terminal launched ###
foreground cmdline: ['/bin/bash', '--posix']
window cmdline      : ['/bin/bash', '--posix']
window env SHELL     : None

### grounding: kitty inserts --posix at shell_integration.py:146 ###
        env['HISTFILE'] = os.path.expanduser('~/.bash_history')
        env['KITTY_BASH_UNEXPORT_HISTFILE'] = '1'
    argv.insert(1, '--posix')
```

**Source authority — `kitty.bash` control flow (`shell-integration/bash/kitty.bash`,
391 lines in this build):**

```text
L3    if [[ "$-" != *i* ]] ; then builtin return; fi   # only run in interactive mode
L4    if [[ -z "$KITTY_SHELL_INTEGRATION" ]]; then builtin return; fi   # guard: no integration -> stop
L10       builtin unset KITTY_SHELL_INTEGRATION   # ensure manual re-sourcing has no effect
L77       builtin export KITTY_SHELL_INTEGRATION="$ksi_val"   # RE-EXPORT for child processes
L117      builtin unset KITTY_SHELL_INTEGRATION   # FINAL unset after setup completes
```

**Interpretation (observed + source).** kitty runs `/bin/bash --posix` — it inserts
`--posix` into the shell argv at `argv.insert(1, '--posix')`
[kitty/shell_integration.py:146] — and the wrapper `kitty.bash` guards itself to
interactive shells (L3), bails out when `KITTY_SHELL_INTEGRATION` is unset (L4),
unsets the variable early so manual sourcing is inert (L10), **re-exports** it while
configuring child-visible state (L77), and performs a **final unset** once setup is
complete (L117). The earlier document omitted the L77 re-export and the L117 final
unset; both are shown above from this build.

### 4.5 The child-byte read path: read → parse worker → `consume_normal` → screen (observed via source)

```text
kitty/vt-parser.c:1495  void
kitty/vt-parser.c:1496  parse_worker(void *p, ParseData *pd, bool flush) { run_worker(p, pd, flush); }
kitty/vt-parser.c:230   consume_normal(PS *self) {          // ordinary (non-control) text
kitty/vt-parser.c:231       do {
kitty/screen.c:866      screen_draw_text(Screen *self, const uint32_t *chars, size_t num_chars) {
kitty/vt-parser.c:224   dispatch_single_byte_control(PS *self, uint32_t ch) {   // C0/C1 controls only
kitty/vt-parser.c:457   dispatch_osc(PS *self, uint8_t *buf, size_t limit, bool is_extended_osc) {   // OSC only
```

**Interpretation (observed via source; runtime effect shown in §4.3/§4.6).** After the
I/O thread reads bytes from the PTY master, the parser dispatches them: **ordinary
printable text is handled by `consume_normal()` [vt-parser.c:230], which calls
`screen_draw_text()` [screen.c:866]** to write codepoints into cells. Control bytes go
through `dispatch_single_byte_control()` [vt-parser.c:224] and OSC sequences through
`dispatch_osc()` [vt-parser.c:457] — so normal text is **not** routed through the
control/OSC handlers. The parse loop itself runs from `parse_worker`
[vt-parser.c:1495-1496] (correcting an earlier miscitation to `child-monitor.c`). The
successful `get-text` readback in §4.3 is the runtime confirmation that this path
carried the child's text into the screen model.

### 4.6 Visible first output: synchronized before / during / after (observed)

A child was launched that delays its first output by 5 s, so a genuine *before* state
could be captured. The framebuffer was grabbed from the host (`import -window root`)
and the model was read with `get-text` at each state. Full-frame captures are
1280x800; the embedded images are the deterministic top-left crop
(`convert <frame>.png -crop 560x64+0+0`) for legibility.

```text
Child: sh -c 'sleep 5; printf "FIRST_OUTPUT_MARKER_Q3\n"; exec sleep 25'
T0 = launch (window appeared ~0.3 s later)

### BEFORE first output (t=+3.75s) ###
get-text model readback: (screen empty - no cells drawn yet)
full framebuffer 1280x800  sha256: f3bdcc59f1a08b8f599bc2d653ce1bdeaafc2bfdc23eaa6d15fe20b7bd6f5101

### AFTER first output (t=+6.20s) ###
get-text model readback: FIRST_OUTPUT_MARKER_Q3
full framebuffer 1280x800  sha256: c0c34d6f1b25f50034c7796329392b4a193153166cdd815d8f5dbaaec9623195
```

**Before** (top-left crop; entirely black — no glyphs drawn yet):

![Before first output: an entirely black 560x64 top-left crop of the 1280x800 kitty framebuffer, containing no glyphs, confirming the screen was empty before the child printed.](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAjAAAABAAQAAAAAsN3AqAAAAIGNIUk0AAHomAACAhAAA+gAAAIDoAAB1MAAA6mAAADqYAAAXcJy6UTwAAAACYktHRAAB3YoTpAAAAAd0SU1FB+oHDhUEJ/J/KJMAAAAbSURBVFjD7cExAQAAAMKg9U9tDQ+gAAAAAHg2EcAAAaIcQ1cAAAAldEVYdGRhdGU6Y3JlYXRlADIwMjYtMDctMTRUMjE6MDQ6MzkrMDA6MDBJwM+fAAAAJXRFWHRkYXRlOm1vZGlmeQAyMDI2LTA3LTE0VDIxOjAzOjU4KzAwOjAwulluaQAAACh0RVh0ZGF0ZTp0aW1lc3RhbXAAMjAyNi0wNy0xNFQyMTowNDozOSswMDowMG+IVvwAAAAASUVORK5CYII=)

**After** (top-left crop; the child's first line is drawn, with the block cursor on
the next row):

![After first output: a 560x64 top-left crop of the 1280x800 kitty framebuffer showing the text FIRST_OUTPUT_MARKER_Q3 in light-gray anti-aliased monospaced glyphs on a black background, with a filled block cursor on the row below.](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAjAAAABACAAAAAAhJxJbAAAAIGNIUk0AAHomAACAhAAA+gAAAIDoAAB1MAAA6mAAADqYAAAXcJy6UTwAAAACYktHRAD/h4/MvwAAAAd0SU1FB+oHDhUEJ/J/KJMAAARXSURBVHja7dd9TFV1HMfx9wWywIciN7VyabpkAaEpMzNGoaaW0dY07QFtmYXLVis3Vmqt1UAwc+kCxwQRszYCh4bACCTZQAqlhC31AleX2E3XgwY+IBn0Bx7uAQ64Y807dz+vP9i55/v7ns9vP767OxdERERERETEw0HSIoB/JpA7GWD32zT4036mfA2Ergob3ObOyCue0LW4aomnMTMy6K+yd8D5/RIoGRzlWZMfQef545tLLEpG88Ynmx9lZFnQhlTYE1KxDLpDR1eUxUNOROpn+REAv0Z59tM3PX3SrZ1/FCV5+xR9SACQfRo64POyybN2uJ3AD3uGRj/bmkLS+MLGkVNGkTmcSbO/PEmzpy/1kb11MQv+XNd9w7Tm3Ab/0MdXl1iVDJeHxxa82gEwdnxtGJhCAXLDNm2Gc2lAa69Sz/S/q9ydMS/9tsXbx+hLklzPGJcfuZ4HaNgGHM6Fw+ndqz5wvdijq7YYOPANOLcDJZWmNfk/AhmuaIuSYaMzL4OiHa4V8H79nKanTaGjXens/OkV4zk99tM3HWBs02Zvn6EP8bO+HZI86DK03RPeT9vU234G3KP6fW47QwfM/Xbi1DGVAJHHS36faw6F/JBPen9lGCWL9Mg1HPX2KfqQACA5GarjTDejmhy05EBOXJ67qaDAom0crcCFwH6eGrxw2pnCAXNTlyacaAFGj99FY5g5lOlBp7u+O4a5gIOLTKW+6UtXOS5t3+jtU/QhxjtMs/nmoeK7Z2XugpS8hfdNjA5Z37fNMdAzhzXhOJVxleD66GyAxYMKOfDQvEJPKJ3ZLyTHwZV3GLdpP73SO4CqtcHRC47t8PYx+o4A4Ehur5vnMvF/s7YOXGsJLIq1GBgXQ4Ggi13/NeOv0b7p8pEaLEseuZe2RgNT/baBY16hKbT6wxFzE9YBHem999OdPgQIagOcTtbXLNTAXDf9vMOQ2L6y6+Li2UEW5ZqzY4A7T0HbLcDNbeZiR2Z2DViWPIqWnwSC792fmJjYGN4z9PUTcTP6248p3X3lo+Mmb5+iDwnwXD51xwhC4p37ALi4f87MvRUNR1seCKu26vtuTnpdTHAuHLs/xTXlrmKrNQOUDMsCy7MhdP5jpUaoE2D9x++Wg1880J7l2Y8pPe3QjOAvGLm9/pfA6beXINeLaWAWT4Y4du/r+pQ2c/leZ3iUX2vlSqu+FVunzWjZuQ7eWz03sLX0Das1A5QMD17IAornx5YaoW8BFE1/Lu01hiQA57M8+/GkZz482/H1p7S0zxzccearNYhcRWRlzRPe3oMv8vf2Bq6Vu35c9Qlvb0JEBuawtdr4/d3y8jWFebdd/g+Og8ZVpLe3IjcCv//+CPElGhixRQMjtmhgxBYNjNiigRFbNDBiiwZGbNHAiC0aGLFFAyO2aGDEFg2M2KKBEVs0MGKLBkZs0cCIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIiIyA3qX/j/m/+pYXzvAAAAJXRFWHRkYXRlOmNyZWF0ZQAyMDI2LTA3LTE0VDIxOjA0OjM5KzAwOjAwScDPnwAAACV0RVh0ZGF0ZTptb2RpZnkAMjAyNi0wNy0xNFQyMTowNDowMCswMDowMCOKNRMAAAAodEVYdGRhdGU6dGltZXN0YW1wADIwMjYtMDctMTRUMjE6MDQ6MzkrMDA6MDBviFb8AAAAAElFTkSuQmCC)

**Interpretation (observed).** Across the transition the model readback goes from
empty to `FIRST_OUTPUT_MARKER_Q3`, and the two full-frame captures have **different
sha256 digests** (`f3bdcc59…` → `c0c34d6f…`), proving pixels were actually drawn — not
merely that a model field changed. The embedded crops make the change visible: a black
frame before, and the anti-aliased monospaced text plus block cursor after. This is
the concrete "the shell's first output was understood and drawn" evidence Question 3
asks for. (`get-text` is the model-level readback — a separate remote-control endpoint
`GetText` [kitty/rc/get_text.py:13] returning `window.as_text(...)` [kitty/rc/get_text.py:116];
the framebuffer captures are the independent pixel-level proof.)

---

## 5. Q4: Display System Evidence

**Direct answer.** As the first characters appear, four independent signals show the
display system is active and working:

- **Fonts** — `--debug-font-fallback` prints a `Text fonts:` block naming the exact
  font files chosen for Normal/Bold/Italic/Bold-Italic
  [kitty/fonts/render.py:163], and the rendered glyphs are visibly those faces.
- **Layout** — text lands on a fixed monospaced **cell grid**; the durable
  screenshot (§5.4) shows columns aligning exactly, which is the layout system at
  work.
- **Screen updates / GPU** — a real OpenGL 4.5 context is obtained and the frame is
  presented; the pixels in §5.4 (colours, bold, italic, underline, reverse-video
  backgrounds) cannot exist unless the shader pipeline compiled and ran.
- **Scrolling** — forcing 120 lines of output evicts earlier lines into scrollback,
  and an actual scroll moves the viewport (§5.5).

The rendering pipeline that produces those pixels is: **shaping** with HarfBuzz
(`hb_shape()` [kitty/fonts.c:813]) → **rasterization** with FreeType
(`render_bitmap()`/`FT_Render_Glyph` [kitty/freetype.c:507,904]) → a **CPU-side hash
cache** of glyph→sprite positions (`kitty/glyph-cache.c`) → a **GPU texture atlas**
(`alloc_sprite_map` [kitty/shaders.c:51], `glTexStorage3D` [kitty/shaders.c:123],
`glTexSubImage3D` [kitty/shaders.c:99,155]) → **cell shaders** (`cell_vertex.glsl` /
`cell_fragment.glsl`). The atlas upload is orchestrated from `kitty/fonts.c` via the
`current_send_sprite_to_gpu` macro [kitty/fonts.c:21] at call sites
[kitty/fonts.c:667,744,1455,1470], keyed by `sprite_position_for` [kitty/fonts.c:257].

### 5.1 Font selection: the `Text fonts:` block (observed)

`--debug-font-fallback` was launched and stdout/stderr captured separately to label
the stream correctly:

```text
===STDOUT (fd1)===
===STDERR (fd2) - Text fonts block lives here===
[0.264] Failed to open systemd user bus with error: No medium found
[0.268] Text fonts:
[0.268]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.268]   Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
[0.268]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.268]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

**Source authority.** `dump_font_debug()` emits the block: `log_error('Text fonts:')`
then one line per style via `identify_for_debug()` [kitty/fonts/render.py:163]. Family
resolution on Linux is done by fontconfig through `_fc_match()`
[kitty/fontconfig.c:269], which calls `FcFontMatch()` [kitty/fontconfig.c:276]
(enumeration via `fc_list()` [kitty/fontconfig.c:235]).

**Interpretation (observed).** The `Text fonts:` block is on **stderr** (fd2), not
stdout (fd1 was empty) — correcting an earlier stream mislabel. In this container the
four faces resolve to the DejaVu Sans Mono family (Normal/Bold/Oblique/BoldOblique).
Those are exactly the faces visible in the durable screenshot in §5.4 (the bold and
italic words use the Bold and Oblique files listed here).

### 5.2 The display pipeline — correct component ownership (observed via source + runtime frame)

The pipeline stages and their true owners (each `file:line` verified in this build):

```text
shaping (HarfBuzz)     kitty/fonts.c:813     hb_shape(font, harfbuzz_buffer, fobj->ffs_hb_features, num_features);
rasterization (FreeType) kitty/freetype.c:507  render_bitmap(Face*, glyph_id, ...)
                        kitty/freetype.c:904  error = FT_Render_Glyph(self->face->glyph, FT_RENDER_MODE_NORMAL);
glyph->sprite HASH cache kitty/glyph-cache.c:34 find_or_create_sprite_position(SpritePosition**head, ...)
                        kitty/glyph-cache.c:57 free_sprite_position_hash_table(SpritePosition**head)
GPU texture atlas       kitty/shaders.c:51    alloc_sprite_map(cell_width, cell_height)
                        kitty/shaders.c:123   glTexStorage3D(GL_TEXTURE_2D_ARRAY, 1, GL_SRGB8_ALPHA8, ...)
                        kitty/shaders.c:99    glTexSubImage3D(GL_TEXTURE_2D_ARRAY, 0, 0,0,0, ...)
                        kitty/shaders.c:155   glTexSubImage3D(GL_TEXTURE_2D_ARRAY, 0, x,y,z, ...)
atlas upload orchestration kitty/fonts.c:21   #define current_send_sprite_to_gpu(...) (... send_sprite_to_gpu(...))
                        kitty/fonts.c:257     sprite_position_for(FontGroup*, Font*, glyphs, ...)
                        kitty/fonts.c:667,744,1455,1470  current_send_sprite_to_gpu((FONTS_DATA_HANDLE)fg, ...)
```

**Interpretation.** **(observed via source)** HarfBuzz shaping lives in `kitty/fonts.c`
(`hb_shape`, line 813) — *not* in `freetype.c`; FreeType only **rasterizes** glyphs
(`render_bitmap` → `FT_Render_Glyph`). `kitty/glyph-cache.c` is a **CPU hash cache**
mapping shaped glyph runs to sprite positions (`find_or_create_sprite_position`,
`free_sprite_position_hash_table`) — it is *not* the GPU atlas. The **GPU texture
atlas** is allocated and uploaded in `kitty/shaders.c`
(`alloc_sprite_map`/`glTexStorage3D`/`glTexSubImage3D`), and the upload is
**orchestrated from `kitty/fonts.c`** through the `current_send_sprite_to_gpu` macro.
**(inferred from source, confirmed by the frame)** I did not instrument the individual
`glTexSubImage3D` calls at runtime, so the per-call execution is inferred from the
source; however, the durable rendered frame in §5.4 — with correctly-shaped,
rasterized, coloured cells — is runtime proof that this entire chain executed
end-to-end (no pixels could appear otherwise). This corrects the earlier document's
material misattribution of these components.

### 5.3 GLSL shaders and the GL context (observed source enumeration + observed runtime context)

**Source enumeration (static — labeled as such):**

```text
$ ls -1 kitty/*.glsl
kitty/alpha_blend.glsl
kitty/bgimage_fragment.glsl
kitty/bgimage_vertex.glsl
kitty/border_fragment.glsl
kitty/border_vertex.glsl
kitty/cell_defines.glsl
kitty/cell_fragment.glsl
kitty/cell_vertex.glsl
kitty/graphics_fragment.glsl
kitty/graphics_vertex.glsl
kitty/linear2srgb.glsl
kitty/tint_fragment.glsl
kitty/tint_vertex.glsl
count: 13
```

**Runtime GL context (observed, from the `debug_config` dump in §3.3):**

```text
OpenGL: '4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1' Detected version: 4.5
```

**Interpretation.** **(observed)** The repository contains **13** GLSL sources; the
cell-drawing shaders among them are `cell_vertex.glsl`, `cell_fragment.glsl` and the
shared `cell_defines.glsl`. **(observed)** A real OpenGL **4.5 Core** context was
obtained from Mesa's software renderer (LLVMpipe) under Xvfb — this satisfies kitty's
GL ≥ 3.3 requirement without a GPU. **(inferred, then confirmed)** The `.glsl`
enumeration alone is *static* evidence and does not by itself prove the shaders
compiled or a frame was presented; the proof that they did is the durable rendered
frame in §5.4, which is impossible to produce without a compiled, linked, executing
cell-shader program.

### 5.4 Durable pixel evidence: fonts, styles, colours, cell grid (observed)

A child printed ANSI-styled content; the framebuffer was captured from the host with
`import -window root`. The model was also read with `get-text` to show, by contrast,
what get-text **cannot** preserve.

```text
Child: sh -c 'printf ...bold/italic/underline, 6 ANSI colours, reverse-video, a grid row...; exec sleep 40'
capture timing: after the OS window appeared (xdotool search --sync --class kitty),
                the script polled get-text until the grid row ('aligned') was present
                in the model, then settled 0.5 s, THEN captured -- so the frame is
                synchronized to a readiness gate, not a fixed wall-clock instant.
capture cmd: import -window root  ->  full frame 1280x800 PNG (ImageMagick, host-side)
  full-frame sha256 : 636c00534c9829bb219db625fa9f6562d19cedd0237fa8b76bf36803583f6d4b
  embedded crop     : 760x100 top-left region, PNG8, sha256 2187955370d4befa2edfda5e0deac7fa5ec5a6599e86d5912fbe08ba75830177

### get-text model readback (TEXT ONLY - no colour, weight, slant, or underline) ###
BOLD ITALIC UNDERLINE
RED GREEN YELLOW BLUE MAGENTA CYAN
 BLACK-ON-WHITE   WHITE-ON-RED
0123456789 ABCDEFGHIJ |aligned|cell|grid|
$ prompt-like line with a block cursor below
```

Durable rendered frame (760x100 top-left crop of the 1280x800 framebuffer):

![Durable screenshot of the kitty framebuffer showing five rows of rendered terminal content on a black background. Row 1: the word BOLD in a heavy weight, ITALIC in a slanted oblique face, and UNDERLINE with a horizontal underline beneath it, all in white. Row 2: the words RED, GREEN, YELLOW, BLUE, MAGENTA and CYAN each drawn in that respective colour. Row 3: BLACK-ON-WHITE shown as dark glyphs on a white cell-background block, and WHITE-ON-RED shown as white glyphs on a red cell-background block. Row 4: the string 0123456789 ABCDEFGHIJ followed by pipe-delimited words aligned=cell=grid where the vertical bars line up in fixed columns. Row 5: a green dollar sign followed by prompt-like text. Glyphs are anti-aliased monospaced DejaVu Sans Mono.](data:image/png;base64,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)

**Interpretation (observed).** Each visual claim is now tied to retained pixels, not
to model text:
- **Font faces** — "BOLD" is drawn heavier than the surrounding text and "ITALIC" is
  slanted, matching the Bold and Oblique files from §5.1; the base face is the
  monospaced DejaVu Sans Mono.
- **Underline** — a horizontal rule sits under "UNDERLINE" (a cell decoration, not a
  glyph).
- **Colours** — the six words render in red/green/yellow/blue/magenta/cyan (the ANSI
  palette); backgrounds show as filled cell blocks (black-on-white, white-on-red),
  proving background/reverse attributes and per-cell compositing.
- **Layout** — in row 4 the `|` characters line up in fixed columns, evidencing the
  monospaced cell grid.
- **Black background** — the default background is black.

Crucially, the `get-text` readback above contains **none** of this styling — it is
plain text (`BOLD ITALIC UNDERLINE`, etc.). That is the point: `get-text` is a
model-level readback and cannot preserve font, weight, slant, colour, background, or
the cell grid, so only the durable framebuffer image proves the display attributes.

### 5.5 Scrolling and scrollback: forced overflow + an actual scroll (observed)

A child printed 120 numbered lines — far more than one viewport — then the viewport
and the full history were read, and finally an **actual scroll** was performed:

```text
Child emitted LINE_1 .. LINE_120 (120 lines, far exceeding one viewport).

### BEFORE scroll -- get-text (viewport / on-screen only) ###
visible LINE_ rows: 21 ; first=LINE_100 ; last=LINE_120

### full scrollback -- get-text --extent all ###
total LINE_ rows: 120 ; first=LINE_1 ; last=LINE_120

### AFTER an actual scroll operation (kitty @ action scroll_home) -- get-text (viewport) ###
visible first=LINE_1 ; last=LINE_22   (viewport moved to the top of history)
```

**Source authority.** Scrollback is stored in the history buffer
(`HistoryBuf`/`add_segment` [kitty/history.c:18-33]); `get-text` is the remote-control
endpoint `GetText` calling `window.as_text(...)` [kitty/rc/get_text.py:13,116];
`scroll_home` is a mappable action invoked through remote control.

**Interpretation (observed).** Before any scroll, the viewport holds only the **tail**
(`LINE_100`–`LINE_120`, ~21 rows) — lines 1–99 have been **evicted off-screen into
scrollback**. `get-text --extent all` returns **all 120** lines (`LINE_1`–`LINE_120`),
proving the scrollback retained the evicted content. After a real
`kitty @ action scroll_home`, the viewport **moves** to show `LINE_1`–`LINE_22` — the
top of history — demonstrating actual viewport movement, not merely a model query.
This also distinguishes the three distinct operations Question 4/Finding #32 asks to
separate: an **action invocation** (`scroll_home`), a **model readback** (`get-text`,
optionally `--extent all`), and **framebuffer observation** (§5.4) — they are not the
same path.

---

## 6. Edge and Error Conditions

Per the exhaustive-coverage rule, the transitional and error states implied by the
questions were exercised and captured. Each is shown as command → complete output →
source authority → interpretation.

### 6.1 systemd user-bus failure, then graceful continuation (observed)

The startup log shows the systemd registration **failing** and kitty **continuing**
to a working terminal. Complete merged (`2>&1`) capture of the relevant run:

```text
$ ./kitty/launcher/kitty --debug-rendering -o confirm_os_window_close=0 \
      sh -c 'echo HELLO_$$; sleep 3'   2>&1
[0.278] OS Window created
[0.292] Failed to open systemd user bus with error: No medium found
[0.296] Child launched
[0.203] GL version string: '4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1' Detected version: 4.5
```

Note the `[0.203]` GL line lands **last** despite the earliest timestamp — the
stdout-buffering artifact dissected in §2 (the GL string is written to buffered
stdout; the `debug()` markers go to unbuffered stderr). The systemd→continuation
transition is the three stderr lines in timestamp order (`0.278 → 0.292 → 0.296`).

**Source authority.** The bus is opened post-fork in the child scope:
`spawn()`→`systemd_move_pid_into_new_scope()` [kitty/child.py:346-353], whose C side
logs `log_error("Failed to open systemd user bus ...")` [kitty/systemd.c:87] and then
raises `PyExc_NotImplementedError` [kitty/systemd.c:185-188]; Python swallows it with
`except NotImplementedError: pass` [kitty/child.py:349-351].

**Interpretation (observed).** The failure is **non-fatal by design**: the child is
still released and `Child launched` [kitty/window.py:871] prints **4 ms after** the
error. The container message is `No medium found` (not the AAP's illustrative
`Connection refused`), reported here exactly as observed. This is the graceful
degradation edge — the terminal reaches readiness despite no systemd bus.

### 6.2 `--debug-config` is not a valid flag at this commit (error path, observed)

```text
$ ./kitty/launcher/kitty --debug-config ; echo "EXIT=$?"
Unknown option: --debug-config
EXIT=1
```

**Source authority.** The debug flags that *do* exist are declared in
`kitty/cli.py:989-1002` (`--debug-rendering`, `--debug-input`, `--debug-font-fallback`,
…); `--debug-config` is absent, so option parsing rejects it. The configuration dump
is instead reached through the `debug_config` **action** [kitty/debug_config.py:231]
(the `ctrl+shift+f6` mapping exercised in §3.3).

**Interpretation (observed).** The non-zero exit status (`EXIT=1`) and the
`Unknown option` message are the concrete error-path signal; this is why §3.3 uses
the in-session action, not a CLI flag. (This corrects any impression that
`--debug-config` is usable at `815df1e210e0`.)

### 6.3 Build strictness (observed in-container; host case disclosed as inferred)

- **Observed (container):** the pure `python3 setup.py build` succeeds at **exit 0**
  with **no** deviation flag; the `-Werror` gate is satisfied (§1.3; full log in
  Appendix C). A `grep -ic 'werror'` over the 332-line log returns `0`.
- **Disclosed (inferred, host):** on a newer host (Ubuntu 25.10) the same build fails
  at `glfw/wl_window.c:668` under `-Werror=switch` and requires the official
  `--ignore-compiler-warnings` flag [setup.py:2003-2004] (§1.3 blockquote). Not
  reproduced in the mandated image; a compile-time-only artifact of the Wayland
  backend that never affects the observed X11 runtime.

### 6.4 First shell output — before / during / after (observed)

Fully evidenced in §4.6 with two durable framebuffer captures. The discriminating
proof is that the **full-frame hashes differ** across the transition:

```text
BEFORE first output (t≈+3.75s): screen empty; full-frame sha256 f3bdcc59...
AFTER  first output (t≈+6.20s): FIRST_OUTPUT_MARKER_Q3 drawn; full-frame sha256 c0c34d6f...
```

**Interpretation (observed).** Identical inputs, different pixels ⇒ the child's first
bytes were parsed (`consume_normal` [kitty/vt-parser.c:230]) and drawn
(`screen_draw_text` [kitty/screen.c:866]); see §4.6 for the embedded images.

### 6.5 Scrollback eviction — before / after an actual scroll (observed)

Fully evidenced in §5.5. The discriminating proof:

```text
BEFORE scroll  : viewport = LINE_100..LINE_120 (21 rows); lines 1..99 evicted off-screen
scrollback all : get-text --extent all = LINE_1..LINE_120 (120 rows) retained
AFTER scroll_home: viewport = LINE_1..LINE_22 (viewport physically moved to top)
```

**Interpretation (observed).** Output that exceeds one viewport is retained in the
history buffer (`HistoryBuf` [kitty/history.c:18-33]) and a real `scroll_home` action
moves the viewport — demonstrating both scrollback and scrolling.

### 6.6 True first launch — no user config present (observed)

```text
config_dir  : /root/.config/kitty
defconf     : /root/.config/kitty/kitty.conf
defconf exists: False
```

**Source authority.** `config_dir` [kitty/constants.py:131] and `defconf`
[kitty/constants.py:133]; the emptiness of the dump's `Config options different from
defaults:` section is produced by `debug_config()` [kitty/debug_config.py:231].

**Interpretation (observed).** With `defconf exists: False`, the launch used **only
built-in defaults** — the genuine first-launch condition Question 2 asks about — and
§3.3's dump shows an empty diff, confirming no file contributed.

---

## 7. Honesty and Compatibility Notes

### 7.1 Software OpenGL (LLVMpipe), not a discrete GPU (observed)

The GL context is `4.5 (Core Profile) Mesa 24.2.8-1ubuntu1~24.04.1`, renderer
LLVMpipe (§3.3, §5.3). This is Mesa's **software** rasterizer under Xvfb. It satisfies
kitty's GL ≥ 3.3 requirement and exercises the *same* shader/render code path a
hardware GPU would; the only difference is that rasterization runs on the CPU.
**(inferred)** Pixel output is therefore representative of the real render pipeline;
performance is not, and no timing claims are made from it.

### 7.2 What the harness overrides change — and why they don't distort the evidence

The launches pass `-o confirm_os_window_close=0` (and, where noted, `cursor_shape`,
`scrollback_lines`). **Its actual effect** is narrow: `confirm_os_window_close`
[kitty/options/definition.py:1219-1231] only controls whether closing a window with
running programs shows a **confirmation prompt**; setting it to `0` **disables that
prompt** — it does **not** cause the window to auto-close or the process to exit on
its own. Termination is therefore **explicit**, never implicit: the child is a bounded
`sh -c '...; sleep N'`, and the harness owns process lifecycle directly — it captures
each background PID via `$!`, and a `trap ... EXIT INT TERM` `kill`s the exact Xvfb
PID and `wait`s on it while Docker `--rm` removes the container (§1.5, §8.2). So the
flag merely prevents a blocking dialog on a headless run; it does not fabricate a
clean exit. **(observed)** The overrides appear honestly in the `debug_config` dump's
`Loaded config overrides:` section when used (§3.4), and the **pure-default** dump in
§3.3 was captured with **no** `-o` config overrides at all, so the first-launch/default
evidence is not contaminated by them.

### 7.3 Platform scope: Linux/X11 via Xvfb (observed); macOS and Wayland unexercised

All observations are the **Linux/X11 (Xvfb)** path at commit `815df1e210e0`, kitty
0.35.2. The macOS CoreText/Cocoa backend (`kitty/core_text.m`) and the Wayland
runtime backend were **not** exercised and are **not** represented as observed. The
bundled-GLFW Wayland unit is *compiled* (§1.3, `[3/122]`) but never *entered* at
runtime on the X11 path.

### 7.4 Build deviation: none required in the mandated container (observed)

Contrary to the AAP's precautionary note, the mandated Ubuntu 24.04 image builds
cleanly with the pure command and **no** `--ignore-compiler-warnings` (§1.3,
Appendix C). The deviation is disclosed only as an inferred host-specific case (§6.3);
it is not part of the observed build.

### 7.5 Remote control is model/IPC readback, not a render bypass (observed)

`kitty @ get-text`, `kitty @ ls`, and `kitty @ action …` reach the running instance
over its control socket and read the **screen model** or invoke **mapped actions**;
`GetText` calls `window.as_text()` [kitty/rc/get_text.py:13,116]. **(observed)** They
are *not* a shortcut around the real display path: they cannot produce or inspect
pixels, which is exactly why the durable framebuffer captures (§4.6, §5.4) — taken
with host-side `import` against the same live window — are used for every visual
claim. Model readback and framebuffer observation are treated as distinct evidence
throughout.

---

## 8. Cleanup and Final Repository Proof

The investigation created only **transient** artifacts; the source tree was treated as
strictly read-only. This section shows the actual teardown (commands and output) and
the final repository proof.

### 8.1 What the investigation created (transient)

- **Container build outputs** — `kitty/fast_data_types.so`, `kitty/launcher/kitty`,
  `kitty/launcher/kitten`, and `build/` were produced **inside** the mandated
  `--rm` container (§1.3); they vanish with the container and never touch the host
  repository.
- **Host HOME state** — kitty wrote `/root/.cache/kitty/` (a `main.json` window-size
  memory and an empty `run/`) during earlier host-side probing; `/root/.config/kitty/`
  existed as an empty directory (no `kitty.conf` — consistent with the observed
  `defconf exists: False`).
- **Agent scratch** — a `/tmp` working directory held the harness, capture scripts,
  and evidence; it is removed at finalization and is never part of the repository.

### 8.2 Teardown — before, commands, after (observed)

```text
### BEFORE ###
$ ls -la /root/.cache/kitty ; ls -la /root/.config/kitty
-rw------- 1 root root 28 ... main.json         # {"window-size": [1000, 350]}
drwx------ 2 root root ...  run/                 # empty
drwxr-xr-x 2 root root ...  /root/.config/kitty  # empty
$ ps -eo pid,comm | grep -E 'Xvfb|kitty|import'  # -> (none: harness EXIT-trap already reaped them)
$ docker ps -aq                                   # -> (none: every run used --rm)

### TEARDOWN ###
$ rm -rf /root/.cache/kitty /root/.config/kitty
exit=0

### AFTER ###
$ ls -la /root/.cache/kitty     # -> removed (absent)
$ ls -la /root/.config/kitty    # -> removed (absent)
$ ps -eo pid,comm | grep -E 'Xvfb|kitty|import'   # -> no kitty/Xvfb/import processes
$ docker ps -aq                                    # -> none
```

**Interpretation (observed).** The investigation-created HOME state is gone, no
harness process (Xvfb, kitty, `import`, `xdotool`) is left running, and no container
remains — the harness's `trap ... EXIT INT TERM` and Docker `--rm` had already
prevented leaks; the explicit `rm -rf` cleared the persisted HOME residue.

### 8.3 Final repository proof (observed)

```text
$ git status --porcelain
 M blitzy/documentation/kitty_815df1e210e0.md

$ git diff --check
(clean)

$ git diff --stat
 blitzy/documentation/kitty_815df1e210e0.md | 1700 +++++++++++++++++++++++-----
 1 file changed, 1422 insertions(+), 278 deletions(-)
```

**Interpretation (observed).** Exactly **one** file is modified — the answer document
`blitzy/documentation/kitty_815df1e210e0.md`. **No source file, dependency manifest,
test, or configuration was created, changed, or deleted**, satisfying the read-only
mandate. `git diff --check` reports no whitespace/EOF defects. This is the pristine
end state; the change is committed as the sole deliverable.

---


## Appendix A: Observed vs Inferred Ledger

Every load-bearing statement is classified. **Observed** = captured at runtime in the
mandated container; **Inferred** = derived from reading source (not directly executed),
always confirmed by an adjacent observed signal where one exists.

| # | Claim | Status | Evidence / Section |
|---|-------|--------|--------------------|
| 1 | Version banner `kitty 0.35.2 (815df1e210) created by Kovid Goyal` | Observed | §1.4, §3.3 dump |
| 2 | Container = Ubuntu 24.04.2, Python 3.12.3, Go 1.23.4, gcc 13.3.0 | Observed | §1.1 |
| 3 | Pure `setup.py build` succeeds at exit 0, no deviation flag | Observed | §1.3, Appendix C |
| 4 | Host (Ubuntu 25.10) needs `--ignore-compiler-warnings` for `wl_window.c:668` | Inferred | §1.3, §6.3 |
| 5 | Startup order: GL context → OS Window created → systemd attempt → Child launched | Observed | §2, §6.1 |
| 6 | `OS Window created` is a completion marker emitted at end of window setup | Observed+Inferred | §2 (line), source `kitty/glfw.c:1321` |
| 7 | systemd bus fails (`No medium found`) and kitty continues | Observed | §6.1 |
| 8 | systemd op is post-fork child-scope; NotImplementedError swallowed | Inferred | source `kitty/child.py:346-353`, `kitty/systemd.c:87,185-188` |
| 9 | Config precedence: defaults → SYSTEM_CONF → defconf|`-c` (replace) → `-o` (last); `NONE` suppresses | Observed | §3.2, §3.4 |
| 10 | First launch used only built-in defaults (`defconf exists: False`, empty diff) | Observed | §3.1, §3.3, §6.6 |
| 11 | GL context `4.5 (Core Profile) Mesa 24.2.8` (LLVMpipe, software) | Observed | §3.3, §5.3, §7.1 |
| 12 | `--debug-config` is not a valid flag (`EXIT=1`) | Observed | §3.6, §6.2 |
| 13 | PTY allocated via `openpty()`; child forked; controlling TTY set; `execvp` shell | Observed+Inferred | §4.1, source `kitty/child.py:170`, `kitty/child.c:97-159` |
| 14 | `Child launched` = terminal-ready release (after geometry + `mark_terminal_ready`) | Observed+Inferred | §4.2, source `kitty/window.py:861-871`, `kitty/child.py:362-364` |
| 15 | `TERM=xterm-kitty` visible to the child; `TERMINFO=/app/terminfo` | Observed | §4.3 (`kitty @ get-text` round-trip) |
| 16 | Interactive shell launched as `/bin/bash --posix` | Observed | §4.3 (`kitty @ ls`), source `kitty/shell_integration.py:146` |
| 17 | shell-integration guard L4 / re-export L77 / final unset L117 | Observed+Inferred | §4.4, source `shell-integration/bash/kitty.bash` |
| 18 | First shell bytes parsed (`consume_normal`) and drawn (`screen_draw_text`) | Observed | §4.5, §4.6 (differing frame hashes) |
| 19 | Font faces: DejaVuSansMono Normal/Bold/Oblique/BoldOblique | Observed | §5.1 (`--debug-font-fallback`), §3.3 |
| 20 | Pipeline: `hb_shape` shaping → FreeType raster → glyph-cache hash → GPU atlas (shaders.c) → cell shaders | Observed(frame)+Inferred(internals) | §5.2 |
| 21 | 13 GLSL shader sources in the repo | Observed (static enumeration) | §5.3 |
| 22 | Shaders compiled and a frame presented | Inferred, confirmed by durable frame | §5.3, §5.4 |
| 23 | Bold/italic/underline, 6 ANSI colours, reverse/bg blocks, aligned cell grid | Observed (durable pixels) | §5.4 |
| 24 | Scrollback retains evicted lines; `scroll_home` moves viewport | Observed | §5.5, §6.5 |
| 25 | Remote control is model/IPC readback, not a render bypass | Observed+Inferred | §7.5, source `kitty/rc/get_text.py:13,116` |
| 26 | Linux/X11 (Xvfb) only; macOS CoreText/Cocoa and Wayland runtime unexercised | Observed (scope) | §7.3 |

## Appendix B: Citation Index

Exact symbols and line ranges for every load-bearing code fact (verified against the
source at commit `815df1e210e0`).

| Domain | Symbol | File:line |
|--------|--------|-----------|
| Launcher | `main()` | `kitty/launcher/main.c:439` |
| Bootstrap | mode dispatch → `kitty.main.main()` | `kitty/entry_points.py:140-164` |
| GUI init | ten-step init / `main()` | `kitty/main.py:90-95,524` |
| OS window | `debug("OS Window created\n")` | `kitty/glfw.c:1321` |
| GL context | GL version string build/print | `kitty/gl.c:47,72` |
| systemd | `log_error("Failed to open systemd user bus...")` | `kitty/systemd.c:87` |
| systemd | raises `PyExc_NotImplementedError` | `kitty/systemd.c:185-188` |
| systemd (py) | `systemd_move_pid_into_new_scope()`; `except NotImplementedError: pass` | `kitty/child.py:346-353` |
| Boss | `Boss.start()` | `kitty/boss.py:1181` |
| Child monitor | I/O threads created in `start()` | `kitty/child-monitor.c:281-289` |
| VT parser worker | `parse_worker` | `kitty/vt-parser.c:1495-1496` |
| Config dir | `config_dir` / `defconf` | `kitty/constants.py:131,133` |
| Config dir env | `_get_config_dir` / `KITTY_CONFIG_DIRECTORY` | `kitty/constants.py:87-89` |
| System conf | `SYSTEM_CONF = /etc/xdg/kitty/kitty.conf` | `kitty/cli.py:1064` |
| Config resolve | `default_config_paths` | `kitty/cli.py:1067-1068` |
| Config resolve | `resolve_config` (branch logic) | `kitty/conf/utils.py:322-329` |
| Config load | `load_config` (overrides merged last) | `kitty/config.py:163` |
| Defaults schema | `opt('term','xterm-kitty',...)` | `kitty/options/definition.py:3242` |
| Options object | `term: str = 'xterm-kitty'` | `kitty/options/types.py:602` |
| Option parser | `Parser.cursor_shape()` → `to_cursor_shape()` | `kitty/options/parse.py:919-920` |
| Debug config | `debug_config()` | `kitty/debug_config.py:231` |
| Debug flags | `--debug-rendering`/`--debug-input`/`--debug-font-fallback` | `kitty/cli.py:989-1002` |
| Native transfer | `set_options()` in `AppRunner.__call__` | `kitty/main.py:249` |
| PTY alloc | `openpty()` → `os.openpty()` | `kitty/child.py:170-171` |
| Fork | `Child.fork()` | `kitty/child.py:276` |
| Child C | `fork`/`setsid`/`ioctl TIOCSCTTY`/`execvp` | `kitty/child.c:97,123,129,159` |
| Ready release | `mark_terminal_ready()` closes ready fd | `kitty/child.py:362-364` |
| Child launched | `print('[..] Child launched', ...)` | `kitty/window.py:871` |
| TERM env | `env['TERM'] = opts.term` | `kitty/child.py:242` |
| TERMINFO env | set in child env | `kitty/child.py:255-260` |
| terminfo db | `xterm-kitty|KovIdTTY,` | `terminfo/kitty.terminfo:1` |
| shell integ | `argv.insert(1, '--posix')` | `kitty/shell_integration.py:146` |
| VT control | `dispatch_single_byte_control` | `kitty/vt-parser.c:224` |
| VT text | `consume_normal` | `kitty/vt-parser.c:230` |
| VT osc | `dispatch_osc` | `kitty/vt-parser.c:457` |
| Screen draw | `screen_draw_text` | `kitty/screen.c:866` |
| Font debug | `dump_font_debug()` → `log_error('Text fonts:')` | `kitty/fonts/render.py:163` |
| Font match | `_fc_match()` → `FcFontMatch()`; `fc_list()` | `kitty/fontconfig.c:269,276,235` |
| Shaping | `hb_shape(...)` | `kitty/fonts.c:813` |
| Raster | `render_bitmap` / `FT_Render_Glyph` | `kitty/freetype.c:507,904` |
| Glyph cache | `find_or_create_sprite_position` / `free_sprite_position_hash_table` | `kitty/glyph-cache.c:34,57` |
| GPU atlas | `alloc_sprite_map` / `glTexStorage3D` / `glTexSubImage3D` | `kitty/shaders.c:51,123,99,155` |
| Atlas upload | `current_send_sprite_to_gpu` macro / `sprite_position_for` / call sites | `kitty/fonts.c:21,257,667,744,1455,1470` |
| Scrollback | `HistoryBuf` / `add_segment` | `kitty/history.c:18-33` |
| Remote get-text | `GetText` → `window.as_text()` | `kitty/rc/get_text.py:13,116` |
| Build gate | `werror = '' if ignore_compiler_warnings else '-pedantic-errors -Werror'` | `setup.py:491,1231` |
| Build flag | `--ignore-compiler-warnings` | `setup.py:2003-2004` |
| Build target | `all:` → `python3 setup.py $(VVAL)` | `Makefile:12-13` |

## Appendix C: Complete Build Log

Verbatim, complete output of the from-scratch canonical build in the mandated
container (referenced by §1.3). Command:
`cd /app && rm -rf build kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so && python3 setup.py build ; echo BUILD_EXIT=$?`. Total 332 lines; ends with `BUILD_EXIT=0`.

```text
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
[13/122] Compiling [x11] glfw/input.c ...
[14/122] Compiling [wayland] glfw/input.c ...
[15/122] Compiling kitty/mouse.c ...
[16/122] Compiling [x11] glfw/xkb_glfw.c ...
[17/122] Compiling [wayland] glfw/xkb_glfw.c ...
[18/122] Compiling kitty/freetype.c ...
[19/122] Compiling [wayland] glfw/wl_client_side_decorations.c ...
[20/122] Compiling [x11] glfw/window.c ...
[21/122] Compiling [wayland] glfw/window.c ...
[22/122] Compiling kitty/line.c ...
[23/122] Compiling kitty/glfw-wrapper.c ...
[24/122] Compiling kittens/transfer/algorithm.c ...
[25/122] Compiling [wayland] glfw/wl_init.c ...
[26/122] Compiling [x11] glfw/x11_init.c ...
[27/122] Compiling kitty/freetype_render_ui_text.c ...
[28/122] Compiling [x11] glfw/egl_context.c ...
[29/122] Compiling [wayland] glfw/egl_context.c ...
[30/122] Compiling kitty/disk-cache.c ...
[31/122] Compiling [x11] glfw/glx_context.c ...
[32/122] Compiling kitty/line-buf.c ...
[33/122] Compiling kitty/data-types.c ...
[34/122] Compiling kitty/colors.c ...
[35/122] Compiling kitty/history.c ...
[36/122] Compiling kitty/keys.c ...
[37/122] Compiling [x11] glfw/x11_monitor.c ...
[38/122] Compiling kitty/fontconfig.c ...
[39/122] Compiling [x11] glfw/context.c ...
[40/122] Compiling [wayland] glfw/context.c ...
[41/122] Compiling kitty/crypto.c ...
[42/122] Compiling [x11] glfw/ibus_glfw.c ...
[43/122] Compiling [wayland] glfw/ibus_glfw.c ...
[44/122] Compiling kitty/key_encoding.c ...
[45/122] Compiling kitty/launcher/main.c ...
[46/122] Compiling [x11] glfw/monitor.c ...
[47/122] Compiling [wayland] glfw/monitor.c ...
[48/122] Compiling kitty/font-names.c ...
[49/122] Compiling [x11] glfw/backend_utils.c ...
[50/122] Compiling [wayland] glfw/backend_utils.c ...
[51/122] Compiling kitty/charsets.c ...
[52/122] Compiling [x11] glfw/linux_joystick.c ...
[53/122] Compiling [wayland] glfw/linux_joystick.c ...
[54/122] Compiling [x11] glfw/init.c ...
[55/122] Compiling [wayland] glfw/init.c ...
[56/122] Compiling [x11] glfw/dbus_glfw.c ...
[57/122] Compiling [wayland] glfw/dbus_glfw.c ...
[58/122] Compiling kitty/gl.c ...
[59/122] Compiling [x11] glfw/vulkan.c ...
[60/122] Compiling [wayland] glfw/vulkan.c ...
[61/122] Compiling [x11] glfw/osmesa_context.c ...
[62/122] Compiling [wayland] glfw/osmesa_context.c ...
[63/122] Compiling kitty/cursor.c ...
[64/122] Compiling kitty/launcher/single-instance.c ...
[65/122] Compiling kitty/desktop.c ...
[66/122] Compiling kitty/loop-utils.c ...
[67/122] Compiling 3rdparty/ringbuf/ringbuf.c ...
[68/122] Compiling kitty/simd-string.c ...
[69/122] Compiling kitty/systemd.c ...
[70/122] Compiling kitty/shlex.c ...
[71/122] Compiling [wayland] glfw/wayland-tablet-unstable-v2-client-protocol.c ...
[72/122] Compiling kitty/child.c ...
[73/122] Compiling [wayland] glfw/linux_desktop_settings.c ...
[74/122] Compiling [wayland] glfw/wl_text_input.c ...
[75/122] Compiling [wayland] glfw/wl_monitor.c ...
[76/122] Compiling kitty/kittens.c ...
[77/122] Compiling 3rdparty/base64/lib/codec_choose.c ...
[78/122] Compiling kitty/png-reader.c ...
[79/122] Compiling [wayland] glfw/wayland-xdg-shell-client-protocol.c ...
[80/122] Compiling [x11] glfw/linux_notify.c ...
[81/122] Compiling [wayland] glfw/linux_notify.c ...
[82/122] Compiling kitty/rowcolumn-diacritics.c ...
[83/122] Compiling kitty/hyperlink.c ...
[84/122] Compiling [wayland] glfw/wayland-primary-selection-unstable-v1-client-protocol.c ...
[85/122] Compiling kitty/wcswidth.c ...
[86/122] Compiling [wayland] glfw/wayland-pointer-constraints-unstable-v1-client-protocol.c ...
[87/122] Compiling kitty/fast-file-copy.c ...
[88/122] Compiling [wayland] glfw/wayland-text-input-unstable-v3-client-protocol.c ...
[89/122] Compiling [wayland] glfw/wayland-wlr-layer-shell-unstable-v1-client-protocol.c ...
[90/122] Compiling 3rdparty/base64/lib/lib.c ...
[91/122] Compiling [x11] glfw/posix_thread.c ...
[92/122] Compiling [wayland] glfw/posix_thread.c ...
[93/122] Compiling kitty/window_logo.c ...
[94/122] Compiling [wayland] glfw/wayland-xdg-activation-v1-client-protocol.c ...
[95/122] Compiling [wayland] glfw/wayland-xdg-decoration-unstable-v1-client-protocol.c ...
[96/122] Compiling [wayland] glfw/wayland-relative-pointer-unstable-v1-client-protocol.c ...
[97/122] Compiling [wayland] glfw/wayland-cursor-shape-v1-client-protocol.c ...
[98/122] Compiling [wayland] glfw/wayland-fractional-scale-v1-client-protocol.c ...
[99/122] Compiling kitty/glyph-cache.c ...
[100/122] Compiling [wayland] glfw/wayland-viewporter-client-protocol.c ...
[101/122] Compiling kitty/logging.c ...
[102/122] Compiling 3rdparty/base64/lib/arch/neon64/codec.c ...
[103/122] Compiling [wayland] glfw/wayland-single-pixel-buffer-v1-client-protocol.c ...
[104/122] Compiling 3rdparty/base64/lib/tables/tables.c ...
[105/122] Compiling [wayland] glfw/wl_cursors.c ...
[106/122] Compiling 3rdparty/base64/lib/arch/neon32/codec.c ...
[107/122] Compiling [wayland] glfw/wayland-kwin-blur-v1-client-protocol.c ...
[108/122] Compiling 3rdparty/base64/lib/arch/avx/codec.c ...
[109/122] Compiling 3rdparty/base64/lib/arch/ssse3/codec.c ...
[110/122] Compiling 3rdparty/base64/lib/arch/sse42/codec.c ...
[111/122] Compiling 3rdparty/base64/lib/arch/sse41/codec.c ...
[112/122] Compiling 3rdparty/base64/lib/arch/avx2/codec.c ...
[113/122] Compiling kitty/utmp.c ...
[114/122] Compiling 3rdparty/base64/lib/arch/avx512/codec.c ...
[115/122] Compiling 3rdparty/base64/lib/arch/generic/codec.c ...
[116/122] Compiling kitty/cleanup.c ...
[117/122] Compiling [x11] glfw/monotonic.c ...
[118/122] Compiling [wayland] glfw/monotonic.c ...
[119/122] Compiling kitty/monotonic.c ...
[120/122] Compiling kitty/simd-string-128.c ...
[121/122] Compiling kitty/simd-string-256.c ...
[122/122] Compiling kitty/gl-wrapper.c ...
 done
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
log/internal
container/list
github.com/shirou/gopsutil/v3/common
github.com/seancfoley/ipaddress-go/ipaddr/addrerr
internal/nettrace
unicode/utf16
vendor/golang.org/x/crypto/cryptobyte/asn1
crypto/subtle
kitty
encoding
github.com/seancfoley/ipaddress-go/ipaddr/addrstr
image/color
vendor/golang.org/x/crypto/internal/alias
golang.org/x/exp/constraints
crypto/internal/alias
github.com/seancfoley/ipaddress-go/ipaddr/addrstrparam
crypto/internal/boring/sig
internal/weak
maps
vendor/golang.org/x/net/dns/dnsmessage
math/rand/v2
internal/singleflight
hash
encoding/base32
crypto/rc4
crypto/internal/randutil
vendor/golang.org/x/text/transform
net/http/internal/ascii
bufio
regexp/syntax
encoding/binary
context
crypto/cipher
crypto/internal/edwards25519/field
embed
crypto/internal/nistec/fiat
runtime/cgo
io/ioutil
encoding/hex
log
github.com/bmatcuk/doublestar/v4
flag
kitty/tools/utils/shlex
net/url
vendor/golang.org/x/sys/cpu
vendor/golang.org/x/net/http2/hpack
github.com/ALTree/bigfloat
crypto/internal/bigmod
encoding/asn1
github.com/seancfoley/bintree/tree
crypto/dsa
crypto
hash/adler32
hash/crc32
image/color/palette
crypto/md5
golang.org/x/image/riff
compress/bzip2
internal/concurrent
compress/flate
os/exec
database/sql/driver
crypto/des
crypto/internal/boring
compress/lzw
golang.org/x/image/tiff/lzw
net/http/internal
encoding/xml
crypto/internal/edwards25519
os/signal
mime/quotedprintable
image
vendor/golang.org/x/text/unicode/bidi
encoding/base64
unique
vendor/golang.org/x/crypto/chacha20
github.com/rwcarlsen/goexif/tiff
crypto/internal/boring/bbig
crypto/rand
crypto/sha512
vendor/golang.org/x/crypto/internal/poly1305
github.com/klauspost/cpuid/v2
crypto/sha256
crypto/hmac
crypto/sha1
github.com/dlclark/regexp2/syntax
crypto/aes
vendor/golang.org/x/crypto/sha3
vendor/golang.org/x/text/unicode/norm
golang.org/x/sys/unix
crypto/x509/pkix
vendor/golang.org/x/crypto/cryptobyte
regexp
vendor/golang.org/x/crypto/hkdf
crypto/rsa
encoding/pem
kitty/tools/utils/secrets
mime
encoding/json
vendor/golang.org/x/crypto/chacha20poly1305
github.com/shirou/gopsutil/v3/internal/common
net/netip
crypto/ed25519
compress/gzip
compress/zlib
archive/zip
crypto/internal/mlkem768
vendor/golang.org/x/text/secure/bidirule
golang.org/x/image/bmp
golang.org/x/image/ccitt
image/internal/imageutil
golang.org/x/image/vp8l
image/png
golang.org/x/image/vp8
crypto/internal/nistec
image/draw
image/jpeg
golang.org/x/image/tiff
golang.org/x/image/webp
image/gif
github.com/zeebo/xxh3
howett.net/plist
vendor/golang.org/x/net/idna
github.com/disintegration/imaging
github.com/kovidgoyal/imaging
github.com/dlclark/regexp2
crypto/ecdh
crypto/elliptic
crypto/internal/hpke
github.com/rwcarlsen/goexif/exif
crypto/ecdsa
github.com/edwvee/exiffix
github.com/alecthomas/chroma/v2
github.com/tklauser/numcpus
github.com/shirou/gopsutil/v3/mem
github.com/tklauser/go-sysconf
os/user
net
github.com/shirou/gopsutil/v3/cpu
github.com/alecthomas/chroma/v2/styles
github.com/alecthomas/chroma/v2/lexers
archive/tar
github.com/shirou/gopsutil/v3/net
vendor/golang.org/x/net/http/httpproxy
net/textproto
github.com/google/uuid
crypto/x509
github.com/seancfoley/ipaddress-go/ipaddr
vendor/golang.org/x/net/http/httpguts
mime/multipart
github.com/shirou/gopsutil/v3/process
crypto/tls
net/http/httptrace
net/http
kitty/tools/utils
kitty/tools/tty
kitty/tools/utils/base85
kitty/tools/utils/paths
kitty/tools/rsync
kitty/tools/wcswidth
kitty/tools/crypto
kitty/tools/tui/shell_integration
kitty/tools/utils/humanize
kitty/tools/utils/style
kitty/tools/cli/markup
kitty/tools/tui/sgr
kitty/tools/tui/loop
kitty/tools/cli
kitty/tools/config
kitty/tools/tui/shortcuts
kitty/tools/cmd/mouse_demo
kitty/tools/utils/shm
kitty/kittens/query_terminal
kitty/kittens/hyperlinked_grep
kitty/tools/tui/readline
kitty/kittens/show_key
kitty/tools/tui
kitty/tools/utils/images
kitty/tools/tui/subseq
kitty/kittens/clipboard
kitty/tools/unicode_names
kitty/tools/tui/graphics
kitty/kittens/ask
kitty/kittens/hints
kitty/tools/cmd/run_shell
kitty/tools/cmd/show_error
kitty/tools/cmd/update_self
kitty/tools/cmd/edit_in_kitty
kitty/tools/cmd/at
kitty/tools/themes
kitty/kittens/unicode_input
kitty/kittens/icat
kitty/tools/cmd/benchmark
kitty/kittens/choose_fonts
kitty/kittens/themes
kitty/kittens/ssh
kitty/kittens/transfer
kitty/tools/cmd/pytest
kitty/kittens/diff
kitty/tools/cmd/tool
kitty/tools/cmd/completion
kitty/tools/cmd
BUILD_EXIT=0
```
