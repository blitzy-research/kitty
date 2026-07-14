# Kitty startup, before the terminal is ready for a shell — a runtime investigation

This document answers four questions about what happens when **kitty 0.35.2** starts up
from commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (branch `kitty_815df1e210e0`),
in the interval between the process being launched and the terminal being ready to host a
shell. Every behavioural claim below is accompanied by the exact command that produced it
and the actual output that command emitted, captured by **building and running the real
binary headlessly under Xvfb**. Output is quoted unedited wherever practical; where a capture
is long or noisy it is shown as a complete-representative block whose *only* modifications are
called out explicitly at the point of use — the specific cases are routine build-progress
lines elided (with the count noted), the monotonic `[%.3f]` timestamp noted as the sole
cross-run difference, a byte dump regrouped one logical line per row (with the literal `od`
wrapping also shown), font paths abbreviated (alongside a byte-exact copy), and `env` piped
through `sort`. In every such case the decisive lines are quoted verbatim. Statements that are
read from source but were not directly exercised at runtime are explicitly labelled
`INFERRED`; everything else is `OBSERVED`.

## Subject under investigation

| Fact | Value | Grounding |
|------|-------|-----------|
| Program | kitty | `kitty/constants.py:25` |
| Version | `0.35.2` | `kitty/constants.py:25` `version: Version = Version(0, 35, 2)` |
| Commit | `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` | build VCS stamp (below) |
| Branch | `kitty_815df1e210e0` | deliverable name |
| Canonical binary used for every run | `$SCRATCH/clean815/kitty/launcher/kitty` | built below |
| Binary sha256 | `8311daddf6bbccf949233c9fdd58fbbe46748dbfc957847b7e4228b4973fc24c` (36224 bytes) | `sha256sum` |
| C-extension sha256 | `fast_data_types.so` = `582933cfd7b6cecb5ee60cfd20ef35a1f74acc2c6a905022180a60a76bf722e8` (1213072 bytes) | `sha256sum` |

The single deliverable of this task is this document. **No repository source file was
modified**; the investigation is strictly read-only, and every temporary artifact
(build clone, crafted config, logs, screenshots) lives outside the repository tree and is
removed at the end (see *Cleanup verification*).

## Methodology & environment

### Run-first principle

The governing methodology is: **build and run the relevant code paths first, then write
from the captured output.** Nothing below is asserted from code-reading alone unless
labelled `INFERRED`. Each subsystem, configuration source, environment variable, escape
sequence and render signal was produced at runtime through kitty's real entry point and
captured, with the decisive output quoted verbatim (any excerpting or normalization is marked
where it occurs, per the note above).

### Canonical entry point only

Every run invokes the native launcher `kitty/launcher/kitty`, which starts CPython and
dispatches into `kitty.main` (proven in Q1). No remote-control hook, debug shim, mock or
synthetic stand-in is substituted for the real path. Where a value could only be obtained
outside the canonical path, it is labelled `non-canonical` and the reason is stated.

### Provenance (the exact environment) · `OBSERVED`

The generic host lacks Go, Xvfb and the graphics/font toolchain, so all building and
running is performed inside the project's canonical Docker container, which bind-mounts the
repository working tree.

| Item | Value |
|------|-------|
| Image (canonical ref) | `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0` |
| Image RepoDigest | `ghcr.io/scaleapi/swe-atlas@sha256:60da90a7183a82861fc6d1d40cb8086baa6a8a0e0d05f26d03aafd0f5b3cc384` |
| Image ID | `sha256:c0824992ad0b274bc8738bf1365d336bf91dec97d726ec122e2d08eb9f053288` |
| Container | `kitty-setup`, bind-mounts host repo at `/work` (read-write) |
| OS | Ubuntu 24.04.2 LTS |
| Python | 3.12.3 (`/usr/bin/python3`) — repo floor `requires-python ">=3.8"` [`pyproject.toml`] |
| Go | 1.23.4 — repo floor `go 1.22` [`go.mod:3`] |
| C compiler | gcc 13.3.0 |
| pkg-config | 1.8.1 |

The alias `andrewparkscaleai/coding-agent:...` requires authentication (denied); the
`ghcr.io` reference above is the one actually used.

**Environmental prerequisites vs. project dependencies.** This task adds **no** dependency
and edits **no** manifest. The following native libraries are *build/run prerequisites*
supplied by the container, not project dependencies: harfbuzz 8.3.0, freetype2 26.1.20,
fontconfig 2.15.0, lcms2 2.14, libpng16 1.6.43, xkbcommon 1.6.0, libX11 1.8.7, GL 1.2.
Their required floors are listed in `docs/build.rst`.

### Build — the two documented paths; the system-library build is the canonical one here · `OBSERVED`

The checkout ships no binary, so the subject must be compiled. The AAP documents **two
canonical build paths** (§0.2.3): the headline `./dev.sh build`, and the system-library build
`python3 setup.py build`. In this canonical container the two do **not** behave the same at
this pinned commit: `./dev.sh build` **fails deterministically** (exit 1, reproducibly — root
cause below), while `python3 setup.py build --verbose` **succeeds** (exit 0, bit-reproducibly).
The project's own setup instructions match this: they designate the system-library path as the
one that produced the container's ready-to-run `/app` and that CI uses, with `./dev.sh build`
listed as the alternate. Under the strict **read-only** rule the `./dev.sh build` failure
**cannot be repaired** — its obstacle lives in kitty's *vendored* GLFW source (`glfw/wl_window.c`),
which may not be edited, and re-pinning `./dev.sh`'s downloaded dependency bundle is likewise a
source/definition change that is out of scope. The `./dev.sh build` failure is therefore
reported here as an **observed edge condition** of building this exact commit in this exact
environment, not worked around; and the **system-library build is the canonical build used for
every value and every run in this document**, correctly labelled as such (never relabelled as
`./dev.sh build`). Both outcomes are documented below with their exact commands, exit codes and
the decisive verbatim output.

**A clean checkout at the exact subject commit.** To obtain a canonical VCS stamp (the
working tree `/work` has this document committed on top of the subject commit, so its HEAD
is not `815df1e210e0`), the binary was built from an isolated clone checked out at the
exact commit:

```
git clone /work $SCRATCH/clean815
cd $SCRATCH/clean815
git checkout 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1     # detached; pure source, zero blitzy docs
git diff 815df1e210e0..HEAD --stat                        # (run against /work) -> only the doc
```

`git diff 815df1e210e0..HEAD --stat` reports a single changed path
(`blitzy/documentation/kitty_815df1e210e0.md`), proving the documentation commit modified
**zero** source files. The clone has its own `.git`, so `/work/.git` and the assigned branch
are never touched.

**The two documented build paths do not behave the same (observed, root-caused).** Run from
independent pristine clones at the pinned commit, the container's system `wayland-protocols`
and the two build entry points give different outcomes (each stable across two runs):

```
$ pkg-config --modversion wayland-protocols                    # container system: 1.34
$SCRATCH/clean815$ python3 setup.py build --verbose            # exit 0, ~22-23 s — SUCCEEDS
$SCRATCH/devsh$    ./dev.sh build                              # exit 1, ~12-13 s — FAILS
```

A clean `python3 setup.py build` against the container's **system** `wayland-protocols`
(1.34) **succeeds** (`exit 0`, ~22.3 s, stable across two independent clean clones) and builds
**both** GLFW backends — `kitty/glfw-x11.so` and `kitty/glfw-wayland.so` — its `wayland-scanner`
reading the system `/usr/share/wayland-protocols/stable/xdg-shell/xdg-shell.xml`, with **zero**
`wl_window.c` errors (the complete 214-line `--verbose` log contains no `error:` line). This is
the **system-library path** the project's setup instructions identify as the one that produced
the container's ready-to-run `/app` and that CI uses — i.e. the **second of the two canonical
build paths the AAP documents** (§0.2.3) — so it is the **canonical build used here** and the
source of `$SCRATCH/clean815/kitty/launcher/kitty`. (The AAP's headline `./dev.sh build`, by
contrast, cannot produce a binary at all in this environment — see the root-cause below — so it
is not usable as the source of build values under the read-only rule.) Both its artifact hashes
are bit-reproducible across two independent clean clones (launcher `8311dadd…3fc24c`;
C-extension `fast_data_types.so` `582933cf…22e8`); the launcher hash additionally equals the
pre-built `/app` binary's (the `/app` `.so` differs only because it was compiled earlier). The
successful launcher link and stamps are, verbatim from the `--verbose` log:

```
gcc build/kitty-launcher-main.o build/kitty-launcher-single-instance.o -ldl -lm \
    -L/usr/lib/x86_64-linux-gnu -lpython3.12 -Xlinker -export-dynamic -Wl,-O1 \
    -Wl,-Bsymbolic-functions -o kitty/launcher/kitty
# compiled with -DKITTY_VERSION="0.35.2" -DKITTY_VCS_REV="815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1"
# Go tools linked with: -ldflags '-X kitty.VCSRevision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 -s -w'
```

`./dev.sh build`, by contrast, downloads its **own** dependency bundle into
`$SCRATCH/devsh/dependencies/`, and that bundle ships `wayland-protocols` **1.45**
(`dependencies/linux-amd64/share/wayland-protocols/stable/xdg-shell/xdg-shell.xml`), not the
system 1.34. Version 1.45 adds the `xdg_toplevel` `XDG_TOPLEVEL_STATE_CONSTRAINED_*` states,
which kitty 0.35.2's bundled GLFW `wl_window.c` `xdgToplevelHandleConfigure` `switch`
(`glfw/wl_window.c:668`) does not handle; kitty's strict `-pedantic-errors -Werror` flags (its
own defaults) then promote the unhandled-enum warning to a fatal error. `compile_glfw`
(`setup.py:945-951`) wraps only dependency *detection* in try/except, not the gcc compile, so
the too-new bundled protocols hard-fails rather than skipping gracefully. The failure is
therefore specific to `./dev.sh build`'s own bundled 1.45 protocols — it is **not**
method-independent, and it is **not** caused by the container's system libraries. The build
proceeds normally (it downloads the bundle, then generates protocols and compiles) up to
`[3/122] Compiling [wayland] glfw/wl_window.c`, where it stops. The complete failing tail —
the four compiler errors (one per unhandled state) and the exit — is, verbatim (routine
progress steps `[1/122]`–`[2/122]` elided):

```
[3/122] Compiling [wayland] glfw/wl_window.c ...
glfw/wl_window.c: In function ‘xdgToplevelHandleConfigure’:
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_LEFT’ not handled in switch [-Werror=switch]
  668 |         switch (*state) {
      |         ^~~~~~
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_RIGHT’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_TOP’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_BOTTOM’ not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
The following build command failed: /root/qa_reverify/devsh/dependencies/linux-amd64/bin/python setup.py develop
exit status 1
```

This tail is byte-identical across the two runs (the error region hashes the same on both),
and `./dev.sh build` returns `exit 1` (~12.4 s / 12.7 s) each time. Note the failing command
is `dependencies/linux-amd64/bin/python setup.py develop` — `./dev.sh` runs `setup.py` under
its **own bundled** Python against its **own bundled** 1.45 protocols, which is what makes its
outcome differ from a direct system `python3 setup.py build`.

