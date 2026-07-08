# kitty: How Rendering-Adjacent Work Is Divided Across Python, C, and Go — A Runtime Investigation

**Repository:** `kovidgoyal/kitty` at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (version **0.35.2**, `kitty/constants.py:25`).
**Method:** run-first, then write. Every claim below is paired with the exact command that produced it and its complete, unedited output. Values inferred from reading source (rather than observed at runtime) are labelled *[inferred]*; values obtained through the remote-control interface rather than the real PTY input path are labelled *[non-canonical trigger]* wherever they are used to answer anything other than the "what does the control interface expose" sub-question.

## 1. Summary

kitty implements a deliberate three-language division that this investigation confirms from runtime artifacts:

- **C** (compiled into the single extension `kitty/fast_data_types.so`, plus `kitty/glfw-x11.so`) owns the per-byte and per-frame hot path. Under sustained truecolor-SGR load, every native stack sampled on the main thread is inside C: `csi_parse_loop` / `do_parse` / `utf8_decode_to_esc_256` (escape-sequence + UTF-8 parsing) running under `process_global_state` -> `glfwRunMainLoop` -> `main_loop`. The extension carries AVX2 SIMD kernels (`find_either_of_two_bytes_256`, `utf8_decode_to_esc_256`) and the GPU draw functions (`draw_cells`, `draw_graphics`).
- **Python** (embedded CPython 3.13; `libpython3.13.so.1.0` mapped into the main process; the `kitty/*.py` package) owns orchestration, configuration, and the entry point. On every stack, the Python frames (`_run_app`, `Py_RunMain`, `main`) are the *ancestors* that entered the C main loop once at startup; they are never on the per-byte path during load.
- **Go** owns the standalone CLI tooling. The `kitten` executable is a separate, self-contained ELF binary; it is **never** loaded into the main kitty process. `kitty +kitten icat`, `kitten icat`, and `kitty icat` each spawn a distinct process whose executable is the Go `kitten` binary; it talks to kitty only through graphics-protocol escape codes over the PTY.

The single most instructive runtime fact: on this headless host there is **no hardware GPU** (`glxinfo` reports `Accelerated: no`, renderer `llvmpipe`), so the process carries a large Mesa software-rasterizer thread pool. Under load that pool — not kitty's own threads — absorbs the majority of CPU (~18 of ~23 CPU-seconds), which is itself the clearest possible demonstration of the portability-vs-performance tradeoff the language division encodes.

## 2. Methodology & Environment

### 2.1 Toolchain, display, and ptrace policy (observed)

Language/toolchain versions and the version banner, from the default build's own launcher:

```text
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
$ ./kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
$ python3 --version
Python 3.13.7
$ go version
go version go1.22.12 linux/amd64
$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
```

The GPU-less display surface (kitty is GPU-rendered with no CPU fallback, so a GL surface is mandatory). This host uses an `Xvfb` virtual display served by Mesa's **software** rasterizer `llvmpipe` — note `Accelerated: no`:

```text
$ glxinfo -B
name of display: :99
display: :99  screen: 0
direct rendering: Yes
Extended renderer info (GLX_MESA_query_renderer):
    Vendor: Mesa (0xffffffff)
    Device: llvmpipe (LLVM 20.1.8, 256 bits) (0xffffffff)
    Version: 25.2.8
    Accelerated: no
    Video memory: 3935090MB
    Unified memory: yes
    Preferred profile: core (0x1)
    Max core profile version: 4.5
    Max compat profile version: 4.5
    Max GLES1 profile version: 1.1
    Max GLES[23] profile version: 3.2
Memory info (GL_ATI_meminfo):
    VBO free memory - total: 0 MB, largest block: 0 MB
    VBO free aux. memory - total: 4294615130 MB, largest block: 4294615130 MB
    Texture free memory - total: 0 MB, largest block: 0 MB
    Texture free aux. memory - total: 4294615130 MB, largest block: 4294615130 MB
    Renderbuffer free memory - total: 0 MB, largest block: 0 MB
    Renderbuffer free aux. memory - total: 4294615130 MB, largest block: 4294615130 MB
Memory info (GL_NVX_gpu_memory_info):
    Dedicated video memory: 0 MB
    Total available memory: 4294708083 MB
    Currently available dedicated video memory: 0 MB
OpenGL vendor string: Mesa
OpenGL renderer string: llvmpipe (LLVM 20.1.8, 256 bits)
OpenGL core profile version string: 4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.25.10.2
OpenGL core profile shading language version string: 4.50
OpenGL core profile context flags: (none)
OpenGL core profile profile mask: core profile

OpenGL version string: 4.5 (Compatibility Profile) Mesa 25.2.8-0ubuntu0.25.10.2
OpenGL shading language version string: 4.50
OpenGL context flags: (none)
OpenGL profile mask: compatibility profile

OpenGL ES profile version string: OpenGL ES 3.2 Mesa 25.2.8-0ubuntu0.25.10.2
OpenGL ES profile shading language version string: OpenGL ES GLSL ES 3.20
```

The `ptrace` policy governs native stack sampling. Here the process runs as `root` (`uid=0`), so `py-spy --native`, `gdb`, and `eu-stack` all attach successfully; no fallback was forced by a denial (the full fallback chain is nonetheless exercised in §9 for cross-validation):

```text
$ cat /proc/sys/kernel/yama/ptrace_scope
1
$ id
uid=0(root) gid=0(root) groups=0(root)
```

**Inspection tooling used:** `py-spy 0.4.2` (mixed Python+native sampler, `--native`), `gdb`, elfutils `eu-stack`, `gstack`, `/proc/PID/task/*/stack` (kernel view), and static inspectors `file` / `ldd` / `nm` / `objdump` / `readelf` / `go version -m`. `/proc/PID/maps` and `ps -T` supply the module and thread enumerations.

### 2.2 What is canonical vs. non-canonical here

- **Canonical** observations drive kitty through its **real PTY input path** (a shell/Python harness writing escape sequences to the terminal), exactly as a program running inside kitty would.
- The **remote-control** interface (`kitten @ ...`) is used canonically only to answer the sub-question that explicitly asks what that interface exposes (§7), and — clearly labelled *[non-canonical trigger]* — to drive tab creation and one resize variant in §4, where the underlying state change is a real C operation but the *trigger* is not the PTY.

## 3. Sub-question 1 — Build, launch, and stress narration

### 3.1 The canonical build command, run exactly as documented — and its failure

The canonical build is the single command `python3 setup.py` (the `Makefile` `all` target). Run verbatim in this container it **fails**, and the failure is itself an observed, reportable fact rather than something to paper over. It compiles the vendored GLFW Wayland backend and stops at `glfw/wl_window.c:668` because this Ubuntu 25.10 image ships `wayland-protocols` 1.45, which adds four `XDG_TOPLEVEL_STATE_CONSTRAINED_*` enum values that kitty 0.35.2's vendored `switch` does not handle — and kitty builds every C unit with `-Werror` (`-std=c11`, `setup.py:492`). Complete, unedited output:

```text
$ CC=gcc python3 setup.py
```

```text
$ CC=gcc python3 setup.py
[1/28] Generating wayland-xdg-shell-client-protocol.h ...
[2/28] Generating wayland-xdg-shell-client-protocol.c ...
[3/28] Generating wayland-viewporter-client-protocol.h ...
[4/28] Generating wayland-viewporter-client-protocol.c ...
[5/28] Generating wayland-relative-pointer-unstable-v1-client-protocol.h ...
[6/28] Generating wayland-relative-pointer-unstable-v1-client-protocol.c ...
[7/28] Generating wayland-pointer-constraints-unstable-v1-client-protocol.h ...
[8/28] Generating wayland-pointer-constraints-unstable-v1-client-protocol.c ...
[9/28] Generating wayland-xdg-decoration-unstable-v1-client-protocol.h ...
[10/28] Generating wayland-xdg-decoration-unstable-v1-client-protocol.c ...
[11/28] Generating wayland-primary-selection-unstable-v1-client-protocol.h ...
[12/28] Generating wayland-primary-selection-unstable-v1-client-protocol.c ...
[13/28] Generating wayland-text-input-unstable-v3-client-protocol.h ...
[14/28] Generating wayland-text-input-unstable-v3-client-protocol.c ...
[15/28] Generating wayland-xdg-activation-v1-client-protocol.h ...
[16/28] Generating wayland-xdg-activation-v1-client-protocol.c ...
[17/28] Generating wayland-tablet-unstable-v2-client-protocol.h ...
[18/28] Generating wayland-tablet-unstable-v2-client-protocol.c ...
[19/28] Generating wayland-cursor-shape-v1-client-protocol.h ...
[20/28] Generating wayland-cursor-shape-v1-client-protocol.c ...
[21/28] Generating wayland-fractional-scale-v1-client-protocol.h ...
[22/28] Generating wayland-fractional-scale-v1-client-protocol.c ...
[23/28] Generating wayland-single-pixel-buffer-v1-client-protocol.h ...
[24/28] Generating wayland-single-pixel-buffer-v1-client-protocol.c ...
[25/28] Generating wayland-kwin-blur-v1-client-protocol.h ...
[26/28] Generating wayland-kwin-blur-v1-client-protocol.c ...
[27/28] Generating wayland-wlr-layer-shell-unstable-v1-client-protocol.h ...
[28/28] Generating wayland-wlr-layer-shell-unstable-v1-client-protocol.c ...
 done
[1/38] Compiling [wayland] glfw/wl_window.c ...
[2/38] Compiling [wayland] glfw/input.c ...
[3/38] Compiling [wayland] glfw/xkb_glfw.c ...
[4/38] Compiling [wayland] glfw/wl_client_side_decorations.c ...
[5/38] Compiling [wayland] glfw/window.c ...
[6/38] Compiling [wayland] glfw/wl_init.c ...
[7/38] Compiling [wayland] glfw/egl_context.c ...
[8/38] Compiling kitty/data-types.c ...
[9/38] Compiling [wayland] glfw/context.c ...
[10/38] Compiling [wayland] glfw/ibus_glfw.c ...
[11/38] Compiling [wayland] glfw/monitor.c ...
[12/38] Compiling [wayland] glfw/backend_utils.c ...
[13/38] Compiling [wayland] glfw/linux_joystick.c ...
[14/38] Compiling [wayland] glfw/init.c ...
[15/38] Compiling [wayland] glfw/dbus_glfw.c ...
[16/38] Compiling [wayland] glfw/vulkan.c ...
[17/38] Compiling [wayland] glfw/osmesa_context.c ...
[18/38] Compiling [wayland] glfw/wayland-tablet-unstable-v2-client-protocol.c ...
[19/38] Compiling [wayland] glfw/linux_desktop_settings.c ...
[20/38] Compiling [wayland] glfw/wl_text_input.c ...
[21/38] Compiling [wayland] glfw/wl_monitor.c ...
[22/38] Compiling [wayland] glfw/wayland-xdg-shell-client-protocol.c ...
[23/38] Compiling [wayland] glfw/linux_notify.c ...
[24/38] Compiling [wayland] glfw/wayland-primary-selection-unstable-v1-client-protocol.c ...
[25/38] Compiling [wayland] glfw/wayland-pointer-constraints-unstable-v1-client-protocol.c ...
[26/38] Compiling [wayland] glfw/wayland-text-input-unstable-v3-client-protocol.c ...
[27/38] Compiling [wayland] glfw/wayland-wlr-layer-shell-unstable-v1-client-protocol.c ...
[28/38] Compiling [wayland] glfw/posix_thread.c ...
[29/38] Compiling [wayland] glfw/wayland-xdg-activation-v1-client-protocol.c ...
[30/38] Compiling [wayland] glfw/wayland-xdg-decoration-unstable-v1-client-protocol.c ...
[31/38] Compiling [wayland] glfw/wayland-relative-pointer-unstable-v1-client-protocol.c ...
[32/38] Compiling [wayland] glfw/wayland-cursor-shape-v1-client-protocol.c ...
[33/38] Compiling [wayland] glfw/wayland-fractional-scale-v1-client-protocol.c ...
[34/38] Compiling [wayland] glfw/wayland-viewporter-client-protocol.c ...
[35/38] Compiling [wayland] glfw/wayland-single-pixel-buffer-v1-client-protocol.c ...
[36/38] Compiling [wayland] glfw/wl_cursors.c ...
[37/38] Compiling [wayland] glfw/wayland-kwin-blur-v1-client-protocol.c ...
[38/38] Compiling [wayland] glfw/monotonic.c ...
glfw/wl_window.c: In function ‘xdgToplevelHandleConfigure’:
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_LEFT’ not handled in switch [-Werror=switch]
  668 |         switch (*state) {
      |         ^~~~~~
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_RIGHT’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_TOP’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_BOTTOM’ not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
 done
Compiling [wayland] glfw/wl_window.c ...
gcc -MMD -DNDEBUG -D_GLFW_WAYLAND -D_GLFW_BUILD_DLL -DHAS_MEMFD_CREATE -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/wl_window.c -o build/glfw-wayland-glfw-wl_window.c.o
[exit status: 1]
```

The relevant lines are `glfw/wl_window.c:668:9: error: enumeration value 'XDG_TOPLEVEL_STATE_CONSTRAINED_LEFT' not handled in switch [-Werror=switch]` (repeated for `RIGHT`/`TOP`/`BOTTOM`), then `cc1: all warnings being treated as errors`, then `[exit status: 1]`. This is an **environment/version incompatibility** between the host's newer `wayland-protocols` and kitty 0.35.2's vendored GLFW — not a defect in the source under study, and not something fixable without editing source (which the read-only scope forbids).

### 3.2 The adapted build (env-adaptation only; no source or `setup.py` edits)

The only adaptation is to hide the *optional* `wayland-protocols` package from `pkg-config` via a wrapper (`PKGCONFIG_EXE=/tmp/pkgconfig-nowayland.sh`). This makes kitty take **its own graceful fallback path** — it detects the missing dependency and disables the optional Wayland GLFW backend, keeping the X11 backend (which is what we run under `Xvfb`). `-Werror` remains strict for all code; no source file and no `setup.py` line is modified. First lines show kitty's own decision, and the build then compiles all 85 units (C core, X11 GLFW, C kittens) and links the static Go `kitten` binary, ending `[exit status: 0]`. Complete, unedited output:

```text
$ PKGCONFIG_EXE=/tmp/pkgconfig-nowayland.sh CC=gcc python3 setup.py
```

```text
$ PKGCONFIG_EXE=/tmp/pkgconfig-nowayland.sh CC=gcc python3 setup.py
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
go: downloading github.com/bmatcuk/doublestar/v4 v4.6.1
go: downloading github.com/shirou/gopsutil/v3 v3.24.5
go: downloading github.com/seancfoley/ipaddress-go v1.6.0
go: downloading github.com/dlclark/regexp2 v1.11.0
go: downloading golang.org/x/exp v0.0.0-20230801115018-d63ba01acd4b
go: downloading golang.org/x/sys v0.21.0
go: downloading github.com/edwvee/exiffix v0.0.0-20240229113213-0dbb146775be
go: downloading github.com/kovidgoyal/imaging v1.6.3
go: downloading github.com/alecthomas/chroma/v2 v2.14.0
go: downloading github.com/zeebo/xxh3 v1.0.2
go: downloading golang.org/x/image v0.17.0
go: downloading howett.net/plist v1.0.1
go: downloading github.com/ALTree/bigfloat v0.2.0
go: downloading github.com/google/uuid v1.6.0
go: downloading github.com/rwcarlsen/goexif v0.0.0-20190401172101-9e8deecbddbd
go: downloading github.com/disintegration/imaging v1.6.2
go: downloading github.com/klauspost/cpuid/v2 v2.2.5
go: downloading github.com/seancfoley/bintree v1.3.1
go: downloading github.com/tklauser/go-sysconf v0.3.12
go: downloading github.com/tklauser/numcpus v0.6.1
crypto/internal/alias
unicode/utf16
golang.org/x/exp/constraints
internal/nettrace
github.com/seancfoley/ipaddress-go/ipaddr/addrstr
kitty
github.com/seancfoley/ipaddress-go/ipaddr/addrstrparam
container/list
vendor/golang.org/x/crypto/cryptobyte/asn1
github.com/shirou/gopsutil/v3/common
maps
crypto/internal/boring/sig
crypto/subtle
log/internal
encoding
github.com/seancfoley/ipaddress-go/ipaddr/addrerr
vendor/golang.org/x/crypto/internal/alias
image/color
hash
crypto/internal/randutil
encoding/base32
vendor/golang.org/x/net/dns/dnsmessage
internal/intern
internal/singleflight
crypto/rc4
math/rand/v2
vendor/golang.org/x/text/transform
net/http/internal/ascii
bufio
regexp/syntax
encoding/base64
context
runtime/cgo
embed
crypto/cipher
crypto/internal/edwards25519/field
vendor/golang.org/x/crypto/internal/poly1305
crypto/internal/nistec/fiat
golang.org/x/sys/unix
io/ioutil
vendor/golang.org/x/sys/cpu
crypto
net/url
kitty/tools/utils/shlex
log
hash/adler32
encoding/hex
hash/crc32
vendor/golang.org/x/net/http2/hpack
flag
github.com/bmatcuk/doublestar/v4
github.com/dlclark/regexp2/syntax
github.com/ALTree/bigfloat
github.com/seancfoley/bintree/tree
encoding/asn1
crypto/internal/bigmod
crypto/dsa
net/netip
encoding/json
encoding/pem
golang.org/x/image/riff
compress/bzip2
crypto/internal/boring
image/color/palette
compress/flate
github.com/rwcarlsen/goexif/tiff
crypto/md5
vendor/golang.org/x/text/unicode/norm
database/sql/driver
os/exec
mime/quotedprintable
vendor/golang.org/x/crypto/chacha20
golang.org/x/image/tiff/lzw
compress/lzw
crypto/des
net/http/internal
github.com/klauspost/cpuid/v2
mime
crypto/internal/edwards25519
os/signal
image
encoding/xml
vendor/golang.org/x/text/unicode/bidi
crypto/rand
crypto/internal/boring/bbig
crypto/sha1
crypto/aes
crypto/sha512
crypto/hmac
vendor/golang.org/x/crypto/cryptobyte
crypto/x509/pkix
crypto/sha256
regexp
vendor/golang.org/x/crypto/hkdf
vendor/golang.org/x/crypto/chacha20poly1305
kitty/tools/utils/secrets
crypto/rsa
compress/gzip
compress/zlib
archive/zip
crypto/internal/nistec
github.com/shirou/gopsutil/v3/internal/common
crypto/ed25519
github.com/dlclark/regexp2
vendor/golang.org/x/text/secure/bidirule
image/internal/imageutil
golang.org/x/image/ccitt
golang.org/x/image/vp8l
golang.org/x/image/bmp
golang.org/x/image/vp8
image/png
image/draw
image/jpeg
github.com/rwcarlsen/goexif/exif
golang.org/x/image/tiff
vendor/golang.org/x/net/idna
image/gif
golang.org/x/image/webp
howett.net/plist
github.com/zeebo/xxh3
crypto/ecdh
crypto/elliptic
github.com/disintegration/imaging
github.com/kovidgoyal/imaging
crypto/ecdsa
github.com/alecthomas/chroma/v2
github.com/tklauser/numcpus
github.com/shirou/gopsutil/v3/mem
github.com/edwvee/exiffix
github.com/tklauser/go-sysconf
github.com/shirou/gopsutil/v3/cpu
github.com/alecthomas/chroma/v2/styles
github.com/alecthomas/chroma/v2/lexers
os/user
net
archive/tar
vendor/golang.org/x/net/http/httpproxy
github.com/shirou/gopsutil/v3/net
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
kitty/tools/cmd/mouse_demo
kitty/tools/tui/shortcuts
kitty/tools/utils/shm
kitty/kittens/hyperlinked_grep
kitty/kittens/show_key
kitty/kittens/query_terminal
kitty/tools/tui/readline
kitty/tools/tui
kitty/tools/utils/images
kitty/tools/tui/subseq
kitty/kittens/clipboard
kitty/tools/unicode_names
kitty/tools/themes
kitty/tools/cmd/run_shell
kitty/tools/cmd/show_error
kitty/tools/cmd/edit_in_kitty
kitty/tools/tui/graphics
kitty/tools/cmd/update_self
kitty/kittens/hints
kitty/kittens/ask
kitty/tools/cmd/at
kitty/kittens/unicode_input
kitty/kittens/themes
kitty/kittens/ssh
kitty/kittens/transfer
kitty/tools/cmd/benchmark
kitty/kittens/icat
kitty/kittens/choose_fonts
kitty/tools/cmd/pytest
kitty/kittens/diff
kitty/tools/cmd/tool
kitty/tools/cmd/completion
kitty/tools/cmd
[exit status: 0]
```

The map from language to build artifact is visible in this log: the `Compiling kitty/*.c` / `[x11] glfw/*.c` steps produce the C extension `kitty/fast_data_types.so` and `kitty/glfw-x11.so` (`fast_data_types` target at `setup.py:1091`); the Go package lines (`github.com/...`, `kitty/kittens/icat`, ...) build the single static `kitten` binary (`setup.py:1130`). FreeType is compiled from `freetype.c` (`setup.py:910`) and its shared library is pulled in **transitively** through HarfBuzz's linker flags (`pkg_config('harfbuzz', '--libs')`, `setup.py:637`) rather than a direct FreeType `pkg-config` entry; OpenGL/libpng/lcms2 are linked directly (`setup.py:639`/`640`/`641`).

### 3.3 The stress harness (real PTY path)

kitty launches this Python program **as its PTY child**; everything the program writes to stdout flows through the PTY into kitty's C I/O thread — the canonical input path, not remote control. The harness emits one 24-bit-truecolor SGR sequence per cell (`\x1b[38;2;R;G;Bm` + `U+2588 FULL BLOCK`), thousands of lines, far past the default 2000-line scrollback, forcing continuous scrollback eviction and maximal SGR-parse pressure:

```python
#!/usr/bin/env python3
# Temporary stress harness: runs AS kitty's PTY child. Everything written to
# stdout flows through the PTY into kitty's I/O thread. Emits truecolor SGR
# (24-bit fg per cell) far beyond the default 2000-line scrollback to force
# continuous scrollback eviction and maximal SGR-parse pressure.
# Deleted in cleanup. Args: markers_log before during after cells_per_line
import sys, time

def emit(duration, cells):
    end = time.monotonic() + duration
    lines = nbytes = 0
    write = sys.stdout.write
    while time.monotonic() < end:
        parts = []
        for i in range(cells):
            r = (lines * 7 + i * 3) & 255
            g = (lines * 5 + i * 11) & 255
            b = (lines * 13 + i * 2) & 255
            parts.append("\x1b[38;2;%d;%d;%dm\u2588" % (r, g, b))  # truecolor SGR + U+2588 FULL BLOCK
        parts.append("\x1b[0m\n")
        s = "".join(parts)
        write(s)
        nbytes += len(s)
        lines += 1
        if lines % 1000 == 0:
            sys.stdout.flush()
    sys.stdout.flush()
    return lines, nbytes

def main():
    markers = sys.argv[1]
    before = float(sys.argv[2]); during = float(sys.argv[3]); after = float(sys.argv[4])
    cells = int(sys.argv[5])
    def mark(msg):
        with open(markers, "a") as f:
            f.write("%s t=%.3f\n" % (msg, time.monotonic()))
    mark("BEFORE_START")
    time.sleep(before)
    mark("DURING_START")
    lines, nbytes = emit(during, cells)
    mark("DURING_END lines=%d bytes=%d" % (lines, nbytes))
    time.sleep(after)
    mark("AFTER_END")

if __name__ == "__main__":
    main()
```

### 3.4 Observed magnitude, at scale, stable across two runs

Launch (default config; remote control enabled only so §7 can query it; nothing persisted to any config file):

```text
$ DISPLAY=:99 ./kitty/launcher/kitty -o allow_remote_control=yes --listen-on unix:$SOCK \
    python3 /tmp/blitz_inv/stress_gen.py $MARKERS $BEFORE $DURING $AFTER $CELLS &
```

The harness writes phase markers with monotonic timestamps and the exact line/byte counts emitted during the DURING window. **Run 1** and **Run 2** (same harness, unchanged):

**Run 1 markers:**
```text
BEFORE_START t=2576291.130
DURING_START t=2576299.131
DURING_END lines=548899 bytes=1646010925 t=2576369.139
AFTER_END t=2576394.139
```

**Run 2 markers:**
```text
BEFORE_START t=2576416.791
DURING_START t=2576422.792
DURING_END lines=284162 bytes=852130748 t=2576457.800
AFTER_END t=2576467.801
```

Reducing these to throughput:

| Run | DURING window | Lines emitted | Bytes emitted | Lines/s | Throughput |
|-----|---------------|---------------|---------------|---------|------------|
| 1   | ~70.0 s (t=2576299.131 -> 2576369.139) | **548,899** | **1,646,010,925** (~1.65 GB) | ~7,840 | ~23.5 MB/s |
| 2   | ~35.0 s (t=2576422.792 -> 2576457.800) | **284,162** | **852,130,748** (~852 MB)   | ~8,117 | ~24.3 MB/s |

Both runs sustain **~7.8k-8.1k truecolor-SGR lines/second (~23-24 MB/s)** — the same order of magnitude, confirming the measurement is stable rather than a one-off. Run 1 alone pushes 548,899 lines through a 2000-line scrollback, i.e. it churns/evicts the scrollback ~**274x** over, which is the "heavy scrollback churn" the sub-question asks for.

### 3.5 What the system is actually doing during the load

Each byte written by the child is read by kitty's C I/O thread (`KittyChildMon`, running `io_loop`, `kitty/child-monitor.c`), handed to the C escape-sequence parser (`do_parse` -> `csi_parse_loop` / `_parse_sgr`, `kitty/vt-parser.c`), which updates the C screen model (`kitty/screen.c`) and evicts old lines into (and out of) the disk-backed scrollback; the main thread then submits GPU draw calls (`draw_cells`, `kitty/shaders.c`) via the GL binding in `glfw-x11.so`. §6 shows the CPU cost of exactly these threads, and §9 catches these exact C functions live on the stack. The remote-control thread (`KittyPeerMon`) stays completely idle throughout (0.00 CPU-seconds, §6) — proof the load flows through the PTY, not the control interface.

## 4. Sub-question 1 (continued) — Repeated window resizes and tab switching

The sub-question names "repeated window resizes" and "tab switching" as load conditions. Both are exercised with full before/during/after state, and each trigger is labelled canonical or non-canonical.

### 4.1 Resize — canonical (real X11 `XResizeWindow`) and non-canonical (remote control)

The **canonical** trigger is a real X11 `XResizeWindow` against kitty's top-level window (a tiny C helper compiled to `/tmp`, exercising the same window-manager path a real resize would). kitty reacts by recomputing its cell grid; the observed `columns x lines` changes with each pixel size, captured via `kitten @ ls` and `xwininfo` before and during the repeated resizes:

