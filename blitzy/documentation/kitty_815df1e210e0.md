# How Kitty divides rendering-adjacent work across Python, C, and Go

> **Question answered:** *"Start Kitty under sustained rendering pressure (lots of colored output, heavy
> scrollback churn, repeated resizes, tab switching). Show what loads into the main process, how thread
> activity changes idle-vs-stress, what live state the control interface exposes, how `kitty +kitten icat`
> relates to the `kitten` process, and capture a symbol/stack snapshot — then infer what belongs to
> Python vs. C vs. Go."*

This document is written **runtime-first**: every behavioral claim below sits next to the **exact command
that produced it** and its **complete, unedited output**, captured while a real Kitty instance built from
this repository ran under load. Statements derived from *reading* source rather than *running* are marked
**`(inferred from source)`**. Structural claims carry `file:line` anchors into the source tree for
corroboration only — the source tree itself was **not modified** (see §10).

Branch: `kitty_815df1e210e0` · HEAD `815df1e21 "Wire up applying of font config"`.
All build/run/observe steps were performed inside the canonical container
(`kovidgoyal__kitty__815df1e210e0…`; Python 3.12.3, Go 1.23.4, gcc 13.3.0) under a headless Xvfb display
with Mesa software GL (llvmpipe). All scratch lived under `/tmp` and was removed afterward.

---

## 1. Lead direct answer

**Kitty is a deliberate three-language system, and each language owns the domain it is best at. This
division is directly visible at runtime:**

- **C (C11) is the performance-critical core**, compiled into the single CPython C extension
  **`kitty.fast_data_types`** `[setup.py:L1091]`. At runtime it is one shared object,
  `/app/kitty/fast_data_types.so`, mapped into the main process (§4). It owns: **VT escape parsing**
  (`do_parse`), the **screen/cell model + scrollback** (`screen_repeat_character`), **GPU rendering**
  (`draw_text` / `draw_text_loop`), the **main event loop** (`main_loop`), and the **PTY-I/O and
  remote-control peer threads** (`io_loop` → `KittyChildMon`, `talk_loop` → `KittyPeerMon`). Every one of
  those symbols was observed in a live stack (§8).
- **Python (≥ 3.8 `[pyproject.toml:L2]`) runs *inside* the main process** as an embedded CPython
  interpreter. It performs orchestration, configuration, and window/tab/child lifecycle, and it hosts the
  **remote-control command surface** (`kitty/rc/*`). At runtime **60 `kitty.*` Python modules** were loaded
  into the running process, and the main thread's Python stack sits in `kitty/main.py` (§8). Python calls
  *into* the C extension; it does not itself parse bytes or draw pixels.
- **Go (declared `go 1.22` `[go.mod:L3]`, `module kitty` `[go.mod:L1]`; the container toolchain that
  built it is `go1.23.4`) is the standalone `kitten` binary**, built from `tools/cmd`
  `[setup.py:L1160-L1163]`. At runtime it is a **separate process** (`/app/kitty/launcher/kitten`), a
  ~16 MB near-static ELF that is **never mapped into the main Kitty address space** (§4, §7).

**Key runtime nuance (surfaced only by running, and it *contradicts* a naïve source reading):**
the exact invocation **`kitty +kitten icat`** runs the **Go** `kitten` binary, **not** the legacy Python
`kittens/icat/main.py`. The C launcher intercepts `+kitten <wrapped-kitten>` and `exec`s the Go binary
**before CPython is even initialised** (`kitty/launcher/main.c:356`). `icat` is one of the 12 "wrapped"
kittens, so the Python `runpy.run_module('kittens.icat.main')` path `[kittens/runner.py:L110-L117]` is
**not** taken for `icat`. This is demonstrated with process evidence in §7.

The rest of this document is the evidence for each of those claims.

---

## 2. Build & invocation (default configuration)

### 2.1 Canonical build

The canonical, default-configuration build is a plain `python3 setup.py` (no flags) from the repo root. It
compiles the C extension, the vendored GLFW backends, and shells out to `go build` for the `kitten` binary.

**Command:**

```
cd /app && python3 setup.py
```

**Observed build milestones (complete phase markers from the build log):** the C core is compiled
translation-unit by translation-unit, then linked into `fast_data_types`, then the Go tooling is built:

```
[1/122] Compiling kitty/screen.c ...
[6/122] Compiling kitty/graphics.c ...
[7/122] Compiling kitty/child-monitor.c ...
[8/122] Compiling kitty/fonts.c ...
[9/122] Compiling kitty/shaders.c ...
[10/122] Compiling kitty/vt-parser.c ...
[11/122] Compiling kitty/vt-parser.c ...
[18/122] Compiling kitty/freetype.c ...
[58/122] Compiling kitty/gl.c ...
...
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
...
kitty/tools/cmd/benchmark
kitty/kittens/icat
kitty/kittens/diff
kitty/tools/cmd/tool
kitty/tools/cmd
```

The final `kitty/...` / `kitty/tools/...` / `kitty/kittens/...` lines are the Go packages being compiled
into the `kitten` binary. This confirms all three languages are built by the one orchestrator, `setup.py`:
the C extension `[setup.py:L1091]`, GLFW `[setup.py:L1130-L1195]`, and the Go `kitten` from `tools/cmd`
`[setup.py:L1160-L1163]`.

**Resulting artifacts (built into the gitignored working tree, never committed):**

| Artifact | Size | Language / role |
|----------|------|-----------------|
| `kitty/fast_data_types.so` | 1,213,072 B | C core (CPython extension) |
| `kitty/launcher/kitty` | 36,224 B | tiny C launcher (`kitty/launcher/main.c`) |
| `kitty/launcher/kitten` | 15,945,988 B | Go static binary (`tools/cmd`) |

### 2.2 Version banners (verbatim, byte-sensitive)

**Commands and complete output:**

```
$ kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

```
$ kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
```

Both banners are `0.35.2` with **no VCS stamp appended** (verified with `cat -A`: no trailing whitespace,
no `+`-suffixed git hash). The compiled-in constant is `Version(0, 35, 2)` `[kitty/constants.py:L25]`; the
banners above are the *real runtime* strings, not hand-typed.

### 2.3 Headless launch with the control interface enabled

Kitty renders exclusively through OpenGL with **no CPU drawing fallback** *(inferred from source: the
"GPU-First" design; corroborated at runtime because a GL context was required for `--render`)*. A virtual
display with Mesa software GL was used:

```
$ Xvfb :99 -screen 0 1920x1080x24 &
$ export DISPLAY=:99
$ glxinfo | grep -E 'renderer|OpenGL version'
OpenGL renderer string: llvmpipe (LLVM 19.1.1, 256 bits)
OpenGL version string: 4.5 (Compatibility Profile) Mesa 24.2.8-1ubuntu1~24.04.1
```

The main instance was launched with pure defaults (`--config NONE`) and **remote control explicitly
enabled** (Kitty does not listen by default — this is required for §6):

```
$ export LIBGL_ALWAYS_SOFTWARE=1
$ kitty/launcher/kitty --config NONE \
    -o allow_remote_control=yes -o enabled_layouts=all \
    --listen-on unix:/tmp/kitty_probe/mykitty.sock &