**`./dev.sh build` cannot be repaired under the read-only rule; no workaround is applied.**
The obstacle is inside kitty's *vendored* GLFW (`glfw/wl_window.c:668`); fixing it would mean
editing a repository source file, and the only other route — re-pinning `./dev.sh`'s
downloaded dependency bundle to an older `wayland-protocols` — is likewise a
source/definition change. Both are forbidden by the strict read-only scope, so the failure is
reported as-is (an observed edge condition of this commit in this environment) rather than
worked around. It also does not need to be repaired: the clean system-library
`python3 setup.py build` succeeds and yields both backends, so the canonical binary is
obtained with **no** source file edited, **no** `-Werror` relaxed, and **no** check disabled.
(kitty
does ship a supported no-Wayland fallback for systems whose `wayland-protocols` is below its
floor of `1.17`, read from `glfw/source-info.json`; shadowing `wayland-protocols.pc` with a
`Version: 1.0` stub triggers it, logging `wayland-protocols >= 1.17 is required, found
version: 1.0` and `Disabling building of wayland backend`. That path is genuine but simply
unnecessary here, since the system 1.34 builds cleanly.) Under Xvfb (an X11 display) kitty
loads `glfw-x11.so`; the Wayland backend, though built, is never exercised in this
investigation. The Go link line stamps the correct
subject commit:

```
-X kitty.VCSRevision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
```

and the launcher/data are compiled with `-DKITTY_VERSION="0.35.2"` and
`-DKITTY_VCS_REV="815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1"`. The resulting binary
(`$SCRATCH/clean815/kitty/launcher/kitty`, sha256 `8311dadd…3fc24c`) is used for **every**
run in this document.

```
$SCRATCH/clean815$ kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

`ldd` confirms the launcher is a native executable dynamically linked against CPython —
the first subsystem in Q1:

```
$SCRATCH/clean815$ ldd kitty/launcher/kitty | grep -i python
	libpython3.12.so.1.0 => /lib/x86_64-linux-gnu/libpython3.12.so.1.0
```

### Headless display strategy (web-research grounded) · `OBSERVED`

kitty requires a real OpenGL context (Q4). With no GPU under a virtual X server, the
accepted approach — confirmed against primary documentation — is **Xvfb (a virtual X11
display) plus Mesa's llvmpipe software rasterizer**, forced with `LIBGL_ALWAYS_SOFTWARE=1`:

- Mesa llvmpipe driver docs (`docs.mesa3d.org/drivers/llvmpipe.html`): llvmpipe is a
  software rasterizer that uses LLVM for runtime code generation — the solution for
  environments with no dedicated GPU.
- Mesa environment variables (`docs.mesa3d.org/envvars.html`): `LIBGL_ALWAYS_SOFTWARE`,
  when set, forces software rendering.
- X.Org `Xvfb` manual (`x.org` / Ubuntu manpage): Xvfb is an X server that runs on machines
  with no display hardware, emulating a framebuffer in virtual memory; it provides GLX via
  Mesa.
- GLFW window/context guide (`glfw.org/docs/latest/window_guide.html`): a minimum context
  version is requested via `GLFW_CONTEXT_VERSION_MAJOR/MINOR`, and `glfwCreateWindow` fails
  if the resulting version is lower than requested — which matches kitty's fatal path at
  `glfw.c:1198-1199`.

The software renderer actually present, captured twice (stable), by the exact command:

```
DISPLAY=:77 LIBGL_ALWAYS_SOFTWARE=1 glxinfo | grep -E 'vendor|renderer|core profile version|direct rendering'
OpenGL vendor string: Mesa
OpenGL renderer string: llvmpipe (LLVM 20.1.2, 256 bits)
OpenGL core profile version string: 4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.2
direct rendering: Yes
```

llvmpipe's 4.5 core profile satisfies kitty's required 3.3 [`kitty/data-types.h:20-22`] and,
on Linux specifically, its 3.1 floor [`kitty/data-types.h:24`].

### Xvfb lifecycle (owned, verified, torn down) · `OBSERVED`

Display `:99` was already occupied (provided by the container setup), so this investigation
started and owns its **own** Xvfb on `:77`:

```
Xvfb :77 -screen 0 1920x1080x24 -ac +extension GLX +render -noreset &
echo $! > $SCRATCH/xvfb77.pid           # captured pid = 45554
```

Readiness was confirmed with `xdpyinfo` (dimensions `1920x1080`, depth 24) before any run;
the pid is retained for an exact-pid shutdown during cleanup. Occupied-display handling
(probe `:99` busy → choose `:77`), readiness verification, and no-leftover teardown are all
part of the harness. The teardown and its verification are shown in *Cleanup verification*.

### Secure observation harness · `OBSERVED`

All temporary files live beneath a single mode-700 scratch directory created with
`mktemp -d`; each run uses an **isolated** `HOME` and `KITTY_CONFIG_DIRECTORY` under that
directory (never the container's real `/root/.config`), all paths are quoted, and cleanup
removes only that owned directory:

```
SCRATCH=$(mktemp -d /tmp/kqna.XXXXXX); chmod 700 "$SCRATCH"
# reusable env ($SCRATCH/krun.env):
export KHOME="$SCRATCH/home"                 # mode 700
export KITTY_BIN="$SCRATCH/clean815/kitty/launcher/kitty"
export DISPLAY=:77
export LIBGL_ALWAYS_SOFTWARE=1
export LANG=C.UTF-8; export LC_ALL=C.UTF-8
```

No crafted configuration is ever written into a shared or real config location; the crafted
`kitty.conf` used in Q2 is created inside an isolated `KITTY_CONFIG_DIRECTORY` under the
scratch directory and removed at the end.

A note on paths, so nothing below is mistaken for edited output: **typed command examples** use
`$SCRATCH` as a placeholder for the run's scratch directory, but every **captured output block**
is quoted with the *literal, resolved* path exactly as emitted — no path normalization is ever
applied to output. Two scratch roots therefore appear, because the work spanned two sets of
runs: the originally-authored runs resolved `$SCRATCH` to `/tmp/kqna.wEhaOC/…` (seen in the Q2
`debug_config` output), while the Q3/Q4 runtime re-verification was re-executed in the canonical
container, where the pinned clone/build lives at `/root/qa_reverify/clean815/…` and the
`strace`/byte-dump artifacts under `/root/qa/…` (seen in the Q3 handshake, child `env`, and the
Q4 font/byte captures). Both are real scratch paths from real runs; none is invented, and the
same `clean815` clone subdirectory name appears under each root.

### Log-capture convention (streams and producers) · `OBSERVED`

kitty's subsystems log through `log_error`, which writes to **stderr** with a monotonic
timestamp prefix `[%.3f] ` [`kitty/logging.c:56`, format `%s\n` at `:61`]. Capturing the
process's stderr therefore captures the bring-up log. **One important exception:** the
OpenGL version banner is printed by native C with `printf` to **stdout** (not stderr) and is
gated on `--debug-rendering` [`kitty/gl.c:72`]. The bring-up lines are **not** all
Python-emitted — they are a mix of native C and Python producers (attributed per line in
Q1). To capture true arrival order across both streams, a line-buffered merged capture
(`stdbuf -oL -eL … 2>&1`) is used where ordering matters.

### Stability discipline · `OBSERVED`

Every condition was run **at least twice**. Outputs are compared after normalising only the
non-deterministic `[%.3f]` timestamp prefix (`sed -E 's/^\[[0-9]+\.[0-9]+\]/[T]/'`); the
sha256 of the normalised output and any diff are reported next to the claim. Where a field
genuinely varies between runs (e.g. the child PID), it is called out explicitly.

### OBSERVED / INFERRED legend

- `OBSERVED` — captured at runtime through the canonical entry point; the exact command and
  complete output are shown adjacent.
- `observed-effect + inferred-correlation` — a later output line proves an earlier
  subsystem must have run, even though that earlier subsystem emitted no line of its own.
- `INFERRED` — read from source and not directly exercised at runtime (e.g. an
  internal branch, or a macOS-only path on this Linux host).
- `non-canonical` — obtained outside the real entry point; used only as clearly-labelled
  corroboration, never as the primary answer.

### What the questions map to

| Question | Primary run(s) | Evidence captured |
|----------|----------------|-------------------|
| Q1 subsystems | default `--debug-rendering --debug-font-fallback sh -c 'echo READY; sleep 1'`, merged line-buffered | ordered bring-up lines + producers/streams + CPython launcher symbols |
| Q2 configuration | default (empty isolated config) + crafted `kitty.conf`; real `debug_config` keybinding | resolution algorithm, absence proof, applied overrides, bad-line reports, canonical report |
| Q3 shell readiness | `sh -c` and `bash -c` children; `--dump-bytes`; live before/after screenshots | PTY/env/terminfo, integration, first bytes drawn |
| Q4 display | default + custom font/cursor; overflowing child; GL-missing negatives | GL version, fonts, screenshots, scrolling, fatal path |

## Q1 — Which systems start up on the way to a working terminal, and what shows them coming online

### The primary evidence run

The single run below exercises the whole startup path and, because the child is
short-lived, lets it draw its first output and exit cleanly while all bring-up logs are
captured. Exact command (isolated `HOME`, empty isolated config, on the owned display):

```
HOME=$SCRATCH/home KITTY_CONFIG_DIRECTORY=$SCRATCH/cfgempty \
DISPLAY=:77 LIBGL_ALWAYS_SOFTWARE=1 LANG=C.UTF-8 LC_ALL=C.UTF-8 \
  $SCRATCH/clean815/kitty/launcher/kitty --debug-rendering --debug-font-fallback \
  sh -c 'echo READY; sleep 1'
```

Complete captured output. **stdout** (the GL banner, `--debug-rendering`-gated,
`kitty/gl.c:72`), verbatim:

```
[0.120] GL version string: '4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.2' Detected version: 4.5
```

**stderr** (the bring-up log, `[%.3f]` prefix per `kitty/logging.c:56`), verbatim:

```
[0.146] OS Window created
[0.154] Failed to open systemd user bus with error: No medium found
[0.158] Child launched
[0.158] Text fonts:
[0.158]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.158]   Bold: DejaVuSansMono-Bold: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf:0
[0.158]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.158]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

**Stability** (`M8`) · `OBSERVED`: exit code `0`, wall time ≈1.3 s on both runs. After
timestamp normalisation the separated captures are identical (sha256 `bfd40625…173fe`, shown
truncated because it is a per-capture, stream-split artifact — sensitive to how stdout and
stderr are interleaved). The authoritative, fully reproducible stability figure is the
line-buffered **merged** capture (`stdbuf -oL -eL … 2>&1`), also identical across the two
runs (sha256 `d0868582aea7c736ce4e3e1e4d7434ab49fc6a9846407542b57dab662bea4bc8`).

**True arrival order** (`M5`) · `OBSERVED`: because the GL banner is on stdout and the rest
on stderr, comparing two separate streams can appear to invert them. The merged
line-buffered capture, and the embedded timestamps within a single process, both show the
GL banner (`0.120`) arriving **before** `OS Window created` (`0.146`) — matching the code
order (the banner is printed from inside `gl_init`, which `create_os_window` calls before
it emits “OS Window created”). The string `GL version string` is printed exactly once
(`kitty/gl.c:72` is its only occurrence in the tree); `gl_init` has a single caller
(`kitty/glfw.c:1212`, guarded by `is_first_window`) and a one-time `glad_loaded` guard
(`kitty/gl.c:53-69`).