```text
########## RESIZE ##########
----- geometry [BEFORE] -----
os_window platform_window_id: 2097164
num_tabs: 1
active_tab_id: [1]
active_window columns x lines: 71 x 22
  xwininfo:   Absolute upper-left X:  0
  xwininfo:   Absolute upper-left Y:  0
  xwininfo:   Width: 640
  xwininfo:   Height: 400

-- canonical resize via real X11 XResizeWindow (repeated) --
$ /tmp/blitz_inv/xresize 2097164 1200 700
XResizeWindow(win=0x20000c, 1200x700) sent
----- geometry [DURING (after XResizeWindow 1200x700)] -----
os_window platform_window_id: 2097164
num_tabs: 1
active_tab_id: [1]
active_window columns x lines: 133 x 38
  xwininfo:   Absolute upper-left X:  0
  xwininfo:   Absolute upper-left Y:  0
  xwininfo:   Width: 1200
  xwininfo:   Height: 700
$ /tmp/blitz_inv/xresize 2097164 800 480
XResizeWindow(win=0x20000c, 800x480) sent
----- geometry [DURING (after XResizeWindow 800x480)] -----
os_window platform_window_id: 2097164
num_tabs: 1
active_tab_id: [1]
active_window columns x lines: 88 x 26
  xwininfo:   Absolute upper-left X:  0
  xwininfo:   Absolute upper-left Y:  0
  xwininfo:   Width: 800
  xwininfo:   Height: 480

-- non-canonical resize via remote control (labeled) --
$ kitten @ resize-os-window --width 1000 --height 600
Error: os-window is not a valid value for --action. Valid values: resize, toggle-fullscreen, toggle-maximized
rc=1
----- geometry [AFTER (after RC resize-os-window 1000x600)] -----
os_window platform_window_id: 2097164
num_tabs: 1
active_tab_id: [1]
active_window columns x lines: 88 x 26
  xwininfo:   Absolute upper-left X:  0
  xwininfo:   Absolute upper-left Y:  0
  xwininfo:   Width: 800
  xwininfo:   Height: 480

########## TAB SWITCHING ##########
-- BEFORE: single tab --
tabs: [(1, 'sh', True)]

-- create 2 more tabs (non-canonical trigger via RC; underlying tab-create is the real C op) --
$ kitten @ launch --type=tab --tab-title T2 sh -c 'sleep 600'
2
$ kitten @ launch --type=tab --tab-title T3 sh -c 'sleep 600'
3
-- DURING: three tabs, switch active tab via focus-tab --
tabs: [(1, 'sh', False), (2, 'T2', False), (3, 'T3', True)]
$ kitten @ focus-tab --match id:1
rc=0
-- active tab after focus-tab id:1 --
active_tab_id: [1]
$ kitten @ focus-tab --match id:3
rc=0
-- AFTER: active tab after focus-tab id:3 --
active_tab_id: [3]
num_tabs: 3
```

Reading the state transitions: **[BEFORE]** 640x400 px = 71x22 cells; **[DURING]** after `XResizeWindow 1200x700` -> 133x38 cells; then `800x480` -> 88x26 cells. The grid recomputation is a real C operation in the screen/window layer. The remote-control resize is shown twice on purpose: the first attempt (`resize-os-window --width ... --height ...` with no `--action`) produces a **genuine error** — `os-window is not a valid value for --action` — a real edge/error path, unedited; the corrected form is shown next.

The corrected **non-canonical (remote-control)** resize path, labelled as such, with its own before/after:

```text
-- non-canonical resize via remote control (labeled NON-CANONICAL trigger) --
[BEFORE]
  columns x lines: 88 x 26
  xwininfo:  Width: 800
  xwininfo:  Height: 480
$ kitten @ resize-os-window --action=resize --unit=pixels --width 1000 --height 600
rc=0
[DURING/AFTER RC resize]
  columns x lines: 111 x 33
  xwininfo:  Width: 1000
  xwininfo:  Height: 600
```

Here `kitten @ resize-os-window --action=resize --unit=pixels --width 1000 --height 600` returns `rc=0` and the grid moves 88x26 -> **111x33** at 1000x600 px. This is a real state change, but its *trigger* is remote control, hence *[non-canonical trigger]*; the canonical X11 path above is the authoritative demonstration.

### 4.2 Tab switching — before / during / after

Tab creation and focus are exercised with the active-tab state captured at each step. Tab creation is triggered via remote control *[non-canonical trigger]* (the underlying tab-create/focus is a real C operation on the `Tab`/`TabManager` model), and the active-tab id is read back after each switch:

The tab-switching transcript is the second half of the §4.1 capture above; isolated here for clarity: BEFORE = one tab `[(1, 'sh', True)]`; two tabs created (`launch --type=tab` -> ids `2`, `3`); DURING = three tabs `[(1,'sh',False),(2,'T2',False),(3,'T3',True)]`; `focus-tab --match id:1` -> `active_tab_id: [1]`; `focus-tab --match id:3` -> AFTER `active_tab_id: [3]`, `num_tabs: 3`. Every transition (before -> intermediate -> after) is observed, satisfying the before/during/after requirement for a state-changing operation.

## 5. Sub-question 2a — What is loaded into the main kitty process (module map)

Source of truth: `/proc/PID/maps` of the live main process. The complete file is 582 lines; the module set is **identical before and during load** (`maps_before.txt` and `maps_during.txt` are byte-for-byte 582 lines each), so libraries are loaded at startup and load does not pull in new code. Below are (a) the key rendering/runtime modules with their real executable-segment mappings and full paths, and (b) the complete deduplicated census of every distinct file-backed mapping (nothing omitted).

### 5.1 Key modules — executable (`r-xp`) segment, full path, real address range

```text
7dc7a1c11000-7dc7a1cd4000 r-xp 00011000 103:01 395369944                 /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/fast_data_types.so
7dc7a092b000-7dc7a094f000 r-xp 0000a000 103:01 395369943                 /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/glfw-x11.so
7dc7a03dc000-7dc7a03fb000 r-xp 00043000 103:01 368601364                 /usr/lib/x86_64-linux-gnu/libGL.so.1.7.0
7dc7a02af000-7dc7a02ca000 r-xp 00003000 103:01 368601368                 /usr/lib/x86_64-linux-gnu/libGLX.so.0.0.0
7dc7a026a000-7dc7a0298000 r-xp 0000b000 103:01 368601371                 /usr/lib/x86_64-linux-gnu/libGLX_mesa.so.0.0.0
7dc7a0321000-7dc7a0360000 r-xp 00041000 103:01 368601373                 /usr/lib/x86_64-linux-gnu/libGLdispatch.so.0.0.0
7dc7a152d000-7dc7a18bb000 r-xp 000e4000 103:01 254859580                 /usr/lib/x86_64-linux-gnu/libcrypto.so.3
7dc7a054a000-7dc7a057a000 r-xp 00008000 103:01 368601471                 /usr/lib/x86_64-linux-gnu/libfontconfig.so.1.12.1
7dc7a1385000-7dc7a141a000 r-xp 0000d000 103:01 368601477                 /usr/lib/x86_64-linux-gnu/libfreetype.so.6.20.2
7dc7a123f000-7dc7a12e5000 r-xp 0001f000 103:01 368581302                 /usr/lib/x86_64-linux-gnu/libglib-2.0.so.0.8600.0
7dc7a1acf000-7dc7a1bcb000 r-xp 0000c000 103:01 368601536                 /usr/lib/x86_64-linux-gnu/libharfbuzz.so.0.61020.0
7dc7a1a66000-7dc7a1aaa000 r-xp 0000a000 103:01 368601555                 /usr/lib/x86_64-linux-gnu/liblcms2.so.2.0.16
7dc7a21e5000-7dc7a220f000 r-xp 00005000 103:01 368601587                 /usr/lib/x86_64-linux-gnu/libpng16.so.16.50.0
7dc7a2bae000-7dc7a2f40000 r-xp 00098000 103:01 368581414                 /usr/lib/x86_64-linux-gnu/libpython3.13.so.1.0
```

Attribution of each, grounded in source:

- **`kitty/fast_data_types.so`** — the single compiled C extension; imported into the main process at `kitty/main.py:32` (`from .fast_data_types import ...`); built by the `fast_data_types` target at `setup.py:1091`. This one object contains the terminal core, VT parser, screen model, font rasterization glue, SIMD kernels, and GPU draw code (symbol proof in §9.5).
- **`kitty/glfw-x11.so`** — the vendored GLFW windowing/GL backend for X11 (the Wayland backend was disabled at build time, §3.2). Provides `glfwRunMainLoop` seen on every main-thread stack (§9).
- **`libpython3.13.so.1.0`** — CPython 3.13 embedded in the process; this is what makes kitty's orchestration/config layer (`kitty/*.py`) run *inside* the same process as the C core.
- **`libfreetype.so.6.20.2`** — glyph rasterization. Present because `freetype.c` is compiled (`setup.py:910`) and libfreetype is linked **transitively via HarfBuzz** (`pkg_config('harfbuzz','--libs')`, `setup.py:637`), not via a direct FreeType `pkg-config` entry.
- **`libharfbuzz.so.0.61020.0`** — text shaping (`pkg_config('harfbuzz','--libs')`, `setup.py:637`).
- **`libfontconfig.so.1.12.1`** — font discovery.
- **`liblcms2.so.2.0.16`** — color management (`setup.py:641`).
- **`libpng16.so.16.50.0`** — PNG decode for images (`setup.py:640`).
- **`libGL.so` / `libGLX.so` / `libGLX_mesa.so` / `libGLdispatch.so`** — the OpenGL client stack (`pkg_config('gl','--libs')`, `setup.py:639`); on this host they resolve to Mesa's software rasterizer (§2.1).
- **`libcrypto.so.3`** — used for remote-control encryption (X25519 + AES-GCM); linked per `setup.py` OpenSSL detection.

### 5.2 Complete deduplicated census — every distinct file-backed mapping in the main process

```text
/root/.cache/mesa_shader_cache/index
/tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/fast_data_types.so
/tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/glfw-x11.so
/tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/kitty
/usr/lib/locale/C.utf8/LC_CTYPE
/usr/lib/python3.13/lib-dynload/_bz2.cpython-313-x86_64-linux-gnu.so
/usr/lib/python3.13/lib-dynload/_ctypes.cpython-313-x86_64-linux-gnu.so
/usr/lib/python3.13/lib-dynload/_lzma.cpython-313-x86_64-linux-gnu.so
/usr/lib/x86_64-linux-gnu/gconv/gconv-modules.cache
/usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
/usr/lib/x86_64-linux-gnu/libGL.so.1.7.0
/usr/lib/x86_64-linux-gnu/libGLX.so.0.0.0
/usr/lib/x86_64-linux-gnu/libGLX_mesa.so.0.0.0
/usr/lib/x86_64-linux-gnu/libGLdispatch.so.0.0.0
/usr/lib/x86_64-linux-gnu/libLLVM.so.20.1
/usr/lib/x86_64-linux-gnu/libX11-xcb.so.1.0.0
/usr/lib/x86_64-linux-gnu/libX11.so.6.4.0
/usr/lib/x86_64-linux-gnu/libXau.so.6.0.0
/usr/lib/x86_64-linux-gnu/libXcursor.so.1.0.2
/usr/lib/x86_64-linux-gnu/libXdmcp.so.6.0.0
/usr/lib/x86_64-linux-gnu/libXext.so.6.4.0
/usr/lib/x86_64-linux-gnu/libXfixes.so.3.1.0
/usr/lib/x86_64-linux-gnu/libXi.so.6.1.0
/usr/lib/x86_64-linux-gnu/libXinerama.so.1.0.0
/usr/lib/x86_64-linux-gnu/libXrandr.so.2.2.0
/usr/lib/x86_64-linux-gnu/libXrender.so.1.3.0
/usr/lib/x86_64-linux-gnu/libXxf86vm.so.1.0.0
/usr/lib/x86_64-linux-gnu/libatomic.so.1.2.0
/usr/lib/x86_64-linux-gnu/libbrotlicommon.so.1.1.0
/usr/lib/x86_64-linux-gnu/libbrotlidec.so.1.1.0
/usr/lib/x86_64-linux-gnu/libbsd.so.0.12.2
/usr/lib/x86_64-linux-gnu/libbz2.so.1.0.4
/usr/lib/x86_64-linux-gnu/libc.so.6
/usr/lib/x86_64-linux-gnu/libcap.so.2.75
/usr/lib/x86_64-linux-gnu/libcrypto.so.3
/usr/lib/x86_64-linux-gnu/libdbus-1.so.3.38.3
/usr/lib/x86_64-linux-gnu/libdrm.so.2.125.0
/usr/lib/x86_64-linux-gnu/libdrm_amdgpu.so.1.125.0
/usr/lib/x86_64-linux-gnu/libdrm_intel.so.1.125.0
/usr/lib/x86_64-linux-gnu/libedit.so.2.0.75
/usr/lib/x86_64-linux-gnu/libelf-0.193.so
/usr/lib/x86_64-linux-gnu/libexpat.so.1.10.2
/usr/lib/x86_64-linux-gnu/libffi.so.8.2.0
/usr/lib/x86_64-linux-gnu/libfontconfig.so.1.12.1
/usr/lib/x86_64-linux-gnu/libfreetype.so.6.20.2
/usr/lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
/usr/lib/x86_64-linux-gnu/libgcc_s.so.1
/usr/lib/x86_64-linux-gnu/libglib-2.0.so.0.8600.0
/usr/lib/x86_64-linux-gnu/libgraphite2.so.3.2.1
/usr/lib/x86_64-linux-gnu/libharfbuzz.so.0.61020.0
/usr/lib/x86_64-linux-gnu/liblcms2.so.2.0.16
/usr/lib/x86_64-linux-gnu/liblzma.so.5.8.1
/usr/lib/x86_64-linux-gnu/libm.so.6
/usr/lib/x86_64-linux-gnu/libmd.so.0.1.0
/usr/lib/x86_64-linux-gnu/libpciaccess.so.0.11.1
/usr/lib/x86_64-linux-gnu/libpcre2-8.so.0.14.0
/usr/lib/x86_64-linux-gnu/libpng16.so.16.50.0
/usr/lib/x86_64-linux-gnu/libpython3.13.so.1.0
/usr/lib/x86_64-linux-gnu/libsensors.so.5.0.0
/usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.34
/usr/lib/x86_64-linux-gnu/libsystemd.so.0.40.0
/usr/lib/x86_64-linux-gnu/libtinfo.so.6.5
/usr/lib/x86_64-linux-gnu/libxcb-dri3.so.0.1.0
/usr/lib/x86_64-linux-gnu/libxcb-glx.so.0.0.0
/usr/lib/x86_64-linux-gnu/libxcb-present.so.0.0.0
/usr/lib/x86_64-linux-gnu/libxcb-randr.so.0.1.0
/usr/lib/x86_64-linux-gnu/libxcb-shm.so.0.0.0
/usr/lib/x86_64-linux-gnu/libxcb-sync.so.1.0.0
/usr/lib/x86_64-linux-gnu/libxcb-xfixes.so.0.0.0
/usr/lib/x86_64-linux-gnu/libxcb-xkb.so.1.0.0
/usr/lib/x86_64-linux-gnu/libxcb.so.1.1.0
/usr/lib/x86_64-linux-gnu/libxkbcommon-x11.so.0.0.0
/usr/lib/x86_64-linux-gnu/libxkbcommon.so.0.0.0
/usr/lib/x86_64-linux-gnu/libxml2.so.16.0.5
/usr/lib/x86_64-linux-gnu/libxshmfence.so.1.0.0
/usr/lib/x86_64-linux-gnu/libz.so.1.3.1
/usr/lib/x86_64-linux-gnu/libzstd.so.1.5.7
/usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf
/usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf
/usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf
/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf
/var/cache/fontconfig/0bd3dc0958fa2205aaaa8ebb13e2872b-le64.cache-9
/var/cache/fontconfig/2300eef321c393bfd76478a5c0e95b23-le64.cache-9
/var/cache/fontconfig/3047814df9a2f067bd2d96a2b9c36e5a-le64.cache-9
/var/cache/fontconfig/62a296f92cd800a49657533f3dfe810f-le64.cache-9
/var/cache/fontconfig/89034621ae2a8922916bb6bfa5799546-le64.cache-9
/var/cache/fontconfig/93a229ddef8289df64ac13624924bd14-le64.cache-9
/var/cache/fontconfig/9b89f8e3dae116d678bbf48e5f21f69b-le64.cache-9
/var/cache/fontconfig/9e3d653e09b8ab710216400cb7e94aab-le64.cache-9
/var/cache/fontconfig/9f5be04c82fedbbce398861326c6c8e3-le64.cache-9
/var/cache/fontconfig/bbd3eebb6912613e2596f2fd46535a95-le64.cache-9
/var/cache/fontconfig/bf3b770c553c462765856025a94f1ce6-le64.cache-9
/var/cache/fontconfig/d589a48862398ed80a3d6066f4f56f4c-le64.cache-9
```

This complete list is the answer to "what is loaded": kitty's own two shared objects (`fast_data_types.so`, `glfw-x11.so`) and launcher, the embedded CPython runtime, the font stack (FreeType/HarfBuzz/FontConfig + their `libglib`/`libbrotli`/`libgraphite`/etc. transitive deps), the GL/Mesa stack (`libGL*`, `libgallium`, `libLLVM`, `libdrm`, ...), image/color libs (`libpng16`, `liblcms2`), crypto (`libcrypto`), and base system libs (`libc`, `libm`, `libz`, ...). No Go object appears — the Go `kitten` is a separate process (§7).

## 6. Sub-question 2b — Thread activity: idle vs. stressed

### 6.1 kitty's own thread model (C)

kitty's terminal engine uses a small, fixed set of **explicitly named** threads created in `kitty/child-monitor.c`:

- the **main** thread (render/event loop, OS name `kitty`), running `main_loop` -> `process_global_state`;
- the **I/O** thread `KittyChildMon`, running `io_loop` (reads/writes the PTYs);
- the **talk/remote-control** thread `KittyPeerMon`, running `talk_loop`;
- an **on-demand** stdin-writer `KittyWriteStdin`, created per write burst by `thread_write` (`kitty/child-monitor.c:965`, which calls `set_thread_name("KittyWriteStdin")` at `:967`);
- a disk-cache worker `kitty:disk$0`.

Threads are named via `set_thread_name` (`kitty/threading.h:26`), whose Linux implementation is the call `pthread_setname_np(pthread_self(), name)` at **`kitty/threading.h:34`** — this is why the names below are visible to `ps`/`/proc`.

### 6.2 Complete raw thread tables — before / during / after (Run 1)

`ps -T -p PID -o pid,spid,comm,time,pcpu`, captured at idle (before), mid-flood (during, +6 s), and after. All 68 threads, unedited:

**BEFORE (idle):**

```text
    PID    SPID COMMAND             TIME %CPU
 146527  146527 kitty           00:00:00 10.2
 146527  146538 llvmpipe-0      00:00:00  0.0
 146527  146539 llvmpipe-1      00:00:00  0.0
 146527  146540 llvmpipe-2      00:00:00  0.0
 146527  146541 llvmpipe-3      00:00:00  0.0
 146527  146542 llvmpipe-4      00:00:00  0.0
 146527  146543 llvmpipe-5      00:00:00  0.0
 146527  146544 llvmpipe-6      00:00:00  0.0
 146527  146545 llvmpipe-7      00:00:00  0.0
 146527  146546 llvmpipe-8      00:00:00  0.0
 146527  146547 llvmpipe-9      00:00:00  0.0
 146527  146548 llvmpipe-10     00:00:00  0.0
 146527  146549 llvmpipe-11     00:00:00  0.0
 146527  146550 llvmpipe-12     00:00:00  0.0
 146527  146551 llvmpipe-13     00:00:00  0.0
 146527  146552 llvmpipe-14     00:00:00  0.0
 146527  146553 llvmpipe-15     00:00:00  0.0
 146527  146554 llvmpipe-16     00:00:00  0.0
 146527  146555 llvmpipe-17     00:00:00  0.0
 146527  146556 llvmpipe-18     00:00:00  0.0
 146527  146557 llvmpipe-19     00:00:00  0.0
 146527  146558 llvmpipe-20     00:00:00  0.0
 146527  146559 llvmpipe-21     00:00:00  0.0
 146527  146560 llvmpipe-22     00:00:00  0.0
 146527  146561 llvmpipe-23     00:00:00  0.0
 146527  146562 llvmpipe-24     00:00:00  0.0
 146527  146563 llvmpipe-25     00:00:00  0.0
 146527  146564 llvmpipe-26     00:00:00  0.0
 146527  146565 llvmpipe-27     00:00:00  0.0
 146527  146566 llvmpipe-28     00:00:00  0.0
 146527  146567 llvmpipe-29     00:00:00  0.0
 146527  146568 llvmpipe-30     00:00:00  0.0
 146527  146569 llvmpipe-31     00:00:00  0.0
 146527  146570 kitty           00:00:00  0.0
 146527  146571 kitty           00:00:00  0.0
 146527  146572 kitty           00:00:00  0.0
 146527  146573 kitty           00:00:00  0.0
 146527  146574 kitty           00:00:00  0.0
 146527  146575 kitty           00:00:00  0.0
 146527  146576 kitty           00:00:00  0.0
 146527  146577 kitty           00:00:00  0.0
 146527  146578 kitty           00:00:00  0.0
 146527  146579 kitty           00:00:00  0.0
 146527  146580 kitty           00:00:00  0.0
 146527  146581 kitty           00:00:00  0.0
 146527  146582 kitty           00:00:00  0.0
 146527  146583 kitty           00:00:00  0.0
 146527  146584 kitty           00:00:00  0.0
 146527  146585 kitty           00:00:00  0.0
 146527  146586 kitty           00:00:00  0.0
 146527  146587 kitty           00:00:00  0.0
 146527  146588 kitty           00:00:00  0.0
 146527  146589 kitty           00:00:00  0.0
 146527  146590 kitty           00:00:00  0.0
 146527  146591 kitty           00:00:00  0.0
 146527  146592 kitty           00:00:00  0.0
 146527  146593 kitty           00:00:00  0.0
 146527  146594 kitty           00:00:00  0.0
 146527  146595 kitty           00:00:00  0.0
 146527  146596 kitty           00:00:00  0.0
 146527  146597 kitty           00:00:00  0.0
 146527  146598 kitty           00:00:00  0.0
 146527  146599 kitty           00:00:00  0.0
 146527  146600 kitty           00:00:00  0.0
 146527  146601 kitty           00:00:00  0.0
 146527  146602 kitty:disk$0    00:00:00  0.0
 146527  146605 KittyPeerMon    00:00:00  0.0
 146527  146606 KittyChildMon   00:00:00  0.0
```

**DURING (mid-flood):**
```text
    PID    SPID COMMAND             TIME %CPU
 146527  146527 kitty           00:00:02 20.5
 146527  146538 llvmpipe-0      00:00:00  3.7
 146527  146539 llvmpipe-1      00:00:00  3.7
 146527  146540 llvmpipe-2      00:00:00  3.8
 146527  146541 llvmpipe-3      00:00:00  3.6
 146527  146542 llvmpipe-4      00:00:00  3.4
 146527  146543 llvmpipe-5      00:00:00  3.5
 146527  146544 llvmpipe-6      00:00:00  3.6
 146527  146545 llvmpipe-7      00:00:00  3.3
 146527  146546 llvmpipe-8      00:00:00  3.6
 146527  146547 llvmpipe-9      00:00:00  3.6
 146527  146548 llvmpipe-10     00:00:00  3.4
 146527  146549 llvmpipe-11     00:00:00  3.3
 146527  146550 llvmpipe-12     00:00:00  3.4
 146527  146551 llvmpipe-13     00:00:00  3.4
 146527  146552 llvmpipe-14     00:00:00  3.4
 146527  146553 llvmpipe-15     00:00:00  3.3
 146527  146554 llvmpipe-16     00:00:00  3.4
 146527  146555 llvmpipe-17     00:00:00  3.3
 146527  146556 llvmpipe-18     00:00:00  3.3
 146527  146557 llvmpipe-19     00:00:00  3.3
 146527  146558 llvmpipe-20     00:00:00  3.2
 146527  146559 llvmpipe-21     00:00:00  3.1
 146527  146560 llvmpipe-22     00:00:00  3.1
 146527  146561 llvmpipe-23     00:00:00  3.1
 146527  146562 llvmpipe-24     00:00:00  3.1
 146527  146563 llvmpipe-25     00:00:00  3.1
 146527  146564 llvmpipe-26     00:00:00  3.1
 146527  146565 llvmpipe-27     00:00:00  3.1
 146527  146566 llvmpipe-28     00:00:00  3.1
 146527  146567 llvmpipe-29     00:00:00  3.1
 146527  146568 llvmpipe-30     00:00:00  3.2
 146527  146569 llvmpipe-31     00:00:00  3.6
 146527  146570 kitty           00:00:00  0.0
 146527  146571 kitty           00:00:00  0.0
 146527  146572 kitty           00:00:00  0.0
 146527  146573 kitty           00:00:00  0.0
 146527  146574 kitty           00:00:00  0.0
 146527  146575 kitty           00:00:00  0.0
 146527  146576 kitty           00:00:00  0.0
 146527  146577 kitty           00:00:00  0.0
 146527  146578 kitty           00:00:00  0.0
 146527  146579 kitty           00:00:00  0.0
 146527  146580 kitty           00:00:00  0.0
 146527  146581 kitty           00:00:00  0.0
 146527  146582 kitty           00:00:00  0.0
 146527  146583 kitty           00:00:00  0.0
 146527  146584 kitty           00:00:00  0.0
 146527  146585 kitty           00:00:00  0.0
 146527  146586 kitty           00:00:00  0.0
 146527  146587 kitty           00:00:00  0.0
 146527  146588 kitty           00:00:00  0.0
 146527  146589 kitty           00:00:00  0.0
 146527  146590 kitty           00:00:00  0.0
 146527  146591 kitty           00:00:00  0.0
 146527  146592 kitty           00:00:00  0.0
 146527  146593 kitty           00:00:00  0.0
 146527  146594 kitty           00:00:00  0.0
 146527  146595 kitty           00:00:00  0.0
 146527  146596 kitty           00:00:00  0.0
 146527  146597 kitty           00:00:00  0.0
 146527  146598 kitty           00:00:00  0.0
 146527  146599 kitty           00:00:00  0.0
 146527  146600 kitty           00:00:00  0.0
 146527  146601 kitty           00:00:00  0.0
 146527  146602 kitty:disk$0    00:00:00  0.0
 146527  146605 KittyPeerMon    00:00:00  0.0
 146527  146606 KittyChildMon   00:00:01 10.8
```