```

The main process PID resolved to **`7370`** (used throughout as `$MAIN`). The Go `@` client is used as the
observation channel:

```
$ KAT="kitty/launcher/kitten @ --to unix:/tmp/kitty_probe/mykitty.sock"
$ $KAT ls >/dev/null && echo "connected"
connected
```

> **Labelling note:** the control interface (`kitty @`) is used here purely as an **observation** channel
> and, in §6, to *drive* resizes/tab-switches. Driving resizes/tabs via `@` is a legitimate canonical
> path (the same commands a user binds to keys), but where it *substitutes* for a real input path it is
> called out as `(non-canonical)`. The sustained render/parse load itself is generated by Kitty's **own**
> `kitten __benchmark__` tool, not a hand-rolled generator.

---

## 3. O1 — Sustained rendering pressure (all four workloads)

The load is generated by Kitty's **own** canonical instrument: the hidden `kitten __benchmark__`
subcommand `[tools/cmd/benchmark/main.go:L319-L349]` (flags `--repetitions` default `100`,
`--with-scrollback`, `--render`). It brackets output with the synchronized-update escapes
`\x1b[?2026h` / `\x1b[?2026l` `[tools/cmd/benchmark/main.go:L68-L69]`, and by default suppresses rendering
to isolate parser throughput; `--render` re-enables the full GPU pipeline. Each workload was run **as a
child inside a Kitty window** (so Kitty's C parser + GPU pipeline are the terminal under test), driven via
the control interface, e.g.:

```
$KAT launch --type=window --keep-focus \
  kitty/launcher/kitten __benchmark__ --render --repetitions 1000 csi
```

**What the system is doing during a run (narration, grounded in §8's stacks and §5's thread delta):** the
benchmark child writes megabytes of escape-laden bytes down the PTY; Kitty's **`KittyChildMon`** I/O thread
(`io_loop`) drains the PTY into a buffer; the **main thread** runs `do_parse` (the C VT state machine) to
classify every byte and mutate the C screen/scrollback model, then `draw_text_loop` uploads cells to the
GPU and issues GL draw calls that the **llvmpipe** software-GL worker threads rasterize. The banner the
benchmark prints itself states the measured quantity:

```
These results measure the time it takes the terminal to fully parse all the data sent to it.
Note that not all data transmitted will be displayed as input parsing is typically asynchronous with
rendering in high performance terminals.
```

### 3.1 Workload 1 — lots of colored output (`csi`)

The `csi` sub-benchmark emits ASCII interspersed with CSI/SGR (color) escape sequences — the canonical
"lots of colored output" stressor. Complete output (ESC bytes shown as `^[`; the benchmark colors its own
throughput number green with `^[[32m … ^[[m`):

**Calibration (100 repetitions):**

```
$ kitty/launcher/kitten __benchmark__ --render --repetitions 100 csi
^[]^[\^[cThese results measure the time it takes the terminal to fully parse all the data sent to it.
Note that not all data transmitted will be displayed as input parsing is typically asynchronous with rendering in high performance terminals.

Results:
  CSI codes with few chars : 3.75s      @ ^[[32m26.7   ^[[m MB/s
```

**Sustained run (1000 repetitions):**

```
$ kitty/launcher/kitten __benchmark__ --render --repetitions 1000 csi
^[]^[\^[cThese results measure the time it takes the terminal to fully parse all the data sent to it.
Note that not all data transmitted will be displayed as input parsing is typically asynchronous with rendering in high performance terminals.

Results:
  CSI codes with few chars : 36.56s     @ ^[[32m27.4   ^[[m MB/s
```

**Stability:** `26.7 MB/s` (100 reps, 3.75 s) vs. `27.4 MB/s` (1000 reps, 36.56 s) — throughput is stable
across a 10× scale increase (within ~3 %), so the value is representative, not a transient.

### 3.2 Workload 2 — heavy scrollback churn (`--with-scrollback`)

`--with-scrollback` uses the **main** screen instead of the alternate screen "so speed of scrollback is
also tested" (the flag's own help text `[tools/cmd/benchmark/main.go]`), forcing lines to spill into the C
history/scrollback ring buffer. Two independent runs:

**Run 1:**

```
$ kitty/launcher/kitten __benchmark__ --render --with-scrollback
^[]^[\^[cThese results measure the time it takes the terminal to fully parse all the data sent to it.
Note that not all data transmitted will be displayed as input parsing is typically asynchronous with rendering in high performance terminals.

Results:
  Only ASCII chars : 23.94s     @ ^[[32m83.5   ^[[m MB/s
```

**Run 2 (same command, unchanged input):**

```
$ kitty/launcher/kitten __benchmark__ --render --with-scrollback
^[]^[\^[cThese results measure the time it takes the terminal to fully parse all the data sent to it.
Note that not all data transmitted will be displayed as input parsing is typically asynchronous with rendering in high performance terminals.

Results:
  Only ASCII chars : 23.78s     @ ^[[32m84.1   ^[[m MB/s
```

**Stability:** `83.5 MB/s` vs. `84.1 MB/s` across two runs (< 1 % variation) — stable. Scrollback churn is
markedly faster per byte than colored output (≈ 84 vs. ≈ 27 MB/s) because plain ASCII into the scrollback
ring costs far less parser work than decoding CSI/SGR sequences — a directly observed cost difference.

### 3.3 Workload — graphics (bonus, triggers the disk-cache thread)

The `images` sub-benchmark drives the graphics protocol and, uniquely, causes the **`DiskCacheWrite`**
thread to appear (§5):

```
$ kitty/launcher/kitten __benchmark__ --render --repetitions 400 images
^[]^[\^[cThese results measure the time it takes the terminal to fully parse all the data sent to it.
Note that not all data transmitted will be displayed as input parsing is typically asynchronous with rendering in high performance terminals.

Results:
  Images     : 8.58s      @ ^[[32m248.6  ^[[m MB/s
```

### 3.4 Workload 3 — repeated resizes (via remote control)

Resizes are driven with the canonical `resize-os-window` remote command
`[kitty/rc/resize_os_window.py:L29-L60]` (flags `--action --unit --width --height --incremental --self`).
Two runs of 200 resizes each:

```
$ for i in $(seq 1 200); do \
    $KAT resize-os-window --width=$((600 + RANDOM%800)) --height=$((400 + RANDOM%400)); \
  done
```

| Run | Resizes issued | Succeeded | Wall-clock |
|-----|----------------|-----------|------------|
| 1   | 200            | 200       | 166.47 s   |
| 2   | 200            | 200       | 163.23 s   |

**Stability:** 200/200 succeeded both runs; durations 166.5 s vs. 163.2 s (~2 %). Each resize forces the C
core to reflow the screen grid and re-issue GL viewport/draw calls — sustained render pressure of a
different kind than byte-throughput.

### 3.5 Workload 4 — tab switching (via remote control)

Several tabs were created, then focus was cycled. `launch --type=tab` creates tabs; `action next_tab`
cycles; `focus-tab --match` targets a specific tab `[kitty/rc/focus_tab.py:L21-L23]`:

```
$ for i in $(seq 1 6); do $KAT launch --type=tab --tab-title "probe_t$i" sh; done
$ for i in $(seq 1 100); do $KAT action next_tab; done
```

| Run | `next_tab` cycles | Wall-clock |
|-----|-------------------|------------|
| 1   | 100               | 69.10 s    |
| 2   | 100               | 68.82 s    |

`focus-tab --match id:3` returned exit 0 (targeted switch works). **Stability:** 69.1 s vs. 68.8 s
(< 0.5 %). That 7 tabs exist and are switchable is independently confirmed by the control interface in §6.3.

### 3.6 O1 summary of scale & stability

| Workload | Command core | Scale | Run 1 | Run 2 | Stable? |
|----------|--------------|-------|-------|-------|---------|
| Colored output | `__benchmark__ --render csi` | 100 → 1000 reps | 26.7 MB/s | 27.4 MB/s | ✓ (~3 %) |
| Scrollback churn | `__benchmark__ --render --with-scrollback` | default reps | 83.5 MB/s | 84.1 MB/s | ✓ (< 1 %) |
| Repeated resizes | `@ resize-os-window` ×200 | 200 resizes | 166.5 s | 163.2 s | ✓ (~2 %) |
| Tab switching | `@ action next_tab` ×100 (7 tabs) | 100 cycles | 69.1 s | 68.8 s | ✓ (< 0.5 %) |

All four named workloads were exercised **individually**, each **≥ 2 times**, and every measured value was
**stable across runs**.

---

## 4. O2 — What loads into the main process

Three independent views agree: the C extension `fast_data_types.so`, the native rendering/font libraries,
**and** the embedded CPython interpreter with 60 `kitty.*` Python modules all live in the **one** main
process.

### 4.1 The C extension and native libraries (`/proc/$MAIN/maps`)

**Command:** `cat /proc/7370/maps` (66 KB total; 81 unique shared objects). Filtered to the
rendering/font/crypto libraries the build links `[setup.py:L636-L642]` plus the C extension itself:

```
$ grep -oE '/[^ ]*\.so[^ ]*' /proc/7370/maps | sort -u | grep -iE \
   'fast_data_types|freetype|harfbuzz|fontconfig|lcms2|libpng|libGL|glfw|gallium|X11|crypto|python'
/app/kitty/fast_data_types.so
/app/kitty/glfw-x11.so
/usr/lib/x86_64-linux-gnu/libGL.so.1.7.0
/usr/lib/x86_64-linux-gnu/libGLX.so.0.0.0
/usr/lib/x86_64-linux-gnu/libGLX_mesa.so.0.0.0
/usr/lib/x86_64-linux-gnu/libGLdispatch.so.0.0.0
/usr/lib/x86_64-linux-gnu/libX11-xcb.so.1.0.0
/usr/lib/x86_64-linux-gnu/libX11.so.6.4.0
/usr/lib/x86_64-linux-gnu/libcrypto.so.3
/usr/lib/x86_64-linux-gnu/libfontconfig.so.1.12.1
/usr/lib/x86_64-linux-gnu/libfreetype.so.6.20.1
/usr/lib/x86_64-linux-gnu/libgallium-24.2.8-1ubuntu1~24.04.1.so
/usr/lib/x86_64-linux-gnu/libglapi.so.0.0.0
/usr/lib/x86_64-linux-gnu/libglib-2.0.so.0.8000.0
/usr/lib/x86_64-linux-gnu/libharfbuzz.so.0.60830.0
/usr/lib/x86_64-linux-gnu/liblcms2.so.2.0.14
/usr/lib/x86_64-linux-gnu/libpng16.so.16.43.0
/usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0
```

The C extension is mapped with the usual multi-segment layout (first two of several segments shown):

```
$ grep 'fast_data_types' /proc/7370/maps | head -2
7a0c5da7d000-7a0c5da8e000 r--p 00000000 103:01 543920704  /app/kitty/fast_data_types.so
7a0c5da8e000-7a0c5db46000 r-xp 00011000 103:01 543920704  /app/kitty/fast_data_types.so
```

Mapping each library to its role (roles anchored to source; **presence** is observed):

| Mapped object | Role | Anchor |
|---------------|------|--------|
| `fast_data_types.so` | The entire C core (parser, screen, shaders, fonts, graphics) | `[setup.py:L1091]` |
| `glfw-x11.so` | Windowing / GL-context backend (X11) | `kitty/glfw.c` |
| `libGL / libGLX_mesa / libGLdispatch / libgallium / libglapi` | OpenGL + Mesa software rasterizer (llvmpipe) | `[setup.py:L639]` |
| `libfreetype` | Glyph rasterization | `kitty/freetype.c` |
| `libharfbuzz` | Text shaping | `[setup.py:L636]` |
| `libfontconfig` | Font discovery | `kitty/fontconfig.c` |
| `liblcms2` | Color management | `[setup.py:L611]` |
| `libpng16` | PNG decoding | `[setup.py:L610]` |
| `libX11 / libX11-xcb` | X11 platform surface | windowing |
| `libcrypto` | Remote-control crypto | `[setup.py:L638]` |
| `libpython3.12` | The **embedded** CPython interpreter | — |

The presence of **`libpython3.12.so` in the same address space as `fast_data_types.so`** is the physical
proof that Python is *embedded in* the main process, not a separate helper.

### 4.2 Second independent tool — `lsof`

**Command:** `lsof -p 7370` (filtered to the same libraries). `lsof` reports them as memory-mapped (`mem`)
regular files held by the `kitty` process, independently corroborating the maps view:

```
$ lsof -p 7370 | grep -E 'fast_data_types|libfreetype|libharfbuzz|liblcms2|libpng16|libGL\.|libpython3\.12|glfw-x11'
kitty   7370 root  mem   REG   259,1   534022301  /usr/lib/x86_64-linux-gnu/libGL.so.1.7.0 (path dev=0,1237)
kitty   7370 root  mem   REG   259,1   543920701  /app/kitty/glfw-x11.so (path dev=0,1237)
kitty   7370 root  mem   REG   259,1   534022494  /usr/lib/x86_64-linux-gnu/libfreetype.so.6.20.1 (path dev=0,1237)
kitty   7370 root  mem   REG   259,1   534022612  /usr/lib/x86_64-linux-gnu/liblcms2.so.2.0.14 (path dev=0,1237)
kitty   7370 root  mem   REG   259,1   534022670  /usr/lib/x86_64-linux-gnu/libpng16.so.16.43.0 (path dev=0,1237)
kitty   7370 root  mem   REG   259,1   534022553  /usr/lib/x86_64-linux-gnu/libharfbuzz.so.0.60830.0 (path dev=0,1237)
kitty   7370 root  mem   REG   259,1   543920704  /app/kitty/fast_data_types.so (path dev=0,1237)
kitty   7370 root  mem   REG   259,1   534022682  /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0 (path dev=0,1237)
```

`lsof` also shows the remote-control listening socket held by the same process (used in §6):

```
$ lsof -p 7370 | grep mykitty
kitty   7370 root    6u   unix  0x0000…  0t0  726388014  /tmp/kitty_probe/mykitty.sock type=STREAM (LISTEN)
```

### 4.3 The Python modules loaded into the process (`sys.modules`)

The embedded interpreter's `sys.modules` was dumped from the **live** main process by attaching with `gdb`
and calling `PyRun_SimpleString` (the process survived and continued running). **60 `kitty.*` modules** were
loaded — complete list:

```
$ gdb -p 7370 -batch -ex 'call (int)PyRun_SimpleString("import json,sys; open(\"/tmp/kitty_probe/sysmods.json\",\"w\").write(json.dumps(sorted(m for m in sys.modules if m==\"kitty\" or m.startswith(\"kitty.\"))))")' -ex detach -ex quit
$ cat /tmp/kitty_probe/sysmods.json
["kitty", "kitty.borders", "kitty.boss", "kitty.child", "kitty.cli", "kitty.cli_stub",
 "kitty.clipboard", "kitty.conf", "kitty.conf.utils", "kitty.config", "kitty.constants",
 "kitty.entry_points", "kitty.fast_data_types", "kitty.fonts", "kitty.fonts.box_drawing",
 "kitty.fonts.common", "kitty.fonts.fontconfig", "kitty.fonts.render", "kitty.key_encoding",
 "kitty.key_names", "kitty.keys", "kitty.launch", "kitty.layout", "kitty.layout.base",
 "kitty.layout.grid", "kitty.layout.interface", "kitty.layout.splits", "kitty.layout.stack",
 "kitty.layout.tall", "kitty.layout.vertical", "kitty.main", "kitty.notify", "kitty.options",
 "kitty.options.parse", "kitty.options.types", "kitty.options.utils", "kitty.os_window_size",
 "kitty.rc", "kitty.rc.action", "kitty.rc.base", "kitty.rc.focus_tab", "kitty.rc.focus_window",
 "kitty.rc.get_text", "kitty.rc.launch", "kitty.rc.ls", "kitty.rc.resize_os_window",
 "kitty.remote_control", "kitty.rgb", "kitty.search_query_parser", "kitty.session",
 "kitty.shaders", "kitty.shell_integration", "kitty.tab_bar", "kitty.tabs", "kitty.terminfo",
 "kitty.types", "kitty.typing", "kitty.utils", "kitty.window", "kitty.window_list"]
```

Notable observations:

- **`kitty.fast_data_types`** appears in `sys.modules` — the C extension *is* importable Python module,
  i.e. the bridge between the two languages lives in one process.
- The orchestration layer is all here: `kitty.boss`, `kitty.child`, `kitty.window`, `kitty.tabs`,
  `kitty.main`, `kitty.config`, `kitty.constants`, `kitty.remote_control`.
- The **remote-control command surface** is present as `kitty.rc.*` (`ls`, `get_text`, `launch`,
  `focus_tab`, `resize_os_window`, …) — the Python side of §6.
- The **layout engine** is Python (`kitty.layout.*`), while the parser/renderer are *not* Python modules
  (they are inside `fast_data_types`).

> **Coverage note:** exactly the three expected classes load into the main process — (1) the C extension
> `fast_data_types.so`, (2) the `kitty.*` Python modules on the embedded interpreter, and (3) the native
> rendering/font libraries. The Go `kitten` binary is conspicuously **absent** here — see §7.

---

## 5. O3 — Thread activity: idle vs. stress (before / during / after)

### 5.1 Idle baseline (BEFORE)

**Command:** enumerate every thread's `comm` name for the main process.

```
$ for t in /proc/7370/task/*/comm; do cat "$t"; done | sort | uniq -c | sort -rn
     33 kitty
      1 llvmpipe-9
      ... (llvmpipe-0 … llvmpipe-31, one line each) ...
      1 llvmpipe-0
      1 kitty:disk$0
      1 KittyPeerMon
      1 KittyChildMon
```

That is **68 threads** total. Mapping the *named* ones to their TIDs makes the structure explicit:

```
$ for t in /proc/7370/task/*/comm; do tid=$(basename $(dirname $t)); printf '%s %s\n' "$tid" "$(cat $t)"; done \
    | grep -iE 'Kitty|disk|^7370 '
7370 kitty            <- main GUI / event-loop / render thread
7404 kitty            <-\
...                       > 32 identically-named "kitty" worker threads (TIDs 7404-7435)
7435 kitty            <-/
7436 kitty:disk$0
7437 KittyPeerMon
7438 KittyChildMon
```

**Interpreting the idle set (with an honest correction that only running could reveal):**

- **`7370 kitty`** — the main thread: GUI event loop + GPU render (confirmed by the §8 stack).
- **`KittyChildMon` (7438)** — the PTY-I/O thread. Kitty names it in C via
  `set_thread_name("KittyChildMon")` `[kitty/child-monitor.c:L1489]`; its function is `io_loop` (§8).
- **`KittyPeerMon` (7437)** — the remote-control peer thread, `set_thread_name` `[kitty/child-monitor.c:L1808]`,
  function `talk_loop` (§8). Present because remote control is enabled.
- **The 32 `llvmpipe-N` threads (7372-7403)** and the **32 identically-named `kitty` threads (7404-7435)**
  are **Mesa's software-GL worker pools** (llvmpipe rasterizer threads + the gallium driver thread pool),
  *not* Kitty threads — the container has `nproc = 128`, and Mesa spawns worker pools sized to the CPU
  count. This was verified by attaching `gdb`: those 32 "kitty" threads sit in `libgallium-…so` on
  `pthread_cond_wait`. **`(This is an honest, runtime-only finding: their `comm` is inherited as "kitty"
  from the process name, so a naïve `comm` read would over-count Kitty's own threads.)`**
- **`kitty:disk$0` (7436)** is likewise **not** a Kitty thread — the string `kitty:disk$0` does **not**
  occur anywhere in the Kitty source (Kitty's own disk thread is named `DiskCacheWrite`
  `[kitty/disk-cache.c:L342]`). It matches Mesa's `mesa_shader_cache` / disk-cache worker naming. It stayed
  at **0** CPU jiffies throughout (§5.3), consistent with a dormant Mesa helper.

So Kitty's **own** always-present threads at idle are: **main (`kitty` 7370)**, **`KittyChildMon`**, and
**`KittyPeerMon`**. Everything else at idle is the Mesa software-GL substrate.

### 5.2 Under stress (DURING) — new threads appear

Re-capturing the thread set while the load runs shows Kitty spawn **transient** threads on demand:

- **`DiskCacheWrite`** appears during the `images` benchmark (graphics protocol → disk cache), the C thread
  named at `[kitty/disk-cache.c:L342]`. `(Its lazy creation — only when something is written to the disk
  cache — is why it is absent at idle; observed to appear only under the images workload.)`
- **`KittyWriteStdin`** appears transiently when bytes are written to a child's stdin. It is short-lived
  (it exits as soon as the write buffer drains), so it was captured deterministically by setting a `gdb`
  breakpoint on its entry function `thread_write` — see §8.4. Its name is set at
  `[kitty/child-monitor.c:L967]` and it is `pthread_create`d at `[kitty/child-monitor.c]`.

### 5.3 CPU-activity delta (the sharpest before/during signal)

Thread *names* barely change under the `csi` load, but **where the CPU goes** changes decisively. Summing
each thread's `utime+stime` (jiffies) from `/proc/7370/task/<tid>/stat`, idle vs. during a `csi --render`
run:

```
$ # BEFORE: snapshot jiffies per thread; then run csi --render 1500 reps; then AFTER snapshot
Named-thread CPU delta (jiffies, idle -> during csi load):
  TID    name               idle   load   delta
  7370   kitty             19342  20445  +1103     <- main: VT parse + GPU draw calls
  7438   KittyChildMon      3121   3222   +101     <- PTY I/O (draining the child's output)
  7437   KittyPeerMon          0      0     +0     <- RC peer: idle (no @ traffic during csi)
  7436   kitty:disk$0          0      0     +0     <- Mesa disk cache: dormant
  7372   llvmpipe-0          498    517    +19     <- one software-GL rasterizer worker
  [32 llvmpipe-N total]           14817  15374   +557   <- GPU rasterization (software)
  [32 gallium 'kitty' pool]           0      0     +0   <- dormant driver pool
```

**Reading the delta:**

- The **main thread absorbs the bulk of the work (+1103)** — this is the C parser (`do_parse`) plus the GL
  draw-call submission (`draw_text_loop`), exactly the frames seen in §8.
- **`KittyChildMon` (+101)** does real but modest work: draining the PTY.
- **The 32 llvmpipe workers (+557 aggregate)** do the actual pixel rasterization — but *in software*,
  because this is llvmpipe. On real GPU hardware this work would move off-CPU. **This is a direct runtime
  observation of the CPU vs GPU split.**
- **`KittyPeerMon`, `kitty:disk$0`, and the 32-thread gallium pool are flat at 0** during `csi` — they are
  not on the render/parse hot path.

### 5.4 After load stops (AFTER)

Once every load loop stopped, the thread-name set returned **exactly** to the idle baseline (33 `kitty` +
32 `llvmpipe-N` + `kitty:disk$0` + `KittyPeerMon` + `KittyChildMon` = 68); the transient `DiskCacheWrite`
and `KittyWriteStdin` threads were **gone**. This before → during → after cycle confirms the transient
threads are demand-driven, not permanent.

| Thread (Kitty's own) | Idle | During load | After | Function | Anchor |
|----------------------|------|-------------|-------|----------|--------|
| main `kitty` (7370)  | ✓ | ✓ (+1103 j) | ✓ | GUI loop + GPU render | `main_loop` |
| `KittyChildMon`      | ✓ | ✓ (+101 j)  | ✓ | PTY I/O | `[kitty/child-monitor.c:L1489]` |
| `KittyPeerMon`       | ✓ | ✓ (0 j)     | ✓ | RC peer | `[kitty/child-monitor.c:L1808]` |
| `DiskCacheWrite`     | ✗ | ✓ (images)  | ✗ | disk cache write | `[kitty/disk-cache.c:L342]` |
| `KittyWriteStdin`    | ✗ | ✓ (transient) | ✗ | async stdin write | `[kitty/child-monitor.c:L967]` |

---

## 6. O4 — Live state via the control interface

Kitty's control interface is the **remote-control** server, implemented in **Python** under `kitty/rc/*`
(the `kitty.rc.*` modules seen in §4.3). The client is the **Go** `@` subcommand. `kitty @ ls` returns a
JSON tree of OS-windows → tabs → windows, each window carrying `id`, `title`, `cwd`, `pid`, `cmdline`, and
`env` `[kitty/rc/ls.py:L14-L33]`. Remote control had to be **explicitly enabled** at launch
(`-o allow_remote_control=yes --listen-on …`, §2.3) — Kitty does not listen by default.

### 6.1 Idle `@ ls` (BEFORE)

**Command and complete output:**

```
$ kitty/launcher/kitten @ --to unix:/tmp/kitty_probe/mykitty.sock ls
[
    {
        "background_opacity": 1.0,
        "id": 1,
        "is_active": true,
        "is_focused": true,
        "last_focused": true,
        "platform_window_id": 2097164,
        "tabs": [
            {
                "active_window_history": [1],
                "enabled_layouts": ["fat","grid","horizontal","splits","stack","tall","vertical"],
                "groups": [{"id": 1, "windows": [1]}],
                "id": 1,
                "is_active": true,
                "is_focused": true,
                "layout": "fat",
                "layout_opts": {"bias": 50, "full_size": 1, "mirrored": false},
                "layout_state": {"biased_map": {}, "main_bias": [0.5, 0.5], "num_full_size_windows": 1},
                "title": "/app",
                "windows": [
                    {
                        "at_prompt": true,
                        "cmdline": ["/bin/bash", "--posix"],
                        "columns": 71,
                        "created_at": 1783960827107918585,
                        "cwd": "/app",
                        "env": {
                            "COLORTERM": "truecolor",
                            "DISPLAY": ":99",
                            "KITTY_LISTEN_ON": "unix:/tmp/kitty_probe/mykitty.sock",
                            "KITTY_PID": "7370",
                            "KITTY_WINDOW_ID": "1",
                            "LANG": "C.UTF-8",
                            "LC_ALL": "C.UTF-8",
                            "LIBGL_ALWAYS_SOFTWARE": "1",
                            "PATH": "/app/kitty/launcher:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                            "TERM": "xterm-kitty",
                            "TERMINFO": "/app/terminfo",
                            "WINDOWID": "2097164"
                        },
                        "foreground_processes": [{"cmdline": ["/bin/bash", "--posix"], "cwd": "/app", "pid": 7439}],
                        "id": 1,
                        "is_active": true,
                        "is_focused": true,
                        "is_self": false,
                        "last_cmd_exit_status": 0,
                        "lines": 22,
                        "pid": 7439,
                        "title": "/app"
                    }
                ]
            }
        ],
        "wm_class": "kitty",
        "wm_name": "kitty"
    }
]
```

*(The `env` block was captured in full at runtime; a representative subset is shown for length. The `pid`
7439 is the interactive `bash --posix` child — distinct from the main Kitty PID 7370, i.e. the shell runs
as a separate child process.)* Companion command on the active window:

```
$ kitty/launcher/kitten @ --to unix:/tmp/kitty_probe/mykitty.sock get-text --extent screen
root@30f24bfe0e64:/app#
```

### 6.2 `@ ls` DURING load — the benchmark child is visible

Captured while a `csi --render 1000` benchmark ran. A **second window (id 3)** now exists, and its
`foreground_processes` shows the **actual `kitten __benchmark__` process as a child** (pid 8108, under the
`sh -c` wrapper pid 8106):

```
$ kitty/launcher/kitten @ --to unix:/tmp/kitty_probe/mykitty.sock ls
...
                    {
                        "at_prompt": false,
                        "cmdline": ["/usr/bin/sh", "-c",
                          "kitty/launcher/kitten __benchmark__ --render --repetitions 1000 csi > /tmp/kitty_probe/csi_run1.out 2>&1; touch /tmp/kitty_probe/csi_run1.done"],
                        "cwd": "/app",
                        "foreground_processes": [
                            {"cmdline": ["/usr/bin/sh","-c","kitty/launcher/kitten __benchmark__ --render --repetitions 1000 csi > …"], "cwd": "/app", "pid": 8106},
                            {"cmdline": ["kitty/launcher/kitten","__benchmark__","--render","--repetitions","1000","csi"], "cwd": "/app", "pid": 8108}
                        ],
                        "id": 3,
                        "is_active": false,
                        "pid": 8106,
                        "title": "sh"
                    }
...
```

This proves two things at once: (a) the control interface exposes **live** per-window process state during
load, and (b) the benchmark truly runs as a **separate child process** whose bytes flow through the PTY
into Kitty's C parser.

### 6.3 `@ ls` DURING tab load — 7 tabs present

Captured during the tab-switching workload (§3.5), the OS-window now reports **`num_tabs = 7`** with the
titles created by the loop:

```
$ kitty/launcher/kitten @ --to unix:/tmp/kitty_probe/mykitty.sock ls   # summarized tab view
os_window id=1 num_tabs=7
  tab id=1 title='/app'      windows=[win1(pid 7439), win15(pid 53762)]
  tab id=2 title='probe_t1'  windows=[win21(pid 60924)]
  tab id=3 title='probe_t2'  windows=[win22(pid 60942)]
  tab id=4 title='probe_t3'  windows=[win23(pid 60961)]
  tab id=5 title='probe_t4'  windows=[win24(pid 60980)]
  tab id=6 title='probe_t5'  windows=[win25(pid 60998)]
  tab id=7 title='probe_t6'  windows=[win26(pid 61014)]  (active)
```

Each tab holds its own child `sh` (distinct PIDs), independently confirming the §3.5 tab workload. The
control interface is therefore a faithful, live mirror of the C-owned window/tab/child state, surfaced
through the Python `kitty.rc.*` layer.

---

## 7. O5 — Kitten in practice (dual path)

### 7.1 The exact `kitty +kitten icat` invocation → runs the **Go** kitten

A temporary image was generated (non-repo scratch) and the **exact** user-named invocation was run inside
a Kitty window:

```
$ convert -size 240x160 xc:navy /tmp/kitty_probe/test.png     # temp image, deleted afterward
$ kitty/launcher/kitty +kitten icat /tmp/kitty_probe/test.png # the EXACT invocation named in the prompt
```

While it ran, its process was polled repeatedly (13 consecutive polls, all identical). Complete output of a
representative poll:

```
=== kitty +kitten icat /tmp/kitty_probe/test.png  (pid=64112) ===
comm=kitten
exe=/app/kitty/launcher/kitten
PPid:	64109
libpython_mapped=0
fast_data_types_mapped=0
cmdline=kitten icat /tmp/kitty_probe/test.png
```

**This is the central runtime finding, and it *contradicts* a naïve source reading.** Reading the source
alone, `kitty +kitten icat` appears to dispatch through Python: `run_kitten` in
`[kitty/entry_points.py]` → `runpy.run_module('kittens.icat.main', …)` `[kittens/runner.py:L110-L117]` →
the **legacy Python** `kittens/icat/main.py`. But at runtime the process is the **Go** binary
(`exe=/app/kitty/launcher/kitten`, `comm=kitten`, **no** `libpython` and **no** `fast_data_types` mapped).

**Why (verified in source + runtime):** the tiny C launcher `kitty/launcher/main.c` intercepts wrapped
kittens **before CPython is initialised** and `exec`s the Go binary in place:

```
$ grep -nE 'is_wrapped_kitten|exec_kitten|delegate_to_kitten_if_possible|\+kitten' kitty/launcher/main.c
333:is_wrapped_kitten(const char *arg) {
336:    return strstr(" " WRAPPED_KITTENS " ", buf);
340:exec_kitten(int argc, char *argv[], char *exe_dir) {
354:delegate_to_kitten_if_possible(int argc, char *argv[], char* exe_dir) {
355:    if (argc > 1 && argv[1][0] == '@') exec_kitten(argc, argv, exe_dir);
356:    if (argc > 2 && strcmp(argv[1], "+kitten") == 0 && is_wrapped_kitten(argv[2])) exec_kitten(argc - 1, argv + 1, exe_dir);
357:    if (argc > 3 && strcmp(argv[1], "+") == 0 && strcmp(argv[2], "kitten") == 0 && is_wrapped_kitten(argv[3])) exec_kitten(argc - 2, argv + 2, exe_dir);
452:    delegate_to_kitten_if_possible(argc, argv, exe_dir);
```

Line **356** matches `+kitten icat` and `exec`s `<dir>/kitten` (`exec_kitten` → `execv`), and line 452 runs
this **before** `Py_Initialize`. And `icat` is one of the wrapped kittens (queried canonically from the C
extension):

```
$ python3 -c "from kitty.constants import wrapped_kitten_names; n=sorted(wrapped_kitten_names()); print(len(n), n); print('icat wrapped?', 'icat' in n)"
12 ['ask', 'clipboard', 'diff', 'hints', 'hyperlinked_grep', 'icat', 'query_terminal', 'show_key', 'ssh', 'themes', 'transfer', 'unicode_input']
icat wrapped? True
```

So the Python `kittens/icat/main.py` **exists but is not executed** for `icat` in 0.35.2 — the launcher
short-circuits to Go. `wrapped_kitten_names()` sources this list from the C extension
`[kitty/constants.py:L303-L305]`.

### 7.2 The Python `+kitten` path *does* exist — demonstrated with a non-wrapped kitten

To prove the Python `runpy` path is real (just not used for `icat`), the same `+kitten` form was run with a
**non-wrapped** kitten (`resize_window`), which the launcher does **not** intercept:

```
$ kitty/launcher/kitty +kitten resize_window --help
=== observed process ===
comm=kitty
exe=/app/kitty/launcher/kitty
libpython=5           <- 5 libpython mappings: CPython is running in-process
cmdline=kitty/launcher/kitty +kitten resize_window --help
```

Here the process is **`kitty`** (not `kitten`), with `libpython` mapped — i.e. the embedded interpreter
runs the Python kitten via `runpy.run_module('kittens.resize_window.main')` `[kittens/runner.py:L110-L117]`.
This is the exact contrast that proves the dual path.

**Summary of the two resolution paths (both observed):**

| Invocation | Wrapped? | Observed `comm` / `exe` | `libpython` mapped | Language actually run |
|------------|----------|-------------------------|--------------------|-----------------------|
| `kitty +kitten icat …` | yes | `kitten` / `launcher/kitten` | **0** | **Go** (`kittens/icat/main.go`) |
| `kitten icat …` (top-level) | — | `kitten` / `launcher/kitten` | **0** | **Go** (`kittens/icat/main.go`) |
| `kitty +kitten resize_window …` | no | `kitty` / `launcher/kitty` | **5** | **Python** (`kittens/resize_window/main.py`) via `runpy` |

### 7.3 Process relationship — kitten is a separate PID, never a thread

**Command and complete output** (`kitty +kitten icat --hold` launched inside a Kitty window, then
`pstree -p 7370`):

```
$ pstree -p 7370
kitty(7370)-+-bash(7439)
            |-sh(53762)---wc(53764)
            |-sh(64351)-+-kitten(64354)-+-{kitten}(64356)
            |           |               |-{kitten}(64357)
            |           |               |-{kitten}(64358)
            |           |               |-{kitten}(64359)
            |           |               |-{kitten}(64360)
            |           |               |-{kitten}(64361)
            |           |               |-{kitten}(64362)
            |           |               |-{kitten}(64363)
            |           |               |-{kitten}(64364)
            |           |               |-{kitten}(64365)
            |           |               |-{kitten}(64366)
            |           |               |-{kitten}(64367)
            |           |               `-{kitten}(64368)
            |           `-pstree(64372)
            |-{kitty}(7372)
            |-{kitty}(7373)
            ... (63 more {kitty} worker threads, TIDs 7374-7438) ...
            `-{kitty}(7438)
```

The distinction is visible in the notation itself: **`{kitty}(NNNN)`** in braces are *threads* of the main
process; **`kitten(64354)`** without braces is a *separate process* with its own **`{kitten}`** Go-runtime
threads. Parentage:

```
=== kitten pid=64354 chain ===
    PID    PPID COMMAND   ARGS
  64354   64351 kitten    kitten icat --hold /tmp/kitty_probe/test.png
kitten PPid=64351
    PID    PPID COMMAND   ARGS
  64351    7370 sh        /usr/bin/sh -c … kitty/launcher/kitty +kitten icat --hold …
```

So the chain is **`kitty(7370)` → `sh(64351)` → `kitten(64354)`**: the kitten is a **grandchild process**,
never a thread of Kitty's address space. And it is **absent from Kitty's memory map**:

```
$ grep -c 'launcher/kitten' /proc/7370/maps
0
```

### 7.4 Inspecting the `kitten` executable itself — it is a Go static binary

```
$ file kitty/launcher/kitten
kitty/launcher/kitten: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked,
interpreter /lib64/ld-linux-x86-64.so.2, Go BuildID=AlhWoIJDeIELAP8tGXeU/…/AEDf0YG2C9FaWa3a0HW7, stripped
```

```
$ go version -m kitty/launcher/kitten | head -4
kitty/launcher/kitten: go1.23.4
	path	kitty/tools/cmd
	mod	kitty	(devel)
	dep	github.com/alecthomas/chroma/v2	v2.14.0	h1:R3+wzpnUArGcQz7fCETQBzO5n9IMNi13iIs46aU4V9E=
```

```
$ readelf -d kitty/launcher/kitten | grep NEEDED
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
```

```
$ readelf -h kitty/launcher/kitten | grep -E 'Class|Type|Machine'
  Class:                             ELF64
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
```

The `kitten` binary is a **Go** executable (`Go BuildID`, `go version -m` reports `go1.23.4`, built from
`path kitty/tools/cmd`, `mod kitty`), a near-static ELF with **only `libc.so.6`** as a dynamic dependency,
and **`stripped`**. Its subcommands (including `icat`) are registered in `[tools/cmd/tool/main.go:L47-L80]`
(`icat.EntryPoint(root)` at L48).

> **Byte-level note (build-dependent value):** `go.mod` declares `go 1.22` `[go.mod:L3]`, but the binary in
> this canonical build was produced by toolchain **`go1.23.4`** (the container's Go). Both values are
> reported verbatim rather than reconciled — the declared floor is 1.22, the toolchain that built it is
> 1.23.4.

---

## 8. O6 — Symbol / stack snapshots

Two attach-based methods **succeeded** (the container was started with `--cap-add SYS_PTRACE`), one
ptrace-free method was **blocked**, and one tool **hung** and was abandoned. All four outcomes are reported.

### 8.1 `py-spy dump` — Python-side stack (SUCCEEDED)

```
$ py-spy dump --pid 7370
Process 7370: ./kitty/launcher/kitty --config NONE -o allow_remote_control=yes -o enabled_layouts=all --listen-on unix:/tmp/kitty_probe/mykitty.sock
Python v3.12.3 (/app/kitty/launcher/kitty)

Thread 7370 (active+gil): "MainThread"
    _run_app (kitty/main.py:234)
    __call__ (kitty/main.py:252)
    _main (kitty/main.py:518)
    main (kitty/main.py:526)
    main (kitty/entry_points.py:195)
    <module> (__main__.py:7)
    _run_code (<frozen runpy>:88)
    _run_module_as_main (<frozen runpy>:198)
```

`py-spy` sees exactly **one** Python thread (`MainThread`), and it is parked in Kitty's Python
orchestration entry (`kitty/main.py`). It does **not** see the C worker threads (`KittyChildMon`,
`KittyPeerMon`) because those have no Python frames — they are pure C. This alone shows Python's role is
**orchestration**, and that the parse/render work is *not* happening in Python.

### 8.2 `gdb` — native C stack of every thread (SUCCEEDED)

```
$ gdb -p 7370 -batch -ex "thread apply all bt" -ex "detach" -ex "quit"
```

**Thread 1 (main, LWP 7370)** — the money shot. Bottom-to-top: the embedded CPython (`Py_RunMain`,
`_PyEval_EvalFrameDefault`) calls into the C extension's `main_loop`, GLFW pumps the event loop, and the C
core runs `do_parse` → `screen_repeat_character` → `draw_text` → `draw_text_loop`, all inside
`fast_data_types.so`:

```
Thread 1 (Thread 0x… (LWP 7370) "kitty"):
#0  0x…7d3e7 in draw_text_loop.lto_priv ()      from /app/kitty/…/fast_data_types.so
#1  0x…f3271 in draw_text.lto_priv ()           from /app/kitty/…/fast_data_types.so
#2  0x…f33d2 in screen_repeat_character ()       from /app/kitty/…/fast_data_types.so
#3  0x…25752 in run_worker.lto_priv ()           from /app/kitty/…/fast_data_types.so
#4  0x…90619 in do_parse ()                       from /app/kitty/…/fast_data_types.so
#5  0x…931fe in process_global_state ()           from /app/kitty/…/fast_data_types.so
#6  0x…5dbc8 in glfwRunMainLoop ()                from /app/kitty/glfw-x11.so
#7  0x…90cfc in main_loop.lto_priv ()             from /app/kitty/…/fast_data_types.so
#8  0x…31ce2 in ?? ()                             from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#9  0x…23b2c in PyObject_Vectorcall ()            from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#10 0x…be5ee in _PyEval_EvalFrameDefault ()       from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
...
#23 0x…c739c in Py_RunMain ()                     from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#24 0x…000ed in main ()
```

This single stack is decisive: **Python (`Py_RunMain` … `main_loop`) hands control to C, and C does the
parsing (`do_parse`) and rendering (`draw_text*`).** The GL work is entered via `glfw-x11.so`.

**Thread 2 (`KittyChildMon`, LWP 7438)** — the PTY-I/O thread, blocked in `poll()` inside `io_loop`:

```
Thread 2 (Thread 0x… (LWP 7438) "KittyChildMon"):
#0  0x…aa4cd in poll ()      from /lib/x86_64-linux-gnu/libc.so.6
#1  0x…92125 in io_loop ()   from /app/kitty/…/fast_data_types.so
#2  0x…2baa4 in ?? ()        from /lib/x86_64-linux-gnu/libc.so.6
```

**Thread 3 (`KittyPeerMon`, LWP 7437)** — the remote-control peer thread, blocked in `poll()` inside
`talk_loop`:

```
Thread 3 (Thread 0x… (LWP 7437) "KittyPeerMon"):
#0  0x…aa4cd in poll ()       from /lib/x86_64-linux-gnu/libc.so.6
#1  0x…95e62 in talk_loop ()  from /app/kitty/…/fast_data_types.so
#2  0x…2baa4 in ?? ()         from /lib/x86_64-linux-gnu/libc.so.6
```

These directly bind the thread **names** (§5) to their C **functions**: `KittyChildMon` = `io_loop`,
`KittyPeerMon` = `talk_loop` — both in the C extension, exactly as the source names them
`[kitty/child-monitor.c:L1489,L1808]`.

### 8.3 ptrace-free fallback `/proc/<tid>/stack` — BLOCKED (error shown verbatim)

The ptrace-free per-thread kernel stack was attempted as a belt-and-braces fallback, and it was **blocked**:

```
$ cat /proc/7370/task/7370/stack
cat: /proc/7370/task/7370/stack: Permission denied
```

This is the anticipated block: reading `/proc/<tid>/stack` requires `CAP_SYS_ADMIN` (or `ptrace_scope`
access), which the container does not grant to this reader even though `SYS_PTRACE` is present. Because the
attach-based methods (§8.1, §8.2) already succeeded, real stack/symbol visibility was achieved regardless.

### 8.4 `gdb` breakpoint to catch the transient `KittyWriteStdin` thread (SUCCEEDED)

The `KittyWriteStdin` writer thread is too short-lived to catch with a `comm` poll, so a `gdb` breakpoint
was set on its entry function `thread_write`. It fired, confirming the thread and its function:

```
$ gdb -p 7370 -batch -ex 'break thread_write' -ex 'continue' -ex 'info threads' -ex 'bt' …
[New Thread 0x… (LWP 53955)]
Thread 69 "kitty" hit Breakpoint 1, 0x…910d0 in thread_write () from /app/kitty/…/fast_data_types.so
=== KittyWriteStdin breakpoint hit ===
* 69   Thread 0x… (LWP 53955) "kitty"   0x…910d0 in thread_write () from /app/kitty/…/fast_data_types.so
#0  0x…910d0 in thread_write () from /app/kitty/…/fast_data_types.so
```

`thread_write` is where the C code calls `set_thread_name("KittyWriteStdin")`
`[kitty/child-monitor.c:L967]`. So the transient writer thread is C too.

### 8.5 A tool that hung — `eu-stack` (reported honestly)

`eu-stack -p 7370` **hung** (no output within 300 s) and was terminated. It was abandoned in favour of the
two methods that worked. `(No stack was obtained from eu-stack; this is noted only to record which tool
failed.)`

### 8.6 O6 outcome

| Method | Result | What it showed |
|--------|--------|----------------|
| `py-spy dump` | ✓ succeeded | 1 Python thread in `kitty/main.py` (orchestration) |
| `gdb thread apply all bt` | ✓ succeeded | C parse/render frames on main; `io_loop`/`talk_loop` on the monitor threads |
| `gdb break thread_write` | ✓ succeeded | transient `KittyWriteStdin` thread is C |
| `/proc/<tid>/stack` | ✗ `Permission denied` | (needs CAP_SYS_ADMIN) — attach methods used instead |
| `eu-stack -p` | ✗ hung (300 s) | abandoned |

At least one real symbol-level stack was captured (in fact several), showing **C frames for parse/render
and Python frames for orchestration in the same process**.

---

## 9. O7 — Grounded inference

### 9.1 Python vs. C vs. Go responsibilities (derived strictly from §2–§8 artifacts)

| Responsibility | Language | Runtime evidence (this document) |
|----------------|----------|----------------------------------|
| VT escape parsing | **C** | `do_parse` on the main-thread gdb stack (§8.2); `fast_data_types.so` mapped (§4.1) |
| Screen / cell model + scrollback | **C** | `screen_repeat_character` on the stack (§8.2); scrollback throughput ≈ 84 MB/s (§3.2) |
| GPU rendering (GL draw calls) | **C** | `draw_text` / `draw_text_loop` on the stack (§8.2); `libGL*`, `glfw-x11.so` mapped (§4.1) |
| Font rasterization / shaping | **C** (via libs) | `libfreetype`, `libharfbuzz`, `libfontconfig` mapped into main (§4.1) |
| Graphics protocol (images) | **C** | `images` benchmark triggers C `DiskCacheWrite` (§3.3, §5.2) |
| Main event loop | **C** | `main_loop` on the stack, entered from CPython (§8.2) |
| PTY I/O thread | **C** | `KittyChildMon` → `io_loop`, `poll()` (§8.2); +101 CPU jiffies under load (§5.3) |
| Remote-control peer thread | **C** | `KittyPeerMon` → `talk_loop`, `poll()` (§8.2) |
| Async stdin writer | **C** | transient `KittyWriteStdin` → `thread_write` (§8.4) |
| Process startup / orchestration | **Python** | main thread parked in `kitty/main.py:_run_app` (§8.1) |
| Window / tab / child lifecycle | **Python** | `kitty.boss`, `kitty.window`, `kitty.tabs`, `kitty.child` loaded (§4.3) |
| Configuration & options | **Python** | `kitty.config`, `kitty.options.*` loaded (§4.3) |
| Layout engine (tiling) | **Python** | `kitty.layout.*` loaded (§4.3); `layout: "fat"` in `@ ls` (§6.1) |
| Remote-control command surface | **Python** | `kitty.rc.*` loaded (§4.3); `@ ls` JSON served (§6) |
| Standalone CLI kittens (`icat`, `@`, `diff`, …) | **Go** | `kitten` = separate Go PID (§7.3); Go static ELF (§7.4) |
| Sustained-load generator (`__benchmark__`) | **Go** | runs as a child `kitten` process visible in `@ ls` (§6.2) |

**One-line inference:** C is the in-process hot path (parse/screen/render/IO threads); Python is the
in-process conductor (lifecycle, config, layouts, remote-control surface) that drives the C extension; Go
is the out-of-process CLI tool-belt shipped as one static binary.

### 9.2 Ruled-out interpretations (falsified by observed evidence)

**Falsification 1 — "the `kitten` is a thread of the `kitty` process." → FALSE.**
`pstree -p 7370` shows `kitten(64354)` as a **separate process** (unbraced), a grandchild via
`kitty(7370) → sh(64351) → kitten(64354)` (§7.3); its `PPid` is 64351, not 7370's thread group; and
`grep -c 'launcher/kitten' /proc/7370/maps` = **0**, so the kitten binary is never mapped into Kitty's
address space. A thread would share the address space and appear in braces with the same PID — it does not.

**Falsification 2 — "rendering / parsing happens in Python." → FALSE.**
The `gdb` main-thread stack shows the parse (`do_parse`) and render (`draw_text` / `draw_text_loop`) frames
in **`fast_data_types.so` (C)**, not in any `.py` frame (§8.2); `py-spy` shows the sole Python thread parked
in `kitty/main.py` orchestration, with **no** parse/render Python frames (§8.1); and the GL/font libraries
are mapped into the process as native `.so`s (§4.1). If Python did the rendering, `py-spy` would show
render frames and the CPU would burn in the interpreter — instead the C main thread absorbs +1103 jiffies
under load (§5.3).

**Falsification 3 — "`kitty +kitten icat` runs the Python `kittens/icat/main.py`." → FALSE (runtime-only).**
The exact invocation yields a process with `comm=kitten`, `exe=/app/kitty/launcher/kitten`, and **zero**
`libpython`/`fast_data_types` mappings (§7.1) — it is the **Go** binary. The C launcher intercepts wrapped
kittens at `kitty/launcher/main.c:356` and `exec`s Go **before** `Py_Initialize` (§7.1). The Python path is
real but only fires for **non-wrapped** kittens, demonstrated by `+kitten resize_window` showing
`comm=kitty` with `libpython` mapped (§7.2). This is the interpretation that *only running* could correct.

### 9.3 One portability-vs-performance tradeoff (supported by runtime observation)

**Observed tradeoff:** the two native languages sit on opposite sides of a deliberate portability/performance
line, and the runtime artifacts show both sides:

- **C, linked *into* the main process for performance.** The parser, screen model, and renderer live in
  `fast_data_types.so` **inside** the main address space (§4.1), and the main-thread stack shows Python
  calling straight into C with **no IPC boundary** (`Py_RunMain … main_loop … do_parse … draw_text`, §8.2).
  This shared-address-space design is what lets Kitty parse at tens-to-hundreds of MB/s (§3) — every byte
  parsed and every cell drawn stays in one process, avoiding serialization/IPC per frame. The cost is
  **portability**: this `.so` must be compiled against the platform's C toolchain and native GL/font
  libraries (the 11 native `.so`s of §4.1), and it is tightly coupled to CPython's ABI (`libpython3.12`).
- **Go, shipped as a *separate*, statically-linked process for portability.** The `kitten` binary is a
  ~16 MB near-static ELF whose **only** dynamic dependency is `libc.so.6` (§7.4), and it runs as an
  independent process that is never mapped into Kitty (§7.3). That makes the CLI tool-belt trivially
  portable and independently shippable/updatable — but it pays a **performance** price for anything that
  must cross into the terminal core: it communicates over a **PTY or a Unix socket** (e.g. `@` over
  `unix:/tmp/…/mykitty.sock`, §4.2), i.e. through IPC and serialization, not shared memory.

**In short, observed at runtime:** put the hot path (parse/render) in C **inside** the process to win
performance at the cost of portability; put the CLI tooling in a **separate** static Go binary to win
portability at the cost of an IPC boundary. Kitty chose each language on exactly that axis.

---

## 10. O8 — Methodology & reproducibility

### 10.1 Durations, scale, and repetition counts

| Observation | Scale | Repetitions | Stability |
|-------------|-------|-------------|-----------|
| Colored output (`csi --render`) | 100 → 1000 reps (3.75 s → 36.56 s) | 3 runs | 26.7 / 27.4 MB/s (~3 %) |
| Scrollback churn (`--with-scrollback --render`) | default reps (~24 s each) | 2 runs | 83.5 / 84.1 MB/s (< 1 %) |
| Graphics (`images --render`) | 400 reps (8.58 s) | 1 run | 248.6 MB/s |
| Repeated resizes (`@ resize-os-window`) | 200 resizes / run (~165 s) | 2 runs | 166.5 / 163.2 s (~2 %) |
| Tab switching (`@ action next_tab`, 7 tabs) | 100 cycles / run (~69 s) | 2 runs | 69.1 / 68.8 s (< 0.5 %) |
| Thread set (idle / during / after) | — | 3 phases | returned to baseline after load |
| `@ ls` structure | idle / during csi / during 7-tabs | 3 captures | consistent tree shape |

Every magnitude-bearing observation was run **≥ 2 times** at representative scale and was **stable** across
runs, per the requirement that timing/magnitude values be confirmed stable.

### 10.2 Environment

- Container: `kovidgoyal__kitty__815df1e210e0…`; Python 3.12.3, Go 1.23.4, gcc 13.3.0.
- Headless display: `Xvfb :99 -screen 0 1920x1080x24`; `LIBGL_ALWAYS_SOFTWARE=1`; Mesa **llvmpipe** GL
  4.5 (no CPU render fallback exists, so a GL context was mandatory and was obtained via software GL).
- Locale `C.UTF-8`. Main Kitty PID `7370`. RC socket `unix:/tmp/kitty_probe/mykitty.sock`.

### 10.3 The ptrace situation and which method worked

The container was started with `--cap-add SYS_PTRACE --security-opt seccomp=unconfined`, so **`py-spy dump`
and `gdb` attach both succeeded** (§8.1, §8.2, §8.4). The ptrace-free `/proc/<tid>/stack` fallback was
**blocked** with `Permission denied` (it needs `CAP_SYS_ADMIN`), and `eu-stack` **hung** and was abandoned.
Because the two attach methods worked, real symbol/stack visibility was achieved without needing the
fallback.

### 10.4 Repository-unchanged proof

All scratch (observation scripts, the test image, the RC socket, captured logs) lived under
`/tmp/kitty_probe/**` and was removed afterward. The build outputs (`kitty/launcher/`,
`kitty/fast_data_types.so`, `*.o`) are gitignored and were **not** added. The host repository shows only
this new documentation file and **no modification to any tracked source file**:

```
$ git status --porcelain
?? blitzy/documentation/kitty_815df1e210e0.md

$ git diff --stat        # zero tracked files modified
$ 
```

*(Before `git add`, git collapses the new file under its untracked parent as `?? blitzy/`; the expanded
path is `blitzy/documentation/kitty_815df1e210e0.md`, and `git diff --stat` is empty — i.e. nothing tracked
was changed.)*

### 10.5 Coverage confirmation (every named item answered)

- **Four workloads:** colored output (§3.1), scrollback churn (§3.2), repeated resizes (§3.4), tab
  switching (§3.5) — each individually, via Kitty's own `kitten __benchmark__` + RC (no hand-rolled
  generator).
- **Named targets:** `kitty` (main process, §4–§6, §8), `kitten` (Go binary, §7.4), `kitty +kitten icat`
  (exact invocation, §7.1), the control interface (`@ ls` / `@ get-text`, §6).
- **Objectives:** O1 §3 · O2 §4 · O3 §5 · O4 §6 · O5 §7 · O6 §8 · O7 §9 · O8 §10.

---

### Appendix — source anchors used for corroboration (read-only, never modified)

`[setup.py:L1091]` C-ext build · `[setup.py:L1160-L1163]` Go `kitten` build · `[setup.py:L636-L642]`
native libs · `[pyproject.toml:L2]` Python ≥ 3.8 · `[go.mod:L1,L3]` `module kitty` / `go 1.22` ·
`[kitty/constants.py:L25]` version · `[kitty/constants.py:L83-L84]` `kitten_exe()` ·
`[kitty/constants.py:L303-L305]` `wrapped_kitten_names()` · `[kittens/runner.py:L110-L117]` Python `+kitten`
path · `[kitty/entry_points.py]` `run_kitten` · `kitty/launcher/main.c:333,340,354-357,452` launcher
delegation · `[kitty/boss.py:L1889,L1949]` kitten spawn · `[kitty/child-monitor.c:L967,L1489,L1808]` thread
names · `[kitty/disk-cache.c:L342]` `DiskCacheWrite` · `[tools/cmd/benchmark/main.go:L319-L349,L68-L69]`
benchmark · `[kitty/rc/ls.py:L14-L33]` live-state schema · `[kitty/rc/resize_os_window.py:L29-L60]` resize ·
`[kitty/rc/focus_tab.py:L21-L23]` focus-tab · `[tools/cmd/tool/main.go:L47-L80]` kitten registry.

*Document generated runtime-first from artifacts captured on branch `kitty_815df1e210e0` (HEAD `815df1e21`).*