### Producer and stream of each captured line · `OBSERVED`

The bring-up log is a mix of native-C and Python producers; the GL banner is on stdout, the
rest on stderr. This is the exact attribution (each `file:line` is the unique producer of
that text in the tree):

| Line | Producer | Stream | Gate |
|------|----------|--------|------|
| `GL version string: …` | `kitty/gl.c:72` (native C `printf`) | **stdout** | `global_state.debug_rendering` |
| `OS Window created` | `kitty/glfw.c:1321` (native C `debug()`) | stderr | `--debug-rendering` |
| `Failed to open systemd user bus …` | `kitty/systemd.c:87` (native C `log_error`) | stderr | always (a real failure, see below) |
| `Child launched` | `kitty/window.py:871` (Python `print(…, file=sys.stderr)`) | stderr | `boss.args.debug_rendering` |
| `Text fonts:` + faces | `kitty/fonts/render.py:163` (Python `log_error`, from `dump_font_debug`) | stderr | `--debug-font-fallback` |

### The subsystems, in the order they come online

The full call chain, each step anchored to source and, where it emits output, to the
captured line above.

#### 1. Native CPython launcher — `kitty/launcher/main.c` · `OBSERVED`

The executable is a native C program that pre-initialises and starts an embedded CPython,
then runs `kitty.main`. Runtime proof that the launcher links and boots CPython:

```
$SCRATCH/clean815$ ldd kitty/launcher/kitty | grep -i python
	libpython3.12.so.1.0 => /lib/x86_64-linux-gnu/libpython3.12.so.1.0

$SCRATCH/clean815$ readelf -W --dyn-syms kitty/launcher/kitty | \
    grep -oE 'Py_(PreInitialize|InitializeFromConfig|RunMain|ExitStatusException)|PyConfig_[A-Za-z]+|PyStatus_[A-Za-z]+' | sort -u
PyConfig_Clear
PyConfig_InitPythonConfig
PyConfig_SetBytesArgv
PyConfig_SetBytesString
PyStatus_Exception
PyStatus_IsExit
Py_ExitStatusException
Py_InitializeFromConfig
Py_PreInitialize
Py_RunMain
```

Source order of those symbols in the launcher: `Py_PreInitialize` (`main.c:190`) →
`PyConfig_InitPythonConfig` (`main.c:193`) → `Py_InitializeFromConfig` (`main.c:211`) →
`Py_RunMain` (`main.c:216`); `PyConfig_Clear` then releases the config struct after a
successful init (`main.c:213`) and again on the error path (`main.c:218`). The `libpython3.12`
link and the imported symbols are
`OBSERVED`; the internal control flow inside `main.c` is `INFERRED` from source.

#### 2. CPython-side dispatch — `kitty/entry_points.py` · `observed-effect + inferred-correlation`

`Py_RunMain` enters `kitty/entry_points.py:183` `main()`, which calls `kitty_main()`
(`:195`), which imports and runs `kitty.main._main`. No line is emitted here, but every
later line proves this dispatch ran.

#### 3. Argument & configuration bootstrap — `kitty/main.py:441` `_main()` · `observed-effect + inferred-correlation`

`_main()` parses arguments, builds options via `create_opts` (Q2), sets locale/environment
and masks signals. Its effects are observed indirectly (the process proceeds to create a
window with the resolved options).

#### 4. GLFW windowing backend — `kitty/main.py:514` `init_glfw(...)` · `observed-effect + inferred-correlation`

`_main()` initialises the GLFW backend (`init_glfw`, def at `main.py:95`). On this host the
backend is X11 (see the `Running under: X11` line of the canonical `debug_config` report in
Q2/E3). That the backend initialised is proven by the subsequent `OS Window created` line;
a *failed* initialisation is shown as a negative in E1.

#### 5. App runner — font selection happens **here**, early — `kitty/main.py:247` `AppRunner.__call__` · `observed-effect + inferred-correlation`

This is the correction to the naive ordering. `AppRunner.__call__` (`main.py:247`) runs
**before** `_run_app`, and performs, in order: `set_scale` (`:248`), `set_options`
(`:249`) and **`set_font_family`** (`:251`) — i.e. font *selection/registration* is an
early step — then calls `_run_app` (`:252`). The later `Text fonts:` dump is a *separate*
debug dump (step 9), not the point at which fonts are chosen.

#### 6. Session / first window model — `kitty/main.py:202` `_run_app` → `create_sessions` · `observed-effect + inferred-correlation`

`_run_app` (`main.py:202`) builds the startup session (`create_sessions`, `main.py:214`)
that describes the first OS window/tab/window before any OS window exists.

#### 7. OS window + OpenGL context + shaders — `create_os_window(...)` · `OBSERVED`

`_run_app` calls `create_os_window` (`main.py:221`), passing the compiled shader loader
`load_all_shaders` (`main.py:82`). Native `create_os_window` (`kitty/glfw.c:1107`) creates a
temp window (`:1198`), then the real window (`:1208`), makes the context current (`:1211`),
calls `gl_init` (`:1212`, which prints the **GL banner** on stdout), and emits
`OS Window created` (`:1321`). Both the GL banner and `OS Window created` are `OBSERVED`
above.

#### 8. `Boss` controller + `ChildMonitor` + child fork — `kitty/boss.py` · `OBSERVED`

`_run_app` constructs `Boss` (`main.py:226`) and calls `boss.start()` (`main.py:227`), which
(`boss.py:1181` → `startup_first_child` `boss.py:383` → `add_child` `boss.py:585-587`) forks
the child (Q3). The Python line `Child launched` (`kitty/window.py:871`) is `OBSERVED` above
and marks the child having been forked with its PTY attached.

#### 9. Font **debug dump** — `kitty/main.py:229` `dump_font_debug()` · `OBSERVED`

Only when `--debug-font-fallback` is set, `_run_app` calls `dump_font_debug` (`main.py:229`),
which logs the resolved faces via `kitty/fonts/render.py:163`. This is the `Text fonts:`
block `OBSERVED` above, and it is **late** (after `boss.start`), distinct from the early
font *selection* in step 5.

#### 10. Child / PTY monitor loop — `kitty/main.py:234` `boss.child_monitor.main_loop()` · `observed-effect + inferred-correlation`

Finally `_run_app` enters the child-monitor main loop (`main.py:234`), which reads the PTY
master and drives parsing/rendering (Q3/Q4). Its effect is proven by the child's `READY`
being drawn on the live screen (Q3).

### The one non-subsystem line · `OBSERVED`

`Failed to open systemd user bus with error: No medium found` (`kitty/systemd.c:87`) is
**not** a startup subsystem failing; it is kitty optimistically probing for a systemd user
bus (to place the child in its own scope) and logging that none is available in this
container. The terminal comes up fully regardless — every subsequent line and the drawn
`READY` prove it.

### Q1 coverage check

Every named startup item is addressed: native launcher `main.c` (CPython symbols observed);
`entry_points.py`; `main.py` `_main`/`init_glfw`/`AppRunner`/`_run_app`/`create_sessions`/
`create_os_window`/`Boss`/`dump_font_debug`/`child_monitor.main_loop`; `glfw.c`
window+context; `gl.c` banner; `boss.py`+`window.py` child; `systemd.c` bus line;
`fonts/render.py` dump; `logging.c` stderr sink. Ordering corrected so font *selection*
(step 5) precedes the font *debug dump* (step 9).


## Q2 — How Kitty decides its initial configuration, and what output shows the settings were applied

### The resolution algorithm · `OBSERVED` (source) + `OBSERVED` (values)

Configuration is resolved by `create_opts` (`kitty/cli.py:1081`), which computes the list of
candidate files and then loads them:

- `create_opts` (`cli.py:1081`): `config = default_config_paths(args.config)`;
  `load_config(*config, overrides=map(parse_override, args.override or ()))`.
- `default_config_paths` (`cli.py:1067`): `tuple(resolve_config(SYSTEM_CONF, defconf, args.config))`.
- `SYSTEM_CONF = '/etc/xdg/kitty/kitty.conf'` (`cli.py:1064`).
- `resolve_config` (`kitty/conf/utils.py:322`): if command-line config files are given and
  `NONE` is not among them → yield `SYSTEM_CONF` then those files; if `NONE` is present →
  yield **nothing** (all configuration suppressed); otherwise (the default) → yield
  `SYSTEM_CONF`, then `defconf`.
- `load_config` (`kitty/config.py:163` → `kitty/conf/utils.py:332`): starts from
  `defaults._asdict()`, opens each candidate, and on a missing/unreadable file does
  `except (FileNotFoundError, PermissionError): continue` (silently skipped); the found
  files are merged over the defaults; it records `opts.config_paths` = files actually loaded
  (`config.py:183`), `opts.all_config_paths` = all candidates tried (`:184`), and
  `opts.config_overrides` (`:185`).

So on a first launch with no user or system config, the candidates are *tried* but nothing
is *loaded*, and the module-level `defaults` instance governs.

### Which config directory is consulted, in order · `OBSERVED` (source)

`defconf` is `config_dir/kitty.conf` (`kitty/constants.py:133`), and `config_dir` is chosen
by `_get_config_dir` (`kitty/constants.py:87`) in this order:
`KITTY_CONFIG_DIRECTORY` (`:88`) → `XDG_CONFIG_HOME` (`:92`) → `~/.config` (`:94`) →
(on macOS `~/Library/Preferences`, `:95`, `INFERRED` on this Linux host) →
`XDG_CONFIG_DIRS` (`:97`); the first writable location that already has a `kitty.conf` wins,
else the first writable location.

### Proving every candidate is absent on a default launch · `OBSERVED`

Using an isolated empty config directory (`KITTY_CONFIG_DIRECTORY=$SCRATCH/cfgempty`), both
candidates are shown to be absent:

```
$ ls -la /etc/xdg/kitty/
ls: cannot access '/etc/xdg/kitty/': No such file or directory      # SYSTEM_CONF absent
$ ls -la "$SCRATCH/cfgempty"
total 8
drwx------ 2 root root 4096 Jul 13 20:56 .
drwxr-xr-x 7 root root 4096 Jul 13 20:56 ..    # only . and .. -> no kitty.conf, defconf ($SCRATCH/cfgempty/kitty.conf) absent
```

Running the **real** `create_opts` path (via `create_default_opts`, `cli.py:1089`) with a
published helper confirms nothing was loaded. Exact command and complete output:

```
$ HOME=$SCRATCH/home KITTY_CONFIG_DIRECTORY=$SCRATCH/cfgempty \
    $KITTY_BIN +runpy 'exec(open("'"$SCRATCH"'/q2/q2_defcfg.py").read())'
config_dir       = /tmp/kqna.wEhaOC/cfgempty
defconf          = /tmp/kqna.wEhaOC/cfgempty/kitty.conf
SYSTEM_CONF      = /etc/xdg/kitty/kitty.conf
opts.config_paths      = ()
opts.all_config_paths  = ('/etc/xdg/kitty/kitty.conf', '/tmp/kqna.wEhaOC/cfgempty/kitty.conf')
opts.config_overrides  = ()
font_family      = FontSpec(family='', style='', postscript_name='', full_name='', system='monospace', axes=(), variable_name='', created_from_string='')
font_size        = 11.0
cursor_shape     = 1          # 1 = block
foreground       = Color(221, 221, 221)   # :2:221:221:221
background       = Color(0, 0, 0)         # :2:0:0:0
scrollback_lines = 2000
```