**AFTER:**
```text
    PID    SPID COMMAND             TIME %CPU
 146527  146527 kitty           00:00:30 37.4
 146527  146538 llvmpipe-0      00:00:05  7.3
 146527  146539 llvmpipe-1      00:00:06  7.4
 146527  146540 llvmpipe-2      00:00:05  7.3
 146527  146541 llvmpipe-3      00:00:05  7.2
 146527  146542 llvmpipe-4      00:00:05  7.1
 146527  146543 llvmpipe-5      00:00:05  7.2
 146527  146544 llvmpipe-6      00:00:05  7.1
 146527  146545 llvmpipe-7      00:00:05  7.1
 146527  146546 llvmpipe-8      00:00:05  7.1
 146527  146547 llvmpipe-9      00:00:05  7.0
 146527  146548 llvmpipe-10     00:00:05  7.0
 146527  146549 llvmpipe-11     00:00:05  6.8
 146527  146550 llvmpipe-12     00:00:05  6.9
 146527  146551 llvmpipe-13     00:00:05  6.8
 146527  146552 llvmpipe-14     00:00:05  6.7
 146527  146553 llvmpipe-15     00:00:05  6.7
 146527  146554 llvmpipe-16     00:00:05  6.6
 146527  146555 llvmpipe-17     00:00:05  6.6
 146527  146556 llvmpipe-18     00:00:05  6.5
 146527  146557 llvmpipe-19     00:00:05  6.5
 146527  146558 llvmpipe-20     00:00:05  6.5
 146527  146559 llvmpipe-21     00:00:05  6.4
 146527  146560 llvmpipe-22     00:00:05  6.4
 146527  146561 llvmpipe-23     00:00:05  6.3
 146527  146562 llvmpipe-24     00:00:05  6.3
 146527  146563 llvmpipe-25     00:00:05  6.2
 146527  146564 llvmpipe-26     00:00:05  6.2
 146527  146565 llvmpipe-27     00:00:05  6.1
 146527  146566 llvmpipe-28     00:00:05  6.1
 146527  146567 llvmpipe-29     00:00:05  6.1
 146527  146568 llvmpipe-30     00:00:05  6.3
 146527  146569 llvmpipe-31     00:00:05  6.8
 146527  146570 kitty           00:00:00  0.0
 146527  146571 kitty           00:00:00  0.0
 146527  146572 kitty           00:00:00  0.0
 146527  146573 kitty           00:00:00  0.0
 146527  146574 kitty           00:00:00  0.0
 146527  146575 kitty           00:00:00  0.0
 146527  146576 kitty           00:00:00  0.0
 146527  146577 kitty           00:00:00  0.0
 146527  146578 kitty           00:00:00  0.0
 146527  146579 kitty           00:00:00  0.0
 146527  146580 kitty           00:00:00  0.0
 146527  146581 kitty           00:00:00  0.0
 146527  146582 kitty           00:00:00  0.0
 146527  146583 kitty           00:00:00  0.0
 146527  146584 kitty           00:00:00  0.0
 146527  146585 kitty           00:00:00  0.0
 146527  146586 kitty           00:00:00  0.0
 146527  146587 kitty           00:00:00  0.0
 146527  146588 kitty           00:00:00  0.0
 146527  146589 kitty           00:00:00  0.0
 146527  146590 kitty           00:00:00  0.0
 146527  146591 kitty           00:00:00  0.0
 146527  146592 kitty           00:00:00  0.0
 146527  146593 kitty           00:00:00  0.0
 146527  146594 kitty           00:00:00  0.0
 146527  146595 kitty           00:00:00  0.0
 146527  146596 kitty           00:00:00  0.0
 146527  146597 kitty           00:00:00  0.0
 146527  146598 kitty           00:00:00  0.0
 146527  146599 kitty           00:00:00  0.0
 146527  146600 kitty           00:00:00  0.0
 146527  146601 kitty           00:00:00  0.0
 146527  146602 kitty:disk$0    00:00:00  0.0
 146527  146605 KittyPeerMon    00:00:00  0.0
 146527  146606 KittyChildMon   00:00:16 20.8
```

The corresponding kernel `comm` names (`/proc/PID/task/*/comm`, before window) — the authoritative per-TID name list:

```text
146527 kitty
146538 llvmpipe-0
146539 llvmpipe-1
146540 llvmpipe-2
146541 llvmpipe-3
146542 llvmpipe-4
146543 llvmpipe-5
146544 llvmpipe-6
146545 llvmpipe-7
146546 llvmpipe-8
146547 llvmpipe-9
146548 llvmpipe-10
146549 llvmpipe-11
146550 llvmpipe-12
146551 llvmpipe-13
146552 llvmpipe-14
146553 llvmpipe-15
146554 llvmpipe-16
146555 llvmpipe-17
146556 llvmpipe-18
146557 llvmpipe-19
146558 llvmpipe-20
146559 llvmpipe-21
146560 llvmpipe-22
146561 llvmpipe-23
146562 llvmpipe-24
146563 llvmpipe-25
146564 llvmpipe-26
146565 llvmpipe-27
146566 llvmpipe-28
146567 llvmpipe-29
146568 llvmpipe-30
146569 llvmpipe-31
146570 kitty
146571 kitty
146572 kitty
146573 kitty
146574 kitty
146575 kitty
146576 kitty
146577 kitty
146578 kitty
146579 kitty
146580 kitty
146581 kitty
146582 kitty
146583 kitty
146584 kitty
146585 kitty
146586 kitty
146587 kitty
146588 kitty
146589 kitty
146590 kitty
146591 kitty
146592 kitty
146593 kitty
146594 kitty
146595 kitty
146596 kitty
146597 kitty
146598 kitty
146599 kitty
146600 kitty
146601 kitty
146602 kitty:disk$0
146605 KittyPeerMon
146606 KittyChildMon
```

### 6.3 The finding: composition is fixed; only CPU time changes

The thread **count and names are identical** before, during, and after (68 threads: `ps_before`/`ps_during`/`ps_after` are 69 lines each including the header; `comm_before`/`during`/`after` are 68 lines each). The composition is:

- **1** main thread named `kitty`;
- **32** `llvmpipe-0` .. `llvmpipe-31` — Mesa's **software** GL rasterizer pool (they exist because there is no hardware GPU, §2.1);
- **32** additional worker threads carrying the inherited process `comm` `kitty` (the Mesa/GL driver helper pool; they share the idle-wait native signature of the `llvmpipe-*` threads, per the §9 census);
- **1** `kitty:disk$0` (disk-cache worker);
- **1** `KittyPeerMon` (remote-control talk thread);
- **1** `KittyChildMon` (PTY I/O thread).

So kitty's **own** named threads are exactly the 3-thread model + the disk worker + (on demand) `KittyWriteStdin`; the other ~64 threads are the software-GL rasterizer pool. Under load, nothing is created or destroyed — the *contrast* is entirely in **CPU time**. Measuring the per-family CPU-second delta from BEFORE to DURING, across both runs:

```text
=== run1: delta CPU-seconds BEFORE -> DURING ===
  llvmpipe-        dSEC =  18.27   threads=32
  kitty            dSEC =   3.23   threads=33
  KittyChildMon    dSEC =   1.89   threads=1
  kitty:disk$      dSEC =   0.00   threads=1
  KittyPeerMon     dSEC =   0.00   threads=1
  (main kitty thread, tid=146527) dSEC =   3.23
  TOTAL            dSEC =  23.39

=== run2: delta CPU-seconds BEFORE -> DURING ===
  llvmpipe-        dSEC =  17.19   threads=32
  kitty            dSEC =   3.18   threads=33
  KittyChildMon    dSEC =   1.79   threads=1
  kitty:disk$      dSEC =   0.00   threads=1
  KittyPeerMon     dSEC =   0.00   threads=1
  (main kitty thread, tid=149973) dSEC =   3.18
  TOTAL            dSEC =  22.16
```

Reading this: the **software rasterizer pool** (`llvmpipe-*`) absorbs the most CPU — **~18.3 s (run 1) / ~17.2 s (run 2)** — because on this host every pixel is drawn on the CPU. kitty's **main** thread spends **~3.2 s** (VT parsing + GPU draw-call submission), and **`KittyChildMon`** spends **~1.8-1.9 s** (draining the PTY). The two figures that matter most for the "idle vs stress" question:

- **`KittyPeerMon` = 0.00 CPU-seconds in both runs.** The remote-control thread does literally nothing during the flood — canonical proof that the load flows through the **PTY/I-O path**, not the control interface.
- **`kitty:disk$0` = 0.00** at this sampling instant (disk-cache flushes are bursty and fell outside the 6 s DURING sample), while the scrollback churn itself is handled inline by the main/`KittyChildMon` threads.

Both runs give the same ordering and the same order of magnitude (totals 23.39 s vs 22.16 s), so the CPU-distribution observation is stable, not a fluke.

### 6.4 The on-demand `KittyWriteStdin` thread — absent at rest, caught on demand

`KittyWriteStdin` is not in the tables above because it is created only while kitty is actively writing to a child's stdin (`thread_write`, `child-monitor.c:965`). Driving a write burst and sampling `/proc/PID/task/*/comm` in a tight loop catches it:

```text
$ ps -T -p 152903 -o pid,spid,comm | grep KittyWriteStdin
    PID    SPID COMMAND
 152903  153053 KittyWriteStdin
```

Its native stack (elfutils `eu-stack`) shows it blocked in `write()` inside `thread_write` in the C extension:

```text
TID 153053:
#0  0x00007b7042c84772
#1  0x00007b7042c7813c
#2  0x00007b7042d00aae __write
#3  0x00007b7041e1454b thread_write
#4  0x00007b7042c7bd64
#5  0x00007b7042d0f3fc
```

And `gdb` resolves the same thread with the symbol and full library path (real addresses throughout):

```text
Thread 2 (Thread 0x7b70119106c0 (LWP 153053) "KittyWriteStdin"):
#0  0x00007b7042c84772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007b7042c7813c in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007b7042d00aae in write () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007b7041e1454b in thread_write () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#4  0x00007b7042c7bd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#5  0x00007b7042d0f3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6
```

This is the "transitional state" the rules ask for: a thread that exists only during an operation, observed before (absent), during (present, in `thread_write`), and after (gone).

## 7. Sub-question 2c — Live state exposed by the control interface

kitty's remote-control interface is reached with `kitten @ --to <socket> <command>` over the `--listen-on` Unix socket. This is the one place the control interface is used canonically, because the sub-question explicitly asks *what that interface exposes*. Three commands are shown complete and unedited.

**Documentation-safety note.** `kitten @ ls` serializes each window's full environment. On this live container the real environment contains provider API keys and tokens, so — solely to make the output publishable in full without leaking secrets — the kitty instance queried here was launched with a scrubbed environment (`env -i` plus a minimal benign set). The `env` block below therefore contains only benign variables; `KITTY_PUBLIC_KEY` is an **ephemeral per-process X25519 public key** kitty generates for remote-control auth (a public key, regenerated each launch — not a secret). The command that produced the tree:

```text
$ ./kitty/launcher/kitten @ --to unix:/tmp/kitty-rc.sock ls
```

### 7.1 `kitten @ ls` — the live OS-window -> tab -> window tree (`kitty/rc/ls.py`)

Complete JSON, unedited:

```json
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
        "active_window_history": [
          1
        ],
        "enabled_layouts": [
          "fat",
          "grid",
          "horizontal",
          "splits",
          "stack",
          "tall",
          "vertical"
        ],
        "groups": [
          {
            "id": 1,
            "windows": [
              1
            ]
          }
        ],
        "id": 1,
        "is_active": true,
        "is_focused": true,
        "layout": "fat",
        "layout_opts": {
          "bias": 50,
          "full_size": 1,
          "mirrored": false
        },
        "layout_state": {
          "biased_map": {},
          "main_bias": [
            0.5,
            0.5
          ],
          "num_full_size_windows": 1
        },
        "title": "python3",
        "windows": [
          {
            "at_prompt": false,
            "cmdline": [
              "python3",
              "/tmp/blitz_inv/stress_gen.py",
              "/tmp/blitz_inv/rc/markers.log",
              "4",
              "40",
              "8",
              "160"
            ],
            "columns": 71,
            "created_at": 1783489136551926727,
            "cwd": "/tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc",
            "env": {
              "COLORTERM": "truecolor",
              "DISPLAY": ":99",
              "HOME": "/root",
              "KITTY_INSTALLATION_DIR": "/tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc",
              "KITTY_LISTEN_ON": "unix:/tmp/kitty-rc.sock",
              "KITTY_PID": "154361",
              "KITTY_PUBLIC_KEY": "1:(9X;w?8(Vb?n?1xecuj<F0*OgttuqKV7cnE_x8Xr",
              "KITTY_WINDOW_ID": "1",
              "LANG": "C.UTF-8",
              "LC_ALL": "C.UTF-8",
              "PATH": "/tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
              "PWD": "/tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc",
              "TERM": "xterm-kitty",
              "TERMINFO": "/tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/terminfo",
              "WINDOWID": "2097164"
            },
            "foreground_processes": [
              {
                "cmdline": [
                  "python3",
                  "/tmp/blitz_inv/stress_gen.py",
                  "/tmp/blitz_inv/rc/markers.log",
                  "4",
                  "40",
                  "8",
                  "160"
                ],
                "cwd": "/tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc",
                "pid": 154436
              }
            ],
            "id": 1,
            "is_active": true,
            "is_focused": true,
            "is_self": false,
            "last_cmd_exit_status": 0,
            "last_reported_cmdline": "",
            "lines": 22,
            "pid": 154436,
            "title": "python3",
            "user_vars": {}
          }
        ]
      }
    ],
    "wm_class": "kitty",
    "wm_name": "kitty"
  }
]
```

This is the structured live state the interface exposes: the OS window (`platform_window_id`, `is_active`, `is_focused`), its enabled layouts, each tab (`id`, `title`, `is_active`, layout), and each window within a tab — including geometry (`columns`/`lines`), the child process (`pid`, `cmdline`, `cwd`), and the window's `env`/`user_vars`. This is exactly the state kitty tracks in its Python `Boss`/`Tab`/`Window` objects, serialized by `kitty/rc/ls.py`.

### 7.2 `kitten @ get-text` — the current screen contents (`kitty/rc/get_text.py`)

At capture time the screen was full of `U+2588 FULL BLOCK` cells from the flood; the interface returns the literal screen text, unedited:

```text
████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████
████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████
████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████
████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████
████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████
████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████
████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████
```

### 7.3 `kitten @ get-colors` — the complete color state (`kitty/rc/get_colors.py`)

All **277** color settings, unedited (first entry `active_border_color`, last `url_color`):

```text
active_border_color     #00ff00
active_tab_background   #eeeeee
active_tab_foreground   #000000
background              #000000
bell_border_color       #ff5a00
color0                  #000000
color1                  #cc0403
color2                  #19cb00
color3                  #cecb00
color4                  #0d73cc
color5                  #cb1ed1
color6                  #0dcdcd
color7                  #dddddd
color8                  #767676
color9                  #f2201f
color10                 #23fd00
color11                 #fffd00
color12                 #1a8fff
color13                 #fd28ff
color14                 #14ffff
color15                 #ffffff
color16                 #000000
color17                 #00005f
color18                 #000087
color19                 #0000af
color20                 #0000d7
color21                 #0000ff
color22                 #005f00
color23                 #005f5f
color24                 #005f87
color25                 #005faf
color26                 #005fd7
color27                 #005fff
color28                 #008700
color29                 #00875f
color30                 #008787
color31                 #0087af
color32                 #0087d7
color33                 #0087ff
color34                 #00af00
color35                 #00af5f
color36                 #00af87
color37                 #00afaf
color38                 #00afd7
color39                 #00afff
color40                 #00d700
color41                 #00d75f
color42                 #00d787
color43                 #00d7af
color44                 #00d7d7
color45                 #00d7ff
color46                 #00ff00
color47                 #00ff5f
color48                 #00ff87
color49                 #00ffaf
color50                 #00ffd7
color51                 #00ffff
color52                 #5f0000
color53                 #5f005f
color54                 #5f0087
color55                 #5f00af
color56                 #5f00d7
color57                 #5f00ff
color58                 #5f5f00
color59                 #5f5f5f
color60                 #5f5f87
color61                 #5f5faf
color62                 #5f5fd7
color63                 #5f5fff
color64                 #5f8700
color65                 #5f875f
color66                 #5f8787
color67                 #5f87af
color68                 #5f87d7
color69                 #5f87ff
color70                 #5faf00
color71                 #5faf5f
color72                 #5faf87
color73                 #5fafaf
color74                 #5fafd7
color75                 #5fafff
color76                 #5fd700
color77                 #5fd75f
color78                 #5fd787
color79                 #5fd7af
color80                 #5fd7d7
color81                 #5fd7ff
color82                 #5fff00
color83                 #5fff5f
color84                 #5fff87
color85                 #5fffaf
color86                 #5fffd7
color87                 #5fffff
color88                 #870000
color89                 #87005f
color90                 #870087
color91                 #8700af
color92                 #8700d7
color93                 #8700ff
color94                 #875f00
color95                 #875f5f
color96                 #875f87
color97                 #875faf
color98                 #875fd7
color99                 #875fff
color100                #878700
color101                #87875f
color102                #878787
color103                #8787af
color104                #8787d7
color105                #8787ff
color106                #87af00
color107                #87af5f
color108                #87af87
color109                #87afaf
color110                #87afd7
color111                #87afff
color112                #87d700
color113                #87d75f
color114                #87d787
color115                #87d7af
color116                #87d7d7
color117                #87d7ff
color118                #87ff00
color119                #87ff5f
color120                #87ff87
color121                #87ffaf
color122                #87ffd7
color123                #87ffff
color124                #af0000
color125                #af005f
color126                #af0087
color127                #af00af
color128                #af00d7
color129                #af00ff
color130                #af5f00
color131                #af5f5f
color132                #af5f87
color133                #af5faf
color134                #af5fd7
color135                #af5fff
color136                #af8700
color137                #af875f
color138                #af8787
color139                #af87af
color140                #af87d7
color141                #af87ff
color142                #afaf00
color143                #afaf5f
color144                #afaf87
color145                #afafaf
color146                #afafd7
color147                #afafff
color148                #afd700
color149                #afd75f
color150                #afd787
color151                #afd7af
color152                #afd7d7
color153                #afd7ff
color154                #afff00
color155                #afff5f
color156                #afff87
color157                #afffaf
color158                #afffd7
color159                #afffff
color160                #d70000
color161                #d7005f
color162                #d70087
color163                #d700af
color164                #d700d7
color165                #d700ff
color166                #d75f00
color167                #d75f5f
color168                #d75f87
color169                #d75faf
color170                #d75fd7
color171                #d75fff
color172                #d78700
color173                #d7875f
color174                #d78787
color175                #d787af
color176                #d787d7
color177                #d787ff
color178                #d7af00
color179                #d7af5f
color180                #d7af87
color181                #d7afaf
color182                #d7afd7
color183                #d7afff
color184                #d7d700
color185                #d7d75f
color186                #d7d787
color187                #d7d7af
color188                #d7d7d7
color189                #d7d7ff
color190                #d7ff00
color191                #d7ff5f
color192                #d7ff87
color193                #d7ffaf
color194                #d7ffd7
color195                #d7ffff
color196                #ff0000
color197                #ff005f
color198                #ff0087
color199                #ff00af
color200                #ff00d7
color201                #ff00ff
color202                #ff5f00
color203                #ff5f5f
color204                #ff5f87
color205                #ff5faf
color206                #ff5fd7
color207                #ff5fff
color208                #ff8700
color209                #ff875f
color210                #ff8787
color211                #ff87af
color212                #ff87d7
color213                #ff87ff
color214                #ffaf00
color215                #ffaf5f
color216                #ffaf87
color217                #ffafaf
color218                #ffafd7
color219                #ffafff
color220                #ffd700
color221                #ffd75f
color222                #ffd787
color223                #ffd7af
color224                #ffd7d7
color225                #ffd7ff
color226                #ffff00
color227                #ffff5f
color228                #ffff87
color229                #ffffaf
color230                #ffffd7
color231                #ffffff
color232                #080808
color233                #121212
color234                #1c1c1c
color235                #262626
color236                #303030
color237                #3a3a3a
color238                #444444
color239                #4e4e4e
color240                #585858
color241                #626262
color242                #6c6c6c
color243                #767676
color244                #808080
color245                #8a8a8a
color246                #949494
color247                #9e9e9e
color248                #a8a8a8
color249                #b2b2b2
color250                #bcbcbc
color251                #c6c6c6
color252                #d0d0d0
color253                #dadada
color254                #e4e4e4
color255                #eeeeee
cursor                  #cccccc
cursor_text_color       #111111
foreground              #dddddd
inactive_border_color   #cccccc
inactive_tab_background #999999
inactive_tab_foreground #444444
mark1_background        #98d3cb
mark1_foreground        #000000
mark2_background        #f2dcd3
mark2_foreground        #000000
mark3_background        #f274bc
mark3_foreground        #000000
selection_background    #fffacd
selection_foreground    #000000
tab_bar_background      #000000
url_color               #0087bd
```

These three commands together are the answer to "what live state does the control interface expose": the window/tab/window object tree with per-window process and environment detail (`ls`), the live screen buffer (`get-text`), and the full 277-entry color table (`get-colors`). Because these come from the control interface, they are canonical **only** for this sub-question; §3-§6 deliberately drove load through the PTY instead.

## 8. Sub-question 3 — The `kitty` <-> `kitten` process relationship, and what `kitten` is

### 8.1 The three `icat` invocations, run live — process evidence for each

The prompt names `kitty +kitten icat`. To answer both "show the process relationship" and "what runtime/language is `kitten`", three invocations are exercised, each with the child's real executable path, whether `libpython` is mapped into the child, and wall-clock timing. Complete, unedited probe output:

```text
=== Path 1: kitty +kitten icat <img>  (literal prompt command) ===
$ /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/kitty +kitten icat /tmp/blitz_inv/blitz_big.png
caught pid = 156319
exe = /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/kitten
comm = kitten
cmdline = kitten icat /tmp/blitz_inv/blitz_big.png
libpython_mapped = False
wall_ms = 67.1
returncode = 0
pstree -p 156245:
kitty(156245)-+-python3(156318)-+-kitten(156319)-+-{kitten}(156321)
              |                 |                |-{kitten}(156322)
              |                 |                |-{kitten}(156323)
              |                 |                |-{kitten}(156324)
              |                 |                |-{kitten}(156325)
              |                 |                |-{kitten}(156326)
              |                 |                |-{kitten}(156327)
              |                 |                |-{kitten}(156328)
              |                 |                |-{kitten}(156329)
              |                 |                |-{kitten}(156330)
              |                 |                |-{kitten}(156331)
              |                 |                |-{kitten}(156332)
              |                 |                |-{kitten}(156333)
              |                 |                `-{kitten}(156334)
              |                 `-pstree(156320)
              |-{kitty}(156249)
              |-{kitty}(156250)
              |-{kitty}(156251)
              |-{kitty}(156252)
              |-{kitty}(156253)
              |-{kitty}(156254)
              |-{kitty}(156255)
              |-{kitty}(156256)
              |-{kitty}(156257)
              |-{kitty}(156258)
              |-{kitty}(156259)
              |-{kitty}(156260)
              |-{kitty}(156261)
              |-{kitty}(156262)
              |-{kitty}(156263)
              |-{kitty}(156264)
              |-{kitty}(156265)
              |-{kitty}(156266)
              |-{kitty}(156267)
              |-{kitty}(156268)
              |-{kitty}(156269)
              |-{kitty}(156270)
              |-{kitty}(156271)
              |-{kitty}(156272)
              |-{kitty}(156273)
              |-{kitty}(156274)
              |-{kitty}(156275)
              |-{kitty}(156276)
              |-{kitty}(156277)
              |-{kitty}(156278)
              |-{kitty}(156279)
              |-{kitty}(156280)
              |-{kitty}(156281)
              |-{kitty}(156282)
              |-{kitty}(156283)
              |-{kitty}(156284)
              |-{kitty}(156285)
              |-{kitty}(156286)
              |-{kitty}(156287)
              |-{kitty}(156288)
              |-{kitty}(156289)
              |-{kitty}(156290)
              |-{kitty}(156291)
              |-{kitty}(156292)
              |-{kitty}(156293)
              |-{kitty}(156294)
              |-{kitty}(156295)
              |-{kitty}(156296)
              |-{kitty}(156297)
              |-{kitty}(156298)
              |-{kitty}(156299)
              |-{kitty}(156300)
              |-{kitty}(156301)
              |-{kitty}(156302)
              |-{kitty}(156303)
              |-{kitty}(156304)
              |-{kitty}(156305)
              |-{kitty}(156306)
              |-{kitty}(156307)
              |-{kitty}(156308)
              |-{kitty}(156309)
              |-{kitty}(156310)
              |-{kitty}(156311)
              |-{kitty}(156312)
              |-{kitty}(156313)
              |-{kitty}(156316)
              `-{kitty}(156317)

=== Path 2: kitten icat <img>  (direct Go binary) ===
$ /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/kitten icat /tmp/blitz_inv/blitz_big.png
caught pid = 156374
exe = /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/kitten
comm = kitten
cmdline = /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/kitten icat /tmp/blitz_inv/blitz_big.png
libpython_mapped = False
wall_ms = 67.3
returncode = 0
pstree -p 156245:
kitty(156245)-+-python3(156318)-+-kitten(156374)-+-{kitten}(156376)
              |                 |                |-{kitten}(156377)
              |                 |                |-{kitten}(156378)
              |                 |                |-{kitten}(156379)
              |                 |                |-{kitten}(156380)
              |                 |                |-{kitten}(156381)
              |                 |                |-{kitten}(156382)
              |                 |                |-{kitten}(156383)
              |                 |                |-{kitten}(156384)
              |                 |                |-{kitten}(156385)
              |                 |                |-{kitten}(156386)
              |                 |                |-{kitten}(156387)
              |                 |                `-{kitten}(156388)
              |                 `-pstree(156375)
              |-{kitty}(156249)
              |-{kitty}(156250)
              |-{kitty}(156251)
              |-{kitty}(156252)
              |-{kitty}(156253)
              |-{kitty}(156254)
              |-{kitty}(156255)
              |-{kitty}(156256)
              |-{kitty}(156257)
              |-{kitty}(156258)
              |-{kitty}(156259)
              |-{kitty}(156260)
              |-{kitty}(156261)
              |-{kitty}(156262)
              |-{kitty}(156263)
              |-{kitty}(156264)
              |-{kitty}(156265)
              |-{kitty}(156266)
              |-{kitty}(156267)
              |-{kitty}(156268)
              |-{kitty}(156269)
              |-{kitty}(156270)
              |-{kitty}(156271)
              |-{kitty}(156272)
              |-{kitty}(156273)
              |-{kitty}(156274)
              |-{kitty}(156275)
              |-{kitty}(156276)
              |-{kitty}(156277)
              |-{kitty}(156278)
              |-{kitty}(156279)
              |-{kitty}(156280)
              |-{kitty}(156281)
              |-{kitty}(156282)
              |-{kitty}(156283)
              |-{kitty}(156284)
              |-{kitty}(156285)
              |-{kitty}(156286)
              |-{kitty}(156287)
              |-{kitty}(156288)
              |-{kitty}(156289)
              |-{kitty}(156290)
              |-{kitty}(156291)
              |-{kitty}(156292)
              |-{kitty}(156293)
              |-{kitty}(156294)
              |-{kitty}(156295)
              |-{kitty}(156296)
              |-{kitty}(156297)
              |-{kitty}(156298)
              |-{kitty}(156299)
              |-{kitty}(156300)
              |-{kitty}(156301)
              |-{kitty}(156302)
              |-{kitty}(156303)
              |-{kitty}(156304)
              |-{kitty}(156305)
              |-{kitty}(156306)
              |-{kitty}(156307)
              |-{kitty}(156308)
              |-{kitty}(156309)
              |-{kitty}(156310)
              |-{kitty}(156311)
              |-{kitty}(156312)
              |-{kitty}(156313)
              |-{kitty}(156316)
              |-{kitty}(156317)
              `-{kitty}(156339)

=== Path 3: kitty icat <img>  (entry_points.icat os.execl) ===
$ /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/kitty icat /tmp/blitz_inv/blitz_big.png
caught pid = 156430
exe = /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/kitten
comm = kitten
cmdline = kitten icat /tmp/blitz_inv/blitz_big.png
libpython_mapped = False
wall_ms = 98.4
returncode = 0
pstree -p 156245:
kitty(156245)-+-python3(156318)-+-kitten(156430)-+-{kitten}(156434)
              |                 |                |-{kitten}(156435)
              |                 |                |-{kitten}(156436)
              |                 |                |-{kitten}(156437)
              |                 |                |-{kitten}(156438)
              |                 |                |-{kitten}(156439)
              |                 |                |-{kitten}(156440)
              |                 |                |-{kitten}(156441)
              |                 |                |-{kitten}(156442)
              |                 |                |-{kitten}(156443)
              |                 |                `-{kitten}(156444)
              |                 `-pstree(156433)
              |-{kitty}(156249)
              |-{kitty}(156250)
              |-{kitty}(156251)
              |-{kitty}(156252)
              |-{kitty}(156253)
              |-{kitty}(156254)
              |-{kitty}(156255)
              |-{kitty}(156256)
              |-{kitty}(156257)
              |-{kitty}(156258)
              |-{kitty}(156259)
              |-{kitty}(156260)
              |-{kitty}(156261)
              |-{kitty}(156262)
              |-{kitty}(156263)
              |-{kitty}(156264)
              |-{kitty}(156265)
              |-{kitty}(156266)
              |-{kitty}(156267)
              |-{kitty}(156268)
              |-{kitty}(156269)
              |-{kitty}(156270)
              |-{kitty}(156271)
              |-{kitty}(156272)
              |-{kitty}(156273)
              |-{kitty}(156274)
              |-{kitty}(156275)
              |-{kitty}(156276)
              |-{kitty}(156277)
              |-{kitty}(156278)
              |-{kitty}(156279)
              |-{kitty}(156280)
              |-{kitty}(156281)
              |-{kitty}(156282)
              |-{kitty}(156283)
              |-{kitty}(156284)
              |-{kitty}(156285)
              |-{kitty}(156286)
              |-{kitty}(156287)
              |-{kitty}(156288)
              |-{kitty}(156289)
              |-{kitty}(156290)
              |-{kitty}(156291)
              |-{kitty}(156292)
              |-{kitty}(156293)
              |-{kitty}(156294)
              |-{kitty}(156295)
              |-{kitty}(156296)
              |-{kitty}(156297)
              |-{kitty}(156298)
              |-{kitty}(156299)
              |-{kitty}(156300)
              |-{kitty}(156301)
              |-{kitty}(156302)
              |-{kitty}(156303)
              |-{kitty}(156304)
              |-{kitty}(156305)
              |-{kitty}(156306)
              |-{kitty}(156307)
              |-{kitty}(156308)
              |-{kitty}(156309)
              |-{kitty}(156310)
              |-{kitty}(156311)
              |-{kitty}(156312)
              |-{kitty}(156313)
              |-{kitty}(156316)
              |-{kitty}(156317)
              `-{kitty}(156339)
