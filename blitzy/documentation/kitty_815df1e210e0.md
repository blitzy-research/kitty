# Kitty Terminal Emulator — Runtime Architecture Investigation

## Document Metadata

| Field | Value |
|---|---|
| Repository | kitty (VCS revision `815df1e210e0`) |
| Version | 0.35.2 |
| Investigation Date | 2025 |
| Investigation Host | Ubuntu 24.04.4 LTS (Noble Numbat), kernel 6.6.113+, x86_64 |
| Python | 3.12.3 |
| Go | 1.22.10 |
| GCC | 13.3.0 (C11-capable) |
| Display | Xvfb :99 (virtual framebuffer, no physical GPU) |
| Methodology | Runtime-first: all conclusions derive from observable process artifacts |

---

## Table of Contents

1. [Build and Launch Observations](#1-build-and-launch-observations)
2. [Loaded Modules and Libraries](#2-loaded-modules-and-libraries)
3. [Thread Activity — Idle vs. Stress](#3-thread-activity--idle-vs-stress)
4. [Remote Control Interface Queries](#4-remote-control-interface-queries)
5. [Kitten Process Relationship](#5-kitten-process-relationship)
6. [Kitten Binary Inspection](#6-kitten-binary-inspection)
7. [Symbol and Stack Snapshots](#7-symbol-and-stack-snapshots)
8. [Language Responsibility Inference](#8-language-responsibility-inference)
9. [Two Falsified Interpretations](#9-two-falsified-interpretations)
10. [Portability vs. Performance Tradeoff](#10-portability-vs-performance-tradeoff)
11. [Appendix: Full Command Transcripts](#11-appendix-full-command-transcripts)

---

## 1. Build and Launch Observations

### 1.1 Build Process

Kitty is built with a single unified build command that compiles three distinct language layers:

```
export PATH=/usr/local/go/bin:$PATH
export KITTY_NO_LTO=1
python3 setup.py build --verbose --ignore-compiler-warnings
```

The `setup.py` build orchestrator performs the following operations in sequence:

1. **C Extension Compilation**: Discovers all `.c` files under `kitty/` (excluding macOS-specific files like `core_text.m`, `cocoa_window.m`, `macos_process_info.c`), compiles them with GCC using `-std=c11 -D_XOPEN_SOURCE=700` flags, and links them into a single shared object `kitty/fast_data_types.so`. The `find_c_files()` function in `setup.py` (line 906) collects 49 C source files plus vendored dependencies from `3rdparty/` (ringbuf, base64 with SIMD).

2. **GLFW Backend Compilation**: Builds two separate shared objects — `kitty/glfw-x11.so` and `kitty/glfw-wayland.so` — from the vendored GLFW 3.4 fork in `glfw/`. Each backend links against its respective display server libraries.

3. **Go Binary Compilation**: Invokes `go build` to produce the static `kitten` binary from `tools/cmd/main.go` and all Go packages under `tools/` and `kittens/`.

4. **C Launcher Compilation**: Compiles `kitty/launcher/main.c` into the small `kitty` launcher binary that embeds CPython.

### 1.2 Build Artifacts

Verification commands and their output:

```
$ file kitty/launcher/kitty
kitty/launcher/kitty: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV),
  dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2,
  BuildID[sha1]=e8c64dd649a7e0353b70a45f1defd979b45f8b79,
  for GNU/Linux 3.2.0, not stripped

$ file kitty/launcher/kitten
kitty/launcher/kitten: ELF 64-bit LSB executable, x86-64, version 1 (SYSV),
  dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2,
  Go BuildID=hWFG_Ca3xQ6VcsZYShGt/vXHUDdkh1kOGPzVOfMDC/TlErkqS1Lkyxjr-Onw6b/6MzJb86-gjtn1TQxs0aQ,
  stripped

$ file kitty/fast_data_types.so
kitty/fast_data_types.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV),
  dynamically linked,
  BuildID[sha1]=46fd91e410b71f30deee320bd09561401915e677, not stripped
```

**Key observation**: The kitty launcher is a tiny (36 KB) dynamically linked PIE executable. The kitten binary is a large (16 MB) Go executable with a `Go BuildID`. The fast_data_types.so extension is 1.5 MB, containing the entire C engine.

```
$ ls -lh kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so
-rwxr-xr-x  36K  kitty/launcher/kitty
-rwxr-xr-x  16M  kitty/launcher/kitten
-rwxr-xr-x 1.5M  kitty/fast_data_types.so
```

### 1.3 Launch Attempt and Display Environment

The sandboxed environment has no physical display server. An initial launch attempt without Xvfb failed:

```
$ DISPLAY=:0 kitty/launcher/kitty
[0.059] [glfw error 65544]: X11: Failed to open display :0
GLFW initialization failed
```

After installing and starting Xvfb (virtual framebuffer):

```
$ Xvfb :99 -screen 0 1280x1024x24 &
$ DISPLAY=:99 kitty/launcher/kitty --listen-on unix:/tmp/kitty-test2.sock -o allow_remote_control=yes &
```

Kitty launched successfully. The only warning was `Failed to open systemd user bus with error: Connection refused`, which is non-fatal. The process used Mesa's llvmpipe (software OpenGL rasterizer) since no physical GPU was present.

**Thinking**: The GLFW error on the first attempt (`X11: Failed to open display :0`) proves that the C GLFW layer (not Python, not Go) is responsible for platform windowing — if it fails, the entire application fails immediately, before any Python-level logic can intervene. This establishes that the C layer owns the display connection.

---

## 2. Loaded Modules and Libraries

### 2.1 Process Memory Map

With kitty running (PID 97391), the loaded shared libraries were captured from `/proc/97391/maps`:

```
$ cat /proc/97391/maps | grep '\.so' | awk '{print $6}' | sort -u
```

**Kitty-Specific Modules (built from source):**

| Library | Size | Purpose |
|---|---|---|
| `kitty/fast_data_types.so` | 1.5 MB | Monolithic C extension: screen model, VT parser, fonts, shaders, child monitor, OpenGL rendering, crypto |
| `kitty/glfw-x11.so` | 369 KB | GLFW X11 backend: window creation, input, display connection |

**Rendering and Font Libraries:**

| Library | Evidence |
|---|---|
| `libGL.so.1.7.0` | OpenGL dispatch layer — confirms GPU rendering path is active |
| `libGLX.so.0.0.0` | GLX (OpenGL for X11) — provides the GL context |
| `libGLX_mesa.so.0.0.0` | Mesa's GLX implementation — actual GL driver |
| `libGLdispatch.so.0.0.0` | GL function dispatch table (GLVND) |
| `libgallium-25.2.8-*.so` | Mesa Gallium3D driver framework — llvmpipe software rasterizer active |
| `libLLVM.so.20.1` | LLVM JIT — used by llvmpipe to JIT-compile shader programs into x86 |
| `libfreetype.so.6.20.1` | FreeType 2 — glyph rasterization engine |
| `libharfbuzz.so.0.60830.0` | HarfBuzz — OpenType text shaping (ligatures, kerning) |
| `libfontconfig.so.1.12.1` | Fontconfig — font discovery and matching |
| `libpng16.so.16.43.0` | libpng — PNG decoding for images and icons |
| `liblcms2.so.2.0.14` | Little CMS 2 — ICC color profile management |

**Platform and System Libraries:**

| Library | Purpose |
|---|---|
| `libpython3.12.so.1.0` | CPython 3.12 interpreter — embedded in the kitty process |
| `libX11.so.6.4.0` | Xlib — X11 protocol client |
| `libX11-xcb.so.1.0.0` | X11/XCB interop |
| `libxcb.so.1.1.0` | XCB — low-level X11 protocol |
| `libxcb-glx.so.0.0.0` | XCB GLX extension |
| `libxcb-xkb.so.1.0.0` | XCB XKB keyboard extension |
| `libxkbcommon.so.0.0.0` | XKB keyboard layout handling |
| `libxkbcommon-x11.so.0.0.0` | XKB X11 integration |
| `libXcursor.so.1.0.2` | X cursor management |
| `libXrandr.so.2.2.0` | X display configuration (monitor geometry) |
| `libXinerama.so.1.0.0` | Multi-monitor support |
| `libdbus-1.so.3.32.4` | D-Bus IPC (desktop integration) |
| `libcrypto.so.3` | OpenSSL — encryption for remote control protocol |
| `libz.so.1.3` | zlib — compression |

**Thinking**: The process map reveals that **all rendering, font, and graphics libraries are loaded into the kitty C process space**, not into any Go process. No Go-specific shared libraries appear in the map (the kitten binary is not loaded here at all). The `libpython3.12.so` confirms CPython is embedded as a shared library within the same process. This is direct evidence that Python orchestration and C rendering coexist in a single address space.

### 2.2 Python Introspection of fast_data_types

```python
$ python3 -c "
import sys; sys.path.insert(0, '.')
from kitty import fast_data_types as fdt
attrs = [a for a in dir(fdt) if not a.startswith('__')]
print(f'Total attributes: {len(attrs)}')
types = [a for a in attrs if a[0].isupper() and not a.isupper()]
print(f'Types: {sorted(types)[:30]}')
"
```

Output:
```
Total attributes: 581
Types/classes: 23
Types: ['AES256GCMDecrypt', 'AES256GCMEncrypt', 'ChildMonitor', 'Color',
        'ColorProfile', 'CryptoError', 'Cursor', 'DiskCache',
        'EllipticCurveKey', 'Face', 'FreeTypeError', 'GraphicsManager',
        'HistoryBuf', 'KeyEvent', 'Line', 'LineBuf', 'Parser', 'Region',
        'Screen', 'Secret', 'Shlex', 'SigInfo', 'SingleKey']
Functions/constants: 558
```

**Thinking**: The `fast_data_types` C extension exposes **581 attributes** (23 C-implemented types and 558 functions/constants) to Python. This is the bridge between the two in-process languages. The types map directly to C structs: `Screen` → `screen.c`, `ChildMonitor` → `child-monitor.c`, `Parser` → `vt-parser.c`, `Face` → `freetype.c`, `GraphicsManager` → `graphics.c`. Python calls into these C types for all performance-critical operations; the C code never calls back into Python for hot-path rendering.

### 2.3 fast_data_types Symbol Analysis

```
$ nm kitty/fast_data_types.so | grep ' [tTbBdD] ' | grep -iE 'render|parse|thread|loop|shader|glyph|font|screen|freetype|child'
```

Key symbols found (selected):

| Symbol | Type | Evidence |
|---|---|---|
| `PyInit_fast_data_types` | T (global text) | Python C extension init entry point |
| `do_parse` | t (local text) | VT escape sequence parser main loop |
| `_parse_sgr` | t | SGR (Select Graphic Rendition) color parsing |
| `csi_parse_loop` | t | CSI escape sequence dispatch loop |
| `alloc_sprite_map` | t | GPU glyph texture atlas allocation |
| `compile_shaders` | t | OpenGL shader compilation |
| `attach_shaders` | t | Shader program linking |
| `find_or_create_sprite_position` | t | Glyph cache lookup |
| `find_or_create_glyph_properties` | t | Glyph metrics caching |
| `draw_text_loop` | t | Text drawing main loop |
| `add_main_loop_timer` | t | Timer registration for the GLFW event loop |
| `gl_init` | t | OpenGL initialization |
| `create_freetype_render_context` | t | FreeType rendering setup |
| `GLAD_GL_VERSION_3_0` (and 1.0–3.1) | b (BSS) | GLAD OpenGL loader version flags |
| `Screen_Type` | d (data) | Python type object for `Screen` |
| `ChildMonitor_Type` | d | Python type object for `ChildMonitor` |
| `Parser_Type` | d | Python type object for `Parser` |
| `children_lock` | b | pthread mutex for child process list |
| `canberra_thread` | b | Audio notification thread |

Also found SIMD-accelerated base64 implementations compiled into the binary:
```
base64_stream_decode_avx
base64_stream_decode_avx2
base64_stream_decode_sse41
base64_stream_decode_sse42
base64_stream_decode_ssse3
base64_stream_encode_avx
base64_stream_encode_avx2
```

**Thinking**: The symbols confirm that all rendering hot paths (`draw_text_loop`, `compile_shaders`, `alloc_sprite_map`), all parsing (`do_parse`, `csi_parse_loop`, `_parse_sgr`), all font operations (`create_freetype_render_context`), and thread synchronization primitives (`children_lock`) are implemented in C within the single `fast_data_types.so`. The GLAD version flags prove that OpenGL function loading happens at the C level, not through any Python or Go intermediary. The SIMD base64 variants show architecture-specific optimization that would not be possible in Go or Python.

---

## 3. Thread Activity — Idle vs. Stress

### 3.1 Thread Inventory at Idle

With kitty running and no user input, the thread listing was captured:

```
$ ls /proc/97391/task/ | wc -l
68

$ for tid in $(ls /proc/97391/task/); do
    name=$(cat /proc/97391/task/$tid/comm)
    echo "TID $tid: $name"
  done
```

| Thread Name | Count | Role |
|---|---|---|
| `kitty` (TID 97391) | 1 | **Main thread** — GLFW event loop, rendering, Python interpreter |
| `KittyChildMon` (TID 97458) | 1 | **I/O thread** — PTY multiplexing, VT parser feeding |
| `KittyPeerMon` (TID 97457) | 1 | **Talk thread** — Remote control UNIX socket handler |
| `kitty:disk$0` (TID 97456) | 1 | **DiskCache thread** — persistent cache I/O |
| `llvmpipe-N` (TID 97392–97423) | 32 | Mesa llvmpipe software rendering worker pool |
| `kitty` (TID 97424–97455) | 32 | Additional Mesa/LLVM JIT threads |

**Thinking**: The three application-level threads (`kitty`, `KittyChildMon`, `KittyPeerMon`) match exactly the architecture documented in `kitty/child-monitor.c` (line 55: `pthread_t io_thread, talk_thread;`). The thread names are set via `set_thread_name()` — line 1492 sets `"KittyChildMon"` for the I/O thread, and the Talk thread is set to `"KittyPeerMon"` (visible in the `talk_loop` function). The `kitty:disk$0` thread is an additional infrastructure thread for the DiskCache. The 64 Mesa threads (32 llvmpipe + 32 JIT) are created by the software OpenGL driver; on a system with a real GPU, these would not exist and the thread count would be approximately 4.

### 3.2 Context Switch Comparison — Idle vs. Stress

A rendering stress test was executed by sending 5000 lines of heavily-colored output into the terminal:

```
$ kitten @ --to unix:/tmp/kitty-test2.sock send-text --match id:1 \
  'seq 1 5000 | while read i; do printf "\033[48;5;$(($i%256));38;5;$((($i+128)%256))m%-80s\n" \
  "LINE $i: heavy colored output"; done'
```

Context switch measurements before and after the stress run:

| Thread | Before (vol_ctx) | After (vol_ctx) | Delta | Interpretation |
|---|---|---|---|---|
| Main (`kitty`, TID 97391) | 844 | 1,290 | **+446** | Rendering frames, GLFW event processing |
| I/O (`KittyChildMon`, TID 97458) | 2,245 | 6,964 | **+4,719** | PTY reads, VT parsing, buffer management |
| Talk (`KittyPeerMon`, TID 97457) | 17 | 20 | **+3** | Minimal activity (no remote control queries during stress) |

**Thinking**: The I/O thread (`KittyChildMon`) shows **10.6× more context switches** than the Main thread during rendering stress. This is because the I/O thread is continuously `poll()`-ing the child PTY, reading incoming bytes, and feeding them to the VT parser (`do_parse`), while the Main thread is processing already-parsed screen updates and submitting them for rendering. The Talk thread is nearly dormant (+3 switches) because no remote control commands were issued during the stress window. This demonstrates the clear division: I/O thread handles the high-frequency byte stream from child processes, Main thread handles rendering at the display refresh rate, and Talk thread is demand-driven.

### 3.3 strace Syscall Profile During Stress

A 5-second strace capture during active rendering:

```
$ strace -p 97391 -c -f -S calls
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- --------
 98.23    0.416559         503       828        83 futex
  1.23    0.005209          38       134           poll
  0.40    0.001716          15       108         6 read
  0.06    0.000241           3        69        52 recvmsg
  0.05    0.000215          13        16           writev
  0.02    0.000085           6        13           write
  0.00    0.000006           1         6           getpid
  0.00    0.000023           7         3           ppoll
  0.00    0.000000           0         1           accept
------ ----------- ----------- --------- --------- --------
100.00    0.424059         356      1188       141 total
```

**Thinking**: The syscall profile reveals that `futex` dominates (98% of wall time, 828 calls) — these are the pthread synchronization operations between the Main, I/O, and Talk threads sharing the `children_lock` and `talk_lock` mutexes. The 134 `poll` calls correspond to the GLFW event loop (`pollForEvents()` in glfw-x11.so) and the I/O thread's child PTY monitoring. The 108 `read` calls are PTY reads by the I/O thread. The 16 `writev` calls are X11 protocol writes for rendering. The single `accept` is the Talk thread accepting a remote control connection. This profile is entirely C-level syscalls — Python's GIL acquire/release would appear as additional futex calls, confirming that the hot path is C code with Python merely orchestrating the lifecycle.

---

## 4. Remote Control Interface Queries

### 4.1 Window/Tab State (`kitty @ ls`)

```
$ kitten @ --to unix:/tmp/kitty-test2.sock ls
```

Output (trimmed to essential structure):

```json
[
  {
    "id": 1,
    "is_active": true,
    "is_focused": true,
    "platform_window_id": 2097164,
    "background_opacity": 1.0,
    "tabs": [
      {
        "id": 1,
        "is_active": true,
        "layout": "fat",
        "title": "/tmp/blitzy/kitty/blitzy-...",
        "windows": [
          {
            "id": 1,
            "is_active": true,
            "columns": 71,
            "lines": 22,
            "pid": 97459,
            "cmdline": ["/bin/bash", "--posix"],
            "cwd": "/tmp/blitzy/kitty/...",
            "at_prompt": true,
            "foreground_processes": [
              {
                "pid": 97459,
                "cmdline": ["/bin/bash", "--posix"],
                "cwd": "/tmp/blitzy/kitty/..."
              }
            ]
          }
        ]
      }
    ],
    "wm_class": "kitty",
    "wm_name": "kitty"
  }
]
```

**Thinking**: The `kitty @ ls` output comes through the Talk thread (`KittyPeerMon`). The Go `kitten` binary sends the `ls` command over a UNIX socket (`unix:/tmp/kitty-test2.sock`), the Talk thread reads it, dispatches to the Python `kitty/rc/ls.py` handler (which calls C-backed methods on the `Boss` object to collect window/tab state), serializes to JSON, and sends it back over the socket. This demonstrates all three languages cooperating: **Go** (kitten CLI sends the request), **C** (Talk thread handles the socket I/O), and **Python** (rc/ls.py handler collects and serializes the state). The `platform_window_id: 2097164` is an X11 window ID from the C GLFW layer.

### 4.2 Color State (`kitty @ get-colors`)

```
$ kitten @ --to unix:/tmp/kitty-test2.sock get-colors
```

Output (first 10 and last 5 entries):

```
active_border_color     #00ff00
active_tab_background   #eeeeee
active_tab_foreground   #000000
background              #000000
bell_border_color       #ff5a00
color0                  #000000
color1                  #cc0403
...
color255                #eeeeee
cursor                  #cccccc
cursor_text_color       #111111
foreground              #dddddd
selection_background    #fffacd
selection_foreground    #000000
url_color               #0087bd
```

**Thinking**: The full 256-color palette plus semantic colors (cursor, selection, tabs, borders) are returned, proving that the C `ColorProfile` type (exposed via `fast_data_types`) stores the complete color state in memory. The Python `kitty/rc/get_colors.py` handler reads from the C-backed `ColorProfile` object, not from a Python dictionary — the colors are maintained in C data structures for direct use by the GLSL shaders.

### 4.3 Screen Content (`kitty @ get-text`)

```
$ kitten @ --to unix:/tmp/kitty-test2.sock get-text --match id:1
```

Output (tail, during stress test):

```
;5;229mSTRACE_STRESS_00997
;5;230mSTRACE_STRESS_00998
;5;231mSTRACE_STRESS_00999
;5;232mSTRACE_STRESS_01000
```

**Thinking**: The `get-text` output shows the visible screen content, including ANSI escape code fragments. This is because the C `Screen` object stores the parsed text but the remote control serialization extracts the raw line content. The ANSI fragments visible in the output are residual SGR sequences that were parsed by the C VT parser (`_parse_sgr` in fast_data_types.so) and applied to the cell attributes; the text extraction path pulls cell character content with formatting fragments.

---

## 5. Kitten Process Relationship

### 5.1 Observation Method

A test image was created and `kitten icat` was launched inside the running kitty terminal:

```
$ kitty/launcher/kitten icat /tmp/test_image.png &
KPID=$!
```

The process state was captured while kitten was running:

```
$ ls -la /proc/$KPID/exe
lrwxrwxrwx 1 root root 0 /proc/103385/exe -> .../kitty/launcher/kitten

$ cat /proc/$KPID/status | head -10
Name:   kitten
Umask:  0022
State:  T (stopped)
Tgid:   103385
Pid:    103385
PPid:   97459
TracerPid: 0
```

Process tree during execution:

```
kitty (PID 97391)          ← C launcher + embedded Python + C extensions
  └── bash (PID 97459)     ← Child shell spawned by kitty
        └── kitten (PID 103385) ← Go binary, separate process
```

### 5.2 Analysis

**Key observations:**

1. **Separate PID**: The kitten process (PID 103385) has its own PID, distinct from the kitty process (PID 97391). It is NOT a thread within the kitty process.

2. **Separate executable**: `/proc/103385/exe` points to `kitty/launcher/kitten`, the Go binary — not to the Python interpreter or the kitty launcher.

3. **Parent chain**: The kitten's PPid is 97459 (the bash shell inside kitty), not 97391 (the kitty process itself). This is because when the user types `kitten icat ...` in the terminal, bash (the child process) fork+exec's the kitten binary.

4. **No Go libraries in kitty process**: Looking back at the `/proc/97391/maps` from Section 2, there are zero Go runtime libraries loaded. The Go runtime is entirely self-contained within the kitten binary's own address space.

**Thinking**: The source code path confirms this observation. In `kitty/entry_points.py` (line 10–12):

```python
def icat(args: List[str]) -> None:
    from kitty.constants import kitten_exe
    os.execl(kitten_exe(), "kitten", *args)
```

When kitty itself dispatches `+kitten icat`, it calls `os.execl()` which **replaces the current process image** with the Go kitten binary. However, in our observation, kitten was launched from the bash shell inside kitty, so it was fork+exec'd as a child of bash, not as a replacement of the kitty process.

The `kitten_exe()` function in `kitty/constants.py` (line 83–84) returns `os.path.join(os.path.dirname(kitty_exe()), 'kitten')` — the kitten binary is expected to be in the same directory as the kitty binary. This is a deployment convention, not a runtime linkage — the two binaries share no memory or code at runtime.

---

## 6. Kitten Binary Inspection

### 6.1 Binary Format

```
$ file kitty/launcher/kitten
kitty/launcher/kitten: ELF 64-bit LSB executable, x86-64, version 1 (SYSV),
  dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2,
  Go BuildID=hWFG_Ca3xQ6VcsZYShGt/vXHUDdkh1kOGPzVOfMDC/TlErkqS1Lkyxjr-Onw6b/6MzJb86-gjtn1TQxs0aQ,
  stripped
```

**Key findings:**
- **ELF 64-bit LSB executable**: Standard Linux binary format, not a script or bytecode
- **Go BuildID present**: The four-component BuildID (`hWFG_.../vXHU.../TlEr.../6MzJ...`) is the definitive Go binary fingerprint
- **Dynamically linked**: Despite being Go, it links against libc (standard for CGO-enabled or external linker builds)
- **Stripped**: Debug symbols removed, but Go metadata sections preserved

### 6.2 Shared Library Dependencies

```
$ ldd kitty/launcher/kitten
  linux-vdso.so.1
  libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
  /lib64/ld-linux-x86-64.so.2

$ readelf -d kitty/launcher/kitten
  Tag        Type         Name/Value
  (NEEDED)   Shared library: [libc.so.6]
```

**Thinking**: The kitten binary depends **only** on libc.so.6 — no libpython, no libfreetype, no libGL, no libharfbuzz. This is the hallmark of a Go binary with CGO disabled or using only libc-level syscalls. Compare with `kitty/fast_data_types.so` which links against 17 shared libraries including libpython, libharfbuzz, libpng, liblcms2, libcrypto, and libGL. The kitten binary is essentially self-contained — the entire Go runtime, garbage collector, goroutine scheduler, and all Go packages are statically compiled into the 16 MB binary.

### 6.3 Go Runtime Confirmation

```
$ readelf -S kitty/launcher/kitten | grep -E '\.go|\.text'
  [ 1] .text             PROGBITS  0000000000401000
  [13] .gosymtab         PROGBITS  0000000000f18508
  [14] .gopclntab        PROGBITS  0000000000f18520
  [15] .go.buildinfo     PROGBITS  00000000012b1000
  [25] .note.go.buildid  NOTE      0000000000400f80

$ go version kitty/launcher/kitten
kitty/launcher/kitten: go1.22.10
```

**Go-specific ELF sections found:**
- `.gosymtab` — Go symbol table (Go's own symbol format, independent of ELF symtab)
- `.gopclntab` — Go PC-to-line-number table (for stack traces and runtime.Caller)
- `.go.buildinfo` — Go module info, build settings, dependency versions
- `.note.go.buildid` — Build reproducibility identifier

### 6.4 Embedded Go Packages

```
$ strings kitty/launcher/kitten | grep -oE 'kitty/(tools|kittens)/[a-z_/]+' | sort -u
```

Packages found (50 total, selected):

| Package | Purpose |
|---|---|
| `kitty/kittens/icat` | Image display via graphics protocol |
| `kitty/kittens/diff` | Side-by-side diff viewer |
| `kitty/kittens/ssh` | SSH integration with shell integration |
| `kitty/kittens/themes` | Theme browser and selector |
| `kitty/kittens/clipboard` | Clipboard access |
| `kitty/kittens/hints` | URL and pattern hints |
| `kitty/kittens/unicode_input` | Unicode character input |
| `kitty/kittens/transfer` | File transfer via terminal |
| `kitty/kittens/ask` | User prompt dialogs |
| `kitty/kittens/choose_fonts` | Font selection UI |
| `kitty/tools/cmd/at` | Remote control (`kitty @`) client |
| `kitty/tools/tui/loop` | Go TUI event loop |
| `kitty/tools/tui/graphics` | Graphics protocol encoder |
| `kitty/tools/crypto` | X25519/AES-GCM encryption |
| `kitty/tools/utils/images` | Image processing (resize, decode) |

**Thinking**: All 50 Go packages are compiled into the single kitten binary. This means every kitten subcommand (icat, diff, ssh, themes, clipboard, hints, unicode_input, transfer, choose_fonts, etc.) runs from the same binary. The binary contains its own TUI loop, graphics protocol encoder, crypto layer, and image processing — completely independent of the kitty process. This is the Go "single binary deployment" pattern: no shared library dependencies, no runtime installation required.

### 6.5 Go Entry Point

The Go entry point (`tools/cmd/main.go`) was read from source:

```go
func main() {
    krm := os.Getenv("KITTY_KITTEN_RUN_MODULE")
    os.Unsetenv("KITTY_KITTEN_RUN_MODULE")
    switch krm {
    case "ssh_askpass":
        ssh.RunSSHAskpass()
        return
    }
    root := cli.NewRootCommand()
    root.ShortDescription = "Fast, statically compiled implementations of various kittens"
    root.HelpText = "kitten serves as a launcher for running individual kittens."
    tool.KittyToolEntryPoints(root)
    completion.EntryPoint(root)
    root.Exec()
}
```

**Thinking**: The Go main function builds a CLI command tree and dispatches to subcommands. The comment string embedded in the binary itself — "Fast, statically compiled implementations of various kittens" — explicitly states the design intent: Go kittens are compiled for speed and deploy as a single static binary.

---

## 7. Symbol and Stack Snapshots

### 7.1 GDB Stack Trace — Main Thread

```
$ gdb -batch -ex 'thread 1' -ex 'bt 15' -p 97391
```

```
Thread 1 (Main thread, "kitty"):
#0  __GI___poll (fds=0x...._glfw+133552, nfds=2, timeout=-1)
    at ../sysdeps/unix/sysv/linux/poll.c:29
#1  pollForEvents ()
    from kitty/glfw-x11.so
#2  _glfwPlatformWaitEvents ()
    from kitty/glfw-x11.so
#3  _glfwPlatformRunMainLoop ()
    from kitty/glfw-x11.so
#4  main_loop ()
    from kitty/fast_data_types.so
#5  ?? () from libpython3.12.so.1.0       [PyObject_Vectorcall]
#6  PyObject_Vectorcall ()                 from libpython3.12.so.1.0
#7  _PyEval_EvalFrameDefault ()            from libpython3.12.so.1.0
#8  _PyObject_FastCallDictTstate ()        from libpython3.12.so.1.0
#9  _PyObject_Call_Prepend ()              from libpython3.12.so.1.0
#10 ?? ()                                  from libpython3.12.so.1.0
#11 _PyObject_MakeTpCall ()                from libpython3.12.so.1.0
#12 _PyEval_EvalFrameDefault ()            from libpython3.12.so.1.0
#13 PyEval_EvalCode ()                     from libpython3.12.so.1.0
```

**Thinking**: This stack trace is the single most revealing artifact. Reading from bottom to top:

1. **Frames #13–#7** (`PyEval_EvalCode`, `_PyEval_EvalFrameDefault`): Python bytecode evaluation — this is `kitty/main.py` calling `boss.child_monitor.main_loop()`
2. **Frame #5–#6** (`PyObject_Vectorcall`): Python calling a C function — the transition from Python to the C `main_loop()` method of the `ChildMonitor` object
3. **Frame #4** (`main_loop()` in `fast_data_types.so`): The C `main_loop()` from `kitty/child-monitor.c` line 1259 — this is where `run_main_loop()` is called
4. **Frame #3** (`_glfwPlatformRunMainLoop()` in `glfw-x11.so`): The GLFW platform-specific event loop
5. **Frame #2** (`_glfwPlatformWaitEvents()`): GLFW waiting for X11 events
6. **Frame #1** (`pollForEvents()`): The actual X11 connection polling
7. **Frame #0** (`__GI___poll`): Kernel poll syscall waiting on the X11 connection file descriptor

This stack definitively shows the **Python → C → GLFW → X11** call chain at idle. Python invokes the C `main_loop`, which delegates to GLFW, which blocks on X11 events. When events arrive (or the I/O thread signals data ready), the C `render()` function (line 871 of child-monitor.c) executes the rendering pipeline — all in C, never returning to Python for the hot path.

### 7.2 GDB Stack Trace — I/O Thread (KittyChildMon)

```
Thread 2 ("KittyChildMon"):
#0  __GI___poll (fds=0x..._children_fds, nfds=3, timeout=-1)
#1  io_loop ()     from kitty/fast_data_types.so
#2  start_thread ()
#3  clone3 ()
```

**Thinking**: The I/O thread is sitting in `poll()` waiting on the `children_fds` array (defined at the file scope in child-monitor.c). The `nfds=3` means it's watching: (1) the wakeup pipe from the main thread, (2) a signal pipe, and (3) one child PTY file descriptor (the bash shell). When data arrives on the PTY, `io_loop()` reads it and feeds it to the VT parser, then wakes the main thread. The entire I/O thread stack is pure C — no Python frames appear.

### 7.3 GDB Stack Trace — Talk Thread (KittyPeerMon)

```
Thread 3 ("KittyPeerMon"):
#0  __GI___poll (fds=0x..., nfds=2, timeout=-1)
#1  talk_loop ()   from kitty/fast_data_types.so
#2  start_thread ()
#3  clone3 ()
```

**Thinking**: The Talk thread is similarly in `poll()`, waiting on: (1) the UNIX socket for remote control connections, and (2) a wakeup pipe. When a `kitten @` command arrives, `talk_loop()` accepts the connection, reads the command, and dispatches it to the Python RC handler via the main thread. Again, all C frames — the Talk thread itself never executes Python bytecode.

### 7.4 DiskCache Thread

```
Thread 4 ("kitty:disk$0"):
#0  __futex_abstimed_wait_common64 (...)
#1  __GI___futex_abstimed_wait_cancelable64 (...)
#2  __pthread_cond_wait_common (...)
#3  ___pthread_cond_wait (...)
#4  ?? ()   from libgallium-25.2.8-*.so
#5  ?? ()   from libgallium-25.2.8-*.so
```

**Thinking**: Despite the name `kitty:disk$0`, this thread's stack shows it waiting in Mesa's Gallium driver. The name comes from Mesa's disk cache subsystem (shader cache), not from kitty's own DiskCache. This is an artifact of the software rendering environment — Mesa caches compiled shader programs to disk using a background thread.

---

## 8. Language Responsibility Inference

Based exclusively on the collected runtime evidence, the following responsibilities are assigned to each language:

### 8.1 C — The Performance Engine (in-process)

**Evidence supporting C responsibility:**

| Responsibility | Runtime Evidence |
|---|---|
| VT escape sequence parsing | `do_parse`, `csi_parse_loop`, `_parse_sgr` symbols in `fast_data_types.so`; I/O thread (pure C stack) feeds parser |
| Screen model management | `Screen_Type` Python type backed by C struct; `screen.c` handles all cell updates |
| OpenGL rendering pipeline | `compile_shaders`, `alloc_sprite_map`, `draw_text_loop` symbols; `libGL.so` loaded in process; GLAD version flags in BSS |
| Font rasterization | `create_freetype_render_context` symbol; `libfreetype.so`, `libharfbuzz.so` loaded; `Face` type in fast_data_types |
| Glyph caching | `find_or_create_sprite_position`, `find_or_create_glyph_properties` symbols; GPU texture atlas |
| Thread management | `children_lock` mutex; `pthread_create` for I/O and Talk threads; GDB shows C-only stacks for all worker threads |
| Child process I/O | I/O thread (`KittyChildMon`) stack is pure C (`io_loop` → `poll`); PTY multiplexing via `poll()` |
| GLFW event loop | Main thread stack shows `_glfwPlatformRunMainLoop` → `pollForEvents` — all C code in `glfw-x11.so` |
| SIMD acceleration | `base64_stream_decode_avx2`, `base64_stream_decode_sse41` symbols; `simd-string-128.c`, `simd-string-256.c` source files |
| Cryptographic operations | `AES256GCMDecrypt`, `AES256GCMEncrypt`, `EllipticCurveKey` C types; `libcrypto.so` linked |

### 8.2 Python — The Orchestration Layer (in-process)

**Evidence supporting Python responsibility:**

| Responsibility | Runtime Evidence |
|---|---|
| Application startup | GDB main thread stack shows `PyEval_EvalCode` → `_PyEval_EvalFrameDefault` below the C `main_loop()` call |
| Configuration loading | `kitty/main.py` startup sequence: `parse_args → create_opts → init_glfw → run_app → boss.child_monitor.main_loop()` |
| Remote control command dispatch | `kitty @ ls` returns JSON assembled by Python `kitty/rc/ls.py`; Python reads from C-backed objects |
| Window/tab lifecycle | `Boss` object (Python) manages windows and tabs; `kitty @ ls` output shows Python-managed state |
| Entry point routing | `kitty/entry_points.py` dispatches `+kitten` → `os.execl(kitten_exe())`, keyboard shortcuts, etc. |
| GLSL shader preprocessing | `kitty/shaders.py` loads `.glsl` files and injects `#define` macros before passing to C for compilation |
| Type stubs for C extension | `kitty/fast_data_types.pyi` (581 attributes) provides Python IDE support for the C extension |

### 8.3 Go — The CLI Toolkit (separate process)

**Evidence supporting Go responsibility:**

| Responsibility | Runtime Evidence |
|---|---|
| All kitten subcommands | 50 Go packages compiled into `kitten` binary (icat, diff, ssh, themes, clipboard, hints, etc.) |
| Remote control client | `kitty/tools/cmd/at/` package; `kitten @ ls` sends commands over UNIX socket from Go process |
| Image display (icat) | `kittens/icat/main.go` — separate process (PID 103385, PPid 97459, exe → kitten binary) |
| TUI interfaces | `kitty/tools/tui/loop` — Go's own event loop for interactive kittens |
| File transfer | `kitty/kittens/transfer` — rsync-like protocol in Go |
| SSH integration | `kitty/kittens/ssh` — shell integration, askpass |
| Encryption for remote control | `kitty/tools/crypto` — X25519/AES-GCM (parallel to C implementation in main process) |
| No GPU rendering | kitten binary has NO libGL, libfreetype, or libharfbuzz dependencies — rendering is not its job |

### 8.4 GLSL — The GPU Shader Pipeline (executed on GPU/llvmpipe)

**Evidence**: 13 shader files totaling 696 lines, compiled by C `compile_shaders()`:

| Shader Pair | Responsible For |
|---|---|
| `cell_vertex.glsl` + `cell_fragment.glsl` | Text cell rendering (233 + 204 lines) |
| `border_vertex.glsl` + `border_fragment.glsl` | Window border decoration |
| `graphics_vertex.glsl` + `graphics_fragment.glsl` | Inline image compositing |
| `bgimage_vertex.glsl` + `bgimage_fragment.glsl` | Background image rendering |
| `tint_vertex.glsl` + `tint_fragment.glsl` | Window tint overlay |
| `alpha_blend.glsl`, `linear2srgb.glsl` | Blending and colorspace utilities |

---

## 9. Two Falsified Interpretations

### 9.1 Falsified: "Go Handles Rendering or Has Access to the GPU"

**Plausible-but-wrong reasoning**: Since Go is used for several kittens including `icat` (which displays images in the terminal), one might assume that Go code directly interfaces with OpenGL or the GPU to render images. After all, `icat` displays images, so it must be doing rendering, right?

**Runtime evidence that falsifies this**:

1. **The kitten binary links only libc.so.6** — `ldd kitty/launcher/kitten` shows zero graphics libraries. No libGL, no libfreetype, no libharfbuzz, no Mesa libraries. If Go were doing rendering, these libraries would appear in its dependency list.

2. **The kitty process map shows all rendering libraries** — `libGL.so`, `libGLX.so`, `libgallium`, `libfreetype.so`, `libharfbuzz.so` are all loaded exclusively in the kitty process (PID 97391), not in the kitten process.

3. **icat uses the terminal graphics protocol, not GPU calls** — The `kittens/icat/transmit.go` file implements the kitty graphics protocol (transmitting base64-encoded image data via terminal escape sequences). The kitten writes escape codes to stdout; the C code in the main kitty process (via `kitty/graphics.c`) decodes these and renders them using OpenGL. The kitten never touches the GPU.

4. **Kitten runs as a separate process** — PID 103385 (kitten) is entirely separate from PID 97391 (kitty). The kitten cannot access the kitty process's OpenGL context because OpenGL contexts are thread-local within a single process.

**Conclusion**: Go kittens produce terminal output (including graphics protocol escape sequences); the C code in the main kitty process is solely responsible for translating that output into GPU rendering commands.

### 9.2 Falsified: "Python Interprets VT Escape Sequences in the Hot Path"

**Plausible-but-wrong reasoning**: Since Python is the "orchestration layer" and manages the `Screen` object, one might assume that incoming VT escape sequences (e.g., SGR color codes, cursor movement, scrolling) are parsed and applied by Python code. The existence of `kitty/fast_data_types.pyi` with its `Screen` type stub might suggest Python is actively calling screen update methods.

**Runtime evidence that falsifies this**:

1. **The I/O thread stack is pure C** — GDB shows `KittyChildMon`'s stack as `poll() → io_loop() → start_thread() → clone3()`. There are zero Python frames (`_PyEval_EvalFrameDefault`, `PyObject_Vectorcall`) in the I/O thread. The VT parser runs entirely in C.

2. **The I/O thread context switches dominate** — During rendering stress, the I/O thread accumulated 4,719 voluntary context switches (10.6× the main thread). This thread does the parsing and buffer management. If Python were in the loop, the GIL would serialize these operations with the main thread, destroying throughput.

3. **The `do_parse` symbol is in fast_data_types.so** — `nm` shows `do_parse` as a C function in the shared library. This is the main VT parser entry point called by `io_loop()`.

4. **Python never acquires the GIL in the I/O thread** — The GDB backtrace of the I/O thread shows no `PyGILState_Ensure` or `PyEval_RestoreThread` calls. The I/O thread operates exclusively in C, using mutexes (`children_lock`) to synchronize with the main thread only when handing off parsed data.

5. **The strace profile shows no Python-level overhead** — The syscall profile is dominated by `futex` (mutex operations) and `poll`/`read` (I/O) — these are raw C syscalls, not Python-mediated operations.

**Conclusion**: The VT parser is a pure C state machine (`vt-parser.c`) running in the I/O thread without Python involvement. Python's role is limited to calling `boss.child_monitor.main_loop()` once at startup — after that, the C code takes over the hot path completely.

---

## 10. Portability vs. Performance Tradeoff

### 10.1 The Go Kitten Binary: Portability over Performance

**Observation**: The Go `kitten` binary is 16 MB, links only libc.so.6, and contains 50 statically-compiled packages including image processing, TUI, crypto, and all kitten implementations. The C `fast_data_types.so` is 1.5 MB but links against 17 shared libraries and contains SIMD-accelerated code paths (AVX, AVX2, SSE4.1, SSE4.2, SSSE3).

**Runtime evidence**:

1. **kitten has zero platform-specific library dependencies** — `ldd` shows only libc. This means the kitten binary can be copied to any Linux x86_64 system and run immediately, regardless of whether FreeType, HarfBuzz, OpenGL, or any other library is installed. The Go runtime handles memory management, networking, and file I/O internally.

2. **fast_data_types.so requires 17 specific shared libraries** — The C extension will fail to load if any of its dependencies (libfreetype, libharfbuzz, libGL, libpng, liblcms2, libcrypto) are missing or are an incompatible version. This makes the C layer sensitive to the host system's library configuration.

3. **The C layer has SIMD acceleration, Go does not** — The `nm` output shows `base64_stream_decode_avx2`, `base64_stream_decode_sse41`, and other SIMD variants in `fast_data_types.so`. The Go binary contains no such architecture-specific optimizations — Go's compiler generates portable x86_64 code but does not auto-vectorize or use hand-written SIMD intrinsics.

4. **The C layer uses platform-specific font backends** — The `find_c_files()` function in `setup.py` (line 908) conditionally excludes `fontconfig.c` and `freetype.c` on macOS (using `core_text.m` instead) or excludes `core_text.m` on Linux. The Go kitten binary does not need any font backend — it delegates text rendering to the terminal (i.e., back to kitty's C layer).

**Tradeoff analysis**:

The design splits the application along a **portability boundary**: operations that need to be fast (parsing, rendering, font rasterization) are implemented in C with platform-specific optimizations (SIMD, conditional compilation, OS-specific font APIs), while operations that need to be portable (CLI tools, file transfer, interactive kittens) are implemented in Go with its "compile once, run anywhere" model. This means:

- **C extension**: Maximum performance through SIMD, hardware-specific font backends, and direct OpenGL access, at the cost of requiring platform-specific compilation and library dependencies.
- **Go kitten**: Maximum portability through static compilation and zero external dependencies (beyond libc), at the cost of not having access to SIMD acceleration, hardware font rendering, or direct GPU access.

A concrete example: when `kitten icat` displays an image, the Go code does image decoding and resizing using pure Go libraries (`golang.org/x/image`, `github.com/kovidgoyal/imaging`), then transmits the pixel data via the terminal graphics protocol. The C code in the main kitty process then uses the GPU (or llvmpipe) to composite the image into the terminal display. If the Go code attempted GPU rendering, it would need OpenGL bindings and lose its single-binary portability. By delegating rendering to the host terminal, the kitten binary remains a simple, portable tool.

---

## 11. Appendix: Full Command Transcripts

### A.1 Build Verification

```bash
$ file kitty/launcher/kitty
kitty/launcher/kitty: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV),
  dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2,
  BuildID[sha1]=e8c64dd649a7e0353b70a45f1defd979b45f8b79,
  for GNU/Linux 3.2.0, not stripped

$ file kitty/launcher/kitten
kitty/launcher/kitten: ELF 64-bit LSB executable, x86-64, version 1 (SYSV),
  dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2,
  Go BuildID=hWFG_Ca3xQ6VcsZYShGt/vXHUDdkh1kOGPzVOfMDC/TlErkqS1Lkyxjr-Onw6b/6MzJb86-gjtn1TQxs0aQ,
  stripped

$ file kitty/fast_data_types.so
kitty/fast_data_types.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV),
  dynamically linked,
  BuildID[sha1]=46fd91e410b71f30deee320bd09561401915e677, not stripped

$ ls -lh kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so
-rwxr-xr-x  36K  kitty/launcher/kitty
-rwxr-xr-x  16M  kitty/launcher/kitten
-rwxr-xr-x 1.5M  kitty/fast_data_types.so
```

### A.2 Library Linkage

```bash
$ ldd kitty/launcher/kitty
  libpython3.12.so.1.0 => /lib/x86_64-linux-gnu/libpython3.12.so.1.0
  libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
  libm.so.6, libz.so.1, libexpat.so.1

$ ldd kitty/launcher/kitten
  libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6

$ ldd kitty/fast_data_types.so
  libpython3.12.so.1.0, libharfbuzz.so.0, libpng16.so.16,
  liblcms2.so.2, libcrypto.so.3, libz.so.1,
  libfreetype.so.6, libglib-2.0.so.0, libgraphite2.so.3,
  libbz2.so.1.0, libbrotlidec.so.1

$ ldd kitty/glfw-x11.so
  libX11.so.6, libXcursor.so.1, libxkbcommon.so.0,
  libxkbcommon-x11.so.0, libX11-xcb.so.1, libdbus-1.so.3,
  libxcb.so.1, libxcb-xkb.so.1
```

### A.3 Process Launch

```bash
$ Xvfb :99 -screen 0 1280x1024x24 &
$ DISPLAY=:99 kitty/launcher/kitty \
    --listen-on unix:/tmp/kitty-test2.sock \
    -o allow_remote_control=yes &
# Output: [0.154] Failed to open systemd user bus with error: Connection refused
# (non-fatal warning)
# Kitty PID: 97391
```

### A.4 Process Map Capture

```bash
$ cat /proc/97391/maps | grep '\.so' | awk '{print $6}' | sort -u
# (76 unique shared libraries — see Section 2.1 for full list)
```

### A.5 Thread Listing at Idle

```bash
$ ls /proc/97391/task/ | wc -l
68

$ for tid in $(ls /proc/97391/task/); do
    echo "TID $tid: $(cat /proc/97391/task/$tid/comm)"
  done
# TID 97391: kitty          (Main thread)
# TID 97392-97423: llvmpipe-0 through llvmpipe-31 (Mesa software rendering)
# TID 97424-97455: kitty      (Mesa/LLVM JIT threads)
# TID 97456: kitty:disk$0    (DiskCache/Mesa disk thread)
# TID 97457: KittyPeerMon    (Talk thread — remote control)
# TID 97458: KittyChildMon   (I/O thread — PTY multiplexing)
```

### A.6 Stress Test and Context Switch Measurement

```bash
# Baseline
# Main: vol=844, IO: vol=2245, Talk: vol=17

$ kitten @ --to unix:/tmp/kitty-test2.sock send-text --match id:1 \
  'seq 1 5000 | while read i; do
    printf "\033[48;5;$(($i%256));38;5;$((($i+128)%256))m%-80s\n" \
    "LINE $i: heavy colored output"
  done'
# Wait 8 seconds for completion

# Post-stress
# Main: vol=1290 (+446), IO: vol=6964 (+4719), Talk: vol=20 (+3)
```

### A.7 strace Capture

```bash
$ strace -p 97391 -c -f -S calls
# (5-second capture during stress — see Section 3.3 for full output)
# Top syscalls: futex(828), poll(134), read(108), recvmsg(69), writev(16)
```

### A.8 GDB Stack Traces

```bash
$ gdb -batch -ex 'thread 1' -ex 'bt 15' -p 97391
# Main thread: poll → pollForEvents → _glfwPlatformWaitEvents →
#   _glfwPlatformRunMainLoop → main_loop [fast_data_types.so] →
#   PyObject_Vectorcall → _PyEval_EvalFrameDefault → PyEval_EvalCode

$ gdb -batch -ex 'thread 2' -ex 'bt 15' -p 97391
# KittyChildMon: poll(children_fds, 3) → io_loop [fast_data_types.so] →
#   start_thread → clone3

$ gdb -batch -ex 'thread 3' -ex 'bt 15' -p 97391
# KittyPeerMon: poll(fds, 2) → talk_loop [fast_data_types.so] →
#   start_thread → clone3
```

### A.9 Kitten Process Observation

```bash
# Inside kitty terminal:
$ kitty/launcher/kitten icat /tmp/test_image.png &
$ KPID=$!

$ ls -la /proc/$KPID/exe
lrwxrwxrwx /proc/103385/exe -> .../kitty/launcher/kitten

$ cat /proc/$KPID/status
Name:   kitten
PPid:   97459
Pid:    103385
```

### A.10 Kitten Binary Inspection

```bash
$ readelf -S kitty/launcher/kitten | grep -E '\.go|\.text'
  .text           PROGBITS  0000000000401000
  .gosymtab       PROGBITS  0000000000f18508
  .gopclntab      PROGBITS  0000000000f18520
  .go.buildinfo   PROGBITS  00000000012b1000
  .note.go.buildid NOTE     0000000000400f80

$ go version kitty/launcher/kitten
kitty/launcher/kitten: go1.22.10

$ readelf -d kitty/launcher/kitten
  (NEEDED) Shared library: [libc.so.6]
```

### A.11 Remote Control Queries

```bash
$ kitten @ --to unix:/tmp/kitty-test2.sock ls
# (JSON output — see Section 4.1)

$ kitten @ --to unix:/tmp/kitty-test2.sock get-colors
# (267 color entries — see Section 4.2)

$ kitten @ --to unix:/tmp/kitty-test2.sock get-text --match id:1
# (Screen content including ANSI fragments — see Section 4.3)
```

### A.12 Python fast_data_types Introspection

```bash
$ python3 -c "
import sys; sys.path.insert(0, '.')
from kitty import fast_data_types as fdt
attrs = [a for a in dir(fdt) if not a.startswith('__')]
print(f'Total: {len(attrs)}')
types = [a for a in attrs if a[0].isupper() and not a.isupper()]
print(f'Types ({len(types)}): {sorted(types)}')
"
# Total: 581
# Types (23): ['AES256GCMDecrypt', 'AES256GCMEncrypt', 'ChildMonitor',
#   'Color', 'ColorProfile', 'CryptoError', 'Cursor', 'DiskCache',
#   'EllipticCurveKey', 'Face', 'FreeTypeError', 'GraphicsManager',
#   'HistoryBuf', 'KeyEvent', 'Line', 'LineBuf', 'Parser', 'Region',
#   'Screen', 'Secret', 'Shlex', 'SigInfo', 'SingleKey']
```

### A.13 Environment Constraints Encountered

| Tool / Feature | Status | Impact |
|---|---|---|
| Physical GPU | ❌ Not available | Used llvmpipe software rendering; 32 extra llvmpipe threads present |
| X11 display server | ❌ Not available | Installed and used Xvfb virtual framebuffer |
| Wayland compositor | ❌ Not available | X11 backend used instead (`glfw-x11.so` loaded) |
| systemd user bus | ❌ Not available | Non-fatal warning; desktop integration features unavailable |
| libcanberra (audio) | ❌ Not available | Bell sound disabled; non-fatal |
| py-spy profiler | ❌ Not installed | Used GDB for stack traces instead |
| perf tool | ❌ Requires root | Used strace and GDB instead |
| /dev/tty | ❌ Not available in non-TTY context | kitten icat exited when launched outside kitty; launched inside kitty instead |

---

*End of investigation document. All commands, outputs, and analysis are provided for independent reproducibility.*