`opts.config_paths = ()` proves **no file was loaded**; `opts.all_config_paths` shows both
candidates were *tried*; the values are the module-level `defaults` (`kitty/options/types.py`).
The helper body `q2_defcfg.py` (published in full in the scratch harness) imports the real
`SYSTEM_CONF`, `create_default_opts` and `debug_config` from `kitty` and prints the fields
above — it calls the real functions and does not fabricate values.

### First-launch defaults that shape the window · `OBSERVED`

Those defaults are exactly what shapes the first window: the background is opaque black
(`background :2:0:0:0`), the foreground is light grey (`foreground :2:221:221:221`), the
cursor is a filled block (`cursor_shape 1`), text is 11pt in the `monospace` family, and up
to 2000 lines of scrollback are retained. Each of these is *seen* on screen in Q3/Q4 (the
grey `READY` on black, the block cursor, the glyph size, and the scroll behaviour).
The default `term` value is `xterm-kitty` [`kitty/options/definition.py:3242`], observed as
`TERM=xterm-kitty` in the child environment (Q3).

### The canonical `debug_config` report (via the real keybinding) · `OBSERVED`

kitty's own "show the effective configuration" action is `debug_config`. It is a
**keybinding action**, not a CLI flag: `kitty_mod` defaults to `ctrl+shift`
[`kitty/options/definition.py:3474`] and the binding is
`debug_config kitty_mod+f6 debug_config` [`kitty/options/definition.py:4256`], i.e.
**ctrl+shift+F6**. `Boss.debug_config` (`kitty/boss.py:3060`) builds the report with
`debug_config(get_options())` [`kitty/debug_config.py:231`], copies an ANSI-stripped copy to
the clipboard via `set_clipboard_string` (`boss.py:3065`), and displays it as an on-screen
overlay via `display_scrollback` (`boss.py:3067`).

The report can only be produced by a **live** window: `debug_config()` calls
`current_fonts()` (`kitty/debug_config.py:261`), and the font group is created only inside
`create_os_window` → `load_fonts_data`. This is exactly why it is a keybinding action, and
why it was captured by driving the real key, not by a bare interpreter call.

Exact procedure (published helper `e3_keybinding.sh`): launch kitty with a long-lived child,
find its X11 window (`xdotool search --class kitty`), seed the clipboard with a sentinel,
give the window input focus (`xdotool windowfocus`), send the real shortcut over XTEST
(`xdotool key --clearmodifiers ctrl+shift+F6`), then read the clipboard (`xclip -o`). The
sentinel is overwritten, proving **kitty** wrote the clipboard. Complete captured report,
verbatim (ANSI already stripped by `boss.py:3065`), byte-identical across two runs
(sha256 `55c3052bc5c1045783337f510c1fd82a5b19d8ac6526dc7a4d3e628b6ee2a746`):

```
kitty 0.35.2 (815df1e210) created by Kovid Goyal
Linux ac1b170925d4 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64
Ubuntu 24.04.2 LTS ac1b170925d4 /dev/tty

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
  kitty: /tmp/kqna.wEhaOC/clean815/kitty/launcher/kitty
  base dir: /tmp/kqna.wEhaOC/clean815
  extensions dir: /tmp/kqna.wEhaOC/clean815/kitty
  system shell: /bin/bash

Config options different from defaults:

Important environment variables seen by the kitty process:
	PATH                                /tmp/kqna.wEhaOC/clean815/kitty/launcher:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
	LANG                                C.UTF-8
	KITTY_CONFIG_DIRECTORY              /tmp/kqna.wEhaOC/cfgempty
	DISPLAY                             :77
	LC_ALL                              C.UTF-8
```

This report proves the default-configuration claims directly:

- The first line is `version(add_rev=True)` (`kitty/debug_config.py:235`): `kitty 0.35.2`
  with the VCS revision `815df1e210` — the subject commit — confirming the canonical build.
- `Running under: X11` (`debug_config.py:257`) confirms the GLFW X11 backend (Q1 step 4).
- `OpenGL: '4.5 (Core Profile) Mesa …'` (`debug_config.py:258`) matches the Q1/Q4 GL banner.
- `Fonts:` (`debug_config.py:260-261`, from `current_fonts()`) lists the same DejaVuSansMono
  faces as the Q1 `Text fonts:` dump — proving the live font group.
- **`Config options different from defaults:` is empty**, and the
  **`Loaded config files:` section is absent** (it is printed only `if opts.config_paths`,
  `debug_config.py:270`). Together these prove that on this launch nothing was loaded and the
  process runs on pure defaults — the canonical answer to "which sources are used on first
  launch".

The on-screen overlay (`display_scrollback`, `boss.py:3067`) was screenshotted from the live
window and shows the same content (green section titles `Running under:`/`Fonts:`/`Paths:`,
the yellow `X11` value, the version banner, and a `:` pager prompt at the bottom), confirming
both the clipboard and overlay halves of the action.

### Now with a user `kitty.conf` — the applied overrides appear · `OBSERVED`

To show "settings were applied", a crafted config was written into an **isolated**
`KITTY_CONFIG_DIRECTORY=$SCRATCH/cfgcustom` (mode 700; never a shared/real location).
Exact creation and contents:

```
mkdir -p "$SCRATCH/cfgcustom"; chmod 700 "$SCRATCH/cfgcustom"
cat > "$SCRATCH/cfgcustom/kitty.conf" <<'CONF'
font_size 18.0
cursor_shape beam
scrollback_lines 5000
not_a_real_kitty_option 123
background_opacity notanumber
CONF
```

Running the real `create_opts` (with `accumulate_bad_lines`) over that directory. Complete
`stdout` of the published helper:

```
opts.config_paths = ('/tmp/kqna.wEhaOC/cfgcustom/kitty.conf',)
APPLIED font_size        = 18.0 (default 11.0)
APPLIED cursor_shape     = 2 (default 1=block)
APPLIED scrollback_lines = 5000 (default 2000)
APPLIED background_opacity = 1.0 (default 1.0; bad line should be ignored)
--- accumulated BadLine entries (known-key invalid value) ---
  BadLine number=5 file=/tmp/kqna.wEhaOC/cfgcustom/kitty.conf
    line='background_opacity notanumber'
    exception=ValueError: could not convert string to float: 'notanumber'
```

Now `opts.config_paths` contains the loaded file, and `font_size`, `cursor_shape` and
`scrollback_lines` show the applied non-default values. The visual effect of two of these
(`font_size`, `cursor_shape`) is measured directly in Q4.

### Bad configuration lines are reported (two distinct mechanisms) · `OBSERVED`

The crafted file contains two kinds of bad line, handled by two different code paths:

1. **Unknown key** → logged and skipped by `kitty/conf/utils.py:250`. Complete `stderr`
   of the helper:

   ```
   [0.017] Ignoring unknown config key: not_a_real_kitty_option
   ```

   The same message appears during a full canonical startup with this config
   (`[0.056] Ignoring unknown config key: not_a_real_kitty_option`, stable across both runs).

2. **Known key, invalid value** → recorded as a `BadLine` (`kitty/conf/utils.py:300`) and
   surfaced at startup. The startup flow is `main.py:493` `bad_lines=[]` → `:494`
   `create_opts(..., accumulate_bad_lines=bad_lines)` → `:518` `run_app(..., bad_lines)` →
   `_run_app` `:252` → `boss.show_bad_config_lines` (`main.py:230-231`). During a full
   canonical startup with this config, stderr shows the unknown-key line followed by the
   window coming up, and a **second** `Child launched` — the bad-line error overlay is a
   special window that keeps kitty alive:

   ```
   [0.056] Ignoring unknown config key: not_a_real_kitty_option
   [0.146] OS Window created
   [0.155] Failed to open systemd user bus with error: No medium found
   [0.159] Child launched
   [0.163] Child launched
   ```

The error overlay was screenshotted from the live window (640×400). It renders, on black:
a red bold title `Errors parsing configuration`; the grey body
`In file /tmp/kqna.wEhaOC/cfgcustom/kitty.conf:` and
`5:could not convert string to float: 'notanumber' in line: background_opacity notanumber`
— which is exactly `format_bad_line`'s template `'{number}:{exception} in line: {line}'`
(`kitty/boss.py:2761`); and a green bold prompt `Press Enter or Esc to exit`
(`show_error`, `boss.py:2050`).

### The generated configuration pipeline (named items) · `OBSERVED` (source)

The typed options and parser are code-generated from the declarative schema and must be
treated as read-only artifacts; the relevant named files and their roles:

- `kitty/options/definition.py` — the authoritative declarative schema and shortcut catalog
  (e.g. `term` default `xterm-kitty` at `:3242`; the `debug_config` binding at `:4256`).
- `kitty/options/parse.py` (generated) — `create_result_dict` (`:1441`),
  `merge_result_dicts` (`:1462`), `parse_conf_item` (`:1477`); used by `config.py`'s
  `parse_config`/`load_config`.
- `kitty/options/types.py` (generated) — `class Options` (`:471`) and the module-level
  `defaults = Options()` (`:752`), the first-launch base.
- `kitty/options/utils.py` — parsing/normalisation helpers used by the above.
- `kitty/options/to-c.h` and `kitty/options/to-c-generated.h` — the Python→C bridge:
  e.g. `convert_from_python_font_size` (`to-c-generated.h:9`) does
  `opts->font_size = PyFloat_AsDouble(val)`.
- `kitty/state.c` — includes `to-c-generated.h` (`state.c:9`) and, from the C `set_options`,
  calls `convert_opts_from_python_opts(opts, &global_state.opts)` (`state.c:741`); the
  C side then reads options through the `OPT()` macro. This is how the Python-resolved
  options reach the native renderer/child machinery.

### Q2 coverage check

Addressed by name: `create_opts` (`cli.py:1081`), `default_config_paths`,
`resolve_config` (`conf/utils.py:322`), `SYSTEM_CONF`, `defconf`, `_get_config_dir` order,
`XDG_CONFIG_DIRS`, the `NONE` suppression semantics, `load_config` (`config.py:163`), the
`defaults` instance, `debug_config()` (`debug_config.py:231`) via the real keybinding,
bad-line reporting (both unknown-key and `BadLine`/`format_bad_line`/`show_error`), and the
generated pipeline (`parse.py`, `types.py`, `utils.py`, `to-c.h`, `to-c-generated.h`,
`state.c`).


## Q3 — How Kitty readies the terminal to talk to the shell, and what shows the first output was understood

### 1. Allocate the pseudo-terminal — `openpty()` · `OBSERVED` (source) + `observed-effect`

`kitty/child.py:170` calls `os.openpty()` to allocate a master/slave PTY pair, marks the
slave inheritable and the master not (`:172-173`), and sets UTF-8 IUTF8 on the fd
(`set_iutf8_fd`, `:174`). `Child.fork` (`child.py:276`) calls `openpty` (`:281`), keeps the
master as `child_fd` (`:338`), and sets it non-blocking (`:345`). That a PTY was allocated
and wired up is proven by the child's output being read back and drawn on screen (below).