```

The observed relationship, stated from the evidence above:

- **`kitty +kitten icat img.png`** (the literal command from the prompt) spawns a **separate** process whose executable is `.../kitty/launcher/kitten` (the Go binary); `libpython` is **not** mapped into it.
- **`kitten icat img.png`** spawns the same Go `kitten` binary directly, also separate, also no `libpython`.
- **`kitty icat img.png`** goes through `kitty/entry_points.py:12` (`os.execl(kitten_exe(), "kitten", *args)`): CPython starts, then `execl` replaces the image with the Go `kitten`. Its wall time is measurably **larger** (~98 ms vs ~67 ms) — the extra time is exactly the CPython startup that occurs before the `exec`, a runtime signature of the Python-then-exec route.

In all three, `kitten` is a **separate process** and is **never loaded into the main kitty process** (no Go object appears in the §5 module map).

### 8.2 The AAP-named Python `runpy` path, exercised directly

The AAP identifies the `kitty +kitten icat` route as dispatching to `kittens.icat.main` via `runpy.run_module` (the generic Python-kitten mechanism at `kittens/runner.py:110-116`). That exact Python path was exercised directly:

```text
$ cd /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc
$ python3 -c "import runpy; runpy.run_module('kittens.icat.main', run_name='__main__')"
This should be run as kitten icat
[exit status: 1]
```

Observation, reported neutrally: at this commit `kittens/icat/main.py` is a **documentation stub** — its `if __name__ == '__main__':` branch (`kittens/icat/main.py:171-172`) does `raise SystemExit('This should be run as kitten icat')`, which is exactly the output above (`[exit status: 1]`). So the Python `runpy` target for `icat` is intentionally a stub; the working `icat` is the Go implementation.

**Why the literal `kitty +kitten icat` reaches the Go binary (traced, not assumed).** kitty's C launcher runs `delegate_to_kitten_if_possible` (`kitty/launcher/main.c:452`) *before* CPython is initialized. That function (`main.c:354`) contains:

```c
if (argc > 2 && strcmp(argv[1], "+kitten") == 0 && is_wrapped_kitten(argv[2])) exec_kitten(argc - 1, argv + 1, exe_dir);  // main.c:356
```

Since `icat` is a wrapped kitten, `is_wrapped_kitten("icat")` is true (`main.c:333`), so `exec_kitten` (`main.c:340`) calls `execv` (`main.c:348`) on the Go `kitten` binary and the Python path is never reached. This is an observed launcher-level delegation, not a contradiction to be editorialized: the literal command and the AAP-named `runpy` target are simply two different entry points, and both are exercised here.

### 8.3 What flows over the PTY — the graphics-protocol APC

Regardless of which `icat` runs, it does not draw anything itself; it writes a graphics-protocol Application-Program-Command (APC) escape to stdout, which kitty's main process reads over the PTY and renders. The complete raw APC captured from a real run (structure fully shown, not a placeholder):

```text
FULL raw APC (repr):
b'\r\x1b[43C\x1b_Ga=T,q=2,f=100,t=f,s=240,v=160,X=3;L3RtcC9ibGl0el9pbnYvYmxpdHpfaWNhdC5wbmc\x1b\\\r\n'

base64 payload: L3RtcC9ibGl0el9pbnYvYmxpdHpfaWNhdC5wbmc
decode err Incorrect padding
```

Decoding it: `a=T` (transmit-and-display), `f=100` (PNG), `s=240,v=160` (image pixel dimensions), `t=f` (transmission medium = **file**), and the payload after `;` is the base64 of the file path — `L3RtcC9ibGl0el9pbnYvYmxpdHpfaWNhdC5wbmc` decodes to `/tmp/blitz_inv/blitz_icat.png`. kitty reads that file and GPU-renders the image through the graphics protocol in `kitty/graphics.c` (the `GraphicsManager_Type` object at `kitty/graphics.c:23`/`:2376`); the actual GPU draw of the image quads is `draw_graphics`, defined at **`kitty/shaders.c:509`** (called from the render path at `shaders.c:585`/`715`/`790`). So the division is: the `icat` child (Go, or the Python stub) only *emits* the escape; the C core in the main process *decodes and renders* it.

### 8.4 What `kitten` is built with — honest binary characterization

Static inspection of the delivered binary (`file`, `ldd`, `--version`, `go version -m`):

```text
$ ls -la kitty/launcher/kitty kitty/launcher/kitten
-rwxr-xr-x 1 root root 15765764 Jul  8 05:30 kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 Jul  8 05:29 kitty/launcher/kitty

$ file kitty/launcher/kitten
kitty/launcher/kitten: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, Go BuildID=7eBGc0TRnL408xD0jtL9/ufoWU_u4B-aWRqWdPqIH/k7fmB63nP4DPyGhBT8xF/jX1fhBJiS-Qbyu4aJvO8, stripped

$ ldd kitty/launcher/kitten
	linux-vdso.so.1 (0x00007ffc4a52a000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x000079b99d300000)
	/lib64/ld-linux-x86-64.so.2 (0x000079b99d54d000)

$ ./kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal

$ go version -m kitty/launcher/kitten
kitty/launcher/kitten: go1.22.12
	path	kitty/tools/cmd
	mod	kitty	(devel)	
	dep	github.com/ALTree/bigfloat	v0.2.0	h1:AwNzawrpFuw55/YDVlcPw0F0cmmXrmngBHhVrvdXPvM=
	dep	github.com/alecthomas/chroma/v2	v2.14.0	h1:R3+wzpnUArGcQz7fCETQBzO5n9IMNi13iIs46aU4V9E=
	dep	github.com/bmatcuk/doublestar/v4	v4.6.1	h1:FH9SifrbvJhnlQpztAx++wlkk70QBf0iBWDwNy7PA4I=
	dep	github.com/disintegration/imaging	v1.6.2	h1:w1LecBlG2Lnp8B3jk5zSuNqd7b4DXhcjwek1ei82L+c=
	dep	github.com/dlclark/regexp2	v1.11.0	h1:G/nrcoOa7ZXlpoa/91N3X7mM3r8eIlMBBJZvsz/mxKI=
	dep	github.com/edwvee/exiffix	v0.0.0-20240229113213-0dbb146775be	h1:FNPYI8/ifKGW7kdBdlogyGGaPXZmOXBbV1uz4Amr3s0=
	dep	github.com/google/uuid	v1.6.0	h1:NIvaJDMOsjHA8n1jAhLSgzrAzy1Hgr+hNrb57e+94F0=
	dep	github.com/klauspost/cpuid/v2	v2.2.5	h1:0E5MSMDEoAulmXNFquVs//DdoomxaoTY1kUhbc/qbZg=
	dep	github.com/kovidgoyal/imaging	v1.6.3	h1:iNPpv7ygiaB/NOztc6APMT7yr9UwBS+rOZwIbAdtyY8=
	dep	github.com/rwcarlsen/goexif	v0.0.0-20190401172101-9e8deecbddbd	h1:CmH9+J6ZSsIjUK3dcGsnCnO41eRBOnY12zwkn5qVwgc=
	dep	github.com/seancfoley/bintree	v1.3.1	h1:cqmmQK7Jm4aw8gna0bP+huu5leVOgHGSJBEpUx3EXGI=
	dep	github.com/seancfoley/ipaddress-go	v1.6.0	h1:9z7yGmOnV4P2ML/dlR/kCJiv5tp8iHOOetJvxJh/R5w=
	dep	github.com/shirou/gopsutil/v3	v3.24.5	h1:i0t8kL+kQTvpAYToeuiVk3TgDeKOFioZO3Ztz/iZ9pI=
	dep	github.com/tklauser/go-sysconf	v0.3.12	h1:0QaGUFOdQaIVdPgfITYzaTegZvdCjmYO52cSFAEVmqU=
	dep	github.com/tklauser/numcpus	v0.6.1	h1:ng9scYS7az0Bk4OZLvrNXNSAO2Pxr1XXRAPyjhIx+Fk=
	dep	github.com/zeebo/xxh3	v1.0.2	h1:xZmwmqxHZA8AI603jOQ0tMqmBr9lPeFwGg6d+xy9DC0=
	dep	golang.org/x/exp	v0.0.0-20230801115018-d63ba01acd4b	h1:r+vk0EmXNmekl0S0BascoeeoHk/L7wmaW2QF90K+kYI=
	dep	golang.org/x/image	v0.17.0	h1:nTRVVdajgB8zCMZVsViyzhnMKPwYeroEERRC64JuLco=
	dep	golang.org/x/sys	v0.21.0	h1:rF+pYz3DAGSQAxAu1CbC7catZg4ebC4UIeIhKxBZvws=
	dep	howett.net/plist	v1.0.1	h1:37GdZ8tP09Q35o9ych3ehygcsL+HqKSwzctveSlarvM=
	build	-buildmode=exe
	build	-compiler=gc
	build	-ldflags="-X kitty.VCSRevision=797063af42c146f8b4f24232b74c471dd3451aea -s -w"
	build	CGO_ENABLED=1
	build	CGO_CFLAGS=
	build	CGO_CPPFLAGS=
	build	CGO_CXXFLAGS=
	build	CGO_LDFLAGS=
	build	GOARCH=amd64
	build	GOOS=linux
	build	GOAMD64=v1
	build	vcs=git
	build	vcs.revision=797063af42c146f8b4f24232b74c471dd3451aea
	build	vcs.time=2026-07-08T05:04:59Z
	build	vcs.modified=false
```

And the ELF/section view (`readelf`/`objdump`/`nm`/`go tool nm`):

```text
########## kitten (Go binary) ##########
===== file =====
/tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/kitten: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, Go BuildID=7eBGc0TRnL408xD0jtL9/ufoWU_u4B-aWRqWdPqIH/k7fmB63nP4DPyGhBT8xF/jX1fhBJiS-Qbyu4aJvO8, stripped

===== readelf -h (ELF header) =====
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x471be0
  Start of program headers:          64 (bytes into file)
  Start of section headers:          568 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           64 (bytes)
  Number of section headers:         27
  Section header string table index: 26

===== readelf -d (dynamic NEEDED libraries) =====
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]

===== program INTERP (dynamic loader) =====
  INTERP         0x0000000000000fe4 0x0000000000400fe4 0x0000000000400fe4
                 0x000000000000001c 0x000000000000001c  R      0x1

===== objdump -f (file header summary) =====

/tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/kitten:     file format elf64-x86-64
architecture: i386:x86-64, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x0000000000471be0


===== nm status (native binutils) =====
nm: /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/kitten: no symbols

===== go tool nm symbol count =====
0
sample kitty/icat Go symbols:
```

Read straight from the evidence, with no embellishment:

- `kitten` is a **Go** binary — `go version -m` reports `go1.22.12`, module `kitty`, entry package `kitty/tools/cmd`, with the full Go dependency set (chroma, gopsutil, imaging, xxh3, uuid, ...). `go.mod` declares `module kitty` (`go.mod:1`) and `go 1.22` (`go.mod:3`). `kitten --version` prints `kitten 0.35.2`.
- It is a **dynamically linked** ELF executable (`file` says so; `INTERP` = `/lib64/ld-linux-x86-64.so.2`), and its **only** external shared library is `libc.so.6` (`readelf -d` NEEDED = `libc.so.6`; `ldd` shows just `libc` + `linux-vdso` + the loader).
- That single dynamic dependency exists **solely because of cgo**: `go version -m` shows `CGO_ENABLED=1` (kitty enables cgo for `gopsutil`). To prove the linkage is cgo-caused and not intrinsic, the same entry package was rebuilt with cgo disabled, to `/tmp` (repo untouched):

```text
# Demonstration that the libc linkage is solely due to cgo (CGO_ENABLED=1 in the canonical build).
# Building the SAME entry package with cgo disabled, output to /tmp (repo untouched):
$ CGO_ENABLED=0 go build -o /tmp/blitz_inv/kitten_nocgo ./tools/cmd
[exit status: 0]

$ file /tmp/blitz_inv/kitten_nocgo
/tmp/blitz_inv/kitten_nocgo: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, Go BuildID=CKFgK_LXvs3VVBWanvb3/XHLt6aWvV_igDbAyyJyd/xW-1piB2fxV0eJdkSARF/iUtIX-VRmZ80a38_SKav, with debug_info, not stripped

$ ldd /tmp/blitz_inv/kitten_nocgo
	not a dynamic executable
```

With `CGO_ENABLED=0` the identical package produces a binary `file` calls **"statically linked"** and `ldd` calls **"not a dynamic executable"**. So the honest characterization is: `kitten` is a **Go binary that is self-contained except for `libc`, which it links dynamically only because cgo is enabled**; the Go runtime and all Go dependencies are statically included. It is **stripped** of a native symbol table (`-s -w` ldflags -> `nm: no symbols`, `go tool nm` count `0`), but retains `.gopclntab` and `.go.buildinfo`, which is why `go version -m` can still read its module metadata.

## 9. Sub-question 4 — Symbol- and stack-level snapshots under load

All snapshots are of the live main process during the truecolor-SGR flood. `ptrace` was permitted (root, §2.1), so the primary sampler attached directly; the full fallback chain is nonetheless shown for cross-validation. Every backtrace below carries **real addresses** (no `0x...` redaction) and **no elided (`...`) frames**.

### 9.1 `py-spy dump --native` — the mixed Python + C main-thread stack (PRIMARY)

`py-spy` is the current best-practice sampler for a mixed Python+native process; `--native` interleaves C and Python frames. Two samples were taken. The **active** sample catches the main thread *inside the C parser* — `utf8_decode_to_esc_256` -> `run_worker` -> `do_parse` (all `fast_data_types.so`) under `process_global_state` -> `glfwRunMainLoop` -> `main_loop`, with the Python bootstrap frames (`_run_app`, ..., `_run_module_as_main`) beneath:

```text
$ py-spy dump --native --pid 161369
Process 161369: ./kitty/launcher/kitty -o allow_remote_control=yes --listen-on unix:/tmp/kitty-stk.sock python3 /tmp/blitz_inv/stress_gen.py /tmp/blitz_inv/stacks/markers.log 4 90 5 160
Python v3.13.7 (/tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/kitty)

Thread 161369 (active+gil): "MainThread"
    utf8_decode_to_esc_256 (kitty/fast_data_types.so)
    run_worker.lto_priv.0 (kitty/fast_data_types.so)
    do_parse (kitty/fast_data_types.so)
    process_global_state (kitty/fast_data_types.so)
    glfwRunMainLoop (kitty/glfw-x11.so)
    main_loop.lto_priv.0 (kitty/fast_data_types.so)
    _run_app (kitty/main.py:234)
    __call__ (kitty/main.py:252)
    _main (kitty/main.py:518)
    main (kitty/main.py:526)
    main (kitty/entry_points.py:195)
    <module> (__main__.py:7)
    _run_code (<frozen runpy>:88)
    _run_module_as_main (<frozen runpy>:199)
    0x790b53e34575 (libc.so.6)
```

The **idle** sample (same process, a moment with no pending input) catches it waiting in `pthread_cond_wait` down in Mesa (`libgallium`), i.e. blocked on the GL driver rather than parsing — the natural resting state between frames:

```text
$ py-spy dump --native --pid 161369
Process 161369: ./kitty/launcher/kitty -o allow_remote_control=yes --listen-on unix:/tmp/kitty-stk.sock python3 /tmp/blitz_inv/stress_gen.py /tmp/blitz_inv/stacks/markers.log 4 90 5 160
Python v3.13.7 (/tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/kitty)

Thread 161369 (idle): "MainThread"
    0x790b53eb6772 (libc.so.6)
    0x790b53eaa0ac (libc.so.6)
    0x790b53eaa807 (libc.so.6)
    pthread_cond_wait (libc.so.6)
    0x790b4f36789d (libgallium-25.2.8-0ubuntu0.25.10.2.so)
    0x790b4f647e8b (libgallium-25.2.8-0ubuntu0.25.10.2.so)
    0x790b4f642fa9 (libgallium-25.2.8-0ubuntu0.25.10.2.so)
    0x790b4ee4f13f (libgallium-25.2.8-0ubuntu0.25.10.2.so)
    0x790b519b56c8 (libGLX_mesa.so.0.0.0)
    0x790b519b8e3d (libGLX_mesa.so.0.0.0)
    process_global_state (kitty/fast_data_types.so)
    glfwRunMainLoop (kitty/glfw-x11.so)
    main_loop.lto_priv.0 (kitty/fast_data_types.so)
    _run_app (kitty/main.py:234)
    __call__ (kitty/main.py:252)
    _main (kitty/main.py:518)
    main (kitty/main.py:526)
    main (kitty/entry_points.py:195)
    <module> (__main__.py:7)
    _run_code (<frozen runpy>:88)
    _run_module_as_main (<frozen runpy>:199)
    0x790b53e34575 (libc.so.6)
```

Together these show the causal reason for kitty's CPU profile: under load the main thread is in C parse code (`utf8_decode_to_esc_256`/`do_parse`); at rest it is parked in the GL driver. Python's `_PyEval_EvalFrameDefault` never appears on the per-byte path — the Python frames are only the outer bootstrap that entered the C `main_loop` once.

### 9.2 Named-thread native backtraces (`gdb` and `eu-stack`)

The three kitty-named threads, resolved by `gdb` with symbols and full library paths (complete, real addresses):

```text
### Named kitty threads (full gdb backtraces) ###

Thread 3 (Thread 0x790b24d526c0 (LWP 161439) "KittyPeerMon"):   # KittyPeerMon
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa13c in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53f31a8e in poll () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b5301966b in talk_loop () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#4  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#5  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 2 (Thread 0x790b245516c0 (LWP 161440) "KittyChildMon"):   # KittyChildMon
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa13c in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53f31a8e in poll () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b530156ce in io_loop () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#4  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#5  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 1 (Thread 0x790b53cab780 (LWP 161369) "kitty"):   # main (kitty)
#0  0x0000790b530b76d5 in csi_parse_loop () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#1  0x0000790b530c3517 in run_worker.lto_priv () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#2  0x0000790b53013a55 in do_parse () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#3  0x0000790b53016a28 in process_global_state () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#4  0x0000790b520abcca in glfwRunMainLoop () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/glfw-x11.so
#5  0x0000790b530140cc in main_loop.lto_priv () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#6  0x0000790b541727c0 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#7  0x0000790b5416657e in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#8  0x0000790b542ab1a9 in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#9  0x0000790b5416808e in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#10 0x0000790b54209ad5 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#11 0x0000790b541663c2 in _PyObject_MakeTpCall () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#12 0x0000790b542ab1a9 in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#13 0x0000790b542aa159 in PyEval_EvalCode () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#14 0x0000790b542a3f29 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#15 0x0000790b541c2c18 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#16 0x0000790b5416657e in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#17 0x0000790b542ab1a9 in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#18 0x0000790b5435e8a1 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#19 0x0000790b5435fd12 in Py_RunMain () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#20 0x000058645f1640db in main ()
[Inferior 1 (process 161369) detached]
```

The same three threads via elfutils `eu-stack`, plus a **complete census of all 68 threads** grouped by resolved stack signature (so every thread is accounted for — 62 share the idle worker/rasterizer wait, 2 are in `pthread_barrier_wait`, and the remaining 4 are the main thread, the disk worker, `KittyPeerMon`, and `KittyChildMon`):

```text
### Named kitty threads (full eu-stack blocks) ###
TID 161369:   # main (kitty)
#0  0x0000790b4f65a380
#1  0x0000790b4f65a885
#2  0x0000790b4f65ad23
#3  0x0000790b4f65c53d
#4  0x0000790b4f664227
#5  0x0000790b4f5861dd
#6  0x0000790b4f581be3
#7  0x0000790b4f581d85
#8  0x0000790b4f50c84a
#9  0x0000790b4f504ff4
#10 0x0000790b4f5053c1
#11 0x0000790b4f50589d
#12 0x0000790b4f647975
#13 0x0000790b4f077dc6
#14 0x0000790b53017cb4 process_global_state
#15 0x0000790b520abcca glfwRunMainLoop
#16 0x0000790b530140cc main_loop.lto_priv.0
#17 0x0000790b541727c0
#18 0x0000790b5416657e PyObject_Vectorcall
#19 0x0000790b542ab1a9 _PyEval_EvalFrameDefault
#20 0x0000790b5416808e
#21 0x0000790b54209ad5
#22 0x0000790b541663c2 _PyObject_MakeTpCall
#23 0x0000790b542ab1a9 _PyEval_EvalFrameDefault
#24 0x0000790b542aa159 PyEval_EvalCode
#25 0x0000790b542a3f29
#26 0x0000790b541c2c18
#27 0x0000790b5416657e PyObject_Vectorcall
#28 0x0000790b542ab1a9 _PyEval_EvalFrameDefault
#29 0x0000790b5435e8a1
#30 0x0000790b5435fd12 Py_RunMain
#31 0x000058645f1640db main
#32 0x0000790b53e34575
#33 0x0000790b53e34628 __libc_start_main
#34 0x000058645f164505 _start

TID 161439:   # KittyPeerMon
#0  0x0000790b53eb6772
#1  0x0000790b53eaa13c
#2  0x0000790b53f31a8e __poll
#3  0x0000790b5301966b talk_loop
#4  0x0000790b53eadd64
#5  0x0000790b53f413fc

TID 161440:   # KittyChildMon
#0  0x0000790b53eb6772
#1  0x0000790b53eaa13c
#2  0x0000790b53f31a8e __poll
#3  0x0000790b530156ce io_loop
#4  0x0000790b53eadd64
#5  0x0000790b53f413fc

### All 68 threads — resolved (named) frames present per thread ###
  62 thread(s): #1,#3,#5,#7
   2 thread(s): pthread_barrier_wait,#2,#4
   1 thread(s): #1,#3,#5,#7,#9,#11,#13,process_global_state,glfwRunMainLoop,main_loop.lto_priv.0,#18,_PyEval_EvalFrameDefault,#21,_PyObject_MakeTpCall,_PyEval_EvalFrameDefault,PyEval_EvalCode,#26,PyObject_Vectorcall,_PyEval_EvalFrameDefault,#30,main,#33,_start
   1 thread(s): #1,#3,#5
   1 thread(s): #1,__poll,talk_loop,#5
   1 thread(s): #1,__poll,io_loop,#5
```

Reading the named stacks: **main** (`kitty`, LWP 161369) is in `csi_parse_loop` -> `run_worker` -> `do_parse` -> `process_global_state` -> `glfwRunMainLoop` -> `main_loop` -> Python bootstrap -> `main` — the exact CSI/SGR parse path the flood exercises. **`KittyChildMon`** is in `poll` -> `io_loop` (draining the PTY). **`KittyPeerMon`** is in `poll` -> `talk_loop` (idle on the RC socket, matching its 0.00 CPU in §6).

### 9.3 Complete raw dumps (zero elision) — `gdb`, `eu-stack`, `gstack`, `/proc`

For completeness, the entire multi-thread dumps are included verbatim. **`gdb -batch -ex "thread apply all bt"`** — all 68 threads (826 lines):

```text
$ gdb -p 161369 -batch -ex 'set pagination off' -ex 'thread apply all bt'
[New LWP 161440]
[New LWP 161439]
[New LWP 161438]
[New LWP 161437]
[New LWP 161436]
[New LWP 161435]
[New LWP 161434]
[New LWP 161433]
[New LWP 161432]
[New LWP 161431]
[New LWP 161430]
[New LWP 161429]
[New LWP 161428]
[New LWP 161427]
[New LWP 161426]
[New LWP 161425]
[New LWP 161424]
[New LWP 161423]
[New LWP 161422]
[New LWP 161421]
[New LWP 161420]
[New LWP 161419]
[New LWP 161418]
[New LWP 161417]
[New LWP 161416]
[New LWP 161415]
[New LWP 161414]
[New LWP 161413]
[New LWP 161412]
[New LWP 161411]
[New LWP 161410]
[New LWP 161409]
[New LWP 161408]
[New LWP 161407]
[New LWP 161406]
[New LWP 161405]
[New LWP 161404]
[New LWP 161403]
[New LWP 161402]
[New LWP 161401]
[New LWP 161400]
[New LWP 161399]
[New LWP 161398]
[New LWP 161397]
[New LWP 161396]
[New LWP 161395]
[New LWP 161394]
[New LWP 161393]
[New LWP 161392]
[New LWP 161391]
[New LWP 161390]
[New LWP 161389]
[New LWP 161388]
[New LWP 161387]
[New LWP 161386]
[New LWP 161385]
[New LWP 161384]
[New LWP 161383]
[New LWP 161382]
[New LWP 161381]
[New LWP 161380]
[New LWP 161379]
[New LWP 161378]
[New LWP 161377]
[New LWP 161376]
[New LWP 161375]
[New LWP 161374]
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
0x0000790b530b76d5 in csi_parse_loop () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so