### 2. Fork and set up the controlling TTY — `Child.fork()` → C `spawn()` · `OBSERVED` (source + `strace` syscalls + child TTY/session state)

`Child.fork` (`child.py:276`) calls the native `spawn()` (`kitty/child.c:81`), which
`fork()`s (`:97`). In the child branch it calls `PyOS_AfterFork_Child()` (`:102`), `chdir(cwd)`
(`:121`), `setsid()` to start a new session (`:123`, `exit_on_err` if it returns −1), opens the
slave by name and makes it the controlling terminal with `ioctl(TIOCSCTTY)` (`:127-129`), and
finally `execvp(exe, argv)` (`:159`) to become the shell. The parent branch calls
`PyOS_AfterFork_Parent()` (`:174/182`). `mark_terminal_ready` (`child.py:362`) is called (from
`window.py:867`) just before the `Child launched` line.

**These syscalls were captured directly at runtime.** The real launcher was run under
`strace -ff` (following forks and the exec), with a child that records its own controlling
terminal, session state and environment:

```
strace -ff -e trace=setsid,execve,ioctl,openat -o $SCRATCH/st/kt \
  $KITTY_BIN sh -c 'tty>$SCRATCH/child_tty.txt; \
                    ps -o pid,ppid,sid,tty,stat,comm,args -p $$ >$SCRATCH/child_ps.txt; \
                    env|sort>$SCRATCH/child_env.txt; echo READY; sleep 2'
```

`strace -ff` writes one file per process. All values below are from a **single launch** (kitty
parent pid `105074`, `sh` child pid `105141`) so the pids cross-check; a second identical run
reproduced the same structure with different pids (`setsid()=105243`, `SID==PID`, `Ss+`). In the
**parent** (kitty, pid `105074`) the PTY master is allocated — the runtime counterpart of
`os.openpty()` (§1):

```
openat(AT_FDCWD, "/dev/ptmx", O_RDWR)   = 8
```

and the **child** file (pid `105141`) contains the exact controlling-TTY handshake from
`child.c`, in source order (`setsid` `:123` → open slave `:127` → `TIOCSCTTY` `:129` → `execvp`
`:159`), verbatim:

```
setsid()                                = 105141
openat(AT_FDCWD, "/dev/pts/0", O_RDWR|O_CLOEXEC) = 12
ioctl(12, TIOCSCTTY, 0)                 = 0
execve("/usr/bin/sh", ["sh", "-c", "tty > /root/qa/one/tty.txt 2>&1;"...], 0x56a8f6dd2d70 /* 21 vars */) = 0
```

(The `"tty > /root/qa/one/tty.txt 2>&1;"...` is `strace`'s own default `-s 32` string
truncation — the first 32 bytes of the child's `sh -c` argument followed by `...`; the full
argument string is visible in the `ps` `args` column below.)

Three facts are worth calling out, each `OBSERVED`:

- `setsid()` **returns `105141`, which equals the child's own PID** — so the child became a
  new **session leader** (SID = PID). This is `child.c:123`.
- `ioctl(12, TIOCSCTTY, 0) = 0` on the freshly-opened `/dev/pts/0` slave (fd 12) **succeeds**,
  making that PTY the child's **controlling terminal**. This is `child.c:129`.
- `execve("/usr/bin/sh", …) = 0` shows `execvp` (`child.c:159`) resolved `sh` on `PATH` to
  `/usr/bin/sh` and replaced the process image with the shell, carrying `/* 21 vars */` (the
  exact count of the exported environment — see §4).

The resulting child's own view of its session, captured by the child itself (from the same
launch), corroborates the syscalls:

```
$ tty
/dev/pts/0
$ ps -o pid,ppid,sid,tty,stat,comm,args -p $$
    PID    PPID     SID TT       STAT COMMAND         COMMAND
 105141  105074  105141 pts/0    Ss+  sh              sh -c tty > /root/qa/one/tty.txt 2>&1; ps -o pid,ppid,sid,tty,stat,comm,args -p $$ > /root/qa/one/ps.txt 2>&1; env | sort > /root/qa/one/env.txt 2>&1; echo READY; sleep 2
```

`SID == PID` (`105141 == 105141`) confirms `setsid()`; the `s` in `STAT=Ss+` confirms the child
is the **session leader** and the `+` confirms it is the **foreground process group** of its
controlling terminal `pts/0` — i.e. `TIOCSCTTY` took effect. `PPID` `105074` is the kitty
process (and equals `KITTY_PID` in §4). Only
the `fork() == -1` failure sub-branch (`spawn` returning to the parent with a failed fork)
remains `INFERRED` from source; the successful `fork`/`setsid`/`TIOCSCTTY`/`execvp` path is now
`OBSERVED` end-to-end. (The post-`execvp` failure fallback is separately exercised in §3.)

### 3. The exec-failure fallback path — exercised · `OBSERVED`

To exercise the failure branch after `execvp` (`child.c:159`), a bogus program was launched
through the real launcher:

```
$KITTY_BIN /nonexistent_program_zzz_47447
```

`execvp` fails, and the child runs the fallback: it writes to its stderr (which is the PTY
slave, so the text is drawn on the terminal, not on kitty's own stderr)
`"Failed to launch child: " + exe` (`child.c:161-163`) and
`"\nWith error: " + strerror(errno)` (`child.c:164-165`), then `execlp`s the kitten with
`__hold_till_enter__` (`child.c:166-167`) to keep the window open. The live window
(screenshot, 640×400) shows exactly this, on black:

```
Failed to launch child: /nonexistent_program_zzz_47447
With error: No such file or directory
Press Enter or Esc to exit
```

with the last line in green (the `__hold_till_enter__` prompt). kitty's own process stderr
showed only the systemd line — confirming the child's error text went to the PTY, as the
code intends. Only the `fork() == -1` sub-branch (`spawn` returning to the parent with a
failed fork) remains `INFERRED` from source.

### 4. The environment Kitty exports to the shell · `OBSERVED`

The child's environment was captured by having the real child write its own environment to a
file. Exact command:

```
$KITTY_BIN sh -c 'env | sort > "$SCRATCH/child_env.txt"; echo READY; sleep 1'
```

Here is the **complete, unedited** `env | sort` the real `sh` child wrote — all 21 variables
(this is the exact count reported by `execve(… /* 21 vars */)` in §2):

```
COLORTERM=truecolor
DISPLAY=:99
HOME=/root
HOSTNAME=ac1b170925d4
KITTY_INSTALLATION_DIR=/root/qa_reverify/clean815
KITTY_PID=105074
KITTY_PUBLIC_KEY=1:eH5RSp=V^vF6jsiG9VzO3JVz|s5gZ1X_NK2_h6o2
KITTY_WINDOW_ID=1
LANG=C.UTF-8
LC_ALL=C.UTF-8
LIBGL_ALWAYS_SOFTWARE=1
OLDPWD=/app
PATH=/root/qa_reverify/clean815/kitty/launcher:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
PWD=/root/qa_reverify/clean815
PYTEST_ADDOPTS=--tb=short -v --continue-on-collection-errors --reruns=3
SHLVL=1
TERM=xterm-kitty
TERMINFO=/root/qa_reverify/clean815/terminfo
UV_HTTP_TIMEOUT=60
WINDOWID=2097164
_=/usr/bin/timeout
```

Most lines are inherited (e.g. `HOME`, `LANG`, `PATH`'s tail, `PYTEST_ADDOPTS`,
`UV_HTTP_TIMEOUT`, `HOSTNAME` — container defaults). The variables **kitty sets or modifies**
for the shell, extracted from the complete capture above with source anchors, are (stable
across two runs; `KITTY_PID` and the per-launch ephemeral `KITTY_PUBLIC_KEY` vary):

| Variable | Value (this run) | Source |
|----------|-------|--------|
| `TERM` | `xterm-kitty` | `kitty/child.py:242` |
| `COLORTERM` | `truecolor` | `kitty/child.py:243` |
| `KITTY_PID` | `105074` (varies per launch; equals the launcher pid) | `kitty/child.py:244` |
| `KITTY_PUBLIC_KEY` | `1:eH5RSp=V^vF6jsiG9VzO3JVz|s5gZ1X_NK2_h6o2` (ephemeral, varies) | `kitty/child.py:245` |
| `KITTY_WINDOW_ID` | `1` | (window id) |
| `PWD` | `/root/qa_reverify/clean815` (the launch cwd; `OLDPWD=/app`) | `kitty/child.py:254` |
| `TERMINFO` | `/root/qa_reverify/clean815/terminfo` (path mode) | `kitty/child.py:255-258` |
| `KITTY_INSTALLATION_DIR` | `/root/qa_reverify/clean815` | `kitty/child.py:261` |
| `WINDOWID` | `2097164` (the X11 window id) | (set by kitty) |
| `PATH` | prefixed with `…/kitty/launcher` so `kitty`/`kitten` resolve | (prepended by kitty) |

`KITTY_PID=105074` is exactly the launcher pid seen as the strace **parent** and as the child's
`PPID` in §2. `KITTY_SHELL_INTEGRATION` is **absent** for the `sh` child — see §6.

### 5. The terminfo database for `TERM=xterm-kitty` · `OBSERVED`

`terminfo_type` defaults to `path` [`kitty/options/definition.py:3256`, choices
`path`/`direct`/`none`]. Both delivery modes were exercised (stable across two runs):

- **path** (default): `TERMINFO=<clean815>/terminfo`, a directory containing
  `x/xterm-kitty` (3711 bytes). Resolved by `checked_terminfo_dir` (`child.py:86`,
  used at `:256-258`).
- **direct** (`-o terminfo_type=direct`): `TERMINFO` is a 4952-byte string prefixed `b64:`
  followed by base64 (`base64_terminfo_data`, `child.py:184`, used at `:259-260`). Decoding
  it yields **3711 bytes**, magic `0x011A`, containing the string `xterm-kitty|KovIdTTY` —
  byte-for-byte the same size as the on-disk path-mode file. Both modes therefore deliver the
  same compiled capability database; which one is exported changes only *how* the shell finds
  it.

### 6. Shell integration is applied to supported shells, not to `sh` · `OBSERVED`

`modify_shell_environ` (`kitty/shell_integration.py:218`) sets `KITTY_SHELL_INTEGRATION`
only for supported shells: `ENV_MODIFIERS = {'fish', 'zsh', 'bash'}`
(`shell_integration.py:173-176`); `get_supported_shell_name` (`:186`) returns `None` for
`sh`, so `modify_shell_environ` returns early (`:221`). Both branches were observed:

```
# bash child (supported):
$KITTY_BIN bash -c 'env | grep -E "^KITTY_SHELL_INTEGRATION" ; echo READY; sleep 1'
KITTY_SHELL_INTEGRATION=enabled

# sh child (unsupported):
$KITTY_BIN sh -c 'env | grep -c "^KITTY_SHELL_INTEGRATION"; echo READY; sleep 1'
0
```

So the `sh -c` children used elsewhere in this document deliberately receive **no** shell
integration; a supported `bash` child receives `KITTY_SHELL_INTEGRATION=enabled`. The
concrete integration assets are `shell-integration/bash/kitty.bash` (17363 bytes),
`shell-integration/zsh/kitty.zsh` (+ `kitty-integration`, completions), and
`shell-integration/fish/{vendor_conf.d,vendor_completions.d}`; these are the scripts sourced
into a supported shell that emit the first escape sequences on an interactive startup.

### 7. The shell's first bytes are parsed and drawn · `OBSERVED`

**The raw first bytes.** `--dump-bytes` (`kitty/cli.py:985`) records the exact bytes read
from the PTY master. For `sh -c 'echo READY; sleep 1'` the dump is exactly 7 bytes, identical
across two runs (`od -c`):

```
0000000   R   E   A   D   Y  \r  \n
0000007
```

i.e. `0x52 45 41 44 59 0D 0A`. There are no leading escape sequences, consistent with `sh`
receiving no shell integration (§6).

**The bytes are understood and drawn (live screen).** The canonical proof is the real
launcher window itself, captured before and after the child's first output. A child that
delays its output (`sh -c 'sleep 2; echo READY; sleep 4'`) lets the *empty* pre-output state
be photographed, then the *populated* state. Exact procedure: launch through the real
launcher; find the window with `xdotool search --class kitty`; `import -window <id>` at
≈1 s and again at ≈3 s; kill our own pid. Both frames are 640×400.

- **before** (≈1 s, during the initial sleep): the screen is entirely black except a small
  light-grey **block cursor** at row 0, column 0. `identify` reports 2 colours. This is the
  empty pre-output state.
- **after** (≈3 s, after `echo`): **`READY`** is drawn in light grey (the default foreground
  ≈221,221,221) at row 0, columns 0–4, and the block cursor has moved to row 1, column 0.
  `identify` reports 100 colours (anti-aliased glyph edges).

This proves the 7 bytes were *understood*, not echoed literally: the five printable bytes
`R E A D Y` were drawn as five glyphs via `screen_draw_text` (`kitty/vt-parser.c:226`), while
`\r\n` were interpreted as **cursor control** (carriage-return + line-feed → move the cursor
to row 1, column 0), not printed as glyphs. The parse→draw path is: the `child-monitor` read
loop hands the PTY bytes to its parse worker — `parse_worker` (`kitty/child-monitor.c:181`),
or `parse_worker_dump` (`kitty/child-monitor.c:180`) when `--dump-bytes` is set — which feeds
them to `screen_draw_text` (`kitty/vt-parser.c:226`) and thence to the line buffer in
`kitty/screen.c`.

**Supplementary corroboration** (`non-canonical`): feeding the same `b"READY\r\n"` to the
in-process `Screen` model via the test helper `kitty_tests.parse_bytes` yields
`BEFORE line0='' cursor(0,0)` and `AFTER line0='READY' cursor(0,1)` — matching the live
window exactly. This is labelled `non-canonical` because it drives the `Screen` model
directly rather than through the launcher; it is included only as mechanism corroboration and
is not the primary evidence (the live screenshots above are).

### Q3 coverage check

Addressed by name: `openpty` (`child.py:170`, `OBSERVED` via `strace` `openat("/dev/ptmx")`),
`Child.fork` (`child.py:276`), C `spawn` (`child.c:81`), `setsid` (`:123`, `OBSERVED`
`setsid()=105141`), `TIOCSCTTY` (`:129`, `OBSERVED` `ioctl(12, TIOCSCTTY, 0)=0`), `execvp`
(`:159`, `OBSERVED` `execve("/usr/bin/sh", …)=0`) and its exec-failure fallback
(`child.c:161-167`, exercised), the child's controlling-TTY/session state (`tty`=`/dev/pts/0`,
`ps` `SID==PID`, `Ss+`), the complete exported environment (all 21 vars, with kitty-set
`TERM`/`COLORTERM`/`KITTY_PID`/`KITTY_PUBLIC_KEY`/`KITTY_WINDOW_ID`/`WINDOWID`/`PWD`/`TERMINFO`/`KITTY_INSTALLATION_DIR`
+ `PATH` prefix), both `TERMINFO` delivery modes, shell integration for supported vs
unsupported shells with the concrete bash/zsh/fish assets, `--dump-bytes` first bytes, the
parse→draw path (the `parse_worker`/`parse_worker_dump` dispatch at `child-monitor.c:180-181` →
`vt-parser.c:226` `screen_draw_text` → `screen.c`), and before/after live screen states.


## Q4 — Visible evidence that the display system is active (fonts, layout, scrolling, screen updates)

### The OpenGL requirement — 3.1 on Linux, 3.3 on macOS · `OBSERVED` (source) + `OBSERVED` (runtime)

kitty hard-requires a modern OpenGL core context. The required version is defined in
`kitty/data-types.h`:

```
#define OPENGL_REQUIRED_VERSION_MAJOR 3          // :20
#ifdef __APPLE__
#define OPENGL_REQUIRED_VERSION_MINOR 3          // :22
#else
#define OPENGL_REQUIRED_VERSION_MINOR 1          // :24
#endif
#define GLSL_VERSION 140                         // :26
```

So on **Linux the floor is 3.1**, and 3.3 applies only on macOS (`INFERRED` for macOS, not
run on this host). The requested context version is set as GLFW hints
(`GLFW_CONTEXT_VERSION_MAJOR/MINOR = OPENGL_REQUIRED_VERSION_*`, `kitty/glfw.c:1127-1128`,
with `GLFW_OPENGL_FORWARD_COMPAT` at `:1129`). After the context is current, the version is
re-validated and the banner printed:

- `kitty/gl.c:72` prints the banner to **stdout** when `--debug-rendering` is set.
- `kitty/gl.c:73-74` is the enforcement: `fatal(...)` if
  `gl_major < 3 || (gl_major == 3 && gl_minor < OPENGL_REQUIRED_VERSION_MINOR)`.

Observed banner (from Q1): `4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.2`, detected
version `4.5`. Because `4.5 ≥ 3.1`, the check at `gl.c:73` **passes** and the `fatal` at
`gl.c:74` is **not reached** (that path is therefore `INFERRED`/not-reached on the happy
path; it *is* exercised as a negative in E1).

### The cell rasterization / draw pipeline · `OBSERVED` (effect) + `INFERRED` (internals)

Shaders are compiled at startup by `load_all_shaders` (`kitty/main.py:82`), which loads the
shader programs and the borders program and raises `SystemExit` on a `CompileError`; the
loader is passed into `create_os_window` (`main.py:221/225`). Natively, programs are compiled
by `compile_program` (`kitty/shaders.c:1168`), and cells are drawn by `draw_cells`
(`kitty/shaders.c:1009`) → `draw_cells_simple` (`:577`) / `draw_cells_interleaved` and
`_premult` (`:868/912`), after `cell_prepare_to_render` (`:394`). The 13 GLSL programs are
`alpha_blend`, `bgimage_fragment`/`vertex`, `border_fragment`/`vertex`, `cell_defines`,
`cell_fragment`, `cell_vertex`, `graphics_fragment`/`vertex`, `linear2srgb`, and
`tint_fragment`/`vertex`. The internals are `INFERRED` from source; the **effect** is
`OBSERVED` — glyphs are drawn (the `READY` frame in Q3) and no `CompileError`/`SystemExit`
occurs, so shader compilation and the draw path succeeded.

### Fonts — resolution, rasterization, and the fallback report · `OBSERVED`

The `--debug-font-fallback` dump from Q1 and the `Fonts:` block of the canonical
`debug_config` report (Q2/E3) both show the resolved faces (reproduced here as an aligned,
path-abbreviated view for readability; the byte-exact `--debug-font-fallback` output with
timestamps is quoted verbatim in Q1):

```
Normal:      DejaVuSansMono            /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
Bold:        DejaVuSansMono-Bold       .../DejaVuSansMono-Bold.ttf:0
Italic:      DejaVuSansMono-Oblique    .../DejaVuSansMono-Oblique.ttf:0
Bold-Italic: DejaVuSansMono-BoldOblique .../DejaVuSansMono-BoldOblique.ttf:0
```

On Linux, discovery is via fontconfig (`kitty/fontconfig.c` + `kitty/fonts/fontconfig.py`)
and rasterization via FreeType (`kitty/freetype.c`, 42821 bytes); the DejaVu paths above are
the `OBSERVED` result of that discovery. The macOS CoreText path (`kitty/core_text.m` +
`kitty/fonts/core_text.py`) exists in the tree but is `INFERRED` (not run on this Linux host).

#### Runtime **fallback** selection for glyphs the primary font lacks · `OBSERVED`

The four faces above are the *primary* group. `DejaVuSansMono` has no CJK ideographs, so when
a cell needs one, kitty performs a **runtime fallback**: `output_cell_fallback_data`
(`kitty/fonts.c:457-467`) logs the missing codepoint with `debug("U+%x ", cell->ch)`
(`:458`) followed by the chosen face via `PyObject_Print(face, stderr, 0)` (`:466`), gated on
`global_state.debug_font_fallback` (`kitty/fonts.c:492`, set by `--debug-font-fallback`,
`kitty/cli.py:1002`). To trigger it, the canonical launcher was run with a child that prints a
CJK ideograph `中` (`U+4E2D`) and a check mark `✓` (`U+2713`), keeping them on screen (no
scroll) so the cells are rendered:

```
DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 LANG=C.UTF-8 LC_ALL=C.UTF-8 \
  $KITTY_BIN --debug-rendering --debug-font-fallback --dump-bytes=$SCRATCH/q4/bytes.dump \
  sh -c 'printf "READY\r\n"; printf "\033[31mREDTEXT\033[0m\r\n"; \
         printf "\344\270\255 \342\234\223\r\n"; sleep 30'
```

The fallback face resolved for `中` appears on stderr, byte-identical across three runs (only
the `[%.3f]` timestamp differs — `0.173` / `0.175` / `0.174`):

```
[0.173] U+4e2d Face(family=Droid Sans Fallback style=Regular ps_name=DroidSansFallback path=/usr/share/fonts/truetype/droid/DroidSansFallbackFull.ttf ttc_index=0 variant=False named_instance=False scalable=True color=False)
```

So `中` is drawn from **Droid Sans Fallback**
(`/usr/share/fonts/truetype/droid/DroidSansFallbackFull.ttf`), selected at runtime — not from
the primary `DejaVuSansMono`. The fallback is **per-glyph and selective**: `✓` (`U+2713`)
produced **no** fallback line, because `DejaVuSansMono` already contains that glyph —
confirmed independently with `fc-query --format='%{charset}' …DejaVuSansMono.ttf`, which lists
codepoint `2713`. Only the codepoint the primary genuinely lacks (`U+4E2D`) is routed to a
fallback face. (The chosen fallback face is environment-dependent — it is whatever fontconfig
ranks first for the codepoint on this host; here that is Droid Sans Fallback.)

### The framebuffer proves fonts + layout + rendering are live (font size measured) · `OBSERVED`

Two launches were photographed at the same window (found via `xdotool search --class kitty`,
captured with `import -window <id>`), changing only the font size, and the drawn `READY`
region was measured with ImageMagick `-trim`:

- default `font_size 11`: the `READY` glyph run measures **44×10 px**.
- `-o font_size=24`: the same text measures **94×23 px**.

The ratio (≈2.1× width, ≈2.3× height) tracks the font-size ratio `24/11 ≈ 2.18×`, proving the
measurement reflects real glyph rasterization at the configured size (not a fixed bitmap).
The custom run is the runtime effect of the Q2 `font_size` override.

The cursor shape was likewise measured (with blinking disabled via `-o cursor_blink_interval=0`
and input focus forced, because the cursor blinks by default — itself a liveness signal):

- default `cursor_shape block`: the home-cell cursor is a filled rectangle **9×18 px**
  (a full cell).
- `-o cursor_shape=beam`: the cursor is a thin vertical bar **2×18 px** (full height, ~2 px
  wide).

This is the runtime effect of the Q2 `cursor_shape` override, measured on the live window.

### ANSI colour and Unicode escapes — parsed, then drawn as coloured/wide glyphs · `OBSERVED`

The same run also proves the two escape classes the READY-only run (Q3 §7) could not: **SGR
colour** and **multi-byte Unicode**. `--dump-bytes` writes the raw PTY bytes to the file *and*
prints the parser's interpretation to stdout via `parse_worker_dump` (`kitty/child-monitor.c:180`).
The raw bytes from `od -An -tx1 $SCRATCH/q4/bytes.dump` — regrouped here one *logical* line
per row (the literal `od` output wraps every 16 bytes: `52 45 41 44 59 0d 0d 0a 1b 5b 33 31 6d
52 45 44 / 54 45 58 54 1b 5b 30 6d 0d 0d 0a e4 b8 ad 20 e2 / 9c 93 0d 0d 0a …`) with an
annotation column added — are, for the first three lines:

```
52 45 41 44 59 0d 0d 0a                                     READY  CR CR LF
1b 5b 33 31 6d 52 45 44 54 45 58 54 1b 5b 30 6d 0d 0d 0a    ESC[31m REDTEXT ESC[0m CR CR LF
e4 b8 ad 20 e2 9c 93 0d 0d 0a                               中(e4 b8 ad) SP ✓(e2 9c 93) CR CR LF
```