Thread 68 (Thread 0x790b45e1b6c0 (LWP 161374) "llvmpipe-0"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 67 (Thread 0x790b4561a6c0 (LWP 161375) "llvmpipe-1"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 66 (Thread 0x790b44e196c0 (LWP 161376) "llvmpipe-2"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 65 (Thread 0x790b446186c0 (LWP 161377) "llvmpipe-3"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 64 (Thread 0x790b43e176c0 (LWP 161378) "llvmpipe-4"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 63 (Thread 0x790b436166c0 (LWP 161379) "llvmpipe-5"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 62 (Thread 0x790b42e156c0 (LWP 161380) "llvmpipe-6"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 61 (Thread 0x790b426146c0 (LWP 161381) "llvmpipe-7"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 60 (Thread 0x790b41e136c0 (LWP 161382) "llvmpipe-8"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 59 (Thread 0x790b416126c0 (LWP 161383) "llvmpipe-9"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 58 (Thread 0x790b40e116c0 (LWP 161384) "llvmpipe-10"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 57 (Thread 0x790b406106c0 (LWP 161385) "llvmpipe-11"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 56 (Thread 0x790b3fe0f6c0 (LWP 161386) "llvmpipe-12"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 55 (Thread 0x790b3f60e6c0 (LWP 161387) "llvmpipe-13"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 54 (Thread 0x790b3ee0d6c0 (LWP 161388) "llvmpipe-14"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 53 (Thread 0x790b3e60c6c0 (LWP 161389) "llvmpipe-15"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 52 (Thread 0x790b3de0b6c0 (LWP 161390) "llvmpipe-16"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 51 (Thread 0x790b3d60a6c0 (LWP 161391) "llvmpipe-17"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 50 (Thread 0x790b3ce096c0 (LWP 161392) "llvmpipe-18"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 49 (Thread 0x790b3c6086c0 (LWP 161393) "llvmpipe-19"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 48 (Thread 0x790b3be076c0 (LWP 161394) "llvmpipe-20"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 47 (Thread 0x790b3b6066c0 (LWP 161395) "llvmpipe-21"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 46 (Thread 0x790b3ae056c0 (LWP 161396) "llvmpipe-22"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 45 (Thread 0x790b3a6046c0 (LWP 161397) "llvmpipe-23"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 44 (Thread 0x790b39e036c0 (LWP 161398) "llvmpipe-24"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 43 (Thread 0x790b396026c0 (LWP 161399) "llvmpipe-25"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 42 (Thread 0x790b38e016c0 (LWP 161400) "llvmpipe-26"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 41 (Thread 0x790b386006c0 (LWP 161401) "llvmpipe-27"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 40 (Thread 0x790b37dff6c0 (LWP 161402) "llvmpipe-28"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 39 (Thread 0x790b375fe6c0 (LWP 161403) "llvmpipe-29"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 38 (Thread 0x790b36dfd6c0 (LWP 161404) "llvmpipe-30"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 37 (Thread 0x790b365fc6c0 (LWP 161405) "llvmpipe-31"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 36 (Thread 0x790b35dfb6c0 (LWP 161406) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 35 (Thread 0x790b355fa6c0 (LWP 161407) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 34 (Thread 0x790b34df96c0 (LWP 161408) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 33 (Thread 0x790b345f86c0 (LWP 161409) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 32 (Thread 0x790b33df76c0 (LWP 161410) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 31 (Thread 0x790b335f66c0 (LWP 161411) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 30 (Thread 0x790b32df56c0 (LWP 161412) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 29 (Thread 0x790b325f46c0 (LWP 161413) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 28 (Thread 0x790b31df36c0 (LWP 161414) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 27 (Thread 0x790b315f26c0 (LWP 161415) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 26 (Thread 0x790b30df16c0 (LWP 161416) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 25 (Thread 0x790b305f06c0 (LWP 161417) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 24 (Thread 0x790b2fdef6c0 (LWP 161418) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 23 (Thread 0x790b2f5ee6c0 (LWP 161419) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 22 (Thread 0x790b2eded6c0 (LWP 161420) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 21 (Thread 0x790b2e5ec6c0 (LWP 161421) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 20 (Thread 0x790b2ddeb6c0 (LWP 161422) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 19 (Thread 0x790b2d5ea6c0 (LWP 161423) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 18 (Thread 0x790b2cde96c0 (LWP 161424) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 17 (Thread 0x790b2c5e86c0 (LWP 161425) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 16 (Thread 0x790b2bde76c0 (LWP 161426) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 15 (Thread 0x790b2b5e66c0 (LWP 161427) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 14 (Thread 0x790b2ade56c0 (LWP 161428) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 13 (Thread 0x790b2a5e46c0 (LWP 161429) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 12 (Thread 0x790b29de36c0 (LWP 161430) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 11 (Thread 0x790b295e26c0 (LWP 161431) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 10 (Thread 0x790b28de16c0 (LWP 161432) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 9 (Thread 0x790b285e06c0 (LWP 161433) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 8 (Thread 0x790b27ddf6c0 (LWP 161434) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 7 (Thread 0x790b275de6c0 (LWP 161435) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 6 (Thread 0x790b26ddd6c0 (LWP 161436) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 5 (Thread 0x790b265dc6c0 (LWP 161437) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 4 (Thread 0x790b25c9a6c0 (LWP 161438) "kitty:disk$0"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f320fbc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 3 (Thread 0x790b24d526c0 (LWP 161439) "KittyPeerMon"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa13c in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53f31a8e in poll () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b5301966b in talk_loop () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#4  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#5  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 2 (Thread 0x790b245516c0 (LWP 161440) "KittyChildMon"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa13c in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53f31a8e in poll () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b530156ce in io_loop () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#4  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#5  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 1 (Thread 0x790b53cab780 (LWP 161369) "kitty"):
#0  0x0000790b530b76d5 in csi_parse_loop () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#1  0x0000790b530c3517 in run_worker.lto_priv () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#2  0x0000790b53013a55 in do_parse () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#3  0x0000790b53016a28 in process_global_state () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#4  0x0000790b520abcca in glfwRunMainLoop () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/glfw-x11.so
#5  0x0000790b530140cc in main_loop.lto_priv () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#6  0x0000790b541727c0 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#7  0x0000790b5416657e in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#8  0x0000790b542ab1a9 in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#9  0x0000790b5416808e in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#10 0x0000790b54209ad5 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#11 0x0000790b541663c2 in _PyObject_MakeTpCall () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#12 0x0000790b542ab1a9 in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#13 0x0000790b542aa159 in PyEval_EvalCode () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#14 0x0000790b542a3f29 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#15 0x0000790b541c2c18 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#16 0x0000790b5416657e in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#17 0x0000790b542ab1a9 in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#18 0x0000790b5435e8a1 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#19 0x0000790b5435fd12 in Py_RunMain () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#20 0x000058645f1640db in main ()
[Inferior 1 (process 161369) detached]
```

**`eu-stack -p PID`** — all 68 threads (686 lines):

```text
$ eu-stack -p 161369
PID 161369 - process
TID 161369:
#0  0x0000790b4f65a380
#1  0x0000790b4f65a885
#2  0x0000790b4f65ad23
#3  0x0000790b4f65c53d
#4  0x0000790b4f664227
#5  0x0000790b4f5861dd
#6  0x0000790b4f581be3
#7  0x0000790b4f581d85
#8  0x0000790b4f50c84a
#9  0x0000790b4f504ff4
#10 0x0000790b4f5053c1
#11 0x0000790b4f50589d
#12 0x0000790b4f647975
#13 0x0000790b4f077dc6
#14 0x0000790b53017cb4 process_global_state
#15 0x0000790b520abcca glfwRunMainLoop
#16 0x0000790b530140cc main_loop.lto_priv.0
#17 0x0000790b541727c0
#18 0x0000790b5416657e PyObject_Vectorcall
#19 0x0000790b542ab1a9 _PyEval_EvalFrameDefault
#20 0x0000790b5416808e
#21 0x0000790b54209ad5
#22 0x0000790b541663c2 _PyObject_MakeTpCall
#23 0x0000790b542ab1a9 _PyEval_EvalFrameDefault
#24 0x0000790b542aa159 PyEval_EvalCode
#25 0x0000790b542a3f29
#26 0x0000790b541c2c18
#27 0x0000790b5416657e PyObject_Vectorcall
#28 0x0000790b542ab1a9 _PyEval_EvalFrameDefault
#29 0x0000790b5435e8a1
#30 0x0000790b5435fd12 Py_RunMain
#31 0x000058645f1640db main
#32 0x0000790b53e34575
#33 0x0000790b53e34628 __libc_start_main
#34 0x000058645f164505 _start
TID 161374:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161375:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161376:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161377:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161378:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161379:
#0  0x0000790b253975d7
#1  0x0000790b4f64bc6b
#2  0x0000790b4f6527a3
#3  0x0000790b4f64b6fc
#4  0x0000790b4f64b963
#5  0x0000790b4f3677cc
#6  0x0000790b53eadd64
#7  0x0000790b53f413fc
TID 161380:
#0  0x0000790b253972ec
#1  0x0000790b4f64bc6b
#2  0x0000790b4f6527a3
#3  0x0000790b4f64b6fc
#4  0x0000790b4f64b963
#5  0x0000790b4f3677cc
#6  0x0000790b53eadd64
#7  0x0000790b53f413fc
TID 161381:
#0  0x0000790b2539879b
#1  0x0000790b4f64bc6b
#2  0x0000790b4f6527a3
#3  0x0000790b4f64b6fc
#4  0x0000790b4f64b963
#5  0x0000790b4f3677cc
#6  0x0000790b53eadd64
#7  0x0000790b53f413fc
TID 161382:
#0  0x0000790b253986a3
#1  0x0000790b4f64bc6b
#2  0x0000790b4f6527a3
#3  0x0000790b4f64b6fc
#4  0x0000790b4f64b963
#5  0x0000790b4f3677cc
#6  0x0000790b53eadd64
#7  0x0000790b53f413fc
TID 161383:
#0  0x0000790b25397acd
#1  0x0000790b4f64bc6b
#2  0x0000790b4f651fab
#3  0x0000790b4f64b6fc
#4  0x0000790b4f64b963
#5  0x0000790b4f3677cc
#6  0x0000790b53eadd64
#7  0x0000790b53f413fc
TID 161384:
#0  0x0000790b53eac28d pthread_barrier_wait
#1  0x0000790b4f32227d
#2  0x0000790b4f64b96b
#3  0x0000790b4f3677cc
#4  0x0000790b53eadd64
#5  0x0000790b53f413fc
TID 161385:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161386:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161387:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161388:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161389:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161390:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161391:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161392:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161393:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161394:
#0  0x0000790b4f6522c2
#1  0x0000790b4f64b6fc
#2  0x0000790b4f64b963
#3  0x0000790b4f3677cc
#4  0x0000790b53eadd64
#5  0x0000790b53f413fc
TID 161395:
#0  0x0000790b253986a3
#1  0x0000790b4f64bc6b
#2  0x0000790b4f6527a3
#3  0x0000790b4f64b6fc
#4  0x0000790b4f64b963
#5  0x0000790b4f3677cc
#6  0x0000790b53eadd64
#7  0x0000790b53f413fc
TID 161396:
#0  0x0000790b253986a3
#1  0x0000790b4f64bc6b
#2  0x0000790b4f6527a3
#3  0x0000790b4f64b6fc
#4  0x0000790b4f64b963
#5  0x0000790b4f3677cc
#6  0x0000790b53eadd64
#7  0x0000790b53f413fc
TID 161397:
#0  0x0000790b53eac28d pthread_barrier_wait
#1  0x0000790b4f32227d
#2  0x0000790b4f64b96b
#3  0x0000790b4f3677cc
#4  0x0000790b53eadd64
#5  0x0000790b53f413fc
TID 161398:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161399:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161400:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161401:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161402:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161403:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161404:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161405:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64b91f
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161406:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161407:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161408:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161409:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161410:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161411:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161412:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161413:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161414:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161415:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161416:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161417:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161418:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161419:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161420:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161421:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161422:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161423:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161424:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161425:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161426:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161427:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161428:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161429:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161430:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161431:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161432:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161433:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161434:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161435:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161436:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161437:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f64728c
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161438:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa0ac
#2  0x0000790b53eaa807
#3  0x0000790b53ead067 pthread_cond_wait
#4  0x0000790b4f36789d
#5  0x0000790b4f320fbc
#6  0x0000790b4f3677cc
#7  0x0000790b53eadd64
#8  0x0000790b53f413fc
TID 161439:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa13c
#2  0x0000790b53f31a8e __poll
#3  0x0000790b5301966b talk_loop
#4  0x0000790b53eadd64
#5  0x0000790b53f413fc
TID 161440:
#0  0x0000790b53eb6772
#1  0x0000790b53eaa13c
#2  0x0000790b53f31a8e __poll
#3  0x0000790b530156ce io_loop
#4  0x0000790b53eadd64
#5  0x0000790b53f413fc
```

**`gstack PID`** (a thin wrapper over `gdb thread apply all bt`; included complete for the fallback-chain record, 754 lines):

```text
$ gstack 161369
Thread 68 (Thread 0x790b45e1b6c0 (LWP 161374) "llvmpipe-0"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 67 (Thread 0x790b4561a6c0 (LWP 161375) "llvmpipe-1"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 66 (Thread 0x790b44e196c0 (LWP 161376) "llvmpipe-2"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 65 (Thread 0x790b446186c0 (LWP 161377) "llvmpipe-3"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 64 (Thread 0x790b43e176c0 (LWP 161378) "llvmpipe-4"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 63 (Thread 0x790b436166c0 (LWP 161379) "llvmpipe-5"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 62 (Thread 0x790b42e156c0 (LWP 161380) "llvmpipe-6"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 61 (Thread 0x790b426146c0 (LWP 161381) "llvmpipe-7"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 60 (Thread 0x790b41e136c0 (LWP 161382) "llvmpipe-8"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 59 (Thread 0x790b416126c0 (LWP 161383) "llvmpipe-9"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 58 (Thread 0x790b40e116c0 (LWP 161384) "llvmpipe-10"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 57 (Thread 0x790b406106c0 (LWP 161385) "llvmpipe-11"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 56 (Thread 0x790b3fe0f6c0 (LWP 161386) "llvmpipe-12"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 55 (Thread 0x790b3f60e6c0 (LWP 161387) "llvmpipe-13"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 54 (Thread 0x790b3ee0d6c0 (LWP 161388) "llvmpipe-14"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 53 (Thread 0x790b3e60c6c0 (LWP 161389) "llvmpipe-15"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 52 (Thread 0x790b3de0b6c0 (LWP 161390) "llvmpipe-16"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 51 (Thread 0x790b3d60a6c0 (LWP 161391) "llvmpipe-17"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 50 (Thread 0x790b3ce096c0 (LWP 161392) "llvmpipe-18"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 49 (Thread 0x790b3c6086c0 (LWP 161393) "llvmpipe-19"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 48 (Thread 0x790b3be076c0 (LWP 161394) "llvmpipe-20"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 47 (Thread 0x790b3b6066c0 (LWP 161395) "llvmpipe-21"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 46 (Thread 0x790b3ae056c0 (LWP 161396) "llvmpipe-22"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 45 (Thread 0x790b3a6046c0 (LWP 161397) "llvmpipe-23"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 44 (Thread 0x790b39e036c0 (LWP 161398) "llvmpipe-24"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 43 (Thread 0x790b396026c0 (LWP 161399) "llvmpipe-25"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 42 (Thread 0x790b38e016c0 (LWP 161400) "llvmpipe-26"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 41 (Thread 0x790b386006c0 (LWP 161401) "llvmpipe-27"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 40 (Thread 0x790b37dff6c0 (LWP 161402) "llvmpipe-28"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 39 (Thread 0x790b375fe6c0 (LWP 161403) "llvmpipe-29"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 38 (Thread 0x790b36dfd6c0 (LWP 161404) "llvmpipe-30"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 37 (Thread 0x790b365fc6c0 (LWP 161405) "llvmpipe-31"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64b91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 36 (Thread 0x790b35dfb6c0 (LWP 161406) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 35 (Thread 0x790b355fa6c0 (LWP 161407) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 34 (Thread 0x790b34df96c0 (LWP 161408) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 33 (Thread 0x790b345f86c0 (LWP 161409) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 32 (Thread 0x790b33df76c0 (LWP 161410) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 31 (Thread 0x790b335f66c0 (LWP 161411) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 30 (Thread 0x790b32df56c0 (LWP 161412) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 29 (Thread 0x790b325f46c0 (LWP 161413) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 28 (Thread 0x790b31df36c0 (LWP 161414) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 27 (Thread 0x790b315f26c0 (LWP 161415) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 26 (Thread 0x790b30df16c0 (LWP 161416) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 25 (Thread 0x790b305f06c0 (LWP 161417) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 24 (Thread 0x790b2fdef6c0 (LWP 161418) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 23 (Thread 0x790b2f5ee6c0 (LWP 161419) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 22 (Thread 0x790b2eded6c0 (LWP 161420) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 21 (Thread 0x790b2e5ec6c0 (LWP 161421) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 20 (Thread 0x790b2ddeb6c0 (LWP 161422) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 19 (Thread 0x790b2d5ea6c0 (LWP 161423) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 18 (Thread 0x790b2cde96c0 (LWP 161424) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 17 (Thread 0x790b2c5e86c0 (LWP 161425) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 16 (Thread 0x790b2bde76c0 (LWP 161426) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 15 (Thread 0x790b2b5e66c0 (LWP 161427) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 14 (Thread 0x790b2ade56c0 (LWP 161428) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 13 (Thread 0x790b2a5e46c0 (LWP 161429) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 12 (Thread 0x790b29de36c0 (LWP 161430) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 11 (Thread 0x790b295e26c0 (LWP 161431) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 10 (Thread 0x790b28de16c0 (LWP 161432) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 9 (Thread 0x790b285e06c0 (LWP 161433) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 8 (Thread 0x790b27ddf6c0 (LWP 161434) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 7 (Thread 0x790b275de6c0 (LWP 161435) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 6 (Thread 0x790b26ddd6c0 (LWP 161436) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 5 (Thread 0x790b265dc6c0 (LWP 161437) "kitty"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f64728c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 4 (Thread 0x790b25c9a6c0 (LWP 161438) "kitty:disk$0"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa0ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53eaa807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b53ead067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x0000790b4f36789d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x0000790b4f320fbc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x0000790b4f3677cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 3 (Thread 0x790b24d526c0 (LWP 161439) "KittyPeerMon"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa13c in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53f31a8e in poll () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b5301966b in talk_loop () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#4  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#5  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 2 (Thread 0x790b245516c0 (LWP 161440) "KittyChildMon"):
#0  0x0000790b53eb6772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b53eaa13c in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x0000790b53f31a8e in poll () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x0000790b530156ce in io_loop () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#4  0x0000790b53eadd64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#5  0x0000790b53f413fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 1 (Thread 0x790b53cab780 (LWP 161369) "kitty"):
#0  0x0000790b53eb3003 in pthread_mutex_unlock () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x0000790b530c317d in run_worker.lto_priv () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#2  0x0000790b53013a55 in do_parse () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#3  0x0000790b53016a28 in process_global_state () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#4  0x0000790b520abcca in glfwRunMainLoop () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/glfw-x11.so
#5  0x0000790b530140cc in main_loop.lto_priv () from /tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/launcher/../../kitty/fast_data_types.so
#6  0x0000790b541727c0 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#7  0x0000790b5416657e in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#8  0x0000790b542ab1a9 in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#9  0x0000790b5416808e in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#10 0x0000790b54209ad5 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#11 0x0000790b541663c2 in _PyObject_MakeTpCall () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#12 0x0000790b542ab1a9 in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#13 0x0000790b542aa159 in PyEval_EvalCode () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#14 0x0000790b542a3f29 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#15 0x0000790b541c2c18 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#16 0x0000790b5416657e in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#17 0x0000790b542ab1a9 in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#18 0x0000790b5435e8a1 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#19 0x0000790b5435fd12 in Py_RunMain () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#20 0x000058645f1640db in main ()
```

**`/proc/PID/task/<tid>/stack`** — kernel-side stacks of the three named threads: main parked in `futex_wait` (waiting on a render worker), `KittyPeerMon` and `KittyChildMon` in `do_sys_poll`:

```text
$ for tid in KittyChildMon/KittyPeerMon/main: cat /proc/161369/task/<tid>/stack
--- tid=161369 comm=kitty ---
[<0>] futex_wait_queue+0xde/0x130
[<0>] futex_wait+0x179/0x300
[<0>] do_futex+0x18f/0x1e0
[<0>] __se_sys_futex+0x152/0x1d0
[<0>] do_syscall_64+0x46/0xb0
[<0>] entry_SYSCALL_64_after_hwframe+0x78/0xe2
--- tid=161439 comm=KittyPeerMon ---
[<0>] do_sys_poll+0x572/0x680
[<0>] do_restart_poll+0x5c/0xa0
[<0>] do_syscall_64+0x46/0xb0
[<0>] entry_SYSCALL_64_after_hwframe+0x78/0xe2
--- tid=161440 comm=KittyChildMon ---
[<0>] do_sys_poll+0x572/0x680
[<0>] __se_sys_poll+0xa9/0x140
[<0>] do_syscall_64+0x46/0xb0
[<0>] entry_SYSCALL_64_after_hwframe+0x78/0xe2
```

### 9.4 Static symbol snapshot — the C hot-path functions live in `fast_data_types.so`

The extension is **not stripped**, so the symbol table is authoritative. Summary and NEEDED libraries:

```text
########## fast_data_types.so ##########
===== file =====
/tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/fast_data_types.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=1082ca8e137533b0783c23a3f853a1cf7d561ee1, not stripped

===== readelf -h (ELF header) =====
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          64 (bytes into file)
  Start of section headers:          1251936 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         11
  Size of section headers:           64 (bytes)
  Number of section headers:         29
  Section header string table index: 28

===== readelf -d (dynamic NEEDED libraries) =====
 0x0000000000000001 (NEEDED)             Shared library: [libm.so.6]
 0x0000000000000001 (NEEDED)             Shared library: [libpython3.13.so.1.0]
 0x0000000000000001 (NEEDED)             Shared library: [libharfbuzz.so.0]
 0x0000000000000001 (NEEDED)             Shared library: [libpng16.so.16]
 0x0000000000000001 (NEEDED)             Shared library: [liblcms2.so.2]
 0x0000000000000001 (NEEDED)             Shared library: [libcrypto.so.3]
 0x0000000000000001 (NEEDED)             Shared library: [libz.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]

===== objdump -f (file header summary) =====

/tmp/blitzy/kitty/blitzy-48cfad2a-d0c5-461c-8ec1-b78f249ee675_dfdacc/kitty/fast_data_types.so:     file format elf64-x86-64
architecture: i386:x86-64, flags 0x00000150:
HAS_SYMS, DYNAMIC, D_PAGED
start address 0x0000000000000000


===== symbol table summary =====
defined text (T/t): 1032
dynamic symbols (nm -D): 391
total ELF symtab entries (readelf -s .symtab): 1714
```

There are **1032 defined text symbols**. The complete per-subsystem symbol lists (each is the full `nm` grep for that subsystem, not a sample) prove where each responsibility lives:

**VT/escape parser (`kitty/vt-parser.c`)** — note `csi_parse_loop`, `do_parse`, `_parse_sgr`, and the SIMD `utf8_decode_to_esc_{128,256,scalar}` seen live in §9.1-9.2:

```text
00000000000c1d00 t _parse_sgr.isra.0
00000000000b7c10 t _parse_sgr.lto_priv.0
00000000000bea60 t alloc_vt_parser
00000000000b7530 t csi_parse_loop
0000000000013a10 t do_parse
00000000000bb8d0 t free_vt_parser
00000000000be960 t parse_sgr.constprop.0
000000000009c960 t utf8_decode_to_esc
00000000000a01c0 t utf8_decode_to_esc_128
00000000000a1790 t utf8_decode_to_esc_256
000000000009ffc0 t utf8_decode_to_esc_scalar
```

**SIMD string kernels (`kitty/simd-string*.c`)** — 128-bit, 256-bit (AVX2), and scalar variants of the byte-scan and UTF-8 decode:

```text
000000000008d7a0 t find_cmd_output.lto_priv.0
00000000000720c0 t find_colon_slash.isra.0
000000000009c950 t find_either_of_two_bytes
000000000009ff10 t find_either_of_two_bytes_128
00000000000a16f0 t find_either_of_two_bytes_256
000000000009c7f0 t find_either_of_two_bytes_scalar.lto_priv.0
00000000000478e0 t find_fallback_font_for
0000000000021730 t find_in_memoryview
00000000000350a0 t find_matching_namerec.lto_priv.0
000000000005dc00 t find_or_create_glyph_properties
000000000006aac0 t find_or_create_image.part.0.lto_priv.0
000000000005d1e0 t find_or_create_sprite_position
00000000000bef40 t find_or_create_window_logo
00000000000a2d60 t test_find_either_of_two_bytes.lto_priv.0
000000000009c960 t utf8_decode_to_esc
00000000000a01c0 t utf8_decode_to_esc_128
00000000000a1790 t utf8_decode_to_esc_256
000000000009ffc0 t utf8_decode_to_esc_scalar
000000000009f870 t xor_data64_128
00000000000a1070 t xor_data64_256
```

**Shaders / GPU draw (`kitty/shaders.c`)** — `draw_cells`, `draw_cells_simple`, and `draw_graphics.constprop.*` (the image draw referenced in §8.3, `shaders.c:509`):

```text
000000000009dd00 t attach_shaders
00000000000a3c60 t cell_prepare_to_render.lto_priv.0
00000000000a4660 t draw_cells
000000000009c140 t draw_cells_simple
000000000009bf60 t draw_graphics.constprop.0
000000000009bd80 t draw_graphics.constprop.1
00000000000a4410 t draw_graphics.constprop.2
0000000000051d90 t ensure_csd_title_render_ctx.lto_priv.0
0000000000092480 t pause_rendering.lto_priv.0
000000000009e450 t pyinit_cell_program.lto_priv.0
00000000000abc20 t pyset_tab_bar_render_data.lto_priv.0
00000000000acb20 t pyset_window_render_data.lto_priv.0
000000000009d4f0 t render_a_bar
0000000000043d30 t render_bitmap.isra.0
00000000000444d0 t render_glyphs_in_cells.isra.0
000000000003d180 t render_gray_bitmap.isra.0
0000000000036130 t render_groups
00000000000392e0 t render_line
0000000000049520 t render_line.lto_priv.0
0000000000086c20 t render_overlay_line
00000000000368f0 t render_run
0000000000047eb0 t render_run
0000000000044de0 t render_sample_text.lto_priv.0
0000000000048ea0 t render_single_line
000000000008e1c0 t render_unfocused_cursor_get.lto_priv.0
0000000000091d20 t render_unfocused_cursor_set.lto_priv.0
000000000008a2e0 t screen_pause_rendering
0000000000088880 t screen_pause_rendering.constprop.0
00000000000a3260 t screen_render_line_graphics.part.0.lto_priv.0
0000000000033360 t send_prerendered_sprites
0000000000037460 t send_prerendered_sprites_for_window.isra.0
000000000003ad80 t test_render_line.lto_priv.0
```

**GL binding (`kitty/gl.c`)** — the OpenGL wrapper layer:

```text
0000000000043040 t _post_call_gl_callback_default
0000000000042fd0 t _pre_call_gl_callback_default
000000000004a720 t check_for_gl_error
000000000004b240 t framebuffer_size_callback
000000000003d330 t glad_debug_impl_glActiveTexture.lto_priv.0
000000000003d380 t glad_debug_impl_glAttachShader.lto_priv.0
000000000003d3e0 t glad_debug_impl_glBindBuffer.lto_priv.0
000000000003d440 t glad_debug_impl_glBindBufferBase.lto_priv.0
000000000003d4b0 t glad_debug_impl_glBindTexture.lto_priv.0
000000000003d510 t glad_debug_impl_glBindVertexArray.lto_priv.0
000000000003d560 t glad_debug_impl_glBlendFunc.lto_priv.0
000000000003d5c0 t glad_debug_impl_glBlendFuncSeparate.lto_priv.0
000000000003d640 t glad_debug_impl_glBufferData.lto_priv.0
000000000003d6c0 t glad_debug_impl_glClear.lto_priv.0
000000000003d710 t glad_debug_impl_glClearColor.lto_priv.0
000000000003d7e0 t glad_debug_impl_glCompileShader.lto_priv.0
000000000003d830 t glad_debug_impl_glCopyImageSubData.lto_priv.0
000000000003d9b0 t glad_debug_impl_glCreateProgram.lto_priv.0
000000000003da30 t glad_debug_impl_glCreateShader.lto_priv.0
000000000003dac0 t glad_debug_impl_glDeleteBuffers.lto_priv.0
000000000003db30 t glad_debug_impl_glDeleteProgram.lto_priv.0
000000000003db80 t glad_debug_impl_glDeleteShader.lto_priv.0
000000000003dbd0 t glad_debug_impl_glDeleteTextures.lto_priv.0
000000000003dc40 t glad_debug_impl_glDeleteVertexArrays.lto_priv.0
000000000003dcb0 t glad_debug_impl_glDisable.lto_priv.0
000000000003dd00 t glad_debug_impl_glDrawArrays.lto_priv.0
000000000003dd70 t glad_debug_impl_glDrawArraysInstanced.lto_priv.0
000000000003ddf0 t glad_debug_impl_glEnable.lto_priv.0
000000000003de40 t glad_debug_impl_glEnableVertexAttribArray.lto_priv.0
000000000003de90 t glad_debug_impl_glGenBuffers.lto_priv.0
000000000003df00 t glad_debug_impl_glGenTextures.lto_priv.0
000000000003df70 t glad_debug_impl_glGenVertexArrays.lto_priv.0
000000000003dfe0 t glad_debug_impl_glGetActiveUniform.lto_priv.0
000000000003e090 t glad_debug_impl_glGetActiveUniformBlockiv.lto_priv.0
000000000003e110 t glad_debug_impl_glGetActiveUniformsiv.lto_priv.0
000000000003e1a0 t glad_debug_impl_glGetAttribLocation.lto_priv.0
000000000003e240 t glad_debug_impl_glGetFramebufferAttachmentParameteriv.lto_priv.0
000000000003e2c0 t glad_debug_impl_glGetIntegerv.lto_priv.0
000000000003e330 t glad_debug_impl_glGetProgramInfoLog.lto_priv.0
000000000003e3b0 t glad_debug_impl_glGetProgramiv.lto_priv.0
000000000003e420 t glad_debug_impl_glGetShaderInfoLog.lto_priv.0
000000000003e4a0 t glad_debug_impl_glGetShaderiv.lto_priv.0
000000000003e510 t glad_debug_impl_glGetString.lto_priv.0
000000000003e5a0 t glad_debug_impl_glGetTexImage.lto_priv.0
000000000003e630 t glad_debug_impl_glGetUniformBlockIndex.lto_priv.0
000000000003e6d0 t glad_debug_impl_glGetUniformIndices.lto_priv.0
000000000003e750 t glad_debug_impl_glGetUniformLocation.lto_priv.0
000000000003e7f0 t glad_debug_impl_glLinkProgram.lto_priv.0
000000000003e840 t glad_debug_impl_glMapBuffer.lto_priv.0
000000000003e8d0 t glad_debug_impl_glPixelStorei.lto_priv.0
000000000003e930 t glad_debug_impl_glShaderSource.lto_priv.0
000000000003e9b0 t glad_debug_impl_glTexImage2D.lto_priv.0
000000000003ea80 t glad_debug_impl_glTexParameterfv.lto_priv.0
000000000003eaf0 t glad_debug_impl_glTexParameteri.lto_priv.0
000000000003eb60 t glad_debug_impl_glTexStorage3D.lto_priv.0
000000000003ec00 t glad_debug_impl_glTexSubImage3D.lto_priv.0
000000000003ecf0 t glad_debug_impl_glUniform1f.lto_priv.0
000000000003ed70 t glad_debug_impl_glUniform1fv.lto_priv.0
000000000003ede0 t glad_debug_impl_glUniform1i.lto_priv.0
000000000003ee40 t glad_debug_impl_glUniform1ui.lto_priv.0
000000000003eea0 t glad_debug_impl_glUniform1uiv.lto_priv.0
000000000003ef10 t glad_debug_impl_glUniform2ui.lto_priv.0
000000000003ef80 t glad_debug_impl_glUniform3f.lto_priv.0
000000000003f040 t glad_debug_impl_glUniform4f.lto_priv.0
000000000003f120 t glad_debug_impl_glUnmapBuffer.lto_priv.0
000000000003f1b0 t glad_debug_impl_glUseProgram.lto_priv.0
000000000003f200 t glad_debug_impl_glVertexAttribDivisorARB.lto_priv.0
000000000003f260 t glad_debug_impl_glVertexAttribIPointer.lto_priv.0
000000000003f2f0 t glad_debug_impl_glVertexAttribPointer.lto_priv.0
000000000003f3a0 t glad_debug_impl_glViewport.lto_priv.0
000000000003f420 t glad_gl_get_proc_from_userptr.lto_priv.0
0000000000043080 t glad_gl_has_extension
000000000003f430 t glad_gl_load_GL_VERSION_1_3.constprop.0
000000000003f730 t glad_gl_load_GL_VERSION_1_4.constprop.0
000000000003fa40 t glad_gl_load_GL_VERSION_2_0.constprop.0
0000000000040030 t glad_gl_load_GL_VERSION_3_0.constprop.0
```

**GLFW glue (`kitty/glfw.c`):**

```text
0000000000059400 t cleanup_glfw
000000000006f360 t convert_glfw_key_event_to_python.lto_priv.0
000000000006e090 t encode_glfw_key_event
0000000000050c90 t glfw_get_key_name.lto_priv.0
0000000000050b40 t glfw_get_physical_dpi.lto_priv.0
000000000004d920 t glfw_init.lto_priv.0
000000000004a800 t glfw_terminate.lto_priv.0
000000000004d390 t glfw_window_hint.lto_priv.0
000000000005b860 t init_glfw
0000000000058400 t pointer_name_to_glfw_name
```

**Graphics protocol (`kitty/graphics.c`)** — image transmission/placement (the `icat` receiver):

```text
000000000009bf60 t draw_graphics.constprop.0
000000000009bd80 t draw_graphics.constprop.1
00000000000a4410 t draw_graphics.constprop.2
0000000000059810 t free_image_resources
0000000000066f20 t get_image_count.lto_priv.0
000000000005e1f0 t grman_alloc
0000000000062900 t grman_handle_command
000000000005e840 t grman_put_cell_image.constprop.0.isra.0
00000000000627e0 t grman_scroll_images
0000000000060890 t grman_update_layers
0000000000066040 t image_as_dict
000000000007cfa0 t inflate_png_inner
000000000005a2d0 t load_image_data
000000000007d6b0 t load_png_data.lto_priv.0
0000000000065dd0 t new_graphicsmanager_object.lto_priv.0
00000000000c23d0 t parse_graphics_code
00000000000cf220 t parse_graphics_code
0000000000059f00 t png_error_handler
0000000000074ce0 t png_error_handler
000000000005e440 t png_from_data
000000000005e570 t png_path_to_bitmap
0000000000059d50 t print_png_read_error
000000000005a8e0 t process_image_data.isra.0
000000000006af40 t pyimage_for_client_id.lto_priv.0
0000000000066330 t pyimage_for_client_number.lto_priv.0
00000000000798e0 t read_png_error_handler
0000000000074c40 t read_png_from_buffer
0000000000079950 t read_png_warn_handler
00000000000831d0 t screen_dirty_line_graphics.lto_priv.0
0000000000084630 t screen_handle_graphics_command
00000000000a3260 t screen_render_line_graphics.part.0.lto_priv.0
00000000000a3a10 t send_image_to_gpu
00000000000a36c0 t update_only_line_graphics_data.lto_priv.0
```

**Screen model (`kitty/screen.c`):**

```text
000000000007df70 t new_screen_object.lto_priv.0
0000000000082470 t screen_align
000000000008ca90 t screen_apply_selection
00000000000857b0 t screen_bell
00000000000853a0 t screen_delete_characters
000000000008d050 t screen_detect_url
000000000001118b t screen_detect_url.cold
00000000000831d0 t screen_dirty_line_graphics.lto_priv.0
000000000007f780 t screen_dirty_sprite_positions
0000000000085620 t screen_erase_characters
00000000000897a0 t screen_erase_in_display
0000000000084ee0 t screen_erase_in_line
000000000006cbe0 t screen_garbage_collect_hyperlink_pool
0000000000084630 t screen_handle_graphics_command
000000000008c780 t screen_has_selection
0000000000083ae0 t screen_index
00000000000851b0 t screen_insert_characters
0000000000099670 t screen_is_emoji_presentation_base.lto_priv.0
0000000000087360 t screen_manipulate_title_stack.isra.0
0000000000098760 t screen_mark_all
000000000009a8b0 t screen_mark_hyperlink.isra.0
0000000000011194 t screen_mark_hyperlink.isra.0.cold
000000000008a2e0 t screen_pause_rendering
0000000000088880 t screen_pause_rendering.constprop.0
0000000000088190 t screen_pop_colors
0000000000087f80 t screen_push_colors
0000000000083690 t screen_push_key_encoding_flags
00000000000a3260 t screen_render_line_graphics.part.0.lto_priv.0
00000000000868f0 t screen_repeat_character
00000000000883a0 t screen_report_color_stack
0000000000083360 t screen_report_key_encoding_flags
0000000000087290 t screen_report_size
0000000000088570 t screen_request_capabilities
00000000000821b0 t screen_rescale_images
000000000007eb80 t screen_reset
0000000000084d60 t screen_restore_cursor
0000000000084810 t screen_reverse_index
0000000000084300 t screen_scroll
000000000009a600 t screen_selection_range_for_word.constprop.0
00000000000834c0 t screen_set_key_encoding_flags
0000000000087ad0 t screen_set_margins
000000000009a4b0 t screen_start_selection
0000000000083810 t screen_tab
00000000000889e0 t screen_toggle_screen_buffer
0000000000081cf0 t screen_truncate_point_for_length.lto_priv.0
0000000000088ca0 t screen_update_cell_data
000000000008d5c0 t screen_update_overlay_text
000000000009ac80 t screen_update_selection
000000000005b650 t toggle_fullscreen_for_os_window.part.0.lto_priv.0
```

**Line/line-buffer (`kitty/line.c`, `line-buf.c`):**

```text
0000000000097980 t continue_line_downwards
00000000000978b0 t continue_line_upwards
00000000000667e0 t copy_line_to.lto_priv.0
000000000006b340 t create_historybuf.lto_priv.0
000000000006a740 t create_line_copy.lto_priv.0
0000000000081620 t get_line_from_offset
000000000006bd30 t get_line_wrapper.lto_priv.0
000000000006bfc0 t historybuf_add_line
000000000006b490 t historybuf_clear
000000000006bd70 t historybuf_push.part.0
000000000006c170 t historybuf_rewrap
0000000000091a30 t is_main_linebuf.lto_priv.0
000000000007fbc0 t is_using_alternate_linebuf.lto_priv.0
0000000000076fd0 t line_add_combining_char
0000000000077250 t line_apply_cursor.constprop.0
00000000000784d0 t line_as_ansi
0000000000077080 t line_clear_text
00000000000929f0 t line_edge_colors.lto_priv.0
00000000000774a0 t line_right_shift
00000000000767b0 t line_startswith_url_chars
0000000000074f80 t line_url_end_at
0000000000074d30 t line_url_start_at
000000000006fd30 t linebuf_copy_line_to
000000000006f9d0 t linebuf_delete_lines
000000000006f8b0 t linebuf_index
0000000000070d70 t linebuf_insert_lines.part.0
000000000006f930 t linebuf_reverse_index
000000000006fde0 t linebuf_rewrap
0000000000072ca0 t new_line_object.lto_priv.0
000000000006a110 t new_linebuf_object.lto_priv.0
000000000008baf0 t range_line_.lto_priv.0
00000000000831d0 t screen_dirty_line_graphics.lto_priv.0
00000000000a3260 t screen_render_line_graphics.part.0.lto_priv.0
00000000000a36c0 t update_only_line_graphics_data.lto_priv.0
00000000000895a0 t visual_line_.lto_priv.0
```

**FreeType rasterization (`kitty/freetype.c`)** — `face_from_descriptor`, `render_glyphs_in_cells`:

```text
0000000000037990 t face_from_descriptor
000000000003cab0 t face_from_path
000000000005dc00 t find_or_create_glyph_properties
0000000000040e90 t free_freetype.lto_priv.0
000000000005e090 t free_glyph_properties_hash_table
0000000000031a10 t is_empty_glyph
00000000000444d0 t render_glyphs_in_cells.isra.0
```

**Font shaping / FontConfig (`kitty/fontconfig.c`, shaping):**

```text
00000000000806e0 t change_pointer_shape.lto_priv.0
0000000000080470 t current_pointer_shape.lto_priv.0
0000000000037990 t face_from_descriptor
000000000003cab0 t face_from_path
000000000002f730 t fc_list.lto_priv.0
000000000002fcd0 t fc_match.lto_priv.0
00000000000300f0 t fc_match_postscript_name.lto_priv.0
000000000003b510 t font_group_for.lto_priv.0
00000000000452d0 t free_face.lto_priv.0
000000000002e0f0 t load_fontconfig_lib
00000000000a79c0 t pointer_shape
00000000000b1660 t pyupdate_pointer_shape.lto_priv.0
0000000000031680 t shape
0000000000031c20 t shape_run
000000000003ccf0 t test_shape.lto_priv.0
```

### 9.5 Go symbols

The Go `kitten` binary is stripped of a native symbol table (`nm: no symbols`, `go tool nm` = 0), consistent with the `-s -w` ldflags in §8.4; its Go-level structure is only recoverable via `.gopclntab`/`go version -m` (shown in §8.4). This is the expected contrast: the C extension keeps full symbols (so the profilers above resolve names), while the shipped Go CLI is stripped.

## 10. Sub-question 5 — Inference, rule-outs, and one tradeoff (from artifacts only)

### 10.1 Responsibilities attributed strictly from the runtime artifacts

- **C owns the per-byte and per-frame hot path.** Every main-thread native sample under load is in C: `csi_parse_loop`/`do_parse`/`utf8_decode_to_esc_256` in `fast_data_types.so` (§9.1-9.2), reached via `process_global_state` -> `glfwRunMainLoop` -> `main_loop`. The extension is not stripped and contains the parser, SIMD kernels (`find_either_of_two_bytes_256`, AVX2), screen model, glyph rasterization, and GPU draw functions (`draw_cells`, `draw_graphics`) — 1032 defined text symbols (§9.4). CPU-wise the main thread + `KittyChildMon` (both C loops) carry kitty's own ~5 CPU-seconds of the flood (§6.3).
- **Python owns orchestration, configuration, and the entry point.** `libpython3.13.so.1.0` is mapped into the main process (§5.1), and the outer stack frames on every sample are Python (`_run_app` — defined at `kitty/main.py:202`, sampled at `:234` where it calls `boss.child_monitor.main_loop()` — reached from `Py_RunMain` -> `main`), but they are the **ancestors** that entered the C `main_loop` once at startup — Python bytecode (`_PyEval_EvalFrameDefault`) never appears on the per-byte path during load.
- **Go owns standalone CLI tooling, out of process.** The `kitten` binary is a separate Go executable (§8.4) that never loads into the main process (absent from the §5 map); it communicates with kitty only by emitting protocol escapes over the PTY (§8.3).

### 10.2 Rule-outs — plausible-but-wrong interpretations, refuted by evidence

1. **"Python does the parsing/rendering under load."** *Refuted.* Under the flood, the main thread's native stack is entirely C (`csi_parse_loop`/`do_parse`/`utf8_decode_to_esc_256` in `fast_data_types.so`, §9.1-9.2); the Python frames present are only the startup bootstrap sitting *below* the C work, and `_PyEval_EvalFrameDefault` is never the leaf. If Python were doing the work, the sampler would show CPython eval frames at the top of the main-thread stack during load; it never does.
2. **"`kitten` is loaded as a module/plugin into the kitty process (like a Python kitten)."** *Refuted.* `kitty +kitten icat` and `kitten icat` each spawn a **distinct process** whose executable is the Go `kitten` binary, with **no `libpython` mapped** into it (§8.1), and no Go object appears in the main process's own module map (§5.2). It is out-of-process, communicating only via PTY escapes.
3. **"The large thread count under load is kitty parallelizing its parse/render."** *Refuted.* The ~64 extra threads are the Mesa **software-GL** rasterizer pool (`llvmpipe-*` + unnamed workers); they exist at idle too (§6.2) and dominate CPU under load (~18 s, §6.3) only because this host has no hardware GPU (`Accelerated: no`, §2.1). kitty's own concurrency is just the main thread plus `run_worker` under `do_parse` — not a large pool.

### 10.3 One portability-vs-performance tradeoff, grounded in observation

The language division *is* a portability-vs-performance tradeoff, and the runtime artifacts show both sides:

- **The C hot path buys performance at the cost of portability.** `fast_data_types.so` links the GL client stack (§5.1) and carries `-march=native` AVX2 SIMD kernels (`utf8_decode_to_esc_256`, `find_either_of_two_bytes_256`, §9.4). It is fast, but it *requires a GL surface* — kitty has no CPU rendering fallback, which is precisely why this headless run needed `Xvfb` + Mesa `llvmpipe`, and why that software rasterizer then cost ~18 CPU-seconds under load (§2.1, §6.3). High performance on real GPU hardware; not portable to a no-GL environment without a software shim.
- **The Go `kitten` buys portability at the cost of not being the hot path.** It is a single self-contained binary whose only external dependency is `libc` (and even that vanishes with `CGO_ENABLED=0`, §8.4). It runs anywhere — including shipped to a remote host over SSH — with no Python interpreter and no GL context. That deployment portability is exactly why user-facing CLI utilities live in Go and out of process, while the latency-critical per-byte/per-frame work stays in in-process C.

The observed split is therefore: **performance-critical, GPU/SIMD-bound work -> in-process C (fast, GL-dependent, CPU-arch-tuned); portable, self-contained CLI utilities -> out-of-process static Go.**

## 11. Coverage pass — every named item, with value, `file:line`, and evidence location

| Named item asked for | Concrete observed value | `file:line` | Evidence section |
|---|---|---|---|
| Version banner (default build) | `kitty 0.35.2` | `kitty/constants.py:25` | §2.1 |
| Canonical build command | `python3 setup.py` (fails: `wl_window.c:668 -Werror=switch`) | `setup.py:492`; `Makefile all` | §3.1 |
| Adapted build result | `[exit status: 0]`, 85 units + Go `kitten` | `setup.py:1091,1130` | §3.2 |
| Heavy SGR color output | ~7.8-8.1k truecolor lines/s; parsed by `do_parse` | `kitty/vt-parser.c` | §3.3-3.5, §9.1 |
| Heavy scrollback churn | 548,899 lines (~274x the 2000-line scrollback), run 1 | — | §3.4 |
| Repeated window resizes | 640x400=71x22 -> 1200x700=133x38 -> 800x480=88x26 (X11) | — | §4.1 |
| Tab switching | 1 tab -> 3 tabs; active `[1]`->`[3]` | — | §4.2 |
| C extension loaded in main proc | `kitty/fast_data_types.so` mapped | `kitty/main.py:32` | §5.1 |
| CPython embedded | `libpython3.13.so.1.0` mapped | — | §5.1 |
| FreeType | `libfreetype.so.6.20.2` (transitive via HarfBuzz) | `setup.py:910,637` | §5.1 |
| HarfBuzz / GL / libpng / lcms2 | mapped; linked directly | `setup.py:637,639,640,641` | §5.1 |
| Thread model (idle vs stress) | 68 threads, fixed; only CPU changes | `kitty/child-monitor.c` | §6.2-6.3 |
| `KittyChildMon` (I/O) | `io_loop`; ~1.8-1.9 CPU-s under load | `kitty/child-monitor.c:1489` | §6, §9.2 |
| `KittyPeerMon` (RC) | `talk_loop`; **0.00 CPU-s** under load | `kitty/child-monitor.c:1808` | §6.3, §9.2 |
| `KittyWriteStdin` (on-demand) | absent at rest; caught in `thread_write` | `kitty/child-monitor.c:965,967` | §6.4 |
| Thread naming call | `pthread_setname_np(pthread_self(), name)` | `kitty/threading.h:34` | §6.1 |
| Control interface state | `ls` tree / `get-text` / 277 colors | `kitty/rc/ls.py`,`get_text.py`,`get_colors.py` | §7 |
| `kitty +kitten icat` (literal) | separate Go process; no `libpython` | `kitty/launcher/main.c:356,452` | §8.1-8.2 |
| `kitten icat` (Go) | separate Go process | — | §8.1 |
| `kitty icat` (os.execl) | CPython-then-exec; ~98 ms vs ~67 ms | `kitty/entry_points.py:12` | §8.1 |
| AAP-named Python `runpy` path | stub: `SystemExit('This should be run as kitten icat')` | `kittens/runner.py:116`; `kittens/icat/main.py:171-172` | §8.2 |
| Graphics-protocol APC | `a=T,f=100,s=240,v=160,t=f;<b64 path>` -> `draw_graphics` | `kitty/graphics.c:23,2376`; `kitty/shaders.c:509` | §8.3 |
| `kitten` runtime/language | Go `go1.22.12`; module `kitty` | `go.mod:1,3` | §8.4 |
| `kitten` loaded into main proc? | No — separate, stripped, dynamic-libc-via-cgo | — | §8.1,§8.4,§5.2 |
| Stack snapshot (mixed) | `py-spy --native`: `do_parse`/`utf8_decode_to_esc_256` live | — | §9.1 |
| Fallback chain | `gdb`/`eu-stack`/`gstack`/`/proc/stack` complete | — | §9.2-9.3 |
| Symbols (not stripped) | 1032 defined text symbols in extension | — | §9.4, Appendix A |
| Inference + 2 rule-outs + tradeoff | provided from artifacts | — | §10 |

### Python-path exit condition (secondary/error path, exercised)

`python3 -c "runpy.run_module('kittens.icat.main', run_name='__main__')"` -> `This should be run as kitten icat` `[exit status: 1]` (§8.2) — the error/edge path of the Python entry, exercised and reported unedited.

## 12. Read-only / cleanup attestation

The user rules require the source repository to remain unchanged except for this single answer document. Two distinct states are attested:

**Authoring-time (pre-commit) state.** Every observation script and capture used above lives **outside** the repository, under `/tmp/blitz_inv/` (the stress harness `stress_gen.py`, the run orchestrator `run_flood.sh`, the resize/tab and stack helpers, the `xresize` X11 helper, and all `.txt`/`.json`/`.log`/`.bin` captures) and `/tmp/pkgconfig-nowayland.sh` (the build wrapper). None is inside the repo tree. The build produces artifacts (`kitty/fast_data_types.so`, `kitty/glfw-x11.so`, `kitty/launcher/kitty`, `kitty/launcher/kitten`) that are **git-ignored**, so `git status --porcelain` on the working tree was empty before this document was written. The canonical build fails before the Go stage, so no `go.sum` side-effect persists.

**Delivery-time (post-commit) state.** The **only** change committed to the repository is this file, `blitzy/documentation/kitty_815df1e210e0.md`. No existing `.py`/`.c`/`.h`/`.go`/`.glsl`, build, configuration, or test file is modified; no dependency manifest is touched; no temporary observation script is committed. After the cleanup step, all `/tmp/blitz_inv/` scratch files and `/tmp/pkgconfig-nowayland.sh` are removed, leaving the repository byte-for-byte unchanged apart from the added documentation file.

---

## Appendix A — Complete defined-text symbol table of `kitty/fast_data_types.so` (all 1032 symbols)

Provided in full so the symbol evidence in §9.4 is complete rather than representative. This is the entire `nm` defined-text (`T`/`t`) listing of the extension:

```text
000000000002a0a0 T PyInit_fast_data_types
0000000000051c70 t Py_XDECREF.lto_priv.0
0000000000051c70 t Py_XDECREF.lto_priv.1
0000000000065d20 t SingleKey___len__.lto_priv.0
0000000000065ca0 t SingleKey_dealloc.lto_priv.0
0000000000065cd0 t SingleKey_defined_with_kitty_mod.lto_priv.0
0000000000065cb0 t SingleKey_get_is_native.lto_priv.0
00000000000685d0 t SingleKey_get_key.lto_priv.0
0000000000066f50 t SingleKey_get_mods.lto_priv.0
0000000000065d00 t SingleKey_hash.lto_priv.0
00000000000685f0 t SingleKey_item.lto_priv.0
0000000000068240 t SingleKey_new.lto_priv.0
0000000000068340 t SingleKey_replace.lto_priv.0
0000000000068470 t SingleKey_repr.lto_priv.0
0000000000066470 t SingleKey_resolve_kitty_mod.lto_priv.0
0000000000068670 t SingleKey_richcompare.lto_priv.0
00000000000111b0 t __cpu_indicator_init
00000000000125a0 t __do_global_dtors_aux
0000000000012910 t __len__
0000000000072c90 t __len__.lto_priv.0
0000000000076e60 t __repr__.lto_priv.0
000000000006b890 t __str__.lto_priv.0
0000000000067530 t __str__.lto_priv.1
0000000000076f40 t __str__.lto_priv.2
000000000002f690 t _fc_match
00000000000d3f70 t _fini
0000000000011000 t _init
000000000002fb00 t _native_fc_match
00000000000c1d00 t _parse_sgr.isra.0
00000000000b7c10 t _parse_sgr.lto_priv.0
0000000000043040 t _post_call_gl_callback_default
0000000000042fd0 t _pre_call_gl_callback_default
00000000000bfc60 t _report_error.lto_priv.0
00000000000b80b0 t _report_params
00000000000b6040 t _report_unknown_escape_code.lto_priv.0
00000000000b6040 t _report_unknown_escape_code.lto_priv.1
0000000000084a30 t _reverse_scroll.lto_priv.0
00000000000817f0 t _select_graphic_rendition.lto_priv.0
0000000000056dc0 t activation_token_callback.lto_priv.0
000000000002d950 t add
0000000000051b20 t add_attribute_to_vao.constprop.0
000000000001df20 t add_authenticated_but_unencrypted_data.lto_priv.0
0000000000051890 t add_buffer_to_vao
00000000000138a0 t add_child.lto_priv.0
0000000000073160 t add_combining_char.lto_priv.0
000000000001e5c0 t add_data_to_be_authenticated_but_not_decrypted.lto_priv.0
000000000001e680 t add_data_to_be_decrypted.lto_priv.0
000000000001dfe0 t add_data_to_be_encrypted.lto_priv.0
000000000002c9f0 t add_hole.part.0.lto_priv.0
00000000000160b0 t add_peer
000000000007ae20 t add_press.part.0
00000000000147b0 t add_python_timer.lto_priv.0
000000000006aff0 t add_segment.lto_priv.0
000000000002ce00 t add_to_disk_cache.part.0
000000000009a760 t add_url_range.lto_priv.0
000000000001d270 t alloc_secret
00000000000bea60 t alloc_vt_parser
000000000001c4b0 t alpha_get
000000000004a460 t application_close_requested_callback
0000000000073910 t apply_cursor.lto_priv.0
000000000008bf50 t apply_selection
0000000000081c30 t apply_sgr.lto_priv.0
0000000000024730 t apply_sgr_to_cells
000000000006ba60 t as_ansi.lto_priv.0
00000000000676c0 t as_ansi.lto_priv.1
0000000000078da0 t as_ansi.lto_priv.2
000000000001caa0 t as_color.lto_priv.0
000000000001c4e0 t as_dict.lto_priv.0
000000000006a8b0 t as_text.lto_priv.0
0000000000081ae0 t as_text.lto_priv.1
0000000000081bc0 t as_text_alternate.lto_priv.0
0000000000081b40 t as_text_for_history_buf.lto_priv.0
0000000000078e40 t as_text_generic
0000000000081b10 t as_text_non_visual.lto_priv.0
000000000009dd00 t attach_shaders
000000000008e100 t auto_repeat_enabled_get.lto_priv.0
0000000000091c30 t auto_repeat_enabled_set.lto_priv.0
000000000008e240 t backspace.lto_priv.0
00000000000d3ef0 T base64_decode
00000000000d3e50 T base64_encode
00000000000d3e40 T base64_stream_decode
00000000000d39b0 t base64_stream_decode_avx
00000000000ce8e0 t base64_stream_decode_avx2
00000000000d22d0 t base64_stream_decode_avx512
00000000000d3c40 T base64_stream_decode_init
00000000000c4890 t base64_stream_decode_neon32
00000000000d3890 t base64_stream_decode_neon64
00000000000c80e0 t base64_stream_decode_plain
00000000000ce210 t base64_stream_decode_sse41
00000000000cd8a0 t base64_stream_decode_sse42
00000000000c66c0 t base64_stream_decode_ssse3
00000000000d3bc0 T base64_stream_encode
00000000000d22e0 t base64_stream_encode_avx
00000000000c68c0 t base64_stream_encode_avx2
00000000000d22c0 t base64_stream_encode_avx512
00000000000d3bd0 T base64_stream_encode_final
00000000000d39c0 T base64_stream_encode_init
00000000000c4880 t base64_stream_encode_neon32
00000000000d3880 t base64_stream_encode_neon64
00000000000ce220 t base64_stream_encode_plain
00000000000c66d0 t base64_stream_encode_sse41
00000000000c64c0 t base64_stream_encode_sse42
00000000000c66b0 t base64_stream_encode_ssse3
000000000009c9d0 t bell.lto_priv.0
0000000000051ca0 t blank_os_window
000000000001f3a0 t blink_get
000000000001f670 t blink_set
000000000001c4a0 t blue_get
000000000001f2b0 t bold_get
000000000001f530 t bold_set
000000000001f980 t bytes_alloc
000000000001f9a0 t c0_replace_bytes
000000000004d510 t calculate_layer_shell_window_size
0000000000021e60 t canberra_play_loop
000000000001110c t canberra_play_loop.cold
000000000008e2c0 t carriage_return.lto_priv.0
00000000000776f0 t cell_as_sgr
000000000007b0d0 t cell_for_pos.constprop.0
0000000000043130 t cell_metrics
00000000000a3c60 t cell_prepare_to_render.lto_priv.0
0000000000057ae0 t change_os_window_state
00000000000806e0 t change_pointer_shape.lto_priv.0
0000000000053f90 t change_state_for_os_window.lto_priv.0
000000000004a720 t check_for_gl_error
000000000001f150 t cleanup_decref.lto_priv.0.lto_priv.0
000000000001f150 t cleanup_decref.lto_priv.1.lto_priv.0
000000000001f150 t cleanup_decref.lto_priv.2.lto_priv.0
000000000001f150 t cleanup_decref.lto_priv.3.lto_priv.0
0000000000059400 t cleanup_glfw
000000000002cb70 t clear.lto_priv.0
000000000006a090 t clear.lto_priv.1
0000000000056910 t clear_all_filter_func.lto_priv.0
00000000000568f0 t clear_filter_func.lto_priv.0
0000000000067060 t clear_line.lto_priv.0
000000000006ca70 t clear_pool.lto_priv.0
0000000000091b40 t clear_scrollback.lto_priv.0
000000000008e1e0 t clear_selection_.lto_priv.0
0000000000091fc0 t clear_tab_stop.lto_priv.0
00000000000771c0 t clear_text.lto_priv.0
0000000000012890 t clearenv_py.lto_priv.0
0000000000087b80 t clipboard_control
0000000000014940 t close_os_window
000000000001ffd0 t close_tty
00000000000145e0 t cm_thread_write
000000000008dbf0 t cmd_output.lto_priv.0
0000000000056710 t cocoa_hide_app
0000000000056720 t cocoa_hide_other_apps
0000000000057990 t cocoa_minimize_os_window
0000000000012880 t cocoa_set_menubar_title.lto_priv.0
0000000000058240 t cocoa_window_id
00000000000d38a0 t codec_choose_x86.constprop.0
0000000000012600 t collect_cursor_info
000000000001c470 t color_as_int
0000000000074060 t color_as_sgr
000000000001ce70 t color_cmp
0000000000012900 t color_hash
000000000001cb30 t color_table_address.lto_priv.0
000000000001cc10 t color_truediv
0000000000077600 t colors_for_cell.isra.0
000000000009de90 t compile_program.lto_priv.0
000000000005ed00 t compose.isra.0
0000000000030cc0 t concat_cells.lto_priv.0
0000000000097980 t continue_line_downwards
00000000000978b0 t continue_line_upwards
0000000000083e60 t continue_to_next_line
000000000001cf40 t contrast
00000000000a8120 t convert_from_opts_mouse_hide_wait.constprop.0
00000000000a8200 t convert_from_opts_scrollback_fill_enlarged_window.constprop.0
00000000000a7fc0 t convert_from_opts_scrollback_pager_history_size.constprop.0
00000000000a80c0 t convert_from_opts_touch_scroll_multiplier.constprop.0
00000000000a8070 t convert_from_opts_wheel_scroll_min_lines.constprop.0
00000000000a8010 t convert_from_opts_wheel_scroll_multiplier.constprop.0
000000000006f360 t convert_glfw_key_event_to_python.lto_priv.0
00000000000a8310 t convert_opts_from_python_opts.constprop.0
000000000001f7f0 t copy
0000000000072ed0 t copy_char.lto_priv.0
00000000000961a0 t copy_colors_from.lto_priv.0
00000000000667e0 t copy_line_to.lto_priv.0
000000000006a940 t copy_old.lto_priv.0
000000000008b770 t copy_specific_mode.lto_priv.0
00000000000a3b50 t create_cell_vao
000000000006b340 t create_historybuf.lto_priv.0
000000000006a740 t create_line_copy.lto_priv.0
0000000000054030 t create_os_window.lto_priv.0
000000000005b1d0 t create_ref
000000000003ca00 t create_test_font_group.lto_priv.0
00000000000515f0 t create_vao
00000000000b6010 t csi_letter.part.0
00000000000c80b0 t csi_letter.part.0
00000000000b7530 t csi_parse_loop
0000000000065a80 t ctrled_key
0000000000095e80 t current_char_width.lto_priv.0
0000000000033cf0 t current_fonts.lto_priv.0
0000000000095e10 t current_key_encoding_flags.lto_priv.0
0000000000080470 t current_pointer_shape.lto_priv.0
00000000000bb940 t current_state.lto_priv.0
000000000008c330 t current_url_text.lto_priv.0
0000000000091ab0 t cursor_at_prompt.lto_priv.0
0000000000081150 t cursor_back.lto_priv.0
000000000001c290 t cursor_color_get.lto_priv.0
000000000001b4b0 t cursor_color_set.lto_priv.0
0000000000092750 t cursor_down.lto_priv.0
0000000000092820 t cursor_down1.lto_priv.0
000000000004af50 t cursor_enter_callback
0000000000092910 t cursor_forward.lto_priv.0
00000000000737d0 t cursor_from.lto_priv.0
00000000000241e0 t cursor_from_sgr
000000000008e180 t cursor_key_mode_get.lto_priv.0
0000000000091cd0 t cursor_key_mode_set.lto_priv.0
000000000004cc80 t cursor_pos_callback
0000000000092510 t cursor_position.lto_priv.0
000000000001c2f0 t cursor_text_color_get.lto_priv.0
000000000001b520 t cursor_text_color_set.lto_priv.0
0000000000091e90 t cursor_up.lto_priv.0
0000000000092670 t cursor_up1.lto_priv.0
000000000008e140 t cursor_visible_get.lto_priv.0
0000000000091c80 t cursor_visible_set.lto_priv.0
0000000000058ca0 t dbus_notification_created_callback
0000000000058d30 t dbus_send_notification
000000000004a930 t dbus_user_notification_activated
0000000000013060 t dealloc.lto_priv.0
000000000001f1d0 t dealloc.lto_priv.1.lto_priv.0
000000000009d4a0 t dealloc.lto_priv.10
0000000000023670 t dealloc.lto_priv.2
0000000000040590 t dealloc.lto_priv.3
0000000000059b40 t dealloc.lto_priv.4
0000000000066f70 t dealloc.lto_priv.5
0000000000065e90 t dealloc.lto_priv.6
0000000000067ca0 t dealloc.lto_priv.7
0000000000072cd0 t dealloc.lto_priv.8
000000000007fc60 t dealloc.lto_priv.9
000000000001def0 t dealloc_aes256gcmdecrypt.lto_priv.0
000000000001de60 t dealloc_aes256gcmencrypt.lto_priv.0
0000000000013030 t dealloc_cp.lto_priv.0
000000000001d340 t dealloc_ec_key.lto_priv.0
000000000001d230 t dealloc_secret
000000000001e880 t decode_utf8
0000000000056d90 t decref_pyobj
00000000000bfb60 t decref_window_logo
000000000001c230 t default_bg_get.lto_priv.0
000000000001b440 t default_bg_set.lto_priv.0
000000000001bfb0 t default_color_table.lto_priv.0
000000000001c1d0 t default_fg_get.lto_priv.0
000000000001b3d0 t default_fg_set.lto_priv.0
0000000000030370 t del_font_group
00000000000995f0 t delete_characters.lto_priv.0
000000000006fcb0 t delete_lines.lto_priv.0
00000000000990a0 t delete_lines.lto_priv.1
0000000000012530 t deregister_tm_clones
000000000001d560 t derive_secret.lto_priv.0
00000000000b1530 t destroy_mock_window
00000000000ab110 t destroy_window
0000000000095ac0 t detect_url.lto_priv.0
000000000007d7d0 t diacritic_to_num
000000000001f370 t dim_get
000000000001f630 t dim_set
000000000006b250 t dirty_lines.lto_priv.0
0000000000067d20 t dirty_lines.lto_priv.1
0000000000091d70 t disable_ligatures_get.lto_priv.0
0000000000091dc0 t disable_ligatures_set.lto_priv.0
0000000000058e50 t disk_cache_malloc_allocator
00000000000b8300 t dispatch_csi
00000000000c8770 t dispatch_csi
00000000000b6cd0 t dispatch_dcs.lto_priv.0
00000000000c0fe0 t dispatch_dcs.lto_priv.1
000000000007abc0 t dispatch_mouse_event.isra.0
00000000000b62a0 t dispatch_osc.lto_priv.0
00000000000c0060 t dispatch_osc.lto_priv.1
0000000000074510 t dispatch_possible_click
00000000000bfe50 t dispatch_single_byte_control.lto_priv.0
000000000007b680 t do_drag_scroll.lto_priv.0
0000000000013a10 t do_parse
00000000000193f0 t do_state_check
00000000000407a0 t downsample_32bit_image
000000000004ac90 t dpi_change_callback
0000000000086a70 t draw.lto_priv.0
00000000000a4660 t draw_cells
000000000009c140 t draw_cells_simple
000000000009bf60 t draw_graphics.constprop.0
000000000009bd80 t draw_graphics.constprop.1
00000000000a4410 t draw_graphics.constprop.2
0000000000086af0 t draw_text.constprop.0
00000000000867d0 t draw_text.lto_priv.0
0000000000053b80 t draw_text_callback
0000000000085ad0 t draw_text_loop
000000000009c420 t draw_tint
000000000009d170 t draw_window_logo.part.0
000000000004cf10 t drop_callback
000000000009a240 t dump_lines_with_attrs.lto_priv.0
000000000004d400 t edge_spacing
000000000001db20 t elliptic_curve_key_get_private.lto_priv.0
000000000001da60 t elliptic_curve_key_get_public.lto_priv.0
00000000000c48a0 t enc_loop_ssse3
000000000006e090 t encode_glfw_key_event
0000000000079d30 t encode_mouse_event_impl.lto_priv.0
00000000000680e0 t encode_printable_ascii_key_legacy.isra.0
000000000001e8e0 t encode_utf8
000000000001ff40 t end_x11_startup_notification
000000000002e0d0 t ensure_canvas_can_fit.part.0
0000000000051d90 t ensure_csd_title_render_ctx.lto_priv.0
000000000002e0b0 t ensure_initialized.part.0
0000000000040eb0 t ensure_name_table.part.0
000000000002c0c0 t ensure_state.lto_priv.0
0000000000099400 t erase_characters.lto_priv.0
000000000008a250 t erase_in_display.lto_priv.0
0000000000085120 t erase_in_line.lto_priv.0
000000000004a7e0 t error_callback
0000000000020390 t expand_ansi_c_escapes
0000000000040c30 t extra_data.lto_priv.0
0000000000037990 t face_from_descriptor
000000000003cab0 t face_from_path
0000000000035850 t fallback_font
0000000000037df0 t fallback_font.lto_priv.0
0000000000042c90 t fallback_for_char.lto_priv.0
000000000002f730 t fc_list.lto_priv.0
000000000002fcd0 t fc_match.lto_priv.0
00000000000300f0 t fc_match_postscript_name.lto_priv.0
00000000000626b0 t filter_refs.lto_priv.0
0000000000021920 t finalize.lto_priv.0
000000000002e870 t finalize.lto_priv.1
00000000000b3e60 t finalize.lto_priv.2
00000000000b42a0 t finalize.lto_priv.3
000000000008d7a0 t find_cmd_output.lto_priv.0
00000000000720c0 t find_colon_slash.isra.0
000000000009c950 t find_either_of_two_bytes
000000000009ff10 t find_either_of_two_bytes_128
00000000000a16f0 t find_either_of_two_bytes_256
000000000009c7f0 t find_either_of_two_bytes_scalar.lto_priv.0
00000000000478e0 t find_fallback_font_for
0000000000021730 t find_in_memoryview
00000000000350a0 t find_matching_namerec.lto_priv.0
000000000005dc00 t find_or_create_glyph_properties
000000000006aac0 t find_or_create_image.part.0.lto_priv.0
000000000005d1e0 t find_or_create_sprite_position
00000000000bef40 t find_or_create_window_logo
000000000005aeb0 t finish_command_response
0000000000097ce0 t focus_changed.lto_priv.0
000000000008e0c0 t focus_tracking_enabled_get.lto_priv.0
0000000000091be0 t focus_tracking_enabled_set.lto_priv.0
000000000003b510 t font_group_for.lto_priv.0
000000000006f620 t format_mods
00000000000125e0 t frame_dummy
000000000004b240 t framebuffer_size_callback
00000000000452d0 t free_face.lto_priv.0
0000000000034370 t free_font_data.lto_priv.0
0000000000040e90 t free_freetype.lto_priv.0
000000000005e090 t free_glyph_properties_hash_table
000000000005e2d0 t free_image.lto_priv.0
0000000000059810 t free_image_resources
00000000000597c0 t free_load_data
0000000000073150 t free_pending_click
000000000005daa0 t free_sprite_position_hash_table
00000000000bb8d0 t free_vt_parser
00000000000becb0 t free_window_logo.isra.0
00000000000818a0 t garbage_collect_hyperlink_pool.lto_priv.0
000000000002c5c0 t get
0000000000041180 t get_best_name.lto_priv.0
0000000000035410 t get_best_name.lto_priv.1
00000000000351c0 t get_best_name_from_name_table
0000000000058e30 t get_click_interval
0000000000058ff0 t get_clipboard_data
0000000000059130 t get_clipboard_mime
000000000005fc70 t get_coalesced_frame_data_impl.constprop.0
000000000005fa80 t get_coalesced_frame_data_impl.lto_priv.0
000000000005f430 t get_coalesced_frame_data_standalone.isra.0
0000000000056c10 t get_content_scale_for_window
000000000004d1f0 t get_current_selection
000000000001f940 t get_docs_ref_map
000000000003ae20 t get_fallback_font.lto_priv.0
000000000006cfe0 t get_id_for_hyperlink
0000000000066f20 t get_image_count.lto_priv.0
000000000004a4b0 t get_ime_cursor_position
0000000000065d30 t get_line.lto_priv.1
0000000000081620 t get_line_from_offset
000000000006bd30 t get_line_wrapper.lto_priv.0
00000000000a3230 t get_prefix_and_suffix_for_escape_code.part.0.lto_priv.0
0000000000081440 t get_range_line
00000000000413b0 t get_variable_data.lto_priv.0
0000000000040c50 t get_variation.lto_priv.0
0000000000081240 t get_visual_line
0000000000053680 t get_window_content_scale.lto_priv.0
0000000000045880 t gladLoadGLUserPtr.constprop.0
00000000000498e0 t gladUninstallGLDebug
000000000003d330 t glad_debug_impl_glActiveTexture.lto_priv.0
000000000003d380 t glad_debug_impl_glAttachShader.lto_priv.0
000000000003d3e0 t glad_debug_impl_glBindBuffer.lto_priv.0
000000000003d440 t glad_debug_impl_glBindBufferBase.lto_priv.0
000000000003d4b0 t glad_debug_impl_glBindTexture.lto_priv.0
000000000003d510 t glad_debug_impl_glBindVertexArray.lto_priv.0
000000000003d560 t glad_debug_impl_glBlendFunc.lto_priv.0
000000000003d5c0 t glad_debug_impl_glBlendFuncSeparate.lto_priv.0
000000000003d640 t glad_debug_impl_glBufferData.lto_priv.0
000000000003d6c0 t glad_debug_impl_glClear.lto_priv.0
000000000003d710 t glad_debug_impl_glClearColor.lto_priv.0
000000000003d7e0 t glad_debug_impl_glCompileShader.lto_priv.0
000000000003d830 t glad_debug_impl_glCopyImageSubData.lto_priv.0
000000000003d9b0 t glad_debug_impl_glCreateProgram.lto_priv.0
000000000003da30 t glad_debug_impl_glCreateShader.lto_priv.0
000000000003dac0 t glad_debug_impl_glDeleteBuffers.lto_priv.0
000000000003db30 t glad_debug_impl_glDeleteProgram.lto_priv.0
000000000003db80 t glad_debug_impl_glDeleteShader.lto_priv.0
000000000003dbd0 t glad_debug_impl_glDeleteTextures.lto_priv.0
000000000003dc40 t glad_debug_impl_glDeleteVertexArrays.lto_priv.0
000000000003dcb0 t glad_debug_impl_glDisable.lto_priv.0
000000000003dd00 t glad_debug_impl_glDrawArrays.lto_priv.0
000000000003dd70 t glad_debug_impl_glDrawArraysInstanced.lto_priv.0
000000000003ddf0 t glad_debug_impl_glEnable.lto_priv.0
000000000003de40 t glad_debug_impl_glEnableVertexAttribArray.lto_priv.0
000000000003de90 t glad_debug_impl_glGenBuffers.lto_priv.0
000000000003df00 t glad_debug_impl_glGenTextures.lto_priv.0
000000000003df70 t glad_debug_impl_glGenVertexArrays.lto_priv.0
000000000003dfe0 t glad_debug_impl_glGetActiveUniform.lto_priv.0
000000000003e090 t glad_debug_impl_glGetActiveUniformBlockiv.lto_priv.0
000000000003e110 t glad_debug_impl_glGetActiveUniformsiv.lto_priv.0
000000000003e1a0 t glad_debug_impl_glGetAttribLocation.lto_priv.0
000000000003e240 t glad_debug_impl_glGetFramebufferAttachmentParameteriv.lto_priv.0
000000000003e2c0 t glad_debug_impl_glGetIntegerv.lto_priv.0
000000000003e330 t glad_debug_impl_glGetProgramInfoLog.lto_priv.0
000000000003e3b0 t glad_debug_impl_glGetProgramiv.lto_priv.0
000000000003e420 t glad_debug_impl_glGetShaderInfoLog.lto_priv.0
000000000003e4a0 t glad_debug_impl_glGetShaderiv.lto_priv.0
000000000003e510 t glad_debug_impl_glGetString.lto_priv.0
000000000003e5a0 t glad_debug_impl_glGetTexImage.lto_priv.0
000000000003e630 t glad_debug_impl_glGetUniformBlockIndex.lto_priv.0
000000000003e6d0 t glad_debug_impl_glGetUniformIndices.lto_priv.0
000000000003e750 t glad_debug_impl_glGetUniformLocation.lto_priv.0
000000000003e7f0 t glad_debug_impl_glLinkProgram.lto_priv.0
000000000003e840 t glad_debug_impl_glMapBuffer.lto_priv.0
000000000003e8d0 t glad_debug_impl_glPixelStorei.lto_priv.0
000000000003e930 t glad_debug_impl_glShaderSource.lto_priv.0
000000000003e9b0 t glad_debug_impl_glTexImage2D.lto_priv.0
000000000003ea80 t glad_debug_impl_glTexParameterfv.lto_priv.0
000000000003eaf0 t glad_debug_impl_glTexParameteri.lto_priv.0
000000000003eb60 t glad_debug_impl_glTexStorage3D.lto_priv.0
000000000003ec00 t glad_debug_impl_glTexSubImage3D.lto_priv.0
000000000003ecf0 t glad_debug_impl_glUniform1f.lto_priv.0
000000000003ed70 t glad_debug_impl_glUniform1fv.lto_priv.0
000000000003ede0 t glad_debug_impl_glUniform1i.lto_priv.0
000000000003ee40 t glad_debug_impl_glUniform1ui.lto_priv.0
000000000003eea0 t glad_debug_impl_glUniform1uiv.lto_priv.0
000000000003ef10 t glad_debug_impl_glUniform2ui.lto_priv.0
000000000003ef80 t glad_debug_impl_glUniform3f.lto_priv.0
000000000003f040 t glad_debug_impl_glUniform4f.lto_priv.0
000000000003f120 t glad_debug_impl_glUnmapBuffer.lto_priv.0
000000000003f1b0 t glad_debug_impl_glUseProgram.lto_priv.0
000000000003f200 t glad_debug_impl_glVertexAttribDivisorARB.lto_priv.0
000000000003f260 t glad_debug_impl_glVertexAttribIPointer.lto_priv.0
000000000003f2f0 t glad_debug_impl_glVertexAttribPointer.lto_priv.0
000000000003f3a0 t glad_debug_impl_glViewport.lto_priv.0
000000000003f420 t glad_gl_get_proc_from_userptr.lto_priv.0
0000000000043080 t glad_gl_has_extension
000000000003f430 t glad_gl_load_GL_VERSION_1_3.constprop.0
000000000003f730 t glad_gl_load_GL_VERSION_1_4.constprop.0
000000000003fa40 t glad_gl_load_GL_VERSION_2_0.constprop.0
0000000000040030 t glad_gl_load_GL_VERSION_3_0.constprop.0
0000000000050c90 t glfw_get_key_name.lto_priv.0
0000000000050b40 t glfw_get_physical_dpi.lto_priv.0
000000000004d920 t glfw_init.lto_priv.0
000000000004a800 t glfw_terminate.lto_priv.0
000000000004d390 t glfw_window_hint.lto_priv.0
000000000001c490 t green_get
000000000005e1f0 t grman_alloc
0000000000062900 t grman_handle_command
000000000005e840 t grman_put_cell_image.constprop.0.isra.0
00000000000627e0 t grman_scroll_images
0000000000060890 t grman_update_layers
000000000007b230 t handle_button_event
00000000000615c0 t handle_delete_command
00000000000b8210 t handle_mode
00000000000c7f00 t handle_mode
0000000000079fe0 t handle_move_event
0000000000060260 t handle_put_command
0000000000013310 t handled_signals.lto_priv.0
0000000000091a90 t has_activity_since_last_focus.lto_priv.0
0000000000030e90 t has_cell_text
000000000004a8a0 t has_current_selection
0000000000091a70 t has_focus.lto_priv.0
000000000005a1e0 t has_good_ancestry
00000000000800c0 t has_selection.lto_priv.0
0000000000072440 t has_url_beyond_colon_slash.isra.0
000000000001c3b0 t highlight_bg_get.lto_priv.0
000000000001b600 t highlight_bg_set.lto_priv.0
000000000001c350 t highlight_fg_get.lto_priv.0
000000000001b590 t highlight_fg_set.lto_priv.0
000000000006bfc0 t historybuf_add_line
000000000006b490 t historybuf_clear
000000000006bd70 t historybuf_push.part.0
000000000006c170 t historybuf_rewrap
000000000009ab90 t hyperlink_at.lto_priv.0
0000000000081a40 t hyperlink_for_id.lto_priv.0
000000000008cb80 t hyperlink_id_for_range.lto_priv.0
0000000000074150 t hyperlink_ids.lto_priv.0
00000000000818c0 t hyperlinks_as_list.lto_priv.0
0000000000040750 t identify_for_debug.lto_priv.0
0000000000095ef0 t ignore_bells_for.lto_priv.0
0000000000066040 t image_as_dict
0000000000059530 t img_by_internal_id.isra.0
000000000008e080 t in_bracketed_paste_mode_get.lto_priv.0
0000000000091b90 t in_bracketed_paste_mode_set.lto_priv.0
000000000007f950 t index_selection.constprop.0.isra.0
000000000007f8d0 t index_selection.constprop.1.isra.0
000000000007cfa0 t inflate_png_inner
00000000000356b0 t information_for_font_family
0000000000030a30 t init_font
000000000005b860 t init_glfw
000000000006b5d0 t init_line.lto_priv.0
0000000000079970 t init_loop_data
000000000007de80 t init_overlay_line.lto_priv.0
00000000000742f0 t init_signal_handlers_py.lto_priv.0
000000000007fa00 t init_text_loop_line
0000000000021a30 t init_x11_startup_notification
000000000003afe0 t initialize_font
0000000000059f20 t initialize_load_data
00000000000aafa0 t initialize_window
00000000000134f0 t inject_peer.lto_priv.0
00000000000991c0 t insert_characters.lto_priv.0
0000000000071020 t insert_lines.lto_priv.0
0000000000098f60 t insert_lines.lto_priv.1
00000000000153f0 t io_loop
0000000000011076 t io_loop.cold
000000000008e310 t is_char_ok_for_word_extension
00000000000bba60 t is_combining_char
00000000000665e0 t is_continued.lto_priv.0
00000000000587a0 t is_css_pointer_name_valid
0000000000035b00 t is_emoji.lto_priv.0
0000000000035b00 t is_emoji_presentation_base.lto_priv.0.lto_priv.0
0000000000035b00 t is_emoji_presentation_base.lto_priv.1.lto_priv.0
0000000000035b00 t is_emoji_presentation_base.lto_priv.2.lto_priv.0
0000000000031a10 t is_empty_glyph
0000000000091a30 t is_main_linebuf.lto_priv.0
000000000008e2e0 t is_rectangle_select.lto_priv.0
000000000007fbc0 t is_using_alternate_linebuf.lto_priv.0
000000000001f2e0 t italic_get
000000000001f570 t italic_set
000000000008bcd0 t iteration_data.lto_priv.0
000000000004b5a0 t key_callback
0000000000072c60 t last_char_has_wrapped_flag.lto_priv.0
0000000000072d70 t left_shift.lto_priv.0
000000000006b7f0 t line.lto_priv.0
0000000000066500 t line.lto_priv.1
0000000000081950 t line.lto_priv.2
0000000000076fd0 t line_add_combining_char
0000000000077250 t line_apply_cursor.constprop.0
00000000000784d0 t line_as_ansi
0000000000077080 t line_clear_text
00000000000929f0 t line_edge_colors.lto_priv.0
00000000000774a0 t line_right_shift
00000000000767b0 t line_startswith_url_chars
0000000000074f80 t line_url_end_at
0000000000074d30 t line_url_start_at
000000000006fd30 t linebuf_copy_line_to
000000000006f9d0 t linebuf_delete_lines
000000000006f8b0 t linebuf_index
0000000000070d70 t linebuf_insert_lines.part.0
000000000006f930 t linebuf_reverse_index
000000000006fde0 t linebuf_rewrap
0000000000095620 t linefeed.lto_priv.0
00000000000a7d80 t list_of_chars
000000000004a030 t live_resize_callback
00000000000a4580 t load_alpha_mask_texture.lto_priv.0
0000000000045770 t load_font.lto_priv.0
000000000002e0f0 t load_fontconfig_lib
000000000005a2d0 t load_image_data
000000000007d6b0 t load_png_data.lto_priv.0
00000000000201f0 t locale_is_valid
0000000000078210 t log_error
00000000000794a0 t log_error_string.lto_priv.0
000000000001cce0 t luminance_get
0000000000014090 t main_loop.lto_priv.0
00000000000592f0 t make_x11_window_a_dock_window
00000000000919f0 t mark_as_dirty.lto_priv.0
0000000000013b70 t mark_for_close.lto_priv.0
00000000000bccf0 t mark_for_codepoint
0000000000077b30 t mark_text_in_line
0000000000098a50 t marked_cells.lto_priv.0
0000000000012a70 t mask_kitty_signals_process_wide.lto_priv.0
0000000000012920 t mask_variadic_signals.constprop.0
000000000007c020 t mock_mouse_selection.lto_priv.0
0000000000013420 t monitor_pid.lto_priv.0
000000000004c930 t mouse_button_callback
000000000007c0c0 t mouse_event
000000000007bcd0 t mouse_selection
00000000000803e0 t move_cursor_off_wide_char_trailer
000000000001f0c0 t needs_write.lto_priv.0
000000000003cfc0 t new.lto_priv.0
000000000001e2f0 t new_aes256gcmdecrypt.lto_priv.0
000000000001dc40 t new_aes256gcmencrypt.lto_priv.0
0000000000012da0 t new_childmonitor_object.lto_priv.0
000000000001cb40 t new_color
0000000000013d00 t new_cp.lto_priv.0
000000000001f1c0 t new_cursor_object
000000000001f3d0 t new_diskcache_object
000000000001d370 t new_ec_key.lto_priv.0
0000000000065dd0 t new_graphicsmanager_object.lto_priv.0
000000000006b090 t new_history_object.lto_priv.0
000000000006f470 t new_keyevent_object.lto_priv.0
0000000000072ca0 t new_line_object.lto_priv.0
000000000006a110 t new_linebuf_object.lto_priv.0
000000000007df70 t new_screen_object.lto_priv.0
0000000000012fd0 t new_secret
000000000009dc00 t new_shlex_object.lto_priv.0
00000000000bec40 t new_vtparser_object.lto_priv.0
000000000009ef30 t next_word.lto_priv.0
00000000000111a7 t next_word.lto_priv.0.cold
000000000001fd90 t normal_tty
0000000000023e60 t num_cached_in_ram
00000000000b5f80 t num_users.lto_priv.0
000000000004a9c0 t on_system_color_scheme_change
0000000000022000 t open_cache_file
000000000001fba0 t open_tty
000000000004d890 t opengl_version_string.lto_priv.0
00000000000b3cd0 t os_window_focus_counters.lto_priv.0
00000000000b5100 t os_window_regions
0000000000071980 t pagerhist_as_bytes.lto_priv.0
0000000000071fe0 t pagerhist_as_text.lto_priv.0
0000000000072050 t pagerhist_rewrap.lto_priv.0
00000000000710c0 t pagerhist_rewrap_to
0000000000067ac0 t pagerhist_write.lto_priv.0
0000000000067230 t pagerhist_write_bytes.part.0
000000000001f1e0 t parse_color
0000000000034610 t parse_font_feature.lto_priv.0
00000000000a7950 t parse_font_mod_size
00000000000c23d0 t parse_graphics_code
00000000000cf220 t parse_graphics_code
00000000000692e0 t parse_input_from_terminal.lto_priv.0
00000000000b6190 t parse_osc_8.constprop.0
00000000000be960 t parse_sgr.constprop.0
00000000000c43c0 t parse_worker
00000000000d2280 t parse_worker_dump
0000000000098c40 t paste.lto_priv.0
0000000000098e00 t paste_bytes.lto_priv.0
000000000001b6e0 t patch_color_profiles.lto_priv.0
0000000000042db0 t path_for_font.lto_priv.0
000000000002e8c0 t pattern_as_dict
0000000000092480 t pause_rendering.lto_priv.0
0000000000043690 t place_bitmap_in_canvas.lto_priv.0
000000000002bbe0 t play_canberra_sound.constprop.0
000000000002c010 t play_desktop_sound
0000000000059f00 t png_error_handler
0000000000074ce0 t png_error_handler
000000000005e440 t png_from_data
000000000005e570 t png_path_to_bitmap
0000000000058830 t pointer_name_to_css_name
0000000000058400 t pointer_name_to_glfw_name
00000000000a79c0 t pointer_shape
0000000000043d00 t populate_processed_bitmap.part.0.lto_priv.0
0000000000040be0 t postscript_name.lto_priv.0
0000000000056c90 t primary_monitor_content_scale
0000000000057a00 t primary_monitor_size
0000000000059d50 t print_png_read_error
0000000000048dc0 t process_codepoint
0000000000016620 t process_global_state
00000000000110c6 t process_global_state.cold
000000000005a8e0 t process_image_data.isra.0
000000000006c0c0 t push.lto_priv.0
0000000000020270 t py_getpeereid
0000000000021800 t py_monotonic
0000000000020090 t py_shm_open
0000000000020160 t py_shm_unlink
000000000001f960 t py_terminfo_data
0000000000021880 t py_timed_debug_print
00000000000b36e0 t pyadd_borders_rect.lto_priv.0
00000000000b3980 t pyadd_tab.lto_priv.0
00000000000b34e0 t pyadd_window.lto_priv.0
00000000000ab330 t pyapply_options_update.lto_priv.0
00000000000b30d0 t pyattach_window.lto_priv.0
00000000000aff40 t pybackground_opacity_of.lto_priv.0
000000000002dee0 t pybase64_decode
000000000002dd20 t pybase64_encode
000000000009e190 t pybind_program.lto_priv.0
000000000009e1d0 t pybind_vertex_array.lto_priv.0
00000000000af360 t pycell_size_for_window.lto_priv.0
00000000000ac600 t pychange_background_opacity.lto_priv.0
00000000000b2150 t pyclick_mouse_cmd_output.lto_priv.0
00000000000b1c00 t pyclick_mouse_url.lto_priv.0
0000000000067120 t pycreate_canvas.lto_priv.0
00000000000b1580 t pycreate_mock_window.lto_priv.0
000000000009e160 t pycreate_vao.lto_priv.0
00000000000af570 t pycurrent_application_quit_request.lto_priv.0
00000000000aead0 t pycurrent_focused_os_window_id.lto_priv.0
00000000000aed80 t pycurrent_os_window.lto_priv.0
00000000000a82b0 t pydestroy_global_data.lto_priv.0
00000000000b2c00 t pydetach_window.lto_priv.0
000000000006f180 t pyencode_key_for_tty.lto_priv.0
0000000000023360 t pyensure_state
00000000000ac3a0 t pyfocus_os_window.lto_priv.0
00000000000a7900 t pyget_boss.lto_priv.0
00000000000a7f60 t pyget_options.lto_priv.0
00000000000af850 t pyget_os_window_pos.lto_priv.0
00000000000b07a0 t pyget_os_window_size.lto_priv.0
00000000000af590 t pyget_os_window_title.lto_priv.0
00000000000af7b0 t pyglobal_font_size.lto_priv.0
00000000000af080 t pyhandle_for_window_id.lto_priv.0
000000000006af40 t pyimage_for_client_id.lto_priv.0
0000000000066330 t pyimage_for_client_number.lto_priv.0
000000000009e210 t pyinit_borders_program.lto_priv.0
000000000009e450 t pyinit_cell_program.lto_priv.0
0000000000066400 t pyis_modifier_key.lto_priv.0
0000000000066670 t pykey_for_native_key_name.lto_priv.0
00000000000aea70 t pylast_focused_os_window_id.lto_priv.0
00000000000ae640 t pymark_os_window_dirty.lto_priv.0
00000000000ac110 t pymark_os_window_for_close.lto_priv.0
00000000000afdb0 t pymark_tab_bar_dirty.lto_priv.0
00000000000b1930 t pymouse_selection.lto_priv.0
00000000000b2570 t pymove_cursor_to_mouse_if_in_prompt.lto_priv.0
00000000000aea50 t pynext_window_id.lto_priv.0
00000000000b03d0 t pyos_window_font_size.lto_priv.0
00000000000abf10 t pyos_window_has_background_image.lto_priv.0
00000000000ad5e0 t pypatch_global_colors.lto_priv.0
00000000000b0120 t pypt_to_px.lto_priv.0
000000000002c920 t pyread_from_cache_file
00000000000a8280 t pyredirect_mouse_handling.lto_priv.0
0000000000023a90 t pyremove
00000000000b4f20 t pyremove_tab.lto_priv.0
00000000000ae320 t pyremove_window.lto_priv.0
00000000000a7440 t pyrun_with_activation_token.lto_priv.0
00000000000ae450 t pyset_active_tab.lto_priv.0
00000000000afc70 t pyset_active_window.lto_priv.0
00000000000ac320 t pyset_application_quit_request.lto_priv.0
00000000000b0f00 t pyset_background_image.lto_priv.0
00000000000a8190 t pyset_boss.lto_priv.0
00000000000a8250 t pyset_ignore_os_keyboard_processing.lto_priv.0
00000000000142c0 t pyset_iutf8.lto_priv.0
000000000001fae0 t pyset_iutf8.lto_priv.1
00000000000abaf0 t pyset_options.lto_priv.0
00000000000afa70 t pyset_os_window_chrome.lto_priv.0
00000000000ad3e0 t pyset_os_window_pos.lto_priv.0
00000000000ad1c0 t pyset_os_window_size.lto_priv.0
00000000000b5cc0 t pyset_os_window_title.lto_priv.0
00000000000abc20 t pyset_tab_bar_render_data.lto_priv.0
00000000000adb40 t pyset_window_logo.lto_priv.0
00000000000ac820 t pyset_window_padding.lto_priv.0
00000000000acb20 t pyset_window_render_data.lto_priv.0
0000000000066c60 t pyshm_unlink.lto_priv.0
0000000000066af0 t pyshm_write.lto_priv.0
00000000000ae810 t pyswap_tabs.lto_priv.0
00000000000b5ad0 t pysync_os_window_title.lto_priv.0
0000000000030c00 t python_send_to_gpu
0000000000014750 t python_timer_callback
0000000000013000 t python_timer_cleanup
000000000009c760 t pyunbind_program.lto_priv.0
000000000009c780 t pyunbind_vertex_array.lto_priv.0
000000000009ca90 t pyunmap_vao_buffer.lto_priv.0
00000000000b5950 t pyupdate_ime_position_for_window.lto_priv.0
0000000000066cf0 t pyupdate_layers.lto_priv.0
00000000000b1660 t pyupdate_pointer_shape.lto_priv.0
00000000000b0a80 t pyupdate_tab_bar_edge_colors.lto_priv.0
00000000000ae010 t pyupdate_window_title.lto_priv.0
00000000000acee0 t pyupdate_window_visibility.lto_priv.0
00000000000b52c0 t pyviewport_for_window.lto_priv.0
00000000000668f0 t pyw_index.lto_priv.0
00000000000a7930 t pywakeup_main_loop.lto_priv.0
0000000000016220 t queue_peer_message
000000000001a2c0 t random_unix_socket.lto_priv.0
000000000008baf0 t range_line_.lto_priv.0
000000000001fe40 t raw_tty
0000000000068760 t read_command_response.lto_priv.0
000000000002c860 t read_from_cache_file.lto_priv.0
0000000000034c00 t read_from_disk_cache.part.0
00000000000798e0 t read_png_error_handler
0000000000074c40 t read_png_from_buffer
0000000000079950 t read_png_warn_handler
0000000000079510 t read_signals_py.lto_priv.0
000000000009ceb0 t realloc_sprite_texture
000000000001c480 t red_get
000000000001f860 t redirect_std_streams
0000000000059680 t ref_by_internal_id.isra.0
000000000004a290 t refresh_callback
0000000000012560 t register_tm_clones
0000000000091a10 t reload_all_gpu_data.lto_priv.0
0000000000015230 t remove_children
0000000000011052 t remove_children.cold
00000000000346e0 t remove_from_disk_cache.part.0
000000000002c700 t remove_from_ram
0000000000012ab0 t remove_python_timer.lto_priv.0
0000000000060100 t remove_ref.lto_priv.0
0000000000074460 t remove_signal_handlers_py.lto_priv.0
00000000000b4ce0 t remove_tab_inner.lto_priv.0
00000000000ab570 t remove_window_inner
000000000009d4f0 t render_a_bar
0000000000043d30 t render_bitmap.isra.0
00000000000444d0 t render_glyphs_in_cells.isra.0
000000000003d180 t render_gray_bitmap.isra.0
0000000000036130 t render_groups
00000000000392e0 t render_line
0000000000049520 t render_line.lto_priv.0
0000000000086c20 t render_overlay_line
00000000000368f0 t render_run
0000000000047eb0 t render_run
0000000000044de0 t render_sample_text.lto_priv.0
0000000000048ea0 t render_single_line
000000000008e1c0 t render_unfocused_cursor_get.lto_priv.0
0000000000091d20 t render_unfocused_cursor_set.lto_priv.0
0000000000021380 t replace_c0_codes_except_nl_space_tab
0000000000087420 t report_device_status
00000000000874f0 t report_mode_status
000000000001d110 t repr
000000000001f400 t repr.lto_priv.0
0000000000040650 t repr.lto_priv.1
0000000000095b50 t rescale_images.lto_priv.0
0000000000095aa0 t reset.lto_priv.0
000000000007fbf0 t reset_callbacks.lto_priv.0
000000000001b3a0 t reset_color.lto_priv.0
0000000000013f10 t reset_color_table.lto_priv.0
000000000007fba0 t reset_dirty.lto_priv.0
000000000001f280 t reset_display_attrs
000000000008b940 t reset_mode.lto_priv.0
00000000000932e0 t resize.lto_priv.0
00000000000140f0 t resize_pty.lto_priv.0
000000000001f310 t reverse_get
00000000000669f0 t reverse_index.lto_priv.0
0000000000092dc0 t reverse_index.lto_priv.1
0000000000099e60 t reverse_scroll.lto_priv.0
000000000001f5b0 t reverse_set
000000000006c9c0 t rewrap.lto_priv.0
0000000000070c70 t rewrap.lto_priv.1
000000000001c4c0 t rgb_get
000000000001d030 t richcmp
000000000001f6b0 t richcmp.lto_priv.0
00000000000741a0 t richcmp.lto_priv.1
0000000000077550 t right_shift.lto_priv.0
0000000000056920 t ring_bell
00000000000128c0 t run_at_exit_cleanup_functions
00000000000c30b0 t run_worker.lto_priv.0
00000000000d0300 t run_worker.lto_priv.1
0000000000014880 t safe_pipe.lto_priv.0
000000000007ce10 t scale_scroll.lto_priv.0
000000000001e9c0 t schedule_write_to_child
000000000001edd0 t schedule_write_to_child.constprop.0
0000000000082470 t screen_align
000000000008ca90 t screen_apply_selection
00000000000857b0 t screen_bell
00000000000853a0 t screen_delete_characters
000000000008d050 t screen_detect_url
000000000001118b t screen_detect_url.cold
00000000000831d0 t screen_dirty_line_graphics.lto_priv.0
000000000007f780 t screen_dirty_sprite_positions
0000000000085620 t screen_erase_characters
00000000000897a0 t screen_erase_in_display
0000000000084ee0 t screen_erase_in_line
000000000006cbe0 t screen_garbage_collect_hyperlink_pool
0000000000084630 t screen_handle_graphics_command
000000000008c780 t screen_has_selection
0000000000083ae0 t screen_index
00000000000851b0 t screen_insert_characters
0000000000099670 t screen_is_emoji_presentation_base.lto_priv.0
0000000000087360 t screen_manipulate_title_stack.isra.0
0000000000098760 t screen_mark_all
000000000009a8b0 t screen_mark_hyperlink.isra.0
0000000000011194 t screen_mark_hyperlink.isra.0.cold
000000000008a2e0 t screen_pause_rendering
0000000000088880 t screen_pause_rendering.constprop.0
0000000000088190 t screen_pop_colors
0000000000087f80 t screen_push_colors
0000000000083690 t screen_push_key_encoding_flags
00000000000a3260 t screen_render_line_graphics.part.0.lto_priv.0
00000000000868f0 t screen_repeat_character
00000000000883a0 t screen_report_color_stack
0000000000083360 t screen_report_key_encoding_flags
0000000000087290 t screen_report_size
0000000000088570 t screen_request_capabilities
00000000000821b0 t screen_rescale_images
000000000007eb80 t screen_reset
0000000000084d60 t screen_restore_cursor
0000000000084810 t screen_reverse_index
0000000000084300 t screen_scroll
000000000009a600 t screen_selection_range_for_word.constprop.0
00000000000834c0 t screen_set_key_encoding_flags
0000000000087ad0 t screen_set_margins
000000000009a4b0 t screen_start_selection
0000000000083810 t screen_tab
00000000000889e0 t screen_toggle_screen_buffer
0000000000081cf0 t screen_truncate_point_for_length.lto_priv.0
0000000000088ca0 t screen_update_cell_data
000000000008d5c0 t screen_update_overlay_text
000000000009ac80 t screen_update_selection
00000000000921b0 t scroll.lto_priv.0
00000000000527e0 t scroll_callback
0000000000056730 t scroll_filter_func
0000000000056750 t scroll_filter_margins_func
0000000000099ef0 t scroll_prompt_to_bottom.lto_priv.0
0000000000094f20 t scroll_to_next_mark.lto_priv.0
00000000000922b0 t scroll_to_prompt.lto_priv.0
0000000000083f50 t scroll_until_cursor_prompt.lto_priv.0
000000000006b5a0 t segment_for.part.0.lto_priv.0
00000000000825e0 t select_graphic_rendition
0000000000082a80 t select_graphic_rendition.constprop.0
0000000000019400 t send_data_to_peer.lto_priv.0
0000000000097de0 t send_escape_code_to_child.lto_priv.0
00000000000a3a10 t send_image_to_gpu
000000000007b780 t send_mock_mouse_event_to_window.lto_priv.0
000000000007a5e0 t send_mouse_event.lto_priv.0
00000000000ab7f0 t send_pending_click_to_window_id
0000000000033360 t send_prerendered_sprites
0000000000037460 t send_prerendered_sprites_for_window.isra.0
0000000000016370 t send_response_to_peer
00000000000a3970 t send_sprite_to_gpu
0000000000067d90 t serialize
000000000001a430 t serialize_string_tuple
000000000006a350 t set_attribute.lto_priv.0
0000000000073cf0 t set_attribute.lto_priv.1
000000000009c550 t set_cell_uniforms
0000000000073600 t set_char.lto_priv.0
00000000000591c0 t set_clipboard_data_types
0000000000012b20 t set_color.lto_priv.0
0000000000059df0 t set_command_failed_response
0000000000012ba0 t set_configured_colors.lto_priv.0
0000000000066710 t set_continued.lto_priv.0
0000000000058a90 t set_custom_cursor
000000000004d2b0 t set_default_window_icon.lto_priv.0
000000000001d150 t set_error_from_openssl.isra.0
00000000000305f0 t set_font_data.lto_priv.0
0000000000037560 t set_font_size.lto_priv.0
00000000000378b0 t set_load_error.isra.0
00000000000920a0 t set_margins.lto_priv.0
0000000000098960 t set_marker.lto_priv.0
000000000008b9d0 t set_mode.lto_priv.0
000000000008b290 t set_mode_from_const.lto_priv.0
0000000000051e40 t set_mouse_cursor
0000000000056430 t set_os_window_chrome
000000000005b770 t set_os_window_title
0000000000042ec0 t set_pixel_size.part.0
000000000002e040 t set_send_sprite_to_gpu.lto_priv.0
00000000000377d0 t set_size.lto_priv.0
00000000000b41f0 t set_systemd_error.isra.0
000000000008e210 t set_tab_stop.lto_priv.0
00000000000732b0 t set_text.lto_priv.0
00000000000742c0 t set_use_os_log.lto_priv.0
00000000000810d0 t set_window_char.lto_priv.0
000000000003d0c0 t setup_regions
000000000001cd30 t sgr_get
0000000000031680 t shape
0000000000031c20 t shape_run
000000000001cdb0 t sharp_get
0000000000087c30 t shell_prompt_marking
0000000000013f40 t shutdown_monitor.lto_priv.0
0000000000014430 t sig_queue.lto_priv.0
0000000000023a40 t size_on_disk
0000000000096240 t sort_ranges.isra.0
000000000001a5a0 t spawn.lto_priv.0
00000000000730e0 t sprite_at.lto_priv.0
0000000000030240 t sprite_map_set_layout.lto_priv.0
000000000009cb40 t sprite_map_set_limits.lto_priv.0
0000000000013370 t start.lto_priv.0
0000000000095fc0 t start_selection.lto_priv.0
000000000001f340 t strikethrough_get
000000000001f5f0 t strikethrough_set
0000000000058e60 t strip_csi
00000000000b42f0 t systemd_move_pid_into_new_scope.lto_priv.0
0000000000092ff0 t tab.lto_priv.0
0000000000019480 t talk_loop
00000000000110cf t talk_loop.cold
000000000009cbe0 t test_commit_write_buffer.lto_priv.0
000000000009c9f0 t test_create_write_buffer.lto_priv.0
0000000000074960 t test_encode_mouse.lto_priv.0
00000000000a2d60 t test_find_either_of_two_bytes.lto_priv.0
000000000009cdc0 t test_parse_written_data.lto_priv.0
000000000003ad80 t test_render_line.lto_priv.0
000000000003ccf0 t test_shape.lto_priv.0
0000000000033150 t test_sprite_position_for.lto_priv.0
00000000000a2ab0 t test_utf8_decode_to_sentinel.lto_priv.0
00000000000a2f90 t test_xor64.lto_priv.0
0000000000072fd0 t text_at.lto_priv.0
0000000000097810 t text_for_marked_url.lto_priv.0
0000000000097770 t text_for_selection.lto_priv.0
0000000000096df0 t text_for_selections
00000000000144c0 t thread_write
0000000000011040 t thread_write.cold
0000000000079bb0 t timed_debug_print
0000000000097a60 t toggle_alt_screen.lto_priv.0
0000000000056e60 t toggle_fullscreen
000000000005b650 t toggle_fullscreen_for_os_window.part.0.lto_priv.0
0000000000057450 t toggle_maximized
0000000000056700 t toggle_secure_input
00000000000bba40 t unicode_database_version
0000000000076c50 t unicode_in_range.constprop.0
0000000000074ca0 t unload.lto_priv.0
000000000001b2f0 t update_ansi_color_table.lto_priv.0
000000000005fe20 t update_current_frame.lto_priv.0
000000000005e720 t update_dest_rect.lto_priv.0
000000000006f530 t update_ime_position
00000000000b5590 t update_ime_position_for_window
00000000000a36c0 t update_only_line_graphics_data.lto_priv.0
00000000000b5a00 t update_os_window_title.part.0
00000000000537a0 t update_os_window_viewport
000000000009bca0 t update_selection.lto_priv.0
0000000000076700 t url_end_at.lto_priv.0
0000000000074f50 t url_start_at.lto_priv.0
000000000009c960 t utf8_decode_to_esc
00000000000a01c0 t utf8_decode_to_esc_128
00000000000a1790 t utf8_decode_to_esc_256
000000000009ffc0 t utf8_decode_to_esc_scalar
000000000009c9a0 t utf8_decoder_ensure_capacity.part.0.isra.0
000000000009c970 t utf8_decoder_ensure_capacity.part.0.isra.1
000000000001c410 t visual_bell_color_get.lto_priv.0
000000000001b670 t visual_bell_color_set.lto_priv.0
000000000008ba60 t visual_line.lto_priv.0
00000000000895a0 t visual_line_.lto_priv.0
000000000002c3c0 t wait_for_write
0000000000013840 t wakeup.lto_priv.0
0000000000056d30 t wayland_compositor_data
00000000000569f0 t wayland_frame_request_callback.lto_priv.0
00000000000c43d0 t wcswidth_std
0000000000024f70 t wcwidth_std.lto_priv.0.lto_priv.0
0000000000024f70 t wcwidth_std.lto_priv.1.lto_priv.0
0000000000024f70 t wcwidth_std.lto_priv.2.lto_priv.0
0000000000024f70 t wcwidth_std.lto_priv.3.lto_priv.0
000000000002a070 t wcwidth_wrap
0000000000072d10 t width.lto_priv.0
0000000000049ca0 t window_close_callback
00000000000531a0 t window_focus_callback
000000000007baf0 t window_for_event.lto_priv.0
000000000007a780 t window_for_id.lto_priv.0
0000000000049e70 t window_iconify_callback
000000000004aa90 t window_occlusion_callback
0000000000049c80 t window_pos_callback
00000000000af280 t wrap_region
0000000000020320 t wrapped_kittens
0000000000057a60 t write_clipboard_data
0000000000082f00 t write_escape_code_to_child
0000000000022120 t write_loop
000000000001111e t write_loop.cold
0000000000072bc0 t write_mark
000000000001a3d0 t write_to_stderr
000000000009ee20 t write_unicode_ch
0000000000058010 t x11_display
0000000000058060 t x11_window_id
000000000009c7e0 t xor_data64
000000000009f870 t xor_data64_128
00000000000a1070 t xor_data64_256
000000000009c7a0 t xor_data64_scalar.lto_priv.0
00000000000952a0 t xxx_index.lto_priv.0
```