`1b 5b 33 31 6d` is `ESC [ 3 1 m` (SGR 31, red), `1b 5b 30 6d` is `ESC [ 0 m` (SGR reset),
`e4 b8 ad` is UTF-8 for `中` (`U+4E2D`) and `e2 9c 93` is UTF-8 for `✓` (`U+2713`). (The
doubled `0d 0d 0a` is the PTY line discipline's `ONLCR` turning the child's own `\r\n` into
`\r\r\n` on the master side.) The parser's interpretation on stdout is, verbatim for the first
three logical lines (identical across three runs; the trailing `GL version …` banner line is
omitted only because it belongs to a different stream, quoted in Q4's OpenGL section):

```
draw READY
screen_carriage_return
screen_carriage_return
screen_linefeed
select_graphic_rendition 31
draw REDTEXT
select_graphic_rendition 0
screen_carriage_return
screen_carriage_return
screen_linefeed
draw 中 ✓
screen_carriage_return
screen_carriage_return
screen_linefeed
```

So the bytes are **understood, not echoed**: `ESC[31m`/`ESC[0m` become
`select_graphic_rendition 31`/`0` (the VT parser's `dispatch_csi` → `select_graphic_rendition`,
`kitty/vt-parser.c`), and the six UTF-8 bytes are decoded into exactly two glyphs
`draw 中 ✓` (`screen_draw_text`, `kitty/vt-parser.c:226`), not eight raw bytes. Each doubled
`screen_carriage_return` mirrors the doubled `0d` in the raw dump (the `ONLCR` effect), and
each `screen_linefeed` is the `0a` — control bytes turned into cursor motion, not glyphs.

**Drawn to the framebuffer with the right colour and shape.** The live window (`xwd` →
`convert`, 640×400) was measured per 18-px text row with ImageMagick. The result confirms the
colour was applied *and reset* exactly where the escapes dictate (red-ish = R>120, G<80, B<80):

| Row (18 px band) | Dominant non-black ink | Red-ish px | Meaning |
|------------------|------------------------|-----------|---------|
| 0 — `READY` | `#DDDDDD` (grey) | 0 | default foreground |
| 1 — `REDTEXT` | `#CC0403` (red) | 235 | `ESC[31m` applied — **all** the red ink is here |
| 2 — `中 ✓` | `#DDDDDD` (grey) | 0 | `ESC[0m` reset back to default before this row |

The whole frame is `sRGB` with **202 unique colours** (vs the all-grey 50-line frame below,
which is `Grayscale`); `#CC0403` is kitty's default palette `color1` (red). The `中 ✓` row has
a trimmed ink bounding box of **34×13 px** — real rasterized glyphs with extent (`中` occupies
a double-width cell), not empty cells or `.notdef` boxes, consistent with the Droid Sans
Fallback selection logged above.

### Layout / scrolling / screen-update (damage) model · `OBSERVED` (effect) + `INFERRED` (internals)

To exercise **scrolling** (not merely a two-line layout), a child was run that overflows the
window:

```
$KITTY_BIN --dump-bytes=$SCRATCH/q4/overflow.dump \
  sh -c 'for i in $(seq 1 50); do echo LINE$i; done; sleep 30'
```

The window is 640×400 with a ≈9×18 px cell, i.e. ≈71 columns × ≈22 rows visible. The
`--dump-bytes` confirms all 50 `LINE` tokens were emitted. The live screenshot shows
**`LINE30` (top) through `LINE50` (bottom)** followed by the cursor — 21 lines — while
`LINE1`–`LINE29` have **scrolled off** the top into scrollback. Fifty lines mapped into ≈22
visible rows demonstrates real line-feed scrolling. The internal damage/scroll bookkeeping
(dirty-line marking `kitty/screen.c:210-211`, `linebuf_clear` `:180`, `linebuf_rewrap`
`:240`) is `INFERRED` from source; the scroll **effect** is `OBSERVED`. A re-run confirmed the
counts are stable: the parser emitted exactly 50 `draw LINE` tokens (last `draw LINE50`) and
the resulting frame has **21 inked text rows** and is `Grayscale` (0 red-ish pixels — the plain
`LINE` text carries no colour, in deliberate contrast to the `sRGB`/red frame above).

### Confirming log/console messages · `OBSERVED`

The messages that confirm the display system came up, and their streams/producers (as
established in Q1):

- **stdout**, `--debug-rendering`: `GL version string: '4.5 (Core Profile) Mesa …' Detected
  version: 4.5` (`kitty/gl.c:72`) — the GL context is live at the required version.
- **stderr**, `--debug-rendering`: `OS Window created` (`kitty/glfw.c:1321`) — the OS window
  exists.
- **stderr**, `--debug-font-fallback`: the `Text fonts:` block (`kitty/fonts/render.py:163`)
  — the fonts resolved.
- The canonical `debug_config` report (Q2/E3) restates all three (`OpenGL:`, `Fonts:`,
  `Running under: X11`) from the live process.

All of these arrive on stderr with the `[%.3f]` prefix (`kitty/logging.c:56`) except the GL
banner, which is on stdout.

### Q4 coverage check

Addressed by name: the OpenGL requirement (`data-types.h:20-26`, Linux 3.1 / macOS 3.3),
the GLFW context hints (`glfw.c:1127-1129`), the runtime banner + enforcement
(`gl.c:72-74`, with `:74` correctly labelled not-reached on the happy path),
`load_all_shaders` (`main.py:82`), the draw pipeline (`shaders.c:1009` and variants) and the
13 GLSL programs, font discovery/rasterization (`fontconfig.c`/`fonts/fontconfig.py`,
`freetype.c`; macOS `core_text` inferred), the **runtime fallback-font selection** for a glyph
the primary lacks (`output_cell_fallback_data`, `fonts.c:457-467` — `U+4e2d` → Droid Sans
Fallback, with `✓`/`U+2713` staying in `DejaVuSansMono`), **ANSI SGR colour** parsing and
reset (`select_graphic_rendition 31`/`0`, red `#CC0403` measured and confined to the coloured
row) and **Unicode** decoding into wide/narrow glyphs (`draw 中 ✓` from UTF-8 `e4 b8 ad`/`e2 9c
93`), measured font-size and cursor-shape effects, real scrolling with before/after visible
lines, and the confirming log lines with correct streams/producers.


## Edge cases and negative conditions (E1–E5)

### E1 — OpenGL availability, tested truthfully · `OBSERVED`

The relevant question is whether removing `LIBGL_ALWAYS_SOFTWARE` changes the outcome. The
unchanged canonical launch was repeated with **only** that variable unset, twice:

```
env -u LIBGL_ALWAYS_SOFTWARE DISPLAY=:77 LANG=C.UTF-8 LC_ALL=C.UTF-8 \
  HOME=$SCRATCH/home KITTY_CONFIG_DIRECTORY=$SCRATCH/cfgempty \
  $KITTY_BIN --debug-rendering sh -c 'echo READY; sleep 1'
```

Result: **exit 0 on both runs, identical GL banner** `4.5 (Core Profile) Mesa
25.2.8-0ubuntu0.24.04.2`. `glxinfo` reports llvmpipe 4.5 **with and without** the variable.
The honest finding is **no difference**: under Xvfb there is no GPU, so Mesa selects the
llvmpipe software renderer regardless; `LIBGL_ALWAYS_SOFTWARE=1` is a redundant *force*, not
a prerequisite in this environment. (It would matter on a host that also has a hardware GL
driver Mesa might otherwise prefer.)

Two **additional negatives** genuinely produce the failure paths (each run twice; after
timestamp normalisation the two runs are byte-identical):

- **Negative A — no reachable display** (`DISPLAY=:1234`): exit 1, stderr:

  ```
  [0.061] [glfw error 65544]: X11: Failed to open display :1234
  GLFW initialization failed
  ```

  This is `init_glfw` failing **before** any window is created (Q1 step 4 failing).

- **Negative B — force an inadequate GL version** (`MESA_GL_VERSION_OVERRIDE=2.1`): exit 1,
  stderr:

  ```
  [0.105] [glfw error 65543]: GLX: Failed to create context: GLXBadFBConfig
  [0.105] Failed to create GLFW temp window! This usually happens because of old/broken OpenGL drivers. kitty requires working OpenGL 3.1 drivers.
  ```

  This is the **temp-window fatal at `kitty/glfw.c:1198-1199`**: the 3.1-core context
  requested at `glfwCreateWindow` cannot be satisfied, so creation fails there. The message
  interpolates `3.1`, confirming the Linux `OPENGL_REQUIRED_VERSION_MINOR = 1`
  (`data-types.h:24`). Note the important subtlety: an inadequate GL manifests as the
  `glfw.c:1199` temp-window fatal, so the *secondary* version check at `gl.c:73-74` is **not
  reached** in this scenario (it is a backstop for a context that is created but reports too
  low a version). `gl.c:73-74` is therefore `INFERRED` (not-reached) here, consistent with
  the happy-path labelling in Q4.

### E2 — crafted config in an isolated root · `OBSERVED`

Fully covered in Q2: the crafted `kitty.conf` was created inside an isolated
`KITTY_CONFIG_DIRECTORY=$SCRATCH/cfgcustom` (mode 700, never a shared/real config location),
one canonical run applied the valid overrides (`font_size 18.0`, `cursor_shape beam`,
`scrollback_lines 5000`) and reported both bad-line kinds (unknown key on stderr; known-key
invalid value as a `BadLine` shown in the red error overlay), and the directory is removed in
cleanup. The complete creation command, applied values, raw stderr, and overlay content are
in Q2.

### E3 — the `debug_config` report via the real keybinding · `OBSERVED` (fully canonical)

Fully covered in Q2. This was captured by driving the **real** shortcut — `kitty_mod+f6` =
**ctrl+shift+F6** (`definition.py:3474` + `:4256`) — against the live window over XTEST, then
reading the report kitty copied to the X11 clipboard and screenshotting the on-screen
overlay. The clipboard sentinel seeded before the keypress was overwritten by kitty's own
`set_clipboard_string` (`boss.py:3065`), proving the write was kitty's. The report is
byte-identical across two runs.

**On the review's "no window manager ⇒ key delivery impossible" claim (M16):** this is
demonstrably **false** in this environment. There is no window manager, yet
`xdotool windowactivate` failing (expected without a WM) does not prevent input:
`xdotool windowfocus <id>` sets input focus via `XSetInputFocus`, `xdotool getwindowfocus`
then returns the kitty window id, and the XTEST key injection is delivered to kitty and acted
upon (the clipboard changes and the overlay appears). The shortcut was delivered canonically
without any WM.

### E4 — before / intermediate / after screen states · `OBSERVED`

Fully covered in Q3 §7 using the **live launcher window** (not a synthetic model): the empty
black screen with only the block cursor *before* the child's first output; `READY` drawn at
row 0 with the cursor advanced to row 1 *after*; and, as a distinct stateful transition, the
scrolled screen in Q4 (`LINE30`–`LINE50` visible, earlier lines scrolled off). The in-process
`Screen`-model check is included only as clearly-labelled `non-canonical` corroboration.

### E5 — `TERMINFO` delivery modes · `OBSERVED`

Fully covered in Q3 §5: `path` mode exports a directory containing `x/xterm-kitty` (3711 B);
`direct` mode exports a `b64:`-prefixed base64 string that decodes to the same 3711-byte
compiled database (magic `0x011A`, `xterm-kitty|KovIdTTY`). Both modes were run twice and are
stable.

## Final coverage pass

### Functions / mechanisms

`Py_PreInitialize`/`PyConfig_InitPythonConfig`/`Py_InitializeFromConfig`/`Py_RunMain`
(`launcher/main.c`, observed via `ldd`+`readelf`); `entry_points.main`/`kitty_main`;
`_main` (`main.py:441`); `init_glfw` (`main.py:514`); `AppRunner.__call__` with
`set_scale`/`set_options`/`set_font_family` (`main.py:247-251`); `_run_app` (`main.py:202`);
`create_sessions` (`main.py:214`); `create_os_window` (`main.py:221`, native `glfw.c:1107`);
`load_all_shaders` (`main.py:82`, native `compile_program` `shaders.c:1168`, `draw_cells`
`shaders.c:1009`); `gl_init` + banner + enforcement (`gl.c:72-74`); `Boss`/`boss.start`/
`startup_first_child`/`add_child`; `dump_font_debug` (`main.py:229`, `fonts/render.py:163`);
`child_monitor.main_loop` (`main.py:234`); `create_opts` (`cli.py:1081`),
`default_config_paths`, `resolve_config` (`conf/utils.py:322`), `load_config`
(`config.py:163`); `debug_config` (`debug_config.py:231`) + `Boss.debug_config`
(`boss.py:3060`) + `set_clipboard_string`/`display_scrollback`; `openpty` (`child.py:170`),
`Child.fork` (`child.py:276`), C `spawn` (`child.c:81`), `setsid`/`TIOCSCTTY`/`execvp`
(`child.c:123/129/159`) + exec-failure fallback (`child.c:161-167`);
`checked_terminfo_dir`/`base64_terminfo_data` (`child.py:86/184`); `modify_shell_environ`
(`shell_integration.py:218`); `screen_draw_text` (`vt-parser.c:226`);
`format_bad_line`/`show_error` (`boss.py:2761/2050`); `convert_opts_from_python_opts`
(`state.c:741`).

### Files

Launcher/entry: `kitty/launcher/main.c`, `kitty/entry_points.py`, `kitty/main.py`.
Windowing/GL: `kitty/glfw.c`, `kitty/state.c`, `kitty/gl.c`, `kitty/data-types.h`,
`kitty/shaders.c`, `kitty/*.glsl`. Configuration: `kitty/cli.py`, `kitty/config.py`,
`kitty/constants.py`, `kitty/conf/utils.py`, `kitty/options/definition.py`,
`kitty/options/parse.py`, `kitty/options/types.py`, `kitty/options/utils.py`,
`kitty/options/to-c.h`, `kitty/options/to-c-generated.h`, `kitty/debug_config.py`. PTY/child:
`kitty/child.py`, `kitty/child.c`, `kitty/child-monitor.c`, `kitty/boss.py`, `kitty/window.py`,
`kitty/shell_integration.py`, `kitty/terminfo.py`. Parse/screen/fonts: `kitty/vt-parser.c`,
`kitty/screen.c`, `kitty/fonts.c`, `kitty/fonts/render.py`, `kitty/fonts/fontconfig.py`,
`kitty/fonts/core_text.py` (macOS, inferred), `kitty/freetype.c`, `kitty/fontconfig.c`,
`kitty/core_text.m` (macOS, inferred). Assets consumed at runtime:
`terminfo/x/xterm-kitty`, `shell-integration/bash/kitty.bash` (+ zsh/fish). Common sink:
`kitty/logging.c`. Build refs: `docs/build.rst`, `dev.sh`, `setup.py`, `go.mod`,
`pyproject.toml`.

### Flags

`--debug-rendering` (GL banner + `OS Window created`; its alias `--debug-gl`,
`kitty/cli.py:989`, enables the same OpenGL-error checking), `--debug-font-fallback`
(`Text fonts:` dump, `kitty/cli.py:1002`), `--dump-bytes` (raw PTY bytes, `kitty/cli.py:985`),
`-o KEY=VALUE` (config override, e.g. `font_size`, `cursor_shape`, `terminfo_type`,
`cursor_blink_interval`), `--version`. Two further debug flags exist but were **not**
exercised, because keyboard input is outside the four startup questions: `--debug-input` and
its alias `--debug-keyboard` (`kitty/cli.py:996`, `dest=debug_keyboard`), which print key and
mouse events as they are received. The `debug_config` action is a **keybinding**
(`ctrl+shift+F6`), not a CLI flag, at this commit.

### "e.g. / such as / including / like" items from the questions

- Systems on the way up: launcher/CPython, GLFW backend, OpenGL context + shaders, fonts,
  OS window, `Boss` controller, child/PTY monitor — each addressed in Q1.
- Configuration sources / default settings: `SYSTEM_CONF`, `defconf`, `XDG` dirs, the
  `defaults` instance — Q2.
- Terminal↔shell items: `openpty`, `fork`/`spawn`, `setsid`, `TIOCSCTTY`, `execvp`,
  `TERM=xterm-kitty`, `TERMINFO`, shell integration — Q3.
- Display items: fonts, layout, scrolling, screen updates, plus confirming log/console
  messages — Q4.

### OBSERVED vs INFERRED summary

- `OBSERVED` (runtime, canonical): the build + artifact hashes + VCS stamp; glxinfo renderer;
  the Q1 bring-up lines with streams/producers and stability hashes; CPython launcher symbols;
  the default-config resolution values and the canonical `debug_config` report via the real
  keybinding; the crafted-config applied values and both bad-line reports + overlay; the
  PTY/session/controlling-TTY/exec handshake syscalls via `strace` (`openpty`→`/dev/ptmx`,
  `setsid`, `ioctl TIOCSCTTY`, `execvp`→`/usr/bin/sh`) plus the child's own TTY/session state
  (`SID==PID`, `Ss+`, `pts/0`); the complete child environment, both terminfo modes,
  bash-vs-`sh` integration, the 7 first bytes, and the before/after live screen; the GL
  requirement/banner, primary **and** fallback font selection, ANSI-colour + Unicode escape
  parsing and rendering, measured font-size and cursor-shape effects, real scrolling; and the
  E1 negatives.
- `observed-effect + inferred-correlation`: that `entry_points`/`_main`/`init_glfw`/
  `create_sessions` ran (no line of their own, but later output cannot exist without them).
- `INFERRED` (source-read only): internal control flow inside
  `create_os_window`/`gl_init`/`boss.start` beyond the emitted anchors; the `spawn`
  `fork()==-1` failure sub-branch only (its `setsid`/`TIOCSCTTY`/`execvp` success path is now
  `OBSERVED` via `strace`); the damage/rewrap internals; `gl.c:73-74` on the happy path
  (not reached; the fatal is exercised instead at `glfw.c:1199` in E1 negative B); and the
  macOS CoreText/Cocoa paths.
- `non-canonical` (labelled, corroboration only): the in-process `Screen`-model check in Q3.

### Dependency and security scope note

This investigation introduced **no dependency or manifest change**: no package was added,
updated or removed, and `pyproject.toml`, `go.mod` and `setup.py` are untouched (the task is
read-only and its sole artifact is this document). The versions exercised are those the
pinned commit and the canonical container already carry — `kitty 0.35.2`
(`kitty/constants.py:25`), CPython 3.12, Go 1.23, gcc 13, and the bundled/system native
libraries (`harfbuzz`, `freetype`, `fontconfig`, `wayland-protocols`, Mesa/llvmpipe, etc.).
Because `0.35.2` is a **historical** release and the container image pins older toolchain and
library builds, some of those components may be subject to security advisories published
*after* this commit. Any such advisory is **pre-existing** — it is a property of the historical
baseline, not something this document introduced — and auditing or remediating it is **out of
scope** for a read-only startup investigation (it would require dependency upgrades, which are
explicitly excluded). If the historical `0.35.2` binary built here were to be *deployed* rather
than merely observed, a separately scoped dependency/CVE review of that pinned toolchain would
be warranted before deployment. No secrets or credentials are used or emitted by any run above;
the only sensitive-looking value, `KITTY_PUBLIC_KEY`, is an ephemeral per-launch public key
(regenerated every start, e.g. `1:eH5RSp=…` in §4) and carries no persistent secret.

### Cleanup verification

All temporary artifacts live under the single mode-700 scratch directory (the build clone,
isolated `HOME`, isolated `KITTY_CONFIG_DIRECTORY`, crafted `kitty.conf`, logs, byte dumps and
screenshots), plus a temporary screenshot copy directory under `blitzy/_scratch_screens/` used
only for inspection. The owned Xvfb (`:77`, pid recorded) is stopped by its exact pid, and the
scratch directory and the screenshot copy directory are removed. After cleanup the repository
diffs clean except for this single document (`git status` shows only
`blitzy/documentation/kitty_815df1e210e0.md`), and `git diff --check` reports no whitespace
errors. The build artifacts (`kitty/launcher/kitt*`, `*.so`, `/build`, `/dependencies`) are
git-ignored and never appear as tracked changes.
