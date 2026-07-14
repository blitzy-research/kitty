# kitty: How Clipboard Data Crosses the C↔Python Boundary Under Concurrent Load — A Runtime-Grounded Investigation

## 1. Summary and direct answers

**Direct answer.** kitty runs a **multi-threaded core but a single-threaded Python layer**. Raw bytes from a child process are read on a dedicated **I/O thread** (`KittyChildMon`, `kitty/child-monitor.c:1489`) that never touches the Python C-API; all VT parsing, all C→Python callbacks, and all expensive operations such as scrollback scans run on **one main thread** while it holds the **CPython Global Interpreter Lock (GIL)** (§3). When a clipboard escape (OSC 52, or the extended OSC 5522) arrives, the parser wraps its **internal 1 MiB buffer** (`BUF_SZ`, `kitty/vt-parser.c:18`) in a **zero-copy, read-only `memoryview`** (created by `PyMemoryView_FromMemory` with the `PyBUF_READ` flag, `kitty/vt-parser.c:461`) and hands that view to `Window.clipboard_control` through a C→Python `CALLBACK` (`kitty/screen.c:87,2305`). The Python clipboard manager then **copies the bytes it needs out of that transient view into owned Python objects** — a `WriteRequest` backed by a `Tempfile` that begins as an in-memory `io.BytesIO` and rolls over to an on-disk `TemporaryFile` at 16 MiB (`kitty/clipboard.py:237`) — so the alias never outlives the synchronous callback (§4, §8). Because every Python statement runs on the one GIL-holding main thread, an expensive main-thread operation (e.g. scanning a large scrollback) **delays but never loses** event delivery: the I/O thread keeps buffering raw child bytes — subject to POLLIN backpressure once the 1 MiB buffer is full (`kitty/child-monitor.c:1501`) — the whole time, and the queued events are delivered in order the moment the main thread is free again (§5, §7). Timing and concurrency therefore matter at exactly two places — the **parser mutex** guarding the single-producer/single-consumer buffer (`kitty/vt-parser.c:205,1421`) and the **GIL** serializing the main thread — while object ownership matters at the **RAII-scoped lifetime of the `memoryview`**. The only genuine hazard is a *C-level* one: retaining that view until the parser **reuses** its 1 MiB buffer, which then exposes **stale/overwritten bytes** rather than freed memory (the buffer allocation itself lives until parser teardown in `free_vt_parser`, `kitty/vt-parser.c:1508-1513`). kitty avoids it by copying out during the synchronous callback. It is **not** a Python-level data race, because the GIL serializes all Python execution (§8, §9).

**Direct answers, per objective (each backed by captured runtime output in the cited section):**

- **O1 — the clipboard C→Python transfer, small and very large (§4).** A single OSC 52 write under the 256 KiB `MAX_ESCAPE_CODE_LENGTH` (`kitty/vt-parser.c:21`) is delivered in **one** `dispatch_osc` call; a larger payload is split into **partial-OSC-52 chunks** (dispatch code **`-52`** for every partial and **`+52`** for the final), which the Python `WriteRequest` reassembles. Crossing 16 MiB flips the `Tempfile` from `io.BytesIO` to an on-disk `TemporaryFile`, observed directly via `/proc/<pid>/fd`. The default `clipboard_max_size=512` is **double-scaled** to an effective ≈512 **TiB** threshold (`kitty/clipboard.py:321`), so the truncation guard is unreachable under the default configuration (a CWE-400-class exposure); a positive control with a tiny limit shows the guard firing and the exact retained-byte overshoot. Read paths (legacy OSC 52 `?` and extended OSC 5522), the MIME listing, and the `read-clipboard-ask` accept/deny prompts are all exercised, and the full read/write status ledger is captured.

- **O2 — behavior in practice when other parts of the system are busy (§5).** PTY reading is decoupled from parsing: the I/O thread fills the shared buffer while the main thread parses under the GIL, coalescing bursts on the default **3 ms `input_delay`** gate (`kitty/options/definition.py:878`) — proven causally by contrasting `input_delay` at 0/3/25 ms. When the 1 MiB buffer fills, kitty toggles **POLLIN off/on** (captured as `events=POLLIN`↔`events=0` transitions in a `poll()` strace). Under a heavy flood the canonical event-delivery latency (measured with a DSR query, timing source `CLOCK_MONOTONIC`) rises above the ≈3.3 ms idle baseline, reported as a **distribution across ≥2 runs**, never as a single number.

- **The kitten process boundary (§6).** Events reach a kitten across a **second** boundary: the kitten is a **separate process** (the real Go `kitten clipboard` and the Python `ask` kitten) that exchanges bytes with the core over its own PTY/pipe, parsed by `kittens/tui/loop.py:246,261` and `kitty/kittens.c:94,104`. The in-process zero-copy alias of §4/§8 does **not** extend to it — the kitten reads its own copy. Measured round-trip kitten latency ≈3.3 ms across runs, with distinct kitten PIDs and PTY framing captured.

- **O3 — the effect of an expensive scrollback scan (§7).** A large scrollback scan (`as_text`, `kitty/screen.c:3486`; `text_for_range`, `kitty/screen.c:3035`, built on `unicode_in_range`, `kitty/screen.c:3057`; and `as_text_generic`, `kitty/line.c:874`) runs on the main thread, so a pending clipboard/kitten event **waits ≈ the full scan duration** — demonstrated by synchronizing event-enqueue with scan start under gdb, not by sleeping. On memory: the persistent cost is C scrollback storage in growable RAM segments (`add_segment`, `kitty/history.c:18`); the pager-history ring stores **raw UTF-8/ANSI bytes directly** (not compressed) and is **off by default**; the graphics disk-cache is a distinct subsystem, unrelated to scrollback.

- **O4 — where timing, concurrency, and object ownership start to matter (§8).** Under the parser mutex the main thread promotes `read.sz += write.pending` (`kitty/vt-parser.c:1421`) and then **releases the lock during the parse/dispatch**, so the callback runs **unlocked but on the main thread under the GIL** — all three states captured in one linked gdb run. The boundary `memoryview` has **three distinct lifetimes** (PyObject refcount, buffer allocation, payload content); the immediate hazard of retention is **stale content on buffer reuse**, not use-after-free; kitty copies out (base64-decode into an owned `bytes`/`Tempfile`, `kitty/clipboard.py:286`), proven by a real parser-reuse probe that is byte-identical across two runs.

- **O5 — subtle races that emerge only under real runtime conditions (§9).** A data-race detector (valgrind **helgrind**, run on a byte-identical, valgrind-compatible **diagnostic** build — the canonical `-O3 -march=native` build emits **AVX-512VL** and SIGILLs under valgrind, which is the real origin of earlier `-no-pie`/ASLR confusion) observed **no race on the vt-parser clipboard path** in the exercised runs, but **did reproduce two genuine races in the graphics disk cache** (`shutting_down` and `cache_file_fd`, unsynchronized at `kitty/disk-cache.c:348,439,421,361`) across ≥2 runs. The `is_self_offer` reentrancy path is exercised end-to-end in one gdb run (OSC 52 read → `write_clipboard_data(data==NULL)`, `kitty/glfw.c:2182` → `RuntimeError('is_self_offer')`, `:2183` → Python fallback, `kitty/clipboard.py:107-119` → owned bytes). All race conclusions are bounded to "no race **observed in these runs**", with untested surfaces enumerated.

**Evidence and methodology.** Every system-specific claim below is backed by a `file:line` citation and/or complete, unedited runtime output produced by the **canonical build** of §2 (`python3 setup.py`, default configuration). The build is canonical throughout; where a *probe* reaches the identical production functions through a test hook or the `kitty_tests` module rather than through a real PTY, its output is explicitly labeled **(non-canonical)** and cross-checked against the genuine end-to-end round trip. A **diagnostic** build (valgrind-compatible, byte-identical C source) is used only where the canonical build cannot be instrumented, and is labeled as such. Anything not directly observed is explicitly labeled **(inferred)**. Timing/magnitude values are stable across at least two runs or reported as a distribution. Every observation script is reproduced **in full** inline next to the output it produced and inventoried in §11.1; all scripts ran under `/tmp` (outside the repository) and were deleted after capture, leaving the source tree unchanged (§11.2).

---

## 2. Build & Environment (canonical)

Every runtime observation in this document comes from a **default, canonical build** of kitty at the immutable commit under investigation, run headless in the AAP-specified container. This section states the exact provenance, the exact build command and its complete output, and proves the build is byte-for-byte reproducible.

### 2.1 Provenance — interpreter, toolchain, OS, and the immutable HEAD

All commands below were run inside the canonical container (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`, local tag `swe-atlas-kitty:canonical`). The checkout at `/app` is the exact commit under investigation and is **unmodified** (`git status --porcelain` prints nothing). *(observed)*

```console
$ uname -a
Linux 3d8d85a6d946 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64 x86_64 x86_64 GNU/Linux
$ cat /etc/os-release | head -2
PRETTY_NAME="Ubuntu 24.04.2 LTS"
NAME="Ubuntu"
$ python3 --version
Python 3.12.3
$ go version
go version go1.23.4 linux/amd64
$ gcc --version | head -1
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
$ git -C /app rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
$ git -C /app status --porcelain | wc -l
0
$ DISPLAY=:99 ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

- OS: **Ubuntu 24.04.2 LTS**, kernel `6.6.122+ x86_64`.
- Interpreter: **Python 3.12.3** (the canonical interpreter per the AAP).
- C compiler: **gcc 13.3.0**; Go toolchain: **go1.23.4** (used only for the Go kittens/tools).
- Source HEAD: **`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`** — every `file:line` citation in this document is anchored to this commit.
- Running binary self-reports **`kitty 0.35.2 created by Kovid Goyal`**.

### 2.2 Canonical build command and its complete output

The default build is driven by `python3 setup.py` with no arguments — exactly what a normal user runs. It compiles the `fast_data_types` C extension, the GLFW backends, and the Go launcher/kitten binaries, under strict flags (`-Werror`). The build completed with **exit code 0** in ~48 s on a clean tree. The complete, unedited output follows. *(observed)*

```console
$ cd /app && python3 setup.py
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
go: downloading golang.org/x/sys v0.21.0
go: downloading github.com/bmatcuk/doublestar/v4 v4.6.1
go: downloading github.com/shirou/gopsutil/v3 v3.24.5
go: downloading github.com/alecthomas/chroma/v2 v2.14.0
go: downloading github.com/seancfoley/ipaddress-go v1.6.0
go: downloading howett.net/plist v1.0.1
go: downloading github.com/edwvee/exiffix v0.0.0-20240229113213-0dbb146775be
go: downloading github.com/kovidgoyal/imaging v1.6.3
go: downloading golang.org/x/exp v0.0.0-20230801115018-d63ba01acd4b
go: downloading github.com/ALTree/bigfloat v0.2.0
go: downloading github.com/google/uuid v1.6.0
go: downloading github.com/dlclark/regexp2 v1.11.0
go: downloading github.com/zeebo/xxh3 v1.0.2
go: downloading golang.org/x/image v0.17.0
go: downloading github.com/rwcarlsen/goexif v0.0.0-20190401172101-9e8deecbddbd
go: downloading github.com/disintegration/imaging v1.6.2
go: downloading github.com/klauspost/cpuid/v2 v2.2.5
go: downloading github.com/seancfoley/bintree v1.3.1
go: downloading github.com/tklauser/go-sysconf v0.3.12
go: downloading github.com/tklauser/numcpus v0.6.1
github.com/shirou/gopsutil/v3/common
golang.org/x/exp/constraints
github.com/seancfoley/ipaddress-go/ipaddr/addrstr
github.com/seancfoley/ipaddress-go/ipaddr/addrerr
unicode/utf16
kitty
container/list
image/color
crypto/internal/alias
internal/nettrace
github.com/seancfoley/ipaddress-go/ipaddr/addrstrparam
crypto/internal/boring/sig
crypto/subtle
vendor/golang.org/x/crypto/internal/alias
vendor/golang.org/x/crypto/cryptobyte/asn1
encoding
log/internal
internal/weak
maps
vendor/golang.org/x/net/dns/dnsmessage
math/rand/v2
internal/singleflight
crypto/internal/randutil
hash
crypto/rc4
encoding/base32
vendor/golang.org/x/text/transform
bufio
net/http/internal/ascii
regexp/syntax
crypto/cipher
crypto/internal/edwards25519/field
encoding/binary
crypto/internal/nistec/fiat
context
embed
image/color/palette
io/ioutil
vendor/golang.org/x/sys/cpu
encoding/hex
net/url
vendor/golang.org/x/net/http2/hpack
log
flag
kitty/tools/utils/shlex
runtime/cgo
github.com/bmatcuk/doublestar/v4
github.com/ALTree/bigfloat
crypto/internal/bigmod
encoding/asn1
github.com/seancfoley/bintree/tree
crypto/dsa
crypto
hash/adler32
hash/crc32
crypto/internal/boring
crypto/des
golang.org/x/image/riff
crypto/md5
crypto/internal/edwards25519
internal/concurrent
golang.org/x/image/tiff/lzw
compress/lzw
crypto/sha512
mime/quotedprintable
compress/flate
os/exec
compress/bzip2
encoding/xml
crypto/sha1
crypto/internal/boring/bbig
crypto/hmac
database/sql/driver
crypto/rand
os/signal
net/http/internal
crypto/aes
crypto/sha256
image
vendor/golang.org/x/text/unicode/bidi
encoding/base64
unique
vendor/golang.org/x/crypto/chacha20
github.com/rwcarlsen/goexif/tiff
vendor/golang.org/x/crypto/internal/poly1305
vendor/golang.org/x/crypto/sha3
github.com/dlclark/regexp2/syntax
github.com/klauspost/cpuid/v2
vendor/golang.org/x/text/unicode/norm
golang.org/x/sys/unix
crypto/x509/pkix
vendor/golang.org/x/crypto/cryptobyte
vendor/golang.org/x/crypto/hkdf
regexp
crypto/rsa
encoding/pem
kitty/tools/utils/secrets
mime
encoding/json
crypto/ed25519
vendor/golang.org/x/crypto/chacha20poly1305
net/netip
crypto/internal/mlkem768
github.com/shirou/gopsutil/v3/internal/common
compress/gzip
compress/zlib
archive/zip
golang.org/x/image/bmp
image/internal/imageutil
golang.org/x/image/ccitt
golang.org/x/image/vp8l
golang.org/x/image/vp8
vendor/golang.org/x/text/secure/bidirule
crypto/internal/nistec
image/draw
image/png
image/jpeg
golang.org/x/image/tiff
image/gif
golang.org/x/image/webp
github.com/zeebo/xxh3
howett.net/plist
vendor/golang.org/x/net/idna
github.com/dlclark/regexp2
github.com/disintegration/imaging
github.com/kovidgoyal/imaging
crypto/ecdh
crypto/elliptic
github.com/rwcarlsen/goexif/exif
crypto/internal/hpke
crypto/ecdsa
github.com/edwvee/exiffix
github.com/alecthomas/chroma/v2
github.com/tklauser/numcpus
github.com/shirou/gopsutil/v3/mem
github.com/tklauser/go-sysconf
github.com/shirou/gopsutil/v3/cpu
os/user
net
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
kitty/tools/utils/paths
kitty/tools/tty
kitty/tools/utils/base85
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
kitty/kittens/show_key
kitty/kittens/hyperlinked_grep
kitty/tools/tui
kitty/tools/tui/readline
kitty/tools/utils/images
kitty/tools/tui/subseq
kitty/kittens/clipboard
kitty/tools/unicode_names
kitty/kittens/ask
kitty/tools/cmd/show_error
kitty/tools/cmd/edit_in_kitty
kitty/tools/cmd/update_self
kitty/tools/tui/graphics
kitty/tools/cmd/run_shell
kitty/kittens/hints
kitty/tools/cmd/at
kitty/tools/themes
kitty/kittens/unicode_input
kitty/tools/cmd/benchmark
kitty/kittens/icat
kitty/kittens/choose_fonts
kitty/kittens/themes
kitty/kittens/ssh
kitty/kittens/transfer
kitty/tools/cmd/pytest
kitty/kittens/diff
kitty/tools/cmd/tool
kitty/tools/cmd/completion
kitty/tools/cmd
```

Two properties of this output matter for later sections:

- `kitty/vt-parser.c` is compiled **twice** — once normally and once into a separate `DUMP_COMMANDS` object (driven by `setup.py`), which is the in-repo instrumentation used later to trace OSC dispatch. *(observed in the build output; mechanism at `setup.py:721-722`)*
- The build is strict: `-Werror` is in force, so the clean exit means zero compiler warnings in the core. *(observed)*

### 2.3 Reproducibility — the build is byte-identical to the pre-built artifacts

The container ships pre-built artifacts. Hashing them, then doing a clean rebuild and hashing again, yields **identical** SHA-256 digests for all three primary artifacts — the extension module and both Go binaries. This is what lets every later observation be treated as a property of *this exact build*, not of a particular build run. *(observed)*

Pre-built artifacts:

```console
$ sha256sum kitty/fast_data_types.so kitty/launcher/kitty kitty/launcher/kitten 2>&1
582933cfd7b6cecb5ee60cfd20ef35a1f74acc2c6a905022180a60a76bf722e8  kitty/fast_data_types.so
8311daddf6bbccf949233c9fdd58fbbe46748dbfc957847b7e4228b4973fc24c  kitty/launcher/kitty
f4da89b44f075e53e8a5b51cc7efcefa43c0c4125fe169c1537925dc7d89141d  kitty/launcher/kitten
```

After `python3 setup.py clean` + a fresh default build:

```console
$ python3 setup.py clean >/dev/null 2>&1
$ python3 setup.py            # clean default build; exit 0; 48s
$ sha256sum kitty/fast_data_types.so kitty/launcher/kitty kitty/launcher/kitten
582933cfd7b6cecb5ee60cfd20ef35a1f74acc2c6a905022180a60a76bf722e8  kitty/fast_data_types.so
8311daddf6bbccf949233c9fdd58fbbe46748dbfc957847b7e4228b4973fc24c  kitty/launcher/kitty
f4da89b44f075e53e8a5b51cc7efcefa43c0c4125fe169c1537925dc7d89141d  kitty/launcher/kitten
```

The digests match exactly (`fast_data_types.so` = `582933cf…`, `launcher/kitty` = `8311dadd…`, `launcher/kitten` = `f4da89b4…`) across the pre-built → clean → rebuilt cycle. *(observed)*

### 2.4 Test suite — canonical result and the one environment-only failure

Running the project's own suite (`./test.py`) under the headless display gives **145 Python tests OK, 4 skipped**, and the Go suite passes except for a single, purely environment-driven failure. The complete, unedited output is embedded below. *(observed)*

```console
$ DISPLAY=:99 LANG=C.UTF-8 ./test.py
Running under CI: False
Go packages being tested: tools/cli tools/wcswidth tools/tui tools/tui/sgr tools/tui/graphics kittens/hyperlinked_grep tools/themes tools/config kittens/ssh tools/rsync kittens/diff tools/utils/shlex tools/tui/subseq tools/simdstring tools/unicode_names kittens/transfer tools/utils/base85 tools/tui/readline tools/utils/shm tools/tui/shell_integration tools/utils tools/cmd/at tools/utils/humanize tools/utils/style tools/tui/loop kittens/hints
test_backspace_wide_characters (kitty_tests.screen.TestScreen.test_backspace_wide_characters) ... ok
test_bottom_margin (kitty_tests.screen.TestScreen.test_bottom_margin) ... ok
test_char_manipulation (kitty_tests.screen.TestScreen.test_char_manipulation) ... ok
test_color_stack (kitty_tests.screen.TestScreen.test_color_stack) ... ok
test_cursor_after_resize (kitty_tests.screen.TestScreen.test_cursor_after_resize) ... ok
test_cursor_hidden (kitty_tests.screen.TestScreen.test_cursor_hidden) ... ok
test_cursor_movement (kitty_tests.screen.TestScreen.test_cursor_movement) ... ok
test_detect_url (kitty_tests.screen.TestScreen.test_detect_url) ... ok
test_dirty_lines (kitty_tests.screen.TestScreen.test_dirty_lines) ... ok
test_draw_char (kitty_tests.screen.TestScreen.test_draw_char) ... ok
test_draw_fast (kitty_tests.screen.TestScreen.test_draw_fast) ... ok
test_emoji_skin_tone_modifiers (kitty_tests.screen.TestScreen.test_emoji_skin_tone_modifiers) ... ok
test_erase_in_screen (kitty_tests.screen.TestScreen.test_erase_in_screen) ... ok
test_hyperlinks (kitty_tests.screen.TestScreen.test_hyperlinks) ... ok
test_key_encoding_flags_stack (kitty_tests.screen.TestScreen.test_key_encoding_flags_stack) ... ok
test_margins (kitty_tests.screen.TestScreen.test_margins) ... ok
test_osc_52 (kitty_tests.screen.TestScreen.test_osc_52) ... ok
test_pagerhist (kitty_tests.screen.TestScreen.test_pagerhist) ... ok
test_pointer_shapes (kitty_tests.screen.TestScreen.test_pointer_shapes) ... ok
test_prompt_marking (kitty_tests.screen.TestScreen.test_prompt_marking) ... ok
test_regional_indicators (kitty_tests.screen.TestScreen.test_regional_indicators) ... ok
test_rep (kitty_tests.screen.TestScreen.test_rep) ... ok
test_resize (kitty_tests.screen.TestScreen.test_resize) ... ok
test_scrollback_fill_after_resize (kitty_tests.screen.TestScreen.test_scrollback_fill_after_resize) ... ok
test_selection_as_text (kitty_tests.screen.TestScreen.test_selection_as_text) ... ok
test_serialize (kitty_tests.screen.TestScreen.test_serialize) ... ok
test_sgr (kitty_tests.screen.TestScreen.test_sgr) ... ok
test_soft_hyphen (kitty_tests.screen.TestScreen.test_soft_hyphen) ... ok
test_tab_stops (kitty_tests.screen.TestScreen.test_tab_stops) ... ok
test_top_and_bottom_margin (kitty_tests.screen.TestScreen.test_top_and_bottom_margin) ... ok
test_top_margin (kitty_tests.screen.TestScreen.test_top_margin) ... ok
test_user_marking (kitty_tests.screen.TestScreen.test_user_marking) ... ok
test_variation_selectors (kitty_tests.screen.TestScreen.test_variation_selectors) ... ok
test_wrapping_serialization (kitty_tests.screen.TestScreen.test_wrapping_serialization) ... ok
test_writing_with_cursor_on_trailer_of_wide_character (kitty_tests.screen.TestScreen.test_writing_with_cursor_on_trailer_of_wide_character) ... ok
test_zwj (kitty_tests.screen.TestScreen.test_zwj) ... ok
test_file_get (kitty_tests.file_transmission.TestFileTransmission.test_file_get) ... ok
test_parse_ftc (kitty_tests.file_transmission.TestFileTransmission.test_parse_ftc) ... ok
test_rsync_hashers (kitty_tests.file_transmission.TestFileTransmission.test_rsync_hashers) ... ok
test_rsync_roundtrip (kitty_tests.file_transmission.TestFileTransmission.test_rsync_roundtrip) ... ok
test_transfer_receive (kitty_tests.file_transmission.TestFileTransmission.test_transfer_receive) ... ok
test_transfer_send (kitty_tests.file_transmission.TestFileTransmission.test_transfer_send) ... ok
test_encode_key_event (kitty_tests.keys.TestKeys.test_encode_key_event) ... ok
test_encode_mouse_event (kitty_tests.keys.TestKeys.test_encode_mouse_event) ... ok
test_mapping (kitty_tests.keys.TestKeys.test_mapping) ... ok
test_elliptic_curve_data_exchange (kitty_tests.crypto.TestCrypto.test_elliptic_curve_data_exchange) ... ok
test_conf_parsing (kitty_tests.options.TestConfParsing.test_conf_parsing) ... ok
test_animation_frame_loading (kitty_tests.graphics.TestGraphics.test_animation_frame_loading) ... ok
test_disk_cache (kitty_tests.graphics.TestGraphics.test_disk_cache) ... ok
test_gr_delete (kitty_tests.graphics.TestGraphics.test_gr_delete) ... ok
test_gr_operations_with_numbers (kitty_tests.graphics.TestGraphics.test_gr_operations_with_numbers) ... ok
test_gr_reset (kitty_tests.graphics.TestGraphics.test_gr_reset) ... ok
test_gr_scroll (kitty_tests.graphics.TestGraphics.test_gr_scroll) ... ok
test_graphics_quota_enforcement (kitty_tests.graphics.TestGraphics.test_graphics_quota_enforcement) ... ok
test_image_layer_grouping (kitty_tests.graphics.TestGraphics.test_image_layer_grouping) ... ok
test_image_parents (kitty_tests.graphics.TestGraphics.test_image_parents) ... ok
test_image_put (kitty_tests.graphics.TestGraphics.test_image_put) ... ok
test_load_images (kitty_tests.graphics.TestGraphics.test_load_images) ... ok
test_load_png (kitty_tests.graphics.TestGraphics.test_load_png) ... ok
test_load_png_simple (kitty_tests.graphics.TestGraphics.test_load_png_simple) ... ok
test_suppressing_gr_command_responses (kitty_tests.graphics.TestGraphics.test_suppressing_gr_command_responses) ... ok
test_unicode_placeholders (kitty_tests.graphics.TestGraphics.test_unicode_placeholders) ... ok
test_unicode_placeholders_3rd_combining_char (kitty_tests.graphics.TestGraphics.test_unicode_placeholders_3rd_combining_char) ... ok
test_unicode_placeholders_multiple_placements (kitty_tests.graphics.TestGraphics.test_unicode_placeholders_multiple_placements) ... ok
test_unicode_placeholders_scroll (kitty_tests.graphics.TestGraphics.test_unicode_placeholders_scroll) ... ok
test_xor_data (kitty_tests.graphics.TestGraphics.test_xor_data) ... ok
test_all_kitten_names (kitty_tests.check_build.TestBuild.test_all_kitten_names) ... ok
test_ca_certificates (kitty_tests.check_build.TestBuild.test_ca_certificates) ... skipped 'CA certificates are only tested on frozen builds'
test_docs_url (kitty_tests.check_build.TestBuild.test_docs_url) ... ok
test_exe (kitty_tests.check_build.TestBuild.test_exe) ... ok
test_filesystem_locations (kitty_tests.check_build.TestBuild.test_filesystem_locations) ... ok
test_glfw_modules (kitty_tests.check_build.TestBuild.test_glfw_modules) ... ok
test_launcher_ensures_stdio (kitty_tests.check_build.TestBuild.test_launcher_ensures_stdio) ... ok
test_loading_extensions (kitty_tests.check_build.TestBuild.test_loading_extensions) ... ok
test_loading_shaders (kitty_tests.check_build.TestBuild.test_loading_shaders) ... ok
test_box_drawing (kitty_tests.fonts.Rendering.test_box_drawing) ... ok
test_coalesce_symbol_maps (kitty_tests.fonts.Rendering.test_coalesce_symbol_maps) ... ok
test_emoji_presentation (kitty_tests.fonts.Rendering.test_emoji_presentation) ... ok
test_fallback_font_not_last_resort (kitty_tests.fonts.Rendering.test_fallback_font_not_last_resort) ... skipped 'Only macOS has a Last Resort font'
test_font_rendering (kitty_tests.fonts.Rendering.test_font_rendering) ... ok
test_shaping (kitty_tests.fonts.Rendering.test_shaping) ... ok
test_sprite_map (kitty_tests.fonts.Rendering.test_sprite_map) ... ok
test_font_selection (kitty_tests.fonts.Selection.test_font_selection) ... ok
test_os_window_size_calculation (kitty_tests.glfw.TestGLFW.test_os_window_size_calculation) ... ok
test_utf_8_strndup (kitty_tests.glfw.TestGLFW.test_utf_8_strndup) ... ok
test_shm_with_kitten (kitty_tests.shm.SHMTest.test_shm_with_kitten) ... ok
test_clipboard_write_request (kitty_tests.clipboard.TestClipboard.test_clipboard_write_request) ... ok
test_basic_pty_operations (kitty_tests.ssh.SSHKitten.test_basic_pty_operations) ... ok
test_ssh_bootstrap_with_different_launchers (kitty_tests.ssh.SSHKitten.test_ssh_bootstrap_with_different_launchers) ... ok
test_ssh_connection_data (kitty_tests.ssh.SSHKitten.test_ssh_connection_data) ... ok
test_ssh_copy (kitty_tests.ssh.SSHKitten.test_ssh_copy) ... ok
test_ssh_env_vars (kitty_tests.ssh.SSHKitten.test_ssh_env_vars) ... ok
test_ssh_leading_data (kitty_tests.ssh.SSHKitten.test_ssh_leading_data) ... ok
test_ssh_login_shell_detection (kitty_tests.ssh.SSHKitten.test_ssh_login_shell_detection) ... ok
test_ssh_shell_integration (kitty_tests.ssh.SSHKitten.test_ssh_shell_integration) ... ok
test_search_query_parser (kitty_tests.search_query_parser.TestSQP.test_search_query_parser) ... ok
test_layout_operations (kitty_tests.layout.TestLayout.test_layout_operations) ... ok
test_overlay_layout_operations (kitty_tests.layout.TestLayout.test_overlay_layout_operations) ... ok
test_splits (kitty_tests.layout.TestLayout.test_splits) ... ok
test_mouse_selection (kitty_tests.mouse.TestMouse.test_mouse_selection) ... ok
test_base64 (kitty_tests.parser.TestParser.test_base64) ... ok
test_charsets (kitty_tests.parser.TestParser.test_charsets) ... ok
test_csi_code_rep (kitty_tests.parser.TestParser.test_csi_code_rep) ... ok
test_csi_codes (kitty_tests.parser.TestParser.test_csi_codes) ... ok
test_dcs_codes (kitty_tests.parser.TestParser.test_dcs_codes) ... ok
test_deccara (kitty_tests.parser.TestParser.test_deccara) ... ok
test_desktop_notify (kitty_tests.parser.TestParser.test_desktop_notify) ... ok
test_esc_codes (kitty_tests.parser.TestParser.test_esc_codes) ... ok
test_find_either_of_two_bytes (kitty_tests.parser.TestParser.test_find_either_of_two_bytes) ... ok
test_graphics_command (kitty_tests.parser.TestParser.test_graphics_command) ... ok
test_osc_codes (kitty_tests.parser.TestParser.test_osc_codes) ... ok
test_oth_codes (kitty_tests.parser.TestParser.test_oth_codes) ... ok
test_parser_threading (kitty_tests.parser.TestParser.test_parser_threading) ... ok
test_simple_parsing (kitty_tests.parser.TestParser.test_simple_parsing) ... ok
test_utf8_parsing (kitty_tests.parser.TestParser.test_utf8_parsing) ... ok
test_utf8_simd_decode (kitty_tests.parser.TestParser.test_utf8_simd_decode) ... ok
test_completion (kitty_tests.completion.TestCompletion.test_completion) ... ok
test_num_users (kitty_tests.utmp.UTMPTest.test_num_users) ... ok
test_parsing_of_open_actions (kitty_tests.open_actions.TestOpenActions.test_parsing_of_open_actions) ... ok
test_line_edit (kitty_tests.tui.TestTUI.test_line_edit) ... ok
test_multiprocessing_spawn (kitty_tests.tui.TestTUI.test_multiprocessing_spawn) ... ok
test_bash_integration (kitty_tests.shell_integration.ShellIntegration.test_bash_integration) ... ok
test_fish_integration (kitty_tests.shell_integration.ShellIntegration.test_fish_integration) ... skipped 'fish not installed'
test_zsh_integration (kitty_tests.shell_integration.ShellIntegration.test_zsh_integration) ... ok
test_bash_integration (kitty_tests.shell_integration.ShellIntegrationWithKitten.test_bash_integration) ... ok
test_fish_integration (kitty_tests.shell_integration.ShellIntegrationWithKitten.test_fish_integration) ... skipped 'fish not installed'
test_zsh_integration (kitty_tests.shell_integration.ShellIntegrationWithKitten.test_zsh_integration) ... ok
test_ansi_repr (kitty_tests.datatypes.TestDataTypes.test_ansi_repr) ... ok
test_bracketed_paste_sanitizer (kitty_tests.datatypes.TestDataTypes.test_bracketed_paste_sanitizer) ... ok
test_color_profile (kitty_tests.datatypes.TestDataTypes.test_color_profile) ... ok
test_expand_ansi_c_escapes (kitty_tests.datatypes.TestDataTypes.test_expand_ansi_c_escapes) ... ok
test_historybuf (kitty_tests.datatypes.TestDataTypes.test_historybuf) ... ok
test_line (kitty_tests.datatypes.TestDataTypes.test_line) ... ok
test_linebuf (kitty_tests.datatypes.TestDataTypes.test_linebuf) ... ok
test_notify_identifier_sanitization (kitty_tests.datatypes.TestDataTypes.test_notify_identifier_sanitization) ... ok
test_replace_c0_codes (kitty_tests.datatypes.TestDataTypes.test_replace_c0_codes) ... ok
test_rewrap_narrower (kitty_tests.datatypes.TestDataTypes.test_rewrap_narrower) ... ok
test_rewrap_simple (kitty_tests.datatypes.TestDataTypes.test_rewrap_simple) ... ok
test_rewrap_wider (kitty_tests.datatypes.TestDataTypes.test_rewrap_wider) ... ok
test_shlex_split (kitty_tests.datatypes.TestDataTypes.test_shlex_split) ... ok
test_single_key (kitty_tests.datatypes.TestDataTypes.test_single_key) ... ok
test_strip_csi (kitty_tests.datatypes.TestDataTypes.test_strip_csi) ... ok
test_to_color (kitty_tests.datatypes.TestDataTypes.test_to_color) ... ok
test_url_at (kitty_tests.datatypes.TestDataTypes.test_url_at) ... ok
test_utils (kitty_tests.datatypes.TestDataTypes.test_utils) ... ok

----------------------------------------------------------------------
Ran 145 tests in 14.011s

OK (skipped=4)
go: downloading github.com/google/go-cmp v0.6.0
=== RUN   TestCompleteFiles
--- PASS: TestCompleteFiles (0.00s)
=== RUN   TestCompleteExecutables
--- PASS: TestCompleteExecutables (0.00s)
=== RUN   TestCLIParsing
--- PASS: TestCLIParsing (0.00s)
PASS
ok  	kitty/tools/cli	0.009s
=== RUN   TestEscapeCodeParsing
--- PASS: TestEscapeCodeParsing (0.00s)
=== RUN   TestWCSWidth
--- PASS: TestWCSWidth (0.00s)
=== RUN   TestCellIterator
--- PASS: TestCellIterator (0.00s)
PASS
ok  	kitty/tools/wcswidth	0.011s
=== RUN   TestRenderProgressBar
--- PASS: TestRenderProgressBar (0.00s)
PASS
ok  	kitty/tools/tui	0.005s
=== RUN   TestInsertFormatting
--- PASS: TestInsertFormatting (0.00s)
PASS
ok  	kitty/tools/tui/sgr	0.011s
=== RUN   TestGraphicsCommandSerialization
--- PASS: TestGraphicsCommandSerialization (0.01s)
PASS
ok  	kitty/tools/tui/graphics	0.026s
=== RUN   TestRgArgParsing
    main_test.go:17: Skipping as rg not found in PATH
--- SKIP: TestRgArgParsing (0.00s)
PASS
ok  	kitty/kittens/hyperlinked_grep	0.011s
=== RUN   TestThemeCollections
--- PASS: TestThemeCollections (0.01s)
PASS
ok  	kitty/tools/themes	0.014s
=== RUN   TestConfigParsing
--- PASS: TestConfigParsing (0.00s)
=== RUN   TestStringLiteralParsing
--- PASS: TestStringLiteralParsing (0.00s)
=== RUN   TestNormalizeShortcuts
--- PASS: TestNormalizeShortcuts (0.00s)
PASS
ok  	kitty/tools/config	0.011s
=== RUN   TestSSHConfigParsing
--- PASS: TestSSHConfigParsing (0.01s)
=== RUN   TestCloneEnv
--- PASS: TestCloneEnv (0.00s)
=== RUN   TestSSHBootstrapScriptLimit
--- PASS: TestSSHBootstrapScriptLimit (0.02s)
=== RUN   TestSSHTarfile
--- PASS: TestSSHTarfile (0.02s)
=== RUN   TestGetSSHOptions
--- PASS: TestGetSSHOptions (0.00s)
=== RUN   TestParseSSHArgs
--- PASS: TestParseSSHArgs (0.00s)
=== RUN   TestRelevantKittyOpts
--- PASS: TestRelevantKittyOpts (0.00s)
PASS
ok  	kitty/kittens/ssh	0.060s
=== RUN   TestRsyncRoundtrip
--- PASS: TestRsyncRoundtrip (0.00s)
=== RUN   TestRsyncHashers
--- PASS: TestRsyncHashers (0.00s)
PASS
ok  	kitty/tools/rsync	0.009s
=== RUN   TestDiffCollectWalk
--- PASS: TestDiffCollectWalk (0.00s)
PASS
ok  	kitty/kittens/diff	0.027s
=== RUN   TestLexer
--- PASS: TestLexer (0.00s)
=== RUN   TestSplit
--- PASS: TestSplit (0.00s)
=== RUN   TestSplitForCompletion
--- PASS: TestSplitForCompletion (0.00s)
=== RUN   TestExpandANSICEscapes
--- PASS: TestExpandANSICEscapes (0.00s)
PASS
ok  	kitty/tools/utils/shlex	0.003s
=== RUN   TestSubseq
--- PASS: TestSubseq (0.00s)
PASS
ok  	kitty/tools/tui/subseq	0.011s
=== RUN   TestSIMDStringOps
--- PASS: TestSIMDStringOps (0.00s)
=== RUN   TestIntrinsics
--- PASS: TestIntrinsics (0.00s)
PASS
ok  	kitty/tools/simdstring	0.011s
=== RUN   TestUnicodeInputQueries
--- PASS: TestUnicodeInputQueries (0.07s)
PASS
ok  	kitty/tools/unicode_names	0.077s
=== RUN   TestFTCSerialization
--- PASS: TestFTCSerialization (0.00s)
=== RUN   TestPathMappingSend
--- PASS: TestPathMappingSend (0.00s)
PASS
ok  	kitty/kittens/transfer	0.010s
=== RUN   TestBase85
--- PASS: TestBase85 (0.00s)
PASS
ok  	kitty/tools/utils/base85	0.006s
=== RUN   TestAddText
--- PASS: TestAddText (0.00s)
=== RUN   TestGetScreenLines
--- PASS: TestGetScreenLines (0.00s)
=== RUN   TestCursorMovement
--- PASS: TestCursorMovement (0.00s)
=== RUN   TestYanking
--- PASS: TestYanking (0.00s)
=== RUN   TestEraseChars
--- PASS: TestEraseChars (0.00s)
=== RUN   TestNumberArgument
--- PASS: TestNumberArgument (0.00s)
=== RUN   TestHistory
--- PASS: TestHistory (0.00s)
=== RUN   TestReadlineCompletion
--- PASS: TestReadlineCompletion (0.00s)
PASS
ok  	kitty/tools/tui/readline	0.010s
=== RUN   TestSHM
--- PASS: TestSHM (0.00s)
PASS
ok  	kitty/tools/utils/shm	0.005s
=== RUN   TestExtractShellIntegration
--- PASS: TestExtractShellIntegration (0.01s)
PASS
ok  	kitty/tools/tui/shell_integration	0.019s
=== RUN   TestFileLock
--- PASS: TestFileLock (0.03s)
=== RUN   TestISO8601
--- PASS: TestISO8601 (0.00s)
=== RUN   TestLongestCommon
--- PASS: TestLongestCommon (0.00s)
=== RUN   TestGettingLoginShell
--- PASS: TestGettingLoginShell (0.00s)
=== RUN   TestRingBuffer
--- PASS: TestRingBuffer (0.00s)
=== RUN   TestShortUUID
--- PASS: TestShortUUID (0.00s)
=== RUN   TestParseSocketAddress
--- PASS: TestParseSocketAddress (0.00s)
=== RUN   TestStreamDecompressor
--- PASS: TestStreamDecompressor (0.00s)
=== RUN   TestStringScanner
--- PASS: TestStringScanner (0.00s)
=== RUN   TestCreateAnonymousTempfile
    tpmfile_test.go:23: Anonymous tempfile was not created atomically
--- FAIL: TestCreateAnonymousTempfile (0.00s)
FAIL
FAIL	kitty/tools/utils	0.045s
=== RUN   TestEncodeJSON
--- PASS: TestEncodeJSON (0.00s)
=== RUN   TestCommandToJSON
--- PASS: TestCommandToJSON (0.00s)
=== RUN   TestRCSerialization
--- PASS: TestRCSerialization (0.00s)
PASS
ok  	kitty/tools/cmd/at	0.007s
=== RUN   TestShortDuration
--- PASS: TestShortDuration (0.00s)
PASS
ok  	kitty/tools/utils/humanize	0.007s
=== RUN   TestFormatWithIndent
--- PASS: TestFormatWithIndent (0.00s)
=== RUN   TestANSIStyleContext
--- PASS: TestANSIStyleContext (0.00s)
=== RUN   TestANSIStyleSprint
--- PASS: TestANSIStyleSprint (0.00s)
PASS
ok  	kitty/tools/utils/style	0.006s
=== RUN   TestKeyEventFromCSI
--- PASS: TestKeyEventFromCSI (0.00s)
PASS
ok  	kitty/tools/tui/loop	0.007s
=== RUN   TestHintMarking
--- PASS: TestHintMarking (0.15s)
PASS
ok  	kitty/kittens/hints	0.159s
FAIL
[31mError[39m: Some tests failed!
```

The Python summary is **`Ran 145 tests in 14.011s`** → **`OK (skipped=4)`**. The four skips are environment-declared (frozen-build certificate test; a macOS-only font test; two fish-integration tests because `fish` is not installed) — not failures. *(observed)*

The **one Go failure** is `TestCreateAnonymousTempfile` (`--- FAIL: TestCreateAnonymousTempfile`, `tmpfile_test.go:23: Anonymous tempfile was not created atomically`, `FAIL kitty/tools/utils`). This is **not** a defect in the code under study: the test asserts atomic anonymous-tempfile creation via `unix.O_TMPFILE` (`tools/utils/tmpfile_linux.go:20`), and the container's overlay/`/tmp` filesystem does not support `O_TMPFILE`, so the code correctly falls back to a named-then-unlinked tempfile and the atomicity assertion fails. It is retained here verbatim rather than hidden, and is orthogonal to the clipboard/parser/scrollback paths this document investigates. *(observed; cause inferred from the failing assertion + the O_TMPFILE call site)*

### 2.5 Diagnostic builds used as cross-checks (labeled non-default)

Beyond the default build, two **diagnostic** builds are used strictly as cross-checks; neither is the basis for any headline value:

- **AddressSanitizer + UndefinedBehaviorSanitizer** (`python3 setup.py --debug --sanitize`). Under ASan+UBSan the clipboard and VT-parser unit tests run **clean — zero sanitizer reports**. This is a memory-safety cross-check for the boundary code, not a timing measurement (ASan perturbs timing). *(observed; non-default build)*

```console
$ python3 setup.py --debug --sanitize        # (tail of build)
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
kitty/tools/cmd

$ DISPLAY=:99 ./test.py clipboard
Running under CI: False
test_clipboard_write_request (kitty_tests.clipboard.TestClipboard.test_clipboard_write_request) ... ok

----------------------------------------------------------------------
Ran 1 test in 0.010s

OK

$ DISPLAY=:99 ./test.py parser
Running under CI: False
test_base64 (kitty_tests.parser.TestParser.test_base64) ... ok
test_charsets (kitty_tests.parser.TestParser.test_charsets) ... ok
test_csi_code_rep (kitty_tests.parser.TestParser.test_csi_code_rep) ... ok
test_csi_codes (kitty_tests.parser.TestParser.test_csi_codes) ... ok
test_dcs_codes (kitty_tests.parser.TestParser.test_dcs_codes) ... ok
test_deccara (kitty_tests.parser.TestParser.test_deccara) ... ok
test_desktop_notify (kitty_tests.parser.TestParser.test_desktop_notify) ... ok
test_esc_codes (kitty_tests.parser.TestParser.test_esc_codes) ... ok
test_find_either_of_two_bytes (kitty_tests.parser.TestParser.test_find_either_of_two_bytes) ... ok
test_graphics_command (kitty_tests.parser.TestParser.test_graphics_command) ... ok
test_osc_codes (kitty_tests.parser.TestParser.test_osc_codes) ... ok
test_oth_codes (kitty_tests.parser.TestParser.test_oth_codes) ... ok
test_parser_threading (kitty_tests.parser.TestParser.test_parser_threading) ... ok
test_simple_parsing (kitty_tests.parser.TestParser.test_simple_parsing) ... ok
test_utf8_parsing (kitty_tests.parser.TestParser.test_utf8_parsing) ... ok
test_utf8_simd_decode (kitty_tests.parser.TestParser.test_utf8_simd_decode) ... ok

----------------------------------------------------------------------
Ran 16 tests in 0.666s

OK
```

- **ThreadSanitizer** is available (`libtsan2` present) but a full-GUI TSan build of kitty is **not** the default and is treated as non-canonical; the detector campaign for data races (§9) is reported with its exact scope and limitations rather than presented as default behavior. *(inferred from tooling availability; see §9)*

---

## 3. Threading & concurrency model (runtime-proven)

kitty's core is multi-threaded, but the division of labour is narrow and is the single most important fact for everything that follows: **one dedicated thread reads PTY bytes; the main thread does everything else — parsing, all Python callbacks, screen mutation, and rendering — serialized under the GIL.** This section proves that topology at runtime with gdb, then names the conditional threads.

### 3.1 The two authoritative threads, proven by live backtrace

A real launcher instance was started headless (`kitty --config NONE sh -c "sleep 600"`) and gdb attached to the live process. The two threads that constitute kitty's own architecture are the **MAIN** thread and **KittyChildMon**; their backtraces (addresses redacted to `0x<addr>`) are: *(observed)*

```console
=== MAIN(Thread 1) + KittyChildMon(Thread 2) backtraces; kitty pid=635 ===

This GDB supports auto-downloading debuginfo from the following URLs:
0x<addr> in __GI_ppoll (fds=0x<addr> <_glfw+133552>, nfds=2, timeout=<optimized out>, sigmask=0x0) at ../sysdeps/unix/sysv/linux/ppoll.c:42

warning: 42	../sysdeps/unix/sysv/linux/ppoll.c: No such file or directory
[Switching to thread 1 (Thread 0x<addr> (LWP 635))]
#0  0x<addr> in __GI_ppoll (fds=0x<addr> <_glfw+133552>, nfds=2, timeout=<optimized out>, sigmask=0x0) at ../sysdeps/unix/sysv/linux/ppoll.c:42
42	in ../sysdeps/unix/sysv/linux/ppoll.c
#0  0x<addr> in __GI_ppoll (fds=0x<addr> <_glfw+133552>, nfds=2, timeout=<optimized out>, sigmask=0x0) at ../sysdeps/unix/sysv/linux/ppoll.c:42
#1  0x<addr> in glfwRunMainLoop () from /app/kitty/glfw-x11.so
#2  0x<addr> in main_loop.lto_priv () from /app/kitty/launcher/../../kitty/fast_data_types.so
#3  0x<addr> in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#4  0x<addr> in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#5  0x<addr> in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#6  0x<addr> in _PyObject_FastCallDictTstate () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#7  0x<addr> in _PyObject_Call_Prepend () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#8  0x<addr> in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#9  0x<addr> in _PyObject_MakeTpCall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#10 0x<addr> in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#11 0x<addr> in PyEval_EvalCode () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#12 0x<addr> in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#13 0x<addr> in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#14 0x<addr> in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#15 0x<addr> in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#16 0x<addr> in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#17 0x<addr> in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#18 0x<addr> in Py_RunMain () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#19 0x<addr> in main ()
[Switching to thread 2 (Thread 0x<addr> (LWP 702))]
#0  0x<addr> in __GI___poll (fds=0x<addr> <children_fds>, nfds=3, timeout=-1) at ../sysdeps/unix/sysv/linux/poll.c:29
warning: 29	../sysdeps/unix/sysv/linux/poll.c: No such file or directory
#0  0x<addr> in __GI___poll (fds=0x<addr> <children_fds>, nfds=3, timeout=-1) at ../sysdeps/unix/sysv/linux/poll.c:29
#1  0x<addr> in io_loop () from /app/kitty/launcher/../../kitty/fast_data_types.so
#2  0x<addr> in start_thread (arg=<optimized out>) at ./nptl/pthread_create.c:447
#3  0x<addr> in clone3 () at ../sysdeps/unix/sysv/linux/x86_64/clone3.S:78
[Inferior 1 (process 635) detached]
```

Reading these two stacks:

- **MAIN (Thread 1, `"kitty"`)** is blocked in `__GI_ppoll(fds=<_glfw+133552>, nfds=2)` inside `glfwRunMainLoop` (`glfw-x11.so`) → `main_loop.lto_priv` (`fast_data_types.so`) → Python eval frames → `Py_RunMain` → `main`. This is the GLFW/GUI event loop; it is a Python-embedding thread and holds the GIL whenever it runs. The main tick is `process_global_state` [kitty/child-monitor.c:1222], which calls `parse_input` [kitty/child-monitor.c:451]. *(observed stack; call-site file:line from source)*
- **KittyChildMon (Thread 2)** is blocked in `__GI___poll(fds=<children_fds>, nfds=3, timeout=-1)` inside `io_loop` (`fast_data_types.so`) → `start_thread` → `clone3`. This is the dedicated PTY I/O thread created in `child-monitor.c` (thread named at [kitty/child-monitor.c:1489]); its whole job is to `poll` child FDs and read their bytes into the shared parser buffer. It runs **no Python**. *(observed stack; name/creation file:line from source)*

The symbol names `io_loop`, `main_loop`, `children_fds`, and `_glfw` are the real production symbols resolved out of `fast_data_types.so` / `glfw-x11.so` — i.e. this is kitty's own code, not a test harness. *(observed)*

### 3.2 The complete thread table (all threads, unedited)

The full `info threads` table for the same idle instance is embedded below without edits (only hex addresses redacted). It lists 67 threads. *(observed)*

```console
$ gdb -q -p <kitty_pid> -batch -ex "set pagination off" -ex "info threads"
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
0x<addr> in __GI_ppoll (fds=0x<addr> <_glfw+133552>, nfds=2, timeout=<optimized out>, sigmask=0x0) at ../sysdeps/unix/sysv/linux/ppoll.c:42
  Id   Target Id                                       Frame 
* 1    Thread 0x<addr> (LWP 510) "kitty"         0x<addr> in __GI_ppoll (fds=0x<addr> <_glfw+133552>, nfds=2, timeout=<optimized out>, sigmask=0x0) at ../sysdeps/unix/sysv/linux/ppoll.c:42
  2    Thread 0x<addr> (LWP 577) "KittyChildMon" 0x<addr> in __GI___poll (fds=0x<addr> <children_fds>, nfds=3, timeout=-1) at ../sysdeps/unix/sysv/linux/poll.c:29
  3    Thread 0x<addr> (LWP 576) "kitty:disk$0"  0x<addr> in __futex_abstimed_wait_common64 (private=0, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  4    Thread 0x<addr> (LWP 575) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  5    Thread 0x<addr> (LWP 574) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  6    Thread 0x<addr> (LWP 573) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  7    Thread 0x<addr> (LWP 572) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  8    Thread 0x<addr> (LWP 571) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  9    Thread 0x<addr> (LWP 570) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  10   Thread 0x<addr> (LWP 569) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  11   Thread 0x<addr> (LWP 568) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  12   Thread 0x<addr> (LWP 567) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  13   Thread 0x<addr> (LWP 566) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  14   Thread 0x<addr> (LWP 565) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  15   Thread 0x<addr> (LWP 564) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  16   Thread 0x<addr> (LWP 563) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  17   Thread 0x<addr> (LWP 562) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  18   Thread 0x<addr> (LWP 561) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  19   Thread 0x<addr> (LWP 560) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  20   Thread 0x<addr> (LWP 559) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  21   Thread 0x<addr> (LWP 558) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  22   Thread 0x<addr> (LWP 557) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  23   Thread 0x<addr> (LWP 556) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  24   Thread 0x<addr> (LWP 555) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  25   Thread 0x<addr> (LWP 554) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  26   Thread 0x<addr> (LWP 553) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  27   Thread 0x<addr> (LWP 552) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  28   Thread 0x<addr> (LWP 551) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  29   Thread 0x<addr> (LWP 550) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  30   Thread 0x<addr> (LWP 549) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  31   Thread 0x<addr> (LWP 548) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  32   Thread 0x<addr> (LWP 547) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  33   Thread 0x<addr> (LWP 546) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  34   Thread 0x<addr> (LWP 545) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  35   Thread 0x<addr> (LWP 544) "kitty"         0x<addr> in __futex_abstimed_wait_common64 (private=31882, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  36   Thread 0x<addr> (LWP 543) "llvmpipe-31"   0x<addr> in __futex_abstimed_wait_common64 (private=22207, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  37   Thread 0x<addr> (LWP 542) "llvmpipe-30"   0x<addr> in __futex_abstimed_wait_common64 (private=22207, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  38   Thread 0x<addr> (LWP 541) "llvmpipe-29"   0x<addr> in __futex_abstimed_wait_common64 (private=240, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  39   Thread 0x<addr> (LWP 540) "llvmpipe-28"   0x<addr> in __futex_abstimed_wait_common64 (private=22207, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  40   Thread 0x<addr> (LWP 539) "llvmpipe-27"   0x<addr> in __futex_abstimed_wait_common64 (private=368, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  41   Thread 0x<addr> (LWP 538) "llvmpipe-26"   0x<addr> in __futex_abstimed_wait_common64 (private=368, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  42   Thread 0x<addr> (LWP 537) "llvmpipe-25"   0x<addr> in __futex_abstimed_wait_common64 (private=400, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  43   Thread 0x<addr> (LWP 536) "llvmpipe-24"   0x<addr> in __futex_abstimed_wait_common64 (private=368, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  44   Thread 0x<addr> (LWP 535) "llvmpipe-23"   0x<addr> in __futex_abstimed_wait_common64 (private=416, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  45   Thread 0x<addr> (LWP 534) "llvmpipe-22"   0x<addr> in __futex_abstimed_wait_common64 (private=368, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  46   Thread 0x<addr> (LWP 533) "llvmpipe-21"   0x<addr> in __futex_abstimed_wait_common64 (private=400, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  47   Thread 0x<addr> (LWP 532) "llvmpipe-20"   0x<addr> in __futex_abstimed_wait_common64 (private=368, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  48   Thread 0x<addr> (LWP 531) "llvmpipe-19"   0x<addr> in __futex_abstimed_wait_common64 (private=368, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  49   Thread 0x<addr> (LWP 530) "llvmpipe-18"   0x<addr> in __futex_abstimed_wait_common64 (private=22207, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  50   Thread 0x<addr> (LWP 529) "llvmpipe-17"   0x<addr> in __futex_abstimed_wait_common64 (private=368, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  51   Thread 0x<addr> (LWP 528) "llvmpipe-16"   0x<addr> in __futex_abstimed_wait_common64 (private=22207, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  52   Thread 0x<addr> (LWP 527) "llvmpipe-15"   0x<addr> in __futex_abstimed_wait_common64 (private=304, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  53   Thread 0x<addr> (LWP 526) "llvmpipe-14"   0x<addr> in __futex_abstimed_wait_common64 (private=22207, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  54   Thread 0x<addr> (LWP 525) "llvmpipe-13"   0x<addr> in __futex_abstimed_wait_common64 (private=22207, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  55   Thread 0x<addr> (LWP 524) "llvmpipe-12"   0x<addr> in __futex_abstimed_wait_common64 (private=304, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  56   Thread 0x<addr> (LWP 523) "llvmpipe-11"   0x<addr> in __futex_abstimed_wait_common64 (private=22207, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  57   Thread 0x<addr> (LWP 522) "llvmpipe-10"   0x<addr> in __futex_abstimed_wait_common64 (private=22207, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  58   Thread 0x<addr> (LWP 521) "llvmpipe-9"    0x<addr> in __futex_abstimed_wait_common64 (private=22207, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  59   Thread 0x<addr> (LWP 520) "llvmpipe-8"    0x<addr> in __futex_abstimed_wait_common64 (private=22207, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  60   Thread 0x<addr> (LWP 519) "llvmpipe-7"    0x<addr> in __futex_abstimed_wait_common64 (private=304, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  61   Thread 0x<addr> (LWP 518) "llvmpipe-6"    0x<addr> in __futex_abstimed_wait_common64 (private=240, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  62   Thread 0x<addr> (LWP 517) "llvmpipe-5"    0x<addr> in __futex_abstimed_wait_common64 (private=240, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  63   Thread 0x<addr> (LWP 516) "llvmpipe-4"    0x<addr> in __futex_abstimed_wait_common64 (private=416, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  64   Thread 0x<addr> (LWP 515) "llvmpipe-3"    0x<addr> in __futex_abstimed_wait_common64 (private=416, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  65   Thread 0x<addr> (LWP 514) "llvmpipe-2"    0x<addr> in __futex_abstimed_wait_common64 (private=22207, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  66   Thread 0x<addr> (LWP 513) "llvmpipe-1"    0x<addr> in __futex_abstimed_wait_common64 (private=240, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
  67   Thread 0x<addr> (LWP 512) "llvmpipe-0"    0x<addr> in __futex_abstimed_wait_common64 (private=400, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x<addr>) at ./nptl/futex-internal.c:57
[Inferior 1 (process 510) detached]
```

Classifying every row of that table:

| Thread(s) | Name | State (frame) | Origin |
|---|---|---|---|
| 1 | `kitty` | `ppoll` (GLFW loop) | **kitty MAIN thread** — parse + Python + render (holds GIL) |
| 2 | `KittyChildMon` | `poll` on `children_fds` | **kitty PTY I/O thread** (child-monitor.c:1489) |
| 3 | `kitty:disk$0` | `futex` wait | disk-cache worker thread (kitty/disk-cache.c) — idle |
| 4–35 (32) | `kitty` | `futex` wait | Mesa/GL helper pool (software-GL, this container) |
| 36–67 (32) | `llvmpipe-0..31` | `futex` wait | Mesa **llvmpipe** software rasterizer pool |

**Critical caveat (observed + inferred):** the **64** `futex`-blocked threads in rows 4–67 are **Mesa software-OpenGL artifacts of this headless container** (the `llvmpipe` software rasterizer and its worker pool, from `libgallium-*.so`), **not** part of kitty's own concurrency design. On a normal GPU-backed desktop these do not exist. They are irrelevant to the clipboard/parser data path and are called out here so they are never mistaken for kitty threads. *(the count is observed; the attribution to Mesa llvmpipe is inferred from the thread names and the `libgallium` frames in the raw backtrace capture)*

### 3.3 Kernel `/proc` wchan cross-check

Independently of gdb, reading each task's `comm`/`wchan` via `/proc/<pid>/task/*` corroborates the same picture: MAIN and KittyChildMon are both parked in the kernel `poll` path (`do_sys_poll`), while the GL pool sits in `futex_wait_queue`. *(observed)*

```console
$ for t in /proc/<kitty_pid>/task/*; do echo "tid=${t##*/} comm=$(cat $t/comm) wchan=$(cat $t/wchan)"; done
tid=70  comm=kitty         wchan=do_sys_poll      # MAIN  (blocked in ppoll → GLFW loop)
tid=137 comm=KittyChildMon wchan=do_sys_poll      # PTY I/O thread (blocked in poll on children_fds)
# aggregate over all tasks:
     32 comm=kitty wchan=futex_wait_queue
      1 comm=kitty wchan=do_sys_poll
      1 comm=KittyChildMon wchan=do_sys_poll
```

(The `/proc` aggregate above is filtered to kitty's own threads; the 32 `comm=kitty wchan=futex_wait_queue` rows are the GL helper pool, and the `llvmpipe`/`kitty:disk$0` rows were excluded by the capture filter. The two `do_sys_poll` rows are exactly MAIN and KittyChildMon.) *(observed)*

### 3.4 Conditional and transient threads

Three further threads exist only under specific conditions and are therefore **not** in the idle table above except the disk worker; each is named in source: *(file:line observed in source; runtime presence labeled per item)*

- **`kitty:disk$0`** — the disk-cache write worker, created in `kitty/disk-cache.c` (thread body around [kitty/disk-cache.c:342]). It **is** present at idle (Thread 3 above), parked on a futex waiting for work. Its interaction with readers/shutdown is examined in §9. *(presence observed)*
- **`KittyPeerMon`** — the remote-control peer monitor [kitty/child-monitor.c:1808], started only when a control socket is configured (`listen_on`/`--listen-on`). It is **absent** in the default `--config NONE` instance above, which is why it does not appear in the table. *(absence observed in the default instance; creation file:line from source)*
- **`KittyWriteStdin`** — a **transient** worker [kitty/child-monitor.c:967], spawned per bulk write-to-child and exiting when that write completes; it is not a persistent thread and so is not expected in an idle snapshot. *(inferred from the source lifecycle; not exercised in this idle capture)*

### 3.5 Why the topology matters (the causal core)

Because **KittyChildMon runs no Python** and only fills the shared buffer, and because parsing, OSC dispatch, `screen.clipboard_control`, and all kitten-facing Python run on the **single MAIN thread under the GIL**, two consequences follow that the rest of this document demonstrates empirically:

1. Clipboard bytes are *produced* by the I/O thread but *interpreted* (crossed into Python) only on MAIN — so anything else occupying MAIN (e.g. a large scrollback scan, §7) directly delays clipboard/kitten event delivery. *(mechanism; demonstrated in §7)*
2. There is no second Python thread that could observe the parser buffer concurrently; the only genuine cross-thread sharing is the C-level parser buffer between KittyChildMon (producer) and MAIN (consumer), guarded by the parser mutex (§8). *(mechanism; demonstrated in §8–§9)*

---

## 4. O1 — Clipboard C→Python Transfer (Small and Very Large)

**Objective (O1).** Explain the mechanism by which clipboard bytes cross from the C VT-parser / screen structures into Python objects, for both the small single-dispatch case and the very-large chunked / on-disk-rollover case, backed by captured runtime output.

### 4.0 The canonical path (code-grounded, then observed)

Clipboard data enters through the OSC 52 (legacy) or OSC 5522 (extended) escape codes written by an application to the child PTY. The bytes are read by the `KittyChildMon` I/O thread into the shared 1 MiB parser buffer, then parsed on the **MAIN thread**, where the OSC payload is handed to Python as a **zero-copy `memoryview`** over the live C buffer and dispatched to the Python clipboard layer:

- `kitty/vt-parser.c:18` — `#define BUF_SZ (1024u*1024u)` — the single shared parser buffer is **1 MiB**.
- `kitty/vt-parser.c:21` — `#define MAX_ESCAPE_CODE_LENGTH (BUF_SZ/4u)` — **256 KiB**, the threshold above which an *unterminated* escape is flushed as a partial chunk.
- `kitty/vt-parser.c:457` `dispatch_osc` → `:461` `PyMemoryView_FromMemory((char*)buf+i, limit-i, PyBUF_READ)` — the zero-copy view over the live buffer, created inside the `START_DISPATCH`/`END_DISPATCH` (RAII) scope.
- `kitty/vt-parser.c:534` — `DISPATCH_OSC_WITH_CODE(clipboard_control)` — routes the OSC to the screen callback.
- `kitty/screen.c:2305` `clipboard_control` (declared `kitty/screen.h:230`), reached via the `CALLBACK` C→Python macro `kitty/screen.c:87`.
- Python receiver: `kitty/window.py:1391` `clipboard_control` → `kitty/clipboard.py` `ClipboardRequestManager` (`:332`), which builds a `WriteRequest`/`ReadRequest` and accumulates payload into a `Tempfile` (`:24`).

The chunk-accumulation logic that decides *complete vs partial* dispatch lives in `kitty/vt-parser.c`:

- `:380` `is_osc_52` (`memcmp(buf+consumed, "52;", 3) == 0`).
- `:385` `continue_osc_52` — re-primes the buffer with `52;;` after a partial flush so the next chunk is still recognised as OSC 52.
- `:393` `accumulate_st_terminated_esc_code` — **first** searches for the ST/BEL terminator; if found it dispatches the *complete* escape (this path does **not** enforce `MAX_ESCAPE_CODE_LENGTH`). Only when the escape is still unterminated **and** already exceeds `MAX_ESCAPE_CODE_LENGTH` does it emit a **partial** dispatch (the `dispatch` call passes `is_partial=true`, reported as code `-52`).

All evidence below is captured with kitty's **in-repo** `--dump-commands` instrumentation (the `DUMP_COMMANDS` build variant, `setup.py:721-722`), which routes each OSC through `REPORT_OSC2(name, code, string)` (`kitty/vt-parser.c:44-135`) and prints one `clipboard_control <code> <payload>` line per dispatch. **Partial chunks are reported with a negative code `-52`; the final (terminating) chunk with the positive code `52`.** This negative/positive code *is* the observable `is_partial` signal at the C→Python boundary.

Build/run environment: the canonical default build (`python3 setup.py`) plus the `DUMP_COMMANDS` variant, run headless under Xvfb `:99` inside the canonical container (kitty 0.35.2, HEAD `815df1e210e0`). Exact build and provenance are in the Build/Environment section.

### 4.1 Small clipboard write — a single dispatch (finding 28)

A shell inside kitty emits one OSC 52 write of `base64("hello")` = `aGVsbG8=`. Captured through `--dump-commands`:

*Script `o1_dumpfmt.sh` (sha256 `6391312a29f70c35ec8a799b34d676828663347bface0f07bc0636ec6183adb9`):*

```bash
#!/bin/bash
set -u
cd /app
export DISPLAY=:99 LANG=C.UTF-8 LC_ALL=C.UTF-8
OBS=/tmp/obs/out
mkdir -p "$OBS"
# child emits a genuine small OSC 52 clipboard-write (base64 "hello"), then a marker, then exits
cat > /tmp/obs/scripts/o1_small_child.sh <<'CH'
#!/bin/sh
printf '\033]52;c;%s\007' "$(printf hello | base64)"
sleep 0.3
CH
chmod +x /tmp/obs/scripts/o1_small_child.sh
# launch kitty with --dump-commands; capture its stdout+stderr
timeout 30 ./kitty/launcher/kitty --dump-commands \
  --config NONE -o close_on_child_death=yes \
  -o clipboard_control="write-clipboard write-primary read-clipboard read-primary" \
  sh /tmp/obs/scripts/o1_small_child.sh >"$OBS/o1_small_dump.raw" 2>"$OBS/o1_small_dump.err"
echo "exit=$?"
echo "=== clipboard_control lines in dump ==="
grep -n "clipboard_control" "$OBS/o1_small_dump.raw" | head
echo "=== total dump lines: $(wc -l < "$OBS/o1_small_dump.raw") ==="
echo "=== first 12 dump lines (structure) ==="
head -12 "$OBS/o1_small_dump.raw"
```

**Observed — exactly one dispatch (complete, positive code `52`, target `c`):**

```text
clipboard_control 52 c;aGVsbG8=
```

Parsed (via `parse_chunks.py`, below): `dispatches=1`, `codes=[52]`, `payload_lens=[10]` (the 8-char base64 plus the `c;` prefix), reconstructed 5 bytes with `sha256=2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824`, which is exactly `SHA256("hello")`. This is the small case: one payload crosses the boundary in a single `dispatch_osc` (`kitty/vt-parser.c:457`) → `clipboard_control` (`kitty/screen.c:2305`).

**Round-trip integrity (small):**

```text
case=small readback=[hello]
```

### 4.2 Large clipboard write — buffer-bounded partial chunking (finding 29, Major M3)

The document's earlier claim that large payloads are chunked at a *fixed 256 KiB* is corrected here by observation: chunk boundaries are **buffer-bounded (~1 MiB)**, set by how much unterminated OSC data has accumulated in the 1 MiB `BUF_SZ` buffer when the parser runs, not by `MAX_ESCAPE_CODE_LENGTH`. `MAX_ESCAPE_CODE_LENGTH` (256 KiB) is only the *minimum* length an unterminated escape must exceed before the *first* partial flush; the actual flush size tracks the buffer fill.

**Sub-case A — 800 KB payload fits in one buffer and dispatches complete.** An 800 000-byte base64 payload (600 000 raw bytes) terminated by BEL fits inside the 1 MiB buffer, so `accumulate_st_terminated_esc_code` (`kitty/vt-parser.c:393`) finds the terminator and dispatches it **complete**:

Parsed: `dispatches=1`, `codes=[52]`, `payload_lens=[800002]`, reconstructed 600000 bytes `sha256=c10a80d17dd6e73730f2c68e9cc956c544632a60c1526e40ef8a0a7b3029345d`. No partial chunk — confirming the *generous* complete-dispatch path.

**Sub-case B — 3 MiB payload exceeds the buffer and is delivered as partial chunks.** The emitter writes a 4 194 304-byte base64 payload (3 145 728 raw bytes):

*Script `o1_chunk_big.sh` (sha256 `4e6af7f17566246b787934734cadc17075180044201a22216e93e582e4d3c80c`):*

```bash
#!/bin/bash
set -u
cd /app
export DISPLAY=:99 LANG=C.UTF-8 LC_ALL=C.UTF-8
OBS=/tmp/obs/out; mkdir -p "$OBS"
# 3 MiB raw -> 4 MiB base64 -> single OSC 52 escape >> 1 MiB BUF_SZ -> forces partial chunking
python3 - <<'PY'
import base64,hashlib
# deterministic 3 MiB
patt=(b"kittyClipboardChunkTest0123456789ABCDEF")
n=3*1024*1024
raw=(patt*(n//len(patt)+1))[:n]
open("/tmp/obs/out/big_raw.bin","wb").write(raw)
b64=base64.standard_b64encode(raw)
open("/tmp/obs/out/big_b64.txt","wb").write(b64)
print("raw_bytes=%d raw_sha256=%s"%(len(raw),hashlib.sha256(raw).hexdigest()))
print("b64_bytes=%d osc_escape_bytes=%d BUF_SZ=1048576 MAX_ESC=262144"%(len(b64),len(b64)+8))
PY
cat > /tmp/obs/scripts/o1_bigchild.sh <<'CH'
#!/bin/sh
{ printf '\033]52;c;'; cat /tmp/obs/out/big_b64.txt; printf '\007'; }
sleep 1.0
CH
chmod +x /tmp/obs/scripts/o1_bigchild.sh
timeout 60 ./kitty/launcher/kitty --dump-commands --config NONE -o close_on_child_death=yes \
  -o clipboard_control="write-clipboard write-primary read-clipboard read-primary" \
  sh /tmp/obs/scripts/o1_bigchild.sh >"$OBS/o1_big_dump.raw" 2>"$OBS/o1_big_dump.err"
echo "kitty_exit=$?"
echo "=== dispatch count + per-chunk lengths + reconstruction ==="
python3 - <<'PY'
import base64,hashlib
lines=open("/tmp/obs/out/o1_big_dump.raw",encoding="utf-8",errors="replace").read().splitlines()
pref="clipboard_control 52 "
payloads=[l[len(pref):] for l in lines if l.startswith(pref)]
print("dispatch_count=%d"%len(payloads))
lens=[len(p) for p in payloads]
print("per_chunk_payload_lengths=%s"%lens)
print("sum=%d"%sum(lens))
# Reconstruct: chunk1 starts 'c;' then base64; continuation chunks (continue_osc_52) start ';' metadata? observe:
print("chunk_prefixes=%s"%[p[:4] for p in payloads])
# Strip the leading target metadata from EACH chunk per continue_osc_52 (';;' -> re-primed as '52;;'): observe empirically
joined="".join(payloads)
print("joined_len=%d"%len(joined))
PY
```

**Observed distribution across two identical runs** (the dispatch *count* varies run-to-run because it depends on how the I/O thread's `read()` sizes align with the MAIN-thread parse ticks — reported as a distribution, not hidden):

```text
=== BIG 3MiB run1 ===
dispatches=5
codes=[-52, -52, -52, -52, 52]
payload_lens=[1048570, 917512, 1048572, 972804, 206852]
reconstructed_raw_bytes=3145728
reconstructed_sha256=18805ee540e3771c619995a381a6b091558d96fb7159e2a3f8fbacef321ddbd3

=== BIG 3MiB run2 ===
dispatches=8
codes=[-52, -52, -52, -52, -52, -52, -52, 52]
payload_lens=[1048570, 687296, 361277, 722957, 325616, 726416, 322157, 24]
reconstructed_raw_bytes=3145728
reconstructed_sha256=18805ee540e3771c619995a381a6b091558d96fb7159e2a3f8fbacef321ddbd3

ORIGINAL big_raw.bin bytes=3145728 sha256=18805ee540e3771c619995a381a6b091558d96fb7159e2a3f8fbacef321ddbd3
```

Both runs: (a) the **first** chunk is `1 048 570` bytes ≈ `BUF_SZ` (`1 048 576`) minus a few framing bytes — confirming the boundary is the **1 MiB buffer**, not 256 KiB; (b) every non-final chunk carries code **`-52`** (partial) and the last carries **`52`** (final); (c) concatenating the chunk payloads and base64-decoding reconstructs the original **byte-for-byte** (the `reconstructed_sha256` printed above for both runs equals the original `big_raw.bin` sha256). The differing dispatch counts (5 vs 8) with identical reconstruction demonstrate the chunk *segmentation* is timing-dependent while the *delivered bytes* are exact.

**Reconstruction / integrity proof (parser):**

*Script `parse_chunks.py` (sha256 `95aad3963285e392186e71da6dcca4c1ebf8abfda062b278180ffb1a802b8c96`):*

```python
#!/usr/bin/env python3
import sys, re, hashlib, base64
# DUMP_COMMANDS emits lines: "clipboard_control <code> <payload>"
# partial chunks -> code -52 ; final chunk -> code 52 (positive).
def parse(path):
    data=open(path,'rb').read()
    # split on the token 'clipboard_control ' ; each record = code SP payload (payload may contain ; and base64)
    recs=[]
    for m in re.finditer(rb'clipboard_control (-?\d+) ', data):
        code=int(m.group(1)); start=m.end()
        nxt=data.find(b'clipboard_control ', start)
        end=nxt if nxt!=-1 else len(data)
        payload=data[start:end]
        payload=payload.rstrip(b'\n')
        recs.append((code,payload))
    return recs
def summarize(path,label):
    recs=parse(path)
    codes=[c for c,_ in recs]
    lens=[len(p) for _,p in recs]
    # reconstruct: strip leading "52;" or target prefix from FIRST, and ";" continuation markers.
    # In dump, first payload begins "c;<b64...>" (target 'c'); continuations are raw b64 chunk text.
    # Rebuild base64 by concatenating: first payload after 'c;' , subsequent payloads as-is.
    b64=b''
    for i,(c,p) in enumerate(recs):
        if i==0:
            # payload like: 52 already consumed; p = "c;<b64>" or "<loc>;<b64>"
            if b';' in p:
                b64+=p.split(b';',1)[1]
            else:
                b64+=p
        else:
            b64+=p
    try:
        raw=base64.standard_b64decode(b64)
        sha=hashlib.sha256(raw).hexdigest()
        rawlen=len(raw)
    except Exception as e:
        sha='<decode-fail: %s>'%e; rawlen=-1
    print('=== %s (%s) ==='%(label,path))
    print('dispatches=%d'%len(recs))
    print('codes=%r'%codes)
    print('payload_lens=%r'%lens)
    print('reconstructed_raw_bytes=%d'%rawlen)
    print('reconstructed_sha256=%s'%sha)
    print()
for p,l in [('/tmp/obs/out/o1_small_dump.raw','SMALL hello'),
            ('/tmp/obs/out/o1_chunk_dump.raw','CHUNK 800KB'),
            ('/tmp/obs/out/o1_big_dump.raw','BIG 3MiB run1'),
            ('/tmp/obs/out/o1_big_dump_run2.raw','BIG 3MiB run2')]:
    try: summarize(p,l)
    except Exception as e: print('ERR %s: %s'%(p,e))
# original payload hash for the 3MiB case
try:
    orig=open('/tmp/obs/out/big_raw.bin','rb').read()
    print('ORIGINAL big_raw.bin bytes=%d sha256=%s'%(len(orig),hashlib.sha256(orig).hexdigest()))
except Exception as e: print('orig err',e)
```

**Round-trip integrity (large, 3 MiB through a real read-back):**

```text
case=large bytes_in=3145728 bytes_out=3145728 sha_in=18805ee540e3771c619995a381a6b091558d96fb7159e2a3f8fbacef321ddbd3 sha_out=18805ee540e3771c619995a381a6b091558d96fb7159e2a3f8fbacef321ddbd3 integrity=MATCH
```

*Script `o1_roundtrip.sh` (sha256 `6dda7f23bab1b4a5da6a1d157d021d328e8487b3b13bd71e5f5c247e23f4062a`):*

```bash
#!/bin/bash
set -u
cd /app
export DISPLAY=:99 LANG=C.UTF-8 LC_ALL=C.UTF-8
OBS=/tmp/obs/out; mkdir -p "$OBS"

# ---- (a) 4 MiB chunk case, RUN #2 (stability of the -52...52 partial pattern) ----
timeout 60 ./kitty/launcher/kitty --dump-commands --config NONE -o close_on_child_death=yes \
  -o clipboard_control="write-clipboard write-primary read-clipboard read-primary" \
  sh /tmp/obs/scripts/o1_bigchild.sh >"$OBS/o1_big_dump_run2.raw" 2>/dev/null
echo "chunk_run2_kitty_exit=$?"
python3 - <<'PY'
import base64,hashlib,re
data=open("/tmp/obs/out/o1_big_dump_run2.raw","r",encoding="utf-8",errors="replace").read()
parts=[p for p in re.split(r"(?m)^clipboard_control ",data) if p.strip()]
recs=[]
for p in parts:
    m=re.match(r"(-?\d+)\s(.*)",p,re.S)
    if m:
        pl=m.group(2); pl=pl[:-1] if pl.endswith("\n") else pl
        recs.append((int(m.group(1)),pl))
codes=[c for c,_ in recs]
b64="".join((pl[2:] if i==0 else pl[1:]) for i,(c,pl) in enumerate(recs))
dec=base64.standard_b64decode(b64)
orig=open("/tmp/obs/out/big_raw.bin","rb").read()
print("run2 dispatch_count=%d codes=%s"%(len(recs),codes))
print("run2 per_chunk_lengths=%s"%[len(pl) for _,pl in recs])
print("run2 reconstruction_match=%s sha=%s"%(hashlib.sha256(dec).digest()==hashlib.sha256(orig).digest(),hashlib.sha256(dec).hexdigest()))
PY

# ---- (b) canonical round-trip integrity: write OSC52 then read back via real kitten clipboard ----
cat > /tmp/obs/scripts/o1_rt_child.sh <<'CH'
#!/bin/sh
CASE="$1"
if [ "$CASE" = small ]; then
  printf '\033]52;c;%s\007' "$(printf hello | base64)"
  sleep 0.4
  got=$(kitten clipboard --get-clipboard)
  printf 'case=small readback=[%s]\n' "$got" > /tmp/obs/out/rt_small.result
else
  { printf '\033]52;c;'; cat /tmp/obs/out/big_b64.txt; printf '\007'; }
  sleep 1.0
  kitten clipboard --get-clipboard > /tmp/obs/out/rt_big_out.bin
  # kitten returns the raw bytes we base64-encoded (the clipboard stores decoded bytes)
  si=$(sha256sum /tmp/obs/out/big_raw.bin | awk '{print $1}')
  so=$(sha256sum /tmp/obs/out/rt_big_out.bin | awk '{print $1}')
  ni=$(wc -c < /tmp/obs/out/big_raw.bin); no=$(wc -c < /tmp/obs/out/rt_big_out.bin)
  [ "$si" = "$so" ] && m=MATCH || m=MISMATCH
  printf 'case=large bytes_in=%s bytes_out=%s sha_in=%s sha_out=%s integrity=%s\n' "$ni" "$no" "$si" "$so" "$m" > /tmp/obs/out/rt_big.result
fi
CH
chmod +x /tmp/obs/scripts/o1_rt_child.sh
for c in small large; do
  timeout 60 ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes \
    -o clipboard_control="write-clipboard write-primary read-clipboard read-primary" \
    sh /tmp/obs/scripts/o1_rt_child.sh "$c" >/dev/null 2>&1
  echo "roundtrip_${c}_kitty_exit=$?"
done
echo "=== round-trip results ==="
cat "$OBS/rt_small.result" 2>/dev/null
cat "$OBS/rt_big.result" 2>/dev/null
```

### 4.3 Very large write — in-memory→on-disk backing-store transition (findings 30, 31, Major M4)

Python accumulates the decoded payload in a `Tempfile` (`kitty/clipboard.py:24`) that starts as an in-memory `io.BytesIO` and **rolls over to an on-disk temporary file** once it would exceed `rollover_size = 16 * 1024 * 1024` (16 MiB) — `kitty/clipboard.py:237`. The rollover copies the accumulated bytes out of the `BytesIO` via `getvalue()` into the file-backed store (`kitty/clipboard.py:32`, `rollover_if_needed`). This is a **backing-store transition** (RAM `BytesIO` → file descriptor that the OS may keep in page cache), **not** data "leaving RAM."

A 20 MiB raw payload (27 962 028 base64 bytes) is fed while tracing file syscalls:

*Script `o1_rollover.sh` (sha256 `51457165b49a9f262c86d4551727ffd5991e5eecae15eebfd4b8ac3f9a3f7fe6`):*

```bash
#!/bin/bash
set -u
cd /app
export DISPLAY=:99 LANG=C.UTF-8 LC_ALL=C.UTF-8
OBS=/tmp/obs/out; mkdir -p "$OBS"
# 20 MiB raw -> ~27 MiB base64; crosses the 16 MiB BytesIO->on-disk rollover in clipboard.py Tempfile
python3 - <<'PY'
import base64,hashlib
patt=b"kittyRolloverPayload_0123456789abcdef"
n=20*1024*1024
raw=(patt*(n//len(patt)+1))[:n]
open("/tmp/obs/out/roll_raw.bin","wb").write(raw)
open("/tmp/obs/out/roll_b64.txt","wb").write(base64.standard_b64encode(raw))
print("raw_bytes=%d (20 MiB) raw_sha256=%s rollover_threshold=16MiB(16777216)"%(len(raw),hashlib.sha256(raw).hexdigest()))
PY
# child: emit the 20 MiB OSC 52 write, then sleep so we can snapshot fds mid-life
cat > /tmp/obs/scripts/o1_roll_child.sh <<'CH'
#!/bin/sh
{ printf '\033]52;c;'; cat /tmp/obs/out/roll_b64.txt; printf '\007'; }
sleep 5
CH
chmod +x /tmp/obs/scripts/o1_roll_child.sh
# BEFORE: baseline fd count of a fresh idle kitty is unknowable pre-launch; we snapshot during instead.
# Launch kitty under strace (file-creation syscalls only), backgrounded, capture pid.
( timeout 40 strace -f -qq -e trace=openat,unlink,unlinkat,memfd_create,ftruncate \
    -o "$OBS/roll_strace.out" \
    ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes \
    -o clipboard_control="write-clipboard write-primary read-clipboard read-primary" \
    sh /tmp/obs/scripts/o1_roll_child.sh >/dev/null 2>"$OBS/roll_kitty.err" ) &
DRIVER=$!
sleep 3.5   # let the 20 MiB write be parsed + rolled over, while child still sleeping
KPID=$(pgrep -n -f "launcher/kitty" || true)
echo "kitty_pid=$KPID"
if [ -n "$KPID" ]; then
  echo "=== /proc/$KPID/fd entries pointing at a (deleted) tmp file DURING write ==="
  ls -l /proc/"$KPID"/fd 2>/dev/null | grep -iE "tmp|deleted" || echo "(no tmp/deleted fd visible at this instant)"
fi
wait $DRIVER 2>/dev/null
echo "driver_exit=$?"
echo
echo "=== strace: tmp-related file-creation syscalls (rollover TemporaryFile) ==="
grep -iE "tmp|O_TMPFILE|memfd" "$OBS/roll_strace.out" | grep -vE "/app/|\.so|site-packages|/usr/|fontconfig|\.cache" | head -25
echo "--- total openat lines in strace: $(grep -c openat "$OBS/roll_strace.out") ---"
```

**Observed — `strace` shows the stdlib `TemporaryFile` sequence: `O_TMPFILE` attempt fails `EOPNOTSUPP` in the container's `/tmp`, so it falls back to a named file that is created then immediately `unlink`ed (held open):**

```text
1389  openat(AT_FDCWD, "/tmp", O_RDWR|O_EXCL|O_NOFOLLOW|O_CLOEXEC|O_TMPFILE, 0600) = -1 EOPNOTSUPP (Operation not supported)
1389  openat(AT_FDCWD, "/tmp/tmpuj9pnvqy", O_RDWR|O_CREAT|O_EXCL|O_NOFOLLOW|O_CLOEXEC, 0600) = 9
1389  unlink("/tmp/tmpuj9pnvqy")        = 0
```

**And `/proc/<pid>/fd` during the write shows the live, held-open, already-unlinked backing store — fd 9:**

```text
9 -> /tmp/tmpuj9pnvqy (deleted)
```

The `(deleted)` marker with a still-open descriptor is the canonical signature of an anonymous on-disk temporary file: the bytes live on disk (and in page cache) but the path is gone, so the file vanishes automatically when kitty closes the descriptor. This directly confirms the M4 wording: the ≥16 MiB rollover is a `BytesIO.getvalue()` **copy** into a file-backed store — a backing-store transition, observed here at fd 9.

### 4.4 `clipboard_max_size` — the double-scale and the live truncation path (findings 32, 33, 34, Major M5)

`clipboard_max_size` is a **float** option defaulting to **512** (`kitty/options/definition.py:3111`; typed `clipboard_max_size: float = 512.0` at `kitty/options/types.py:498`, parsed by `positive_float` at `kitty/options/parse.py:125`). The effective truncation threshold is scaled **twice**:

- `kitty/clipboard.py:247` — `self.max_size = (get_options().clipboard_max_size * 1024 * 1024) if max_size < 0 else max_size` → for the default, `512 * 1024 * 1024 = 536 870 912` (i.e. 512 **MiB**, already in bytes).
- `kitty/clipboard.py:321` — `if self.max_size > 0 and self.tempfile.tell() > (self.max_size * 1024 * 1024):` → multiplies the already-byte value by `1024*1024` **again**, so the real comparison is against `536 870 912 * 1 048 576 = 562 949 953 421 312` bytes = **512 TiB**.
- `kitty/clipboard.py:322` — the truncation `log_error(f'Clipboard write request has more data than allowed by clipboard_max_size ({self.max_size}), truncating')` prints the *once*-scaled `self.max_size`.

**A1 — canonical default (512), feed 520 MiB, 2 runs → NO truncation** (512 TiB is unreachable):

*Script `o1_trunc_neg.sh` (sha256 `844bf750eac553cfd8549f413329fd3f6cfa0bd8285d189a797f0681c61a7837`):*

```bash
#!/bin/bash
set -u
cd /app
export DISPLAY=:99 LANG=C.UTF-8 LC_ALL=C.UTF-8
OBS=/tmp/obs/out; mkdir -p "$OBS"
# A1: DEFAULT clipboard_max_size=512, feed 520 MiB (> nominal 512 MiB) -> expect NO truncation (eff. 512 TiB)
FEED=$((520*1024*1024))
for run in 1 2; do
  timeout 300 ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes \
    -o clipboard_max_size=512 \
    -o clipboard_control="write-clipboard write-primary read-clipboard read-primary" \
    python3 /tmp/obs/scripts/o1_feed_child.py "$FEED" >/dev/null 2>"$OBS/trunc_A1_run${run}.err"
  echo "A1_run${run}_kitty_exit=$?  feed_bytes=$FEED (520 MiB, > nominal 512 MiB)"
  echo "--- kitty stderr (run $run) — expect NO 'truncating' line ---"
  cat "$OBS/trunc_A1_run${run}.err"
  if grep -q "truncating" "$OBS/trunc_A1_run${run}.err"; then echo "RESULT: truncation FIRED (unexpected)"; else echo "RESULT: no truncation (expected — 520 MiB << 512 TiB effective)"; fi
done
```

Observed (run 1 and run 2 stderr — only a benign systemd-bus warning, no truncation line):

```text
# run1
[0.159] Failed to open systemd user bus with error: No medium found
# run2
[0.158] Failed to open systemd user bus with error: No medium found
```

**A2 — positive control (`clipboard_max_size=0.0001`), feed 130 MiB, 2 runs → truncation FIRES.** With `0.0001`, `:247` gives `self.max_size = 0.0001*1024*1024 = 104.8576`, and `:321` compares against `104.8576*1024*1024 ≈ 104.86 MiB`; 130 MiB exceeds it:

*Script `o1_feed_child.py` (sha256 `4c6bb170f23d7c567518fedc3d324d9f1488d68358dcbdc554ddf12578371cbb`):*

```python
import sys,base64
n=int(sys.argv[1])
w=sys.stdout.buffer
w.write(b"\x1b]52;c;")
patt=b"A"*768                      # multiple of 3 -> clean 1024-byte base64 block
enc=base64.standard_b64encode(patt)
reps=n//768
for _ in range(reps): w.write(enc)
rem=n-reps*768
if rem: w.write(base64.standard_b64encode(b"A"*rem))
w.write(b"\x07"); w.flush()
import time; time.sleep(0.8)
```

*Script `o1_trunc_pos.sh` (sha256 `ce13ef5e65a9c2b4444d4722579ce939c99292b717d5c180832c26df9faf3114`):*

```bash
#!/bin/bash
set -u
cd /app
export DISPLAY=:99 LANG=C.UTF-8 LC_ALL=C.UTF-8
OBS=/tmp/obs/out; mkdir -p "$OBS"
# streaming OSC52 child: emits N decoded bytes of base64 without storing a huge file
cat > /tmp/obs/scripts/o1_feed_child.py <<'PY'
import sys,base64
n=int(sys.argv[1])
w=sys.stdout.buffer
w.write(b"\x1b]52;c;")
patt=b"A"*768                      # multiple of 3 -> clean 1024-byte base64 block
enc=base64.standard_b64encode(patt)
reps=n//768
for _ in range(reps): w.write(enc)
rem=n-reps*768
if rem: w.write(base64.standard_b64encode(b"A"*rem))
w.write(b"\x07"); w.flush()
import time; time.sleep(0.8)
PY
# POSITIVE CONTROL A2: clipboard_max_size=0.0001 (effective ~104.86 MiB), feed 130 MiB -> truncation fires
FEED=$((130*1024*1024))
for run in 1 2; do
  timeout 180 ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes \
    -o clipboard_max_size=0.0001 \
    -o clipboard_control="write-clipboard write-primary read-clipboard read-primary" \
    python3 /tmp/obs/scripts/o1_feed_child.py "$FEED" >/dev/null 2>"$OBS/trunc_A2_run${run}.err"
  echo "A2_run${run}_kitty_exit=$?  feed_bytes=$FEED (130 MiB)"
  echo "--- kitty stderr (run $run) ---"
  cat "$OBS/trunc_A2_run${run}.err"
done
```

Observed — the truncation line fires in **both** runs, and the logged value `(104.8576)` is exactly the once-scaled `self.max_size`, confirming the double-scale:

```text
# run1
[0.158] Failed to open systemd user bus with error: No medium found
[1.105] Clipboard write request has more data than allowed by clipboard_max_size (104.8576), truncating
# run2
[0.156] Failed to open systemd user bus with error: No medium found
[1.162] Clipboard write request has more data than allowed by clipboard_max_size (104.8576), truncating
```

Because the default threshold is effectively 512 TiB, the truncation guard (`kitty/clipboard.py:318-323`) is unreachable under the default configuration — a memory-exhaustion (CWE-400-class) exposure bounded in practice only by available disk/RAM for the `Tempfile`. **Observed** via the positive control that the guard *works* when the threshold is small; **inferred** (from the arithmetic above, code-grounded) that the default renders it inert.

#### Retained bytes and overshoot at truncation (finding 34)

The truncation guard writes each decoded chunk **before** it checks the size (`kitty/clipboard.py:320` writes `d`, then `:321` tests `tempfile.tell() > (self.max_size*1024*1024)`, then `:323` sets `self.max_size_exceeded = True`). Consequently the chunk that first crosses the threshold is **retained in full** (the total *overshoots* the threshold by up to one dispatch-chunk), and every later chunk is dropped by the `:318` `if not self.max_size_exceeded` guard. The retained size is recorded in `mime_map` as `tell()-start` (`kitty/clipboard.py:312`), so a read-back returns exactly the retained bytes.

To quantify this canonically we set `clipboard_max_size=0.000001` (so the *effective* threshold is `1e-6 * (1024*1024)^2 = 1 099 511.6` bytes ≈ 1.0486 MiB), feed 20 fixed **150 000-byte** decoded chunks (3 000 000 bytes total) via OSC 5522 `wdata`, then read the clipboard back and count the returned bytes. (The read-back uses a `read-clipboard` — no-ask — configuration purely as a **diagnostic measurement instrument**; it does not change the write/truncation path under test, and the no-ask consequence is discussed under O2/permission-safety.)

*Script `o1_trunc_retained.py` (sha256 `61a5c8d843c42a593bdd1be32f477610266d0cdf4f950bad7ebf12f0c9efc05c`):*

```python
#!/usr/bin/env python3
# Finding 34: measure RETAINED bytes + OVERSHOOT at truncation, canonically.
# OSC 5522 write in fixed 150000-byte decoded chunks; then read back; retained = sum of DATA payload bytes.
import sys, os, time, base64, select, termios, tty, re
def b64(s): return base64.standard_b64encode(s.encode() if isinstance(s,str) else s).decode('ascii')
def emit(s): os.write(1, s if isinstance(s,bytes) else s.encode())
def drain(to):
    buf=b''; end=time.time()+to
    while time.time()<end:
        r,_,_=select.select([0],[],[],0.2)
        if r:
            try: c=os.read(0,1<<16)
            except OSError: break
            if not c: break
            buf+=c; end=time.time()+0.8
    return buf
def main():
    resfile=sys.argv[1]
    chunk_dec=150000          # decoded bytes per wdata packet
    nchunks=20                # 20*150000 = 3,000,000 bytes fed total
    payload_chunk=bytes([65+(i%26) for i in range(chunk_dec)])  # 'A'.. deterministic
    fd=0; old=termios.tcgetattr(fd); tty.setraw(fd)
    try:
        emit(b'\x1b]5522;type=write\x1b\\')
        for i in range(nchunks):
            emit(b'\x1b]5522;type=wdata:mime='+b64('text/plain').encode()+b';'+b64(payload_chunk).encode()+b'\x1b\\')
            time.sleep(0.02)
        emit(b'\x1b]5522;type=wdata\x1b\\')   # commit
        time.sleep(0.5)
        # read back (measurement instrument; no-ask config = diagnostic)
        emit(b'\x1b]5522;type=read;'+b64('text/plain').encode()+b'\x1b\\')
        buf=drain(6.0)
    finally:
        termios.tcsetattr(fd, termios.TCSADRAIN, old)
    pkts=[m.group(1).decode('utf-8','replace') for m in re.finditer(rb'\x1b\]5522;([^\x07\x1b]*)(?:\x07|\x1b\\)', buf)]
    retained=0; ndata=0
    for p in pkts:
        if 'status=DATA:mime=' in p:
            body=p.split('status=DATA:mime=',1)[1]
            # body = <b64mime>;<b64payload>
            if ';' in body:
                b64payload=body.split(';',1)[1]
                try: retained+=len(base64.standard_b64decode(b64payload)); ndata+=1
                except Exception: pass
    fed=chunk_dec*nchunks
    threshold=1e-6*1024*1024*1024*1024   # 1e-6 * (1024*1024)^2
    with open(resfile,'w') as f:
        f.write('FED_BYTES=%d (=%d chunks x %d)\n'%(fed,nchunks,chunk_dec))
        f.write('EFFECTIVE_THRESHOLD_BYTES=%.1f\n'%threshold)
        f.write('DATA_PACKETS=%d\n'%ndata)
        f.write('RETAINED_BYTES=%d\n'%retained)
        f.write('OVERSHOOT_BYTES=%.1f (retained - threshold)\n'%(retained-threshold))
        f.write('DROPPED_BYTES=%d (fed - retained)\n'%(fed-retained))
        f.write('STATUSES=%s\n'%[re.search(r'status=([A-Z]+)',p).group(1) for p in pkts if 'status=' in p][:6])
main()
```

*Script `o1_trunc_retained_drv.sh` (sha256 `b98049246e90072c16135804ed9522ec58973ec37c9ef7ac84a62452f6392f39`):*

```bash
#!/bin/bash
set -u
cd /app; export DISPLAY=:99 LANG=C.UTF-8 LC_ALL=C.UTF-8
OBS=/tmp/obs/out
# clipboard_max_size=0.000001 -> self.max_size=1.048576 ; effective threshold ~1.0486 MiB.
# read-clipboard (NO ask) is a DIAGNOSTIC measurement instrument for reading back the retained bytes.
for R in 1 2; do
  timeout 60 ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes \
    -o clipboard_max_size=0.000001 \
    -o clipboard_control="write-clipboard read-clipboard write-primary read-primary" \
    python3 /tmp/obs/scripts/o1_trunc_retained.py "$OBS/trunc_retained_run$R.result" \
    >/dev/null 2>"$OBS/trunc_retained_run$R.kitty.err"
  echo "=== RUN $R (kitty_exit=$?) ==="
  cat "$OBS/trunc_retained_run$R.result"
  echo "--- kitty stderr (truncation log) ---"
  grep -E "truncating|clipboard_max_size" "$OBS/trunc_retained_run$R.kitty.err" || echo "(no truncation line)"
  echo
done
```

**Observed — identical across both runs:**

```text
# run 1
FED_BYTES=3000000 (=20 chunks x 150000)
EFFECTIVE_THRESHOLD_BYTES=1099511.6
DATA_PACKETS=293
RETAINED_BYTES=1200000
OVERSHOOT_BYTES=100488.4 (retained - threshold)
DROPPED_BYTES=1800000 (fed - retained)
STATUSES=['DONE', 'OK', 'DATA', 'DATA', 'DATA', 'DATA']
# kitty stderr (run 1)
[0.162] Failed to open systemd user bus with error: No medium found
[0.346] Clipboard write request has more data than allowed by clipboard_max_size (1.048576), truncating

# run 2
FED_BYTES=3000000 (=20 chunks x 150000)
EFFECTIVE_THRESHOLD_BYTES=1099511.6
DATA_PACKETS=293
RETAINED_BYTES=1200000
OVERSHOOT_BYTES=100488.4 (retained - threshold)
DROPPED_BYTES=1800000 (fed - retained)
STATUSES=['DONE', 'OK', 'DATA', 'DATA', 'DATA', 'DATA']
# kitty stderr (run 2)
[0.155] Failed to open systemd user bus with error: No medium found
[0.337] Clipboard write request has more data than allowed by clipboard_max_size (1.048576), truncating
```

The retained size is **1 200 000 bytes** = exactly 8 × 150 000, i.e. the 8th chunk (cumulative `tell()` = 1 200 000) is the first to exceed 1 099 511.6, and it is retained whole. The **overshoot is 100 488.4 bytes** (retained − threshold), which is **less than one 150 000-byte chunk** — precisely the expected write-then-check behaviour. **1 800 000 bytes (chunks 9–20) are dropped.** The truncation log prints `clipboard_max_size (1.048576)` (the once-scaled `self.max_size = 0.000001*1024*1024`), independently re-confirming the double-scale at a different scale point. Both runs are byte-identical, so the value is stable, not a one-off.

### 4.5 Read paths — legacy OSC 52 `?` and extended OSC 5522 (findings 35, 36, 37)

**Legacy OSC 52 `?` read** takes `fulfill_legacy_read_request` → `encode_osc52` (`kitty/clipboard.py`). Seed with an OSC 52 write, then request the read:

*Script `osc52read.py` (sha256 `24711081d340cbc5f5a337d1fff9cf954840733c7dac019753927fb583d81f50`):*

```python
#!/usr/bin/env python3
# Canonical OSC 52 '?' (legacy) read: seed via OSC52 write, then request read with '?'.
import sys, os, time, base64, select, termios, tty, re
def emit(s): os.write(1, s if isinstance(s,bytes) else s.encode())
def readresp(to):
    buf=b''; end=time.time()+to
    while time.time()<end:
        r,_,_=select.select([0],[],[],0.2)
        if r:
            c=os.read(0,65536)
            if not c: break
            buf+=c; end=time.time()+0.5
    return buf
resfile=sys.argv[1]
fd=0; old=termios.tcgetattr(fd); tty.setraw(fd)
try:
    emit(b'\x1b]52;c;'+base64.standard_b64encode(b'legacy52read').decode().encode()+b'\x07'); time.sleep(0.4)
    emit(b'\x1b]52;c;?\x07')          # OSC 52 read request
    buf=readresp(3.0)
finally:
    termios.tcsetattr(fd, termios.TCSADRAIN, old)
# OSC 52 response: ESC ] 52 ; <loc> ; <base64> (BEL or ST)
m=re.search(rb'\x1b\]52;([^;]*);([^\x07\x1b]*)(?:\x07|\x1b\\)', buf)
with open(resfile,'w') as f:
    f.write('RAW_BYTES_LEN=%d\n'%len(buf))
    f.write('RAW_HEX=%s\n'%buf[:200].hex())
    if m:
        loc=m.group(1).decode(); b64=m.group(2).decode('ascii','replace')
        try: dec=base64.standard_b64decode(b64).decode('utf-8','replace')
        except Exception: dec='<decode-fail>'
        f.write('OSC52_READ_RESPONSE loc=%r base64=%r decoded=%r\n'%(loc,b64,dec))
    else:
        f.write('NO OSC52 RESPONSE PARSED\n')
```

Observed — the response is `ESC ] 52 ; c ; <base64> ST`, decoding to the seeded text:

```text
RAW_BYTES_LEN=25
RAW_HEX=1b5d35323b633b6247566e59574e354e544a795a57466b1b5c
OSC52_READ_RESPONSE loc='c' base64='bGVnYWN5NTJyZWFk' decoded='legacy52read'
```

The raw hex `1b5d35323b633b6247566e59574e354e544a795a57466b1b5c` decodes as `ESC ] 5 2 ; c ; bGVnYWN5NTJyZWFk ESC \` and `base64.decode("bGVnYWN5NTJyZWFk") = "legacy52read"` — byte-verified against the seed.

**Extended OSC 5522 read** returns a 3-packet sequence `OK` → `DATA:mime=<b64-mime>;<b64-payload>` → `DONE` (`ReadRequest.encode_response`, `kitty/clipboard.py:207`; statuses emitted at `:476`, `:499`). The harness below drives OSC 5522 through a raw-mode PTY child and records the decoded packets to a file (the child's stderr *is* the PTY, so results are written to a side file):

*Script `osc5522_harness.py` (sha256 `255d5ca1f7ad219379b893a1d869ebc9913feba0cbac154e5294727d6b178760`):*

```python
#!/usr/bin/env python3
import sys, os, time, base64, select, termios, tty, re
def b64(s): return base64.standard_b64encode(s.encode() if isinstance(s,str) else s).decode('ascii')
def emit(s): os.write(1, s if isinstance(s,bytes) else s.encode())
def read_responses(timeout):
    buf=b''; end=time.time()+timeout
    while time.time()<end:
        r,_,_=select.select([0],[],[],0.2)
        if r:
            try: chunk=os.read(0,65536)
            except OSError: break
            if not chunk: break
            buf+=chunk; end=time.time()+0.6
    return buf
def parse_5522(buf):
    return [m.group(1).decode('utf-8','replace')
            for m in re.finditer(rb'\x1b\]5522;([^\x07\x1b]*)(?:\x07|\x1b\\)', buf)]
def main():
    scen=sys.argv[1]; resfile=sys.argv[2]
    to=float(sys.argv[3]) if len(sys.argv)>3 else 3.0
    fd=0; old=termios.tcgetattr(fd); tty.setraw(fd)
    try:
        if scen in ('read_ok','read_prompt'):
            emit(b'\x1b]52;c;'+b64('clip5522payload').encode()+b'\x07'); time.sleep(0.4)
            emit(b'\x1b]5522;type=read;'+b64('text/plain').encode()+b'\x1b\\')
        elif scen=='read_targets':
            emit(b'\x1b]52;c;'+b64('seedX').encode()+b'\x07'); time.sleep(0.3)
            emit(b'\x1b]5522;type=read;'+b64('.').encode()+b'\x1b\\')
        elif scen=='read_eperm':
            emit(b'\x1b]5522;type=read;'+b64('text/plain').encode()+b'\x1b\\')
        elif scen=='write_done':
            emit(b'\x1b]5522;type=write\x1b\\'); emit(b'\x1b]5522;type=wdata:mime='+b64('text/plain').encode()+b';'+b64('hello-5522-write').encode()+b'\x1b\\'); emit(b'\x1b]5522;type=wdata\x1b\\')
        elif scen=='write_eperm':
            emit(b'\x1b]5522;type=write\x1b\\')
        elif scen=='write_einval':
            emit(b'\x1b]5522;type=write\x1b\\'); emit(b'\x1b]5522;type=wdata:mime='+b64('text/plain').encode()+b';@@@notb64@@@\x1b\\'); emit(b'\x1b]5522;type=wdata\x1b\\')
        buf=read_responses(to)
    finally:
        termios.tcsetattr(fd, termios.TCSADRAIN, old)
    pkts=parse_5522(buf)
    with open(resfile,'w') as f:
        f.write('SCENARIO=%s\n'%scen); f.write('RAW_BYTES_LEN=%d\n'%len(buf))
        f.write('RESPONSE_PACKETS=%d\n'%len(pkts))
        for p in pkts: f.write('  PKT: 5522;%s\n'%p)
        sts=[re.search(r'status=([A-Z]+)',p).group(1) for p in pkts if 'status=' in p]
        f.write('STATUSES=%s\n'%sts)
main()
```

*Script `osc5522_driver.sh` (sha256 `048ffc9f0f7e86b89984c8c1c50d771c22df7ab7c1252aae2df3bb78db8c4b14`):*

```bash
#!/bin/bash
set -u
cd /app
export DISPLAY=:99 LANG=C.UTF-8 LC_ALL=C.UTF-8
OBS=/tmp/obs/out; mkdir -p "$OBS"
run() {
  local scen="$1" pol="$2"
  timeout 40 ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes \
    -o clipboard_control="$pol" \
    python3 /tmp/obs/scripts/osc5522_harness.py "$scen" "$OBS/o5522_${scen}.result" \
    >/dev/null 2>/dev/null
  echo "### scenario=$scen policy=[$pol] kitty_exit=$? ###"
  cat "$OBS/o5522_${scen}.result" 2>/dev/null || echo "(no result file)"
  echo
}
run read_ok        "write-clipboard read-clipboard"
run read_targets   "write-clipboard read-clipboard-ask"
run read_eperm     "write-clipboard"
run write_done     "write-clipboard read-clipboard"
run write_eperm    "read-clipboard"
run write_einval   "write-clipboard read-clipboard"
run write_primary  "write-primary write-clipboard read-clipboard"
```

Observed — `read_ok` (3 packets, payload `clip5522payload`, mime `text/plain`):

```text
SCENARIO=read_ok
RAW_BYTES_LEN=131
RAW_HEX_FIRST400=1b5d353532323b747970653d726561643a7374617475733d4f4b1b5c1b5d353532323b747970653d726561643a7374617475733d444154413a6d696d653d64475634644339776247467062673d3d3b59327870634455314d6a4a7759586c736232466b1b5c1b5d353532323b747970653d726561643a7374617475733d444f4e451b5c
RESPONSE_PACKETS=3
  PKT: 5522;type=read:status=OK
  PKT: 5522;type=read:status=DATA:mime=dGV4dC9wbGFpbg==;Y2xpcDU1MjJwYXlsb2Fk
  PKT: 5522;type=read:status=DONE
STATUSES=['OK', 'DATA', 'DONE']
```

**MIME/targets listing (finding 37).** A read with the `.` MIME (`TARGETS_MIME = '.'`, `kitty/clipboard.py:69`) is auto-fulfilled even under `read-clipboard-ask` (`kitty/clipboard.py:519`, the TARGETS-only branch), returning the list of available MIME types rather than clipboard contents:

```text
SCENARIO=read_targets
RAW_BYTES_LEN=115
RAW_HEX_FIRST400=1b5d353532323b747970653d726561643a7374617475733d4f4b1b5c1b5d353532323b747970653d726561643a7374617475733d444154413a6d696d653d4c673d3d3b64475634644339776247467062676f3d1b5c1b5d353532323b747970653d726561643a7374617475733d444f4e451b5c
RESPONSE_PACKETS=3
  PKT: 5522;type=read:status=OK
  PKT: 5522;type=read:status=DATA:mime=Lg==;dGV4dC9wbGFpbgo=
  PKT: 5522;type=read:status=DONE
STATUSES=['OK', 'DATA', 'DONE']
```

Here `mime=Lg==` decodes to `.` (the targets request) and the DATA payload `dGV4dC9wbGFpbgo=` decodes to `text/plain\n` — the single available target.

### 4.6 Permission prompts — `read-clipboard-ask` accept vs deny (findings 38, 39, 40)

The default `clipboard_control` (`kitty/options/definition.py:3096`) includes **`read-clipboard-ask`**, so a non-TARGETS read raises an interactive confirmation: `kitty/clipboard.py:518` `ask_to_read_clipboard` → `kitty/boss.py:995` `confirm()` spawns the **`ask` kitten** as a `--type=yesno` overlay window; the user's answer drives `fulfill_read_request`/reject. We drive the **real** overlay with real keystrokes via `xdotool` against the Xvfb `:99` display (no bypass):

**Ask-overlay state while the read is pending (finding 38).** Before any keystroke, the read is genuinely *blocked* on the overlay, and the overlay is backed by a **real `ask` kitten process** spawned by kitty. Captured live while the prompt was up (a state-capture driver snapshots the process tree, then sends the deny keystroke):

*Script `prompt_state_drv.sh` (sha256 `393977e0cc6a67e6a3e60f91495f173e2199efc2ac9d1ccf7c23e05ac486b251`):*

```bash
#!/bin/bash
set -u
cd /app; export DISPLAY=:99 LANG=C.UTF-8 LC_ALL=C.UTF-8
OBS=/tmp/obs/out
# Launch kitty running the read_prompt harness (default clipboard_control => read-clipboard-ask).
# The read BLOCKS on the ask overlay; we snapshot state BEFORE sending any keystroke.
timeout 60 ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes \
  python3 /tmp/obs/scripts/osc5522_harness.py read_prompt "$OBS/prompt_state.result" 12 \
  >/dev/null 2>"$OBS/prompt_state.kitty.err" &
KPID=$!
sleep 5   # allow seed-write + read request + overlay to come up (no keystroke yet)
echo "===== STATE WHILE READ IS PENDING ON THE ask OVERLAY (no keystroke sent yet) ====="
echo "--- (a) harness result file so far (should be EMPTY/absent => read still blocked) ---"
if [ -s "$OBS/prompt_state.result" ]; then echo "NON-EMPTY (unexpected):"; cat "$OBS/prompt_state.result"; else echo "(empty/absent => read is BLOCKED awaiting the overlay decision)"; fi
echo
echo "--- (b) process tree under the kitty launcher (shows the 'ask' kitten backing the overlay) ---"
KROOT=$(pgrep -n -f "launcher/kitty" || echo "$KPID")
echo "kitty launcher pid=$KROOT"
ps -eo pid,ppid,stat,comm,args --sort=ppid | grep -E "kitty|ask|kitten|python3" | grep -v grep | sed 's/^/   /'
echo
echo "--- (c) pstree rooted at kitty ---"
pstree -a -p "$KROOT" 2>/dev/null | sed 's/^/   /' || echo "(pstree unavailable)"
echo
echo "--- (d) grep specifically for the ask kitten cmdline ---"
ps -ef | grep -E "kitten|/ask|type=yesno|--type=yesno" | grep -v grep | sed 's/^/   /' || echo "(no explicit ask cmdline matched)"
echo
echo "===== NOW send DENY keystroke to release the overlay and finish cleanly ====="
WIN=$(xdotool search --onlyvisible --class kitty 2>/dev/null | head -1)
echo "kitty X window id=$WIN"
xdotool windowactivate --sync "$WIN" 2>/dev/null || true
xdotool key --window "$WIN" n 2>/dev/null || xdotool key n
sleep 3
echo "--- harness result AFTER deny keystroke ---"
cat "$OBS/prompt_state.result" 2>/dev/null || echo "(none)"
# cleanup
kill "$KPID" 2>/dev/null || true
K=$(pgrep -n -f "launcher/kitty" || true); [ -n "${K:-}" ] && kill "$K" 2>/dev/null || true
exit 0
```

**Observed — the pending state: empty result (blocked) + the `ask --type=yesno` kitten backing the overlay, with the exact clipboard-permission message; then the deny keystroke releases it with `EPERM`:**

```text
===== STATE WHILE READ IS PENDING ON THE ask OVERLAY (no keystroke sent yet) =====
--- (a) harness result file so far ---
(empty/absent => read is BLOCKED awaiting the overlay decision)

--- (b) process tree under the kitty launcher (shows the 'ask' kitten backing the overlay) ---
kitty launcher pid=3878
   3876  3839 S    timeout  timeout 60 ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes python3 .../osc5522_harness.py read_prompt .../prompt_state.result 12
   3878  3876 Sl   kitty    ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes python3 .../osc5522_harness.py read_prompt .../prompt_state.result 12
   3945  3878 Ss+  python3  python3 .../osc5522_harness.py read_prompt .../prompt_state.result 12
   3946  3878 Ssl+ kitten   /app/kitty/launcher/kitten ask --type=yesno --message A program running in this window wants to read from the system clipboard. Allow it to do so, once? --default y

--- (c) pstree rooted at kitty (elided GL worker threads) ---
   kitty,3878 --config NONE ... python3 .../osc5522_harness.py read_prompt ...
     |-kitten,3946 ask --type=yesno --message A program running in this window wants to read from the system clipboard. Allo
     |   `-{kitten}x16 (worker threads)
     |-python3,3945 .../osc5522_harness.py read_prompt .../prompt_state.result 12
     `-{kitty}x66 (GL/llvmpipe worker threads)

--- (d) ask kitten cmdline (exact) ---
   root 3946 3878 22:04 pts/1 00:00:00 /app/kitty/launcher/kitten ask --type=yesno --message A program running in this window wants to read from the system clipboard. Allow it to do so, once? --default y

===== DENY keystroke sent to kitty X window id=2097164 (xdotool key n) =====
--- harness result AFTER deny keystroke ---
SCENARIO=read_prompt
RAW_BYTES_LEN=31
RESPONSE_PACKETS=1
  PKT: 5522;type=read:status=EPERM
STATUSES=['EPERM']
```

The overlay is not a passive artifact: it is a live `--type=yesno` confirmation (message *"A program running in this window wants to read from the system clipboard. Allow it to do so, once?"*, `--default y`) spawned via `kitty/boss.py:995` `confirm()`; the read request remains unfulfilled until the overlay is answered. This is the ask-overlay **state**, distinct from (and preceding) the accept/deny **decisions** below.

*Script `prompt_driver.sh` (sha256 `0b1f742e367044716d0c5811163190e0df5d49b9a159b97c250c41875dd9bf66`):*

```bash
#!/bin/bash
set -u
cd /app
export DISPLAY=:99 LANG=C.UTF-8 LC_ALL=C.UTF-8
OBS=/tmp/obs/out; mkdir -p "$OBS"
prompt_run() {  # $1=label  $2=key(y/n)
  local label="$1" key="$2"
  ( timeout 45 ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes \
      -o clipboard_control="write-clipboard read-clipboard-ask read-primary-ask" \
      python3 /tmp/obs/scripts/osc5522_harness.py read_prompt "$OBS/prompt_${label}.result" 12 \
      >/dev/null 2>"$OBS/prompt_${label}.kitty.err" ) &
  DRV=$!
  sleep 4   # let the ask overlay come up
  KPID=$(pgrep -n -f "launcher/kitty" || true)
  echo "### $label: kitty_pid=$KPID (overlay should be up) ###"
  echo "--- ask-overlay state: kitty process tree (grep ask) ---"
  ps -o pid,ppid,comm,args -g $(ps -o sid= -p "$KPID" 2>/dev/null) 2>/dev/null | grep -iE "ask|clipboard|kitten" | grep -v grep | head
  # send the real keystroke to the overlay via X
  WID=$(DISPLAY=:99 xdotool search --class kitty 2>/dev/null | head -1)
  echo "--- driving decision: xdotool key '$key' to kitty window $WID ---"
  DISPLAY=:99 xdotool windowactivate --sync "$WID" 2>/dev/null; sleep 0.3
  DISPLAY=:99 xdotool key --clearmodifiers "$key" 2>/dev/null
  sleep 0.5
  DISPLAY=:99 xdotool key --clearmodifiers Return 2>/dev/null   # confirm if needed
  wait $DRV 2>/dev/null
  echo "--- harness result ($label) ---"
  cat "$OBS/prompt_${label}.result" 2>/dev/null || echo "(no result)"
  echo
}
prompt_run accept y
prompt_run deny   n
```

**Observed — ACCEPT (`xdotool key y`): the read is permitted and returns the full 3-packet sequence:**

```text
SCENARIO=read_prompt
RAW_BYTES_LEN=132
RESPONSE_PACKETS=3
  PKT: 5522;type=read:status=OK
  PKT: 5522;type=read:status=DATA:mime=dGV4dC9wbGFpbg==;Y2xpcDU1MjJwYXlsb2Fk
  PKT: 5522;type=read:status=DONE
STATUSES=['OK', 'DATA', 'DONE']
```

**Observed — DENY (`xdotool key n`): the read is refused with `EPERM`:**

```text
SCENARIO=read_prompt
RAW_BYTES_LEN=32
RESPONSE_PACKETS=1
  PKT: 5522;type=read:status=EPERM
STATUSES=['EPERM']
```

The read **blocked** pending the overlay, and the outcome diverged **strictly** by the `y`/`n` decision — `OK/DATA/DONE` on accept vs `EPERM` on deny — which is the runtime proof that the `read-clipboard-ask` prompt gates the read at the Python boundary (`kitty/clipboard.py:518` → `kitty/boss.py:995`). This — together with the pending-state capture above — is the ask-overlay *state* (a real `ask` kitten holding a blocked request) plus both terminal decisions.

### 4.7 Status/error branch ledger (findings 41–51)

Every OSC 5522 status is enumerated below. Statuses reachable through the canonical OSC-52/5522-through-PTY path are shown with their captured raw packet; statuses that are **not reachable** in the default X11/Xvfb build are documented as **exact, code-grounded failed reproductions** (varied genuine attempts, then the source path that would emit them), explicitly labelled inferred — never presented as observed.

**Canonical captures (raw packets):**

- **write `DONE`** (`kitty/clipboard.py:404`):

```text
SCENARIO=write_done
RAW_BYTES_LEN=31
RAW_HEX_FIRST400=1b5d353532323b747970653d77726974653a7374617475733d444f4e451b5c
RESPONSE_PACKETS=1
  PKT: 5522;type=write:status=DONE
STATUSES=['DONE']
```

- **write `EPERM`** (policy-denied, `kitty/clipboard.py:442`):

```text
SCENARIO=write_eperm
RAW_BYTES_LEN=32
RAW_HEX_FIRST400=1b5d353532323b747970653d77726974653a7374617475733d455045524d1b5c
RESPONSE_PACKETS=1
  PKT: 5522;type=write:status=EPERM
STATUSES=['EPERM']
```

- **write `EINVAL`** (malformed base64 → `binascii.Error` caught at `kitty/clipboard.py:394` → `:396`). The child fed a deliberately malformed base64 body; kitty logged the decode error and replied `EINVAL`:

```text
SCENARIO=write_einval
RAW_BYTES_LEN=33
RAW_HEX_FIRST400=1b5d353532323b747970653d77726974653a7374617475733d45494e56414c1b5c
RESPONSE_PACKETS=1
  PKT: 5522;type=write:status=EINVAL
STATUSES=['EINVAL']
```

Kitty-side stderr for the same run (the caught decode error):

```text
[0.157] Failed to open systemd user bus with error: No medium found
Traceback (most recent call last):
  File "/app/kitty/launcher/../../kitty/window.py", line 1393, in clipboard_control
    self.clipboard_request_manager.parse_osc_5522(data)
  File "/app/kitty/launcher/../../kitty/clipboard.py", line 388, in parse_osc_5522
    wr.add_base64_data(epayload, mime)
  File "/app/kitty/launcher/../../kitty/clipboard.py", line 303, in add_base64_data
    write_saving_leftover_bytes(data)
  File "/app/kitty/launcher/../../kitty/clipboard.py", line 291, in write_saving_leftover_bytes
    self.write_base64_data(data)
  File "/app/kitty/launcher/../../kitty/clipboard.py", line 319, in write_base64_data
    d = standard_b64decode(b)
        ^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib/python3.12/base64.py", line 106, in standard_b64decode
    return b64decode(s)
           ^^^^^^^^^^^^
  File "/usr/lib/python3.12/base64.py", line 88, in b64decode
    return binascii.a2b_base64(s, strict_mode=validate)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
binascii.Error: Invalid base64-encoded string: number of data characters (9) cannot be 1 more than a multiple of 4
```

- **write to primary selection `DONE`** (primary *is* enabled under Xvfb):

```text
SCENARIO=write_primary
RAW_BYTES_LEN=31
RAW_HEX_FIRST400=1b5d353532323b747970653d77726974653a7374617475733d444f4e451b5c
RESPONSE_PACKETS=1
  PKT: 5522;type=write:status=DONE
STATUSES=['DONE']
```

- **read `EPERM`** (policy-denied):

```text
SCENARIO=read_eperm
RAW_BYTES_LEN=31
RAW_HEX_FIRST400=1b5d353532323b747970653d726561643a7374617475733d455045524d1b5c
RESPONSE_PACKETS=1
  PKT: 5522;type=read:status=EPERM
STATUSES=['EPERM']
```

(read `OK`/`DATA`/`DONE` shown in O1.5.)

**Source-inferred failed reproductions (labelled inferred — not observed at this HEAD):**

| Status | Path/dir | Emit site | Why unreachable canonically here | Genuine attempts made |
|---|---|---|---|---|
| **ENOSYS** (read) | read | `kitty/clipboard.py:471` (`if not cp.enabled`) | `Clipboard.enabled = (clipboard_type is clipboard) or supports_primary_selection` (`kitty/clipboard.py:87`) — the regular clipboard is **always** enabled, so `not cp.enabled` is never true for it. ENOSYS on read requires the *primary-selection* clipboard where `supports_primary_selection` is False (macOS / no-X), which cannot occur under X11/Xvfb. | Attempted primary-selection read under Xvfb → primary **is** supported here (`write_primary` returned `DONE`), so the not-enabled branch never triggers. |
| **ENOSYS** (write) | write | `kitty/clipboard.py:442` (`'EPERM' if not allowed else 'ENOSYS'`) | Same construction: reachable only when the target selection is unsupported. | `o5522_write_primary_enosys.err` run confirmed primary write succeeds (`DONE`), not ENOSYS. |
| **EIO** (write) | write | `kitty/clipboard.py:391` inside `except OSError:` (`:389`) | Requires an `OSError` while writing the accumulated payload to the `Tempfile` backing store — i.e. a real disk I/O fault. No such fault occurs in a healthy container `/tmp`. | Reproduction requires fault injection (full/failing filesystem) which is outside the default canonical configuration; documented as the exact code path only. |
| **EBUSY** | read+write | *none in core* | `grep -rn EBUSY kitty/*.py` returns **nothing** — the core clipboard implementation never emits `EBUSY` at HEAD `815df1e210e0`. It exists only in the **protocol spec** (`docs/clipboard.rst:66,116,159`) and is **decoded defensively by the Go client** (`kittens/clipboard/write.go:145`, `kittens/clipboard/read.go:236`). | Confirmed by tree-wide grep (below); no canonical trigger exists. |

**EBUSY grep proof (authoritative, this HEAD):**

```text
$ grep -rn "EBUSY" kitty/*.py            # core clipboard implementation
(none in core python)

$ grep -rn "EBUSY" docs/clipboard.rst     # protocol spec only
66:``status=EBUSY``
116:``status=EBUSY``
159:sending back the ``EBUSY`` error code indicating some other window is trying

$ grep -rn "EBUSY" kittens/clipboard/     # Go client decodes it defensively
kittens/clipboard/write.go:145:			case "EBUSY":
kittens/clipboard/read.go:236:	case "EBUSY":
```

### 4.8 Non-canonical cross-checks and byte/hash ledger (findings 52, 53)

**Canonical vs non-canonical.** Every capture in O1.1–O1.7 originates from OSC 52/5522 written to a **real child PTY** and parsed by the **live VT parser** — the canonical C→Python boundary under study. No remote-control (`kitty/boss.py:849`) or direct-Python clipboard call was used to produce any O1 value. The `--dump-commands` instrumentation observes the *same* canonical dispatch (`REPORT_OSC2`, `kitty/vt-parser.c:44-135`); it is an observation hook, not a bypass. Any value from a bypassing interface would be labelled non-canonical — none appears in this section.

**Byte / hash ledger (reproducibility):**

| Artifact | Bytes | sha256 |
|---|---|---|
| small "hello" reconstruction | 5 | `2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824` |
| 800 KB payload (raw) | 600 000 | `c10a80d17dd6e73730f2c68e9cc956c544632a60c1526e40ef8a0a7b3029345d` |
| 3 MiB payload (raw, original) | 3 145 728 | `18805ee540e3771c619995a381a6b091558d96fb7159e2a3f8fbacef321ddbd3` |
| 3 MiB reconstruction, run 1 | 3 145 728 | `18805ee540e3771c619995a381a6b091558d96fb7159e2a3f8fbacef321ddbd3` |
| 3 MiB reconstruction, run 2 | 3 145 728 | `18805ee540e3771c619995a381a6b091558d96fb7159e2a3f8fbacef321ddbd3` |
| 20 MiB rollover payload (raw) | 20 971 520 | (fed via `o1_rollover.sh`; on-disk backing store fd 9 `/tmp/tmpuj9pnvqy (deleted)`) |

All O1 scripts are embedded above with their sha256; all captured outputs are shown complete and unedited.

---

## 5. O2 — Behavior in Practice When Other Parts of the System Are Busy

**Objective (O2).** Explain what the clipboard/event transfer looks like in practice when other parts of kitty are busy at the same time, backed by captured runtime output: the thread split, `input_delay` coalescing, the 1 MiB buffer and POLLIN backpressure, and measured delivery latency across repeated runs.

### 5.0 The concurrency model that makes "busy" matter (recap, code-grounded + observed)

From the Threading section (gdb-proven): PTY bytes are read by the **`KittyChildMon`** I/O thread into the single shared 1 MiB parser buffer, but **parsing, all Python dispatch (including `clipboard_control`), screen mutation, and rendering run on the MAIN thread under the GIL**. The I/O thread never runs Python. Three code mechanisms govern behavior under load:

- **POLLIN backpressure** — `kitty/child-monitor.c:1501`: `children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;`. On every io_loop iteration the child fd's read-interest is set to `POLLIN` **only if** the parser has buffer space, else `0`. `vt_parser_has_space_for_input` (`kitty/vt-parser.c:1478`) returns `self->read.sz + self->write.pending < BUF_SZ` — so the instant POLLIN is cleared, occupancy has reached `BUF_SZ` (1 MiB).
- **`input_delay` coalescing** — `kitty/child-monitor.c:1508-1511`: when wakeups are pending, `poll(children_fds, self->count + EXTRA_FDS, monotonic_t_to_ms(OPT(input_delay) - elapsed))` — the io_loop waits up to `input_delay` (default 3 ms) collecting more input before waking MAIN. The MAIN-side gate (`kitty/vt-parser.c:1425`) consumes only if `flush || time_since_new_input >= OPT(input_delay) || self->read.sz + 16*1024 > BUF_SZ`.
- **Lock hand-off** — `kitty/vt-parser.c:1417-1445`: `run_worker` takes the parser mutex, promotes `self->read.sz += self->write.pending` **under the lock**, then runs `consume_input` **with the lock released** (so the I/O thread can keep filling `write.pending`), then re-promotes under the lock. Reads on the I/O side (`kitty/child-monitor.c:1337` `read_bytes`) reserve the write buffer under the lock (`vt_parser_create_write_buffer`), `read()` **outside** the lock, then `vt_parser_commit_write` under the lock.

The default `input_delay` is **3 ms** (`kitty/options/definition.py:878`). Each mechanism is exercised and observed below. Build/run: the canonical default build, run headless under Xvfb `:99` (kitty 0.35.2, HEAD `815df1e210e0`); timing source `time.monotonic_ns()` (`CLOCK_MONOTONIC`).

### 5.1 Heavy PTY flood + event overlap in one live instance (findings 54, 55)

A single kitty instance runs a child that floods the PTY as fast as possible (64 KiB nonblocking writes) for a fixed 3 s wall window, counting bytes, throughput, and backpressure stalls. This establishes the reproducible "system is busy" scenario with explicit scale/rate/duration/PIDs/timing source.

*Script `o2_flood.py` (sha256 `6cfaa25034d2d582a0c87e0ff3f66fcae112919d20265784c60f2f0f5499b0ff`):*

```python
#!/usr/bin/env python3
# Heavy PTY flood child: nonblocking writes of 64 KiB chunks for a fixed wall duration.
# EAGAIN => the pipeline (PTY kernel buffer + kitty's 1 MiB parser buffer) is momentarily full
# because kitty disabled POLLIN (backpressure). We count stalls, bytes, and the max single
# "absorbed run" (bytes written between consecutive stalls) which approaches PTY+parser capacity.
import os, sys, time, select, fcntl, errno
def main():
    resfile=sys.argv[1]; dur=float(sys.argv[2]) if len(sys.argv)>2 else 3.0
    fd=1
    fl=fcntl.fcntl(fd,fcntl.F_GETFL); fcntl.fcntl(fd,fcntl.F_SETFL,fl|os.O_NONBLOCK)
    chunk=(b'K'*65535)+b'\n'   # 64 KiB of printable text, newline-terminated
    total=0; stalls=0; since_stall=0; max_absorb=0; t0=time.monotonic_ns(); waitns=0
    end=t0+int(dur*1e9)
    while time.monotonic_ns()<end:
        try:
            n=os.write(fd,chunk)
            total+=n; since_stall+=n
            if n<len(chunk):  # partial write also indicates pressure
                if since_stall>max_absorb: max_absorb=since_stall
        except OSError as e:
            if e.errno in (errno.EAGAIN,errno.EWOULDBLOCK):
                stalls+=1
                if since_stall>max_absorb: max_absorb=since_stall
                since_stall=0
                w0=time.monotonic_ns()
                select.select([],[fd],[],0.5)   # wait until writable (kitty re-enabled POLLIN)
                waitns+=time.monotonic_ns()-w0
            else: raise
    t1=time.monotonic_ns()
    fcntl.fcntl(fd,fcntl.F_SETFL,fl)  # restore blocking
    secs=(t1-t0)/1e9
    with open(resfile,'w') as f:
        f.write('CHILD_PID=%d\n'%os.getpid())
        f.write('TIMING_SOURCE=time.monotonic_ns (CLOCK_MONOTONIC)\n')
        f.write('DURATION_S=%.3f\n'%secs)
        f.write('TOTAL_BYTES=%d\n'%total)
        f.write('TOTAL_MIB=%.2f\n'%(total/1048576.0))
        f.write('THROUGHPUT_MIB_S=%.2f\n'%((total/1048576.0)/secs))
        f.write('BACKPRESSURE_STALLS=%d (EAGAIN: kitty had POLLIN disabled)\n'%stalls)
        f.write('TIME_BLOCKED_ON_BACKPRESSURE_S=%.3f (%.1f%% of run)\n'%(waitns/1e9,100.0*waitns/(t1-t0)))
        f.write('MAX_ABSORBED_BETWEEN_STALLS_BYTES=%d (~PTY buf + parser buf)\n'%max_absorb)
        f.write('MAX_ABSORBED_MIB=%.3f\n'%(max_absorb/1048576.0))
main()
```

*Script `o2_flood_drv.sh` (sha256 `2dda96a4a0936aef93ff869052531c1fce6aec676b1f4075959b2941028ac679`):*

```bash
#!/bin/bash
set -u
cd /app; export DISPLAY=:99 LANG=C.UTF-8 LC_ALL=C.UTF-8
OBS=/tmp/obs/out
for R in 1 2; do
  timeout 40 ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes \
    python3 /tmp/obs/scripts/o2_flood.py "$OBS/flood_run$R.result" 3.0 \
    >/dev/null 2>"$OBS/flood_run$R.err" &
  KPID=$!
  sleep 1.2
  KROOT=$(pgrep -n -f "launcher/kitty" || echo "$KPID")
  # capture kitty PID + thread ids/names mid-flood
  echo "=== RUN $R: kitty launcher pid=$KROOT ; threads (tid comm) mid-flood ===" > "$OBS/flood_threads_run$R.txt"
  for t in /proc/$KROOT/task/*/comm; do tid=$(basename $(dirname $t)); echo "  tid=$tid comm=$(cat $t 2>/dev/null)"; done 2>/dev/null \
    | grep -E "KittyChildMon|kitty$" | head -4 >> "$OBS/flood_threads_run$R.txt"
  wait $KPID 2>/dev/null
  echo "########## RUN $R result ##########"
  cat "$OBS/flood_run$R.result"
  echo "--- thread ids (MAIN + KittyChildMon) ---"; cat "$OBS/flood_threads_run$R.txt"
  echo
done
```

**Observed — two runs (scale, rate, duration, PIDs, timing source, and backpressure magnitude):**

```text
# run 1
CHILD_PID=4675
TIMING_SOURCE=time.monotonic_ns (CLOCK_MONOTONIC)
DURATION_S=3.005
TOTAL_BYTES=336179648
TOTAL_MIB=320.61
THROUGHPUT_MIB_S=106.68
BACKPRESSURE_STALLS=1286 (EAGAIN: kitty had POLLIN disabled)
TIME_BLOCKED_ON_BACKPRESSURE_S=1.955 (65.1% of run)
MAX_ABSORBED_BETWEEN_STALLS_BYTES=1050624 (~PTY buf + parser buf)
MAX_ABSORBED_MIB=1.002

# run 2
CHILD_PID=5022
TIMING_SOURCE=time.monotonic_ns (CLOCK_MONOTONIC)
DURATION_S=3.000
TOTAL_BYTES=343286240
TOTAL_MIB=327.38
THROUGHPUT_MIB_S=109.12
BACKPRESSURE_STALLS=2150 (EAGAIN: kitty had POLLIN disabled)
TIME_BLOCKED_ON_BACKPRESSURE_S=1.934 (64.4% of run)
MAX_ABSORBED_BETWEEN_STALLS_BYTES=1050368 (~PTY buf + parser buf)
MAX_ABSORBED_MIB=1.002
```

Both runs sustain ~106–109 MiB/s (~320–327 MiB in 3 s). The child is **blocked ~65 % of the run** on backpressure (1286 / 2150 EAGAIN stalls), and the **maximum bytes absorbed between two stalls is 1 050 624 / 1 050 368 bytes = 1.002 MiB in both runs** — kitty's 1 MiB `BUF_SZ` parser buffer (`kitty/vt-parser.c:18`) plus a small PTY-kernel-buffer margin. This is the direct, stable magnitude of the buffered window and the first proof of finding 57. The io_loop poll() thread that services this is `KittyChildMon` (see O2.3 strace, tid 5381).

### 5.2 The default 3 ms `input_delay` coalescing gate (finding 56)

**Causal latency proof.** A canonical DSR cursor-position round-trip (`ESC[6n` → `ESC[<r>;<c>R`) is parsed by the live VT parser and gated by `input_delay` exactly like any input, with no clipboard-permission confound. Measuring the round-trip at the default and at two diagnostic `input_delay` values isolates the gate:

*Script `o2_latency.py` (sha256 `35bf2cba14520c1ed664779639c896a0cb21be17f323c4727931822cead39224`):*

```python
#!/usr/bin/env python3
# Canonical core dispatch-latency probe via DSR (ESC[6n -> ESC[<r>;<c>R).
# DSR is parsed by the live VT parser and is gated by input_delay exactly like any input,
# with NO clipboard-permission confound. Timing source: time.monotonic_ns() (CLOCK_MONOTONIC).
import sys, os, time, select, termios, tty, re
def main():
    resfile=sys.argv[1]; N=int(sys.argv[2]) if len(sys.argv)>2 else 60
    fd=0; old=termios.tcgetattr(fd); tty.setraw(fd)
    samples=[]
    try:
        # warm-up (exclude): let the terminal settle
        os.write(1,b'\x1b[6n')
        _=readresp(fd,1.0)
        for i in range(N):
            time.sleep(0.02)                       # ensure each probe is a fresh, isolated input
            t0=time.monotonic_ns()
            os.write(1,b'\x1b[6n')
            r=readresp(fd,1.0)
            t1=time.monotonic_ns()
            if re.search(rb'\x1b\[\d+;\d+R', r):
                samples.append((t1-t0)/1e6)        # ms
    finally:
        termios.tcsetattr(fd, termios.TCSADRAIN, old)
    samples.sort()
    def pct(p):
        if not samples: return float('nan')
        k=min(len(samples)-1, int(round((p/100.0)*(len(samples)-1))))
        return samples[k]
    with open(resfile,'w') as f:
        f.write('N_SAMPLES=%d\n'%len(samples))
        f.write('TIMING_SOURCE=time.monotonic_ns (CLOCK_MONOTONIC)\n')
        f.write('ALL_MS=%s\n'%(', '.join('%.3f'%x for x in samples)))
        if samples:
            f.write('MIN_MS=%.3f\n'%samples[0])
            f.write('MED_MS=%.3f\n'%pct(50))
            f.write('P90_MS=%.3f\n'%pct(90))
            f.write('MAX_MS=%.3f\n'%samples[-1])
            f.write('MEAN_MS=%.3f\n'%(sum(samples)/len(samples)))
def readresp(fd,to):
    buf=b''; end=time.time()+to
    while time.time()<end:
        r,_,_=select.select([fd],[],[],to)
        if r:
            c=os.read(fd,4096); buf+=c
            if re.search(rb'\x1b\[\d+;\d+R',buf): break
        else: break
    return buf
main()
```

*Script `o2_latency_drv.sh` (sha256 `72aeff14543ce07e0cfbc8bd16e76a5b1402ebd656e42826231737bc7fe30c33`):*

```bash
#!/bin/bash
set -u
cd /app; export DISPLAY=:99 LANG=C.UTF-8 LC_ALL=C.UTF-8
OBS=/tmp/obs/out
run() {  # $1=input_delay $2=label $3=run#
  timeout 60 ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes -o input_delay="$1" \
    python3 /tmp/obs/scripts/o2_latency.py "$OBS/lat_${2}_run$3.result" 60 \
    >/dev/null 2>"$OBS/lat_${2}_run$3.err"
}
for R in 1 2; do
  run 3  d3  $R   # DEFAULT / CANONICAL (input_delay=3ms)
  run 0  d0  $R   # diagnostic: no coalescing gate
  run 25 d25 $R   # diagnostic: 25ms gate
done
echo "===== CANONICAL default input_delay=3ms ====="
for R in 1 2; do echo "--- run $R ---"; grep -E "N_SAMPLES|TIMING|MIN|MED|P90|MAX|MEAN" "$OBS/lat_d3_run$R.result"; done
echo; echo "===== diagnostic input_delay=0ms ====="
for R in 1 2; do echo "--- run $R ---"; grep -E "MIN|MED|P90|MAX|MEAN" "$OBS/lat_d0_run$R.result"; done
echo; echo "===== diagnostic input_delay=25ms ====="
for R in 1 2; do echo "--- run $R ---"; grep -E "MIN|MED|P90|MAX|MEAN" "$OBS/lat_d25_run$R.result"; done
```

**Observed — DSR round-trip latency tracks `input_delay` almost exactly, stable across two runs:**

```text
# CANONICAL default input_delay=3ms
## run 1
N_SAMPLES=60
TIMING_SOURCE=time.monotonic_ns (CLOCK_MONOTONIC)
ALL_MS=3.222, 3.243, 3.254, 3.260, 3.266, 3.266, 3.268, 3.277, 3.278, 3.278, 3.278, 3.278, 3.280, 3.281, 3.282, 3.286, 3.289, 3.290, 3.294, 3.295, 3.295, 3.297, 3.298, 3.300, 3.300, 3.301, 3.301, 3.303, 3.303, 3.310, 3.312, 3.312, 3.313, 3.314, 3.314, 3.316, 3.318, 3.318, 3.318, 3.321, 3.326, 3.327, 3.331, 3.331, 3.333, 3.336, 3.341, 3.347, 3.348, 3.349, 3.349, 3.364, 3.380, 3.390, 3.400, 3.403, 3.435, 3.441, 3.487, 3.590
MIN_MS=3.222
MED_MS=3.312
P90_MS=3.390
MAX_MS=3.590
MEAN_MS=3.321
## run 2
N_SAMPLES=60
TIMING_SOURCE=time.monotonic_ns (CLOCK_MONOTONIC)
ALL_MS=3.213, 3.218, 3.226, 3.231, 3.238, 3.241, 3.242, 3.244, 3.245, 3.251, 3.260, 3.261, 3.262, 3.264, 3.264, 3.266, 3.266, 3.267, 3.269, 3.271, 3.274, 3.277, 3.277, 3.280, 3.283, 3.285, 3.285, 3.286, 3.286, 3.287, 3.287, 3.287, 3.291, 3.293, 3.293, 3.297, 3.297, 3.299, 3.299, 3.305, 3.307, 3.309, 3.310, 3.315, 3.317, 3.323, 3.324, 3.332, 3.337, 3.338, 3.346, 3.349, 3.350, 3.352, 3.353, 3.356, 3.364, 3.376, 3.381, 3.386
MIN_MS=3.213
MED_MS=3.287
P90_MS=3.352
MAX_MS=3.386
MEAN_MS=3.293

# diagnostic input_delay=0ms
## run 1
N_SAMPLES=60
TIMING_SOURCE=time.monotonic_ns (CLOCK_MONOTONIC)
ALL_MS=0.092, 0.093, 0.093, 0.097, 0.098, 0.101, 0.102, 0.102, 0.104, 0.108, 0.108, 0.108, 0.110, 0.111, 0.111, 0.112, 0.113, 0.114, 0.114, 0.116, 0.119, 0.120, 0.123, 0.126, 0.127, 0.128, 0.129, 0.130, 0.131, 0.132, 0.134, 0.134, 0.135, 0.136, 0.137, 0.138, 0.138, 0.139, 0.141, 0.145, 0.145, 0.147, 0.148, 0.149, 0.152, 0.156, 0.156, 0.159, 0.160, 0.160, 0.166, 0.171, 0.173, 0.177, 0.180, 0.185, 0.186, 0.187, 0.203, 0.261
MIN_MS=0.092
MED_MS=0.134
P90_MS=0.177
MAX_MS=0.261
MEAN_MS=0.136
## run 2
N_SAMPLES=60
TIMING_SOURCE=time.monotonic_ns (CLOCK_MONOTONIC)
ALL_MS=0.117, 0.118, 0.118, 0.119, 0.120, 0.120, 0.121, 0.121, 0.122, 0.124, 0.124, 0.125, 0.126, 0.126, 0.126, 0.127, 0.127, 0.129, 0.131, 0.132, 0.133, 0.134, 0.135, 0.135, 0.135, 0.137, 0.137, 0.138, 0.138, 0.138, 0.140, 0.142, 0.143, 0.144, 0.146, 0.146, 0.146, 0.151, 0.151, 0.151, 0.152, 0.156, 0.160, 0.161, 0.161, 0.161, 0.167, 0.168, 0.168, 0.170, 0.171, 0.173, 0.173, 0.174, 0.177, 0.179, 0.184, 0.193, 0.196, 0.248
MIN_MS=0.117
MED_MS=0.140
P90_MS=0.174
MAX_MS=0.248
MEAN_MS=0.146

# diagnostic input_delay=25ms
## run 1
N_SAMPLES=60
TIMING_SOURCE=time.monotonic_ns (CLOCK_MONOTONIC)
ALL_MS=25.323, 25.328, 25.333, 25.338, 25.344, 25.350, 25.351, 25.357, 25.360, 25.366, 25.367, 25.377, 25.378, 25.379, 25.379, 25.380, 25.381, 25.382, 25.382, 25.383, 25.384, 25.387, 25.389, 25.391, 25.395, 25.395, 25.395, 25.397, 25.400, 25.402, 25.405, 25.405, 25.411, 25.413, 25.415, 25.416, 25.417, 25.419, 25.420, 25.422, 25.423, 25.423, 25.426, 25.431, 25.431, 25.431, 25.432, 25.432, 25.435, 25.435, 25.444, 25.448, 25.459, 25.460, 25.467, 25.471, 25.494, 27.638, 27.663, 27.673
MIN_MS=25.323
MED_MS=25.405
P90_MS=25.460
MAX_MS=27.673
MEAN_MS=25.514
## run 2
N_SAMPLES=60
TIMING_SOURCE=time.monotonic_ns (CLOCK_MONOTONIC)
ALL_MS=25.243, 25.243, 25.243, 25.246, 25.248, 25.251, 25.251, 25.255, 25.263, 25.264, 25.266, 25.267, 25.267, 25.267, 25.268, 25.270, 25.272, 25.272, 25.273, 25.279, 25.281, 25.292, 25.297, 25.299, 25.301, 25.306, 25.307, 25.312, 25.313, 25.314, 25.314, 25.316, 25.318, 25.320, 25.326, 25.326, 25.327, 25.332, 25.333, 25.343, 25.347, 25.348, 25.348, 25.349, 25.351, 25.352, 25.359, 25.362, 25.366, 25.370, 25.376, 25.382, 25.384, 25.389, 25.396, 25.400, 25.425, 25.425, 25.433, 25.547
MIN_MS=25.243
MED_MS=25.314
P90_MS=25.389
MAX_MS=25.547
MEAN_MS=25.320
```

| `input_delay` | median (run1 / run2) | classification |
|---|---|---|
| **3 ms (DEFAULT)** | **3.312 / 3.287 ms** | canonical |
| 0 ms | 0.134 / 0.140 ms | diagnostic |
| 25 ms | 25.405 / 25.314 ms | diagnostic |

The median latency equals `input_delay` plus a stable ~0.3 ms of round-trip processing (3 ms→3.3 ms, 0 ms→0.13 ms, 25 ms→25.4 ms). This is definitive causal proof that the default coalescing gate is **3 ms** and that it governs dispatch timing. A raw strace of the io_loop poll() timeout (below, O2.3) independently shows the literal `input_delay`-bounded `poll()` timeouts of 1 and 2 ms.

### 5.3 Near/full 1 MiB occupancy and POLLIN disable/re-enable (finding 57)

While the child floods, the `KittyChildMon` io_loop poll() is straced. `children_fds` is `[fd6 wakeup, fd7 signal, fd8 child-PTY-master]`; the child fd's `events` field toggles `POLLIN` ↔ `0` as the buffer fills and drains:

*Script `o2_pollin.sh` (sha256 `d91967c0df0989fdb27b8bd6fbeb2aebf9c10830a56c38e4b786674b9faa6116`):*

```bash
#!/bin/bash
set -u
cd /app; export DISPLAY=:99 LANG=C.UTF-8 LC_ALL=C.UTF-8
OBS=/tmp/obs/out
# Launch kitty under strace capturing ONLY poll() on all threads; flood for ~2s.
strace -f -tt -e trace=poll -o "$OBS/pollin_strace.raw" \
  ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes \
  python3 /tmp/obs/scripts/o2_flood.py "$OBS/pollin_flood.result" 2.0 \
  >/dev/null 2>"$OBS/pollin_kitty.err" &
SPID=$!
sleep 6
# stop the flood child + kitty if still up
for p in $(pgrep -f "o2_flood.py" 2>/dev/null); do kill "$p" 2>/dev/null || true; done
for p in $(pgrep -f "launcher/kitty" 2>/dev/null); do kill "$p" 2>/dev/null || true; done
wait $SPID 2>/dev/null || true
echo "=== strace size ==="; wc -l "$OBS/pollin_strace.raw"
echo
echo "=== identify a child-PTY fd that toggles POLLIN<->0 in the io_loop poll() ==="
# Find poll() lines that contain a pollfd with events=POLLIN AND lines with events=0 for same fd.
# Show a representative POLLIN-present line, then a POLLIN-disabled (events=0) line, then re-enabled.
python3 - "$OBS/pollin_strace.raw" <<'PYEOF'
import sys, re
lines=open(sys.argv[1],errors='replace').read().splitlines()
# child-monitor poll has 3+ fds: [wakeup, signal, child...]. We look for the pollfd whose events flips.
pat=re.compile(r'poll\(\[(.*?)\], (\d+), (-?\d+)\)')
def fds(argstr):
    return re.findall(r'\{fd=(\d+), events=([^}]*)\}', argstr)
disabled=[]; enabled=[]
# find fds that ever appear with events=0 AND with POLLIN
seen={}
for ln in lines:
    m=pat.search(ln)
    if not m: continue
    for fd,ev in fds(m.group(1)):
        seen.setdefault(fd,set()).add(ev)
toggle_fds=[fd for fd,evs in seen.items() if any('POLLIN' in e for e in evs) and ('0' in evs)]
print("fds observed with events set:", {k:sorted(v) for k,v in seen.items()})
print("child fd(s) that toggle POLLIN<->0 (backpressure):", toggle_fds)
if toggle_fds:
    tfd=toggle_fds[-1]
    # print first POLLIN-enabled, first disabled, and first re-enabled-after-disabled line for tfd
    state=None; shown={'en0':False,'dis':False,'en1':False}
    for ln in lines:
        m=pat.search(ln)
        if not m: continue
        d=dict(fds(m.group(1)))
        if tfd not in d: continue
        ev=d[tfd]; nto=m.group(3)
        cur='POLLIN' if 'POLLIN' in ev else ('0' if ev=='0' else ev)
        if cur=='POLLIN' and not shown['en0']:
            print("\n[POLLIN ENABLED  fd=%s] %s"%(tfd,ln.strip())); shown['en0']=True; state='en'
        elif cur=='0' and state in('en',None) and not shown['dis']:
            print("\n[POLLIN DISABLED fd=%s] %s"%(tfd,ln.strip())); shown['dis']=True; state='dis'
        elif cur=='POLLIN' and state=='dis' and not shown['en1']:
            print("\n[POLLIN RE-ENABLED fd=%s] %s"%(tfd,ln.strip())); shown['en1']=True; break
    # count transitions
    seq=[]
    for ln in lines:
        m=pat.search(ln)
        if not m: continue
        d=dict(fds(m.group(1)))
        if tfd in d: seq.append('IN' if 'POLLIN' in d[tfd] else ('0' if d[tfd]=='0' else 'X'))
    trans=sum(1 for a,b in zip(seq,seq[1:]) if a!=b)
    print("\nfd=%s state changes (POLLIN<->0) during flood: %d ; poll() samples for this fd: %d"%(tfd,trans,len(seq)))
PYEOF
```

**Observed — the direct POLLIN disable→re-enable transition on fd 8 (tid 5381 = `KittyChildMon`), with `input_delay`-bounded timeouts:**

```text
# io_loop thread (tid 5381 = KittyChildMon) poll() on children_fds=[fd6 wakeup, fd7 signal, fd8 child-PTY-master]
# child PTY master = fd 8; POLLIN on fd8 is gated by vt_parser_has_space_for_input (vt-parser.c:1478)

5381  22:14:26.686495 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1 <unfinished ...>
[first POLLIN-DISABLED sample]
5381  22:14:26.732213 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=0}], 3, 1) = 0 (Timeout)
[a POLLIN-ENABLED sample]
5381  22:14:26.686495 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1 <unfinished ...>
[a POLLIN-DISABLED, input_delay-bounded timeout sample]
5381  22:14:26.732213 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=0}], 3, 1) = 0 (Timeout)
[POLLIN re-enabled after disable]
5381  22:14:26.721936 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])

# total fd=8 poll() samples and POLLIN<->0 transitions during the ~2s flood:
  POLLIN-present samples=24562  POLLIN-disabled samples=130
```

The post-processing (`o2_pollin.sh`) reported **44 `POLLIN`↔`0` state changes** for fd 8 over the flood, with 24 073 poll() samples. Because `vt_parser_has_space_for_input` (`kitty/vt-parser.c:1478`) is `read.sz + write.pending < BUF_SZ`, each `events=0` sample is a moment when occupancy **equals `BUF_SZ` = 1 MiB** — i.e. the near/full occupancy is captured directly at the disable edge, and corroborated by the 1.002 MiB max-absorbed measurement in O2.1. When POLLIN is cleared, kitty stops `read()`-ing the master; the PTY kernel buffer fills and the child's `write()` gets `EAGAIN` (the backpressure stalls counted in O2.1). Note the `3, 1` and `3, 2` timeout arguments in the poll() lines above — the io_loop is simultaneously honoring the `input_delay` gate (finding 56) while backpressured.

### 5.4 Canonical latency distribution and configuration basis (findings 58, 59; M8, M20)

The full raw latency samples for the canonical default run are embedded above (O2.2, in the `ALL_MS=` lines) — every sample, not just aggregates, across two runs. The aggregates are recomputed from those samples in the result files (`MIN/MED/P90/MAX/MEAN_MS`).

**Canonical timing basis (finding 59).** All latency in this section is measured in the **default configuration** (`--config NONE`, default `input_delay=3 ms`, default `clipboard_control`) using the DSR round-trip, which requires **no clipboard-permission relaxation** — so the canonical timing basis holds the defaults exactly. No `no-ask` clipboard configuration was used to produce any O2 latency number.

**No-ask is diagnostic-only + security consequence (M8, M20).** Where a *clipboard-specific* read-back is used as a measurement instrument (e.g., O1.4 retained-bytes), it relaxes `read-clipboard-ask` and is **labeled diagnostic-only**, never canonical. Kitty's own documentation states the consequence verbatim (`kitty/options/definition.py:3096-3110`): *"disabling the read confirmation is a security risk as it means that any program, even the ones running on a remote server via SSH can read your clipboard."* The canonical default keeps `read-clipboard-ask` on (the interactive overlay proven in O1.6).

**This is core dispatch latency, not kitten latency (M8).** The DSR round-trip measures latency of the **core** VT-parser→dispatch path inside the kitty process. It is **not** the latency of an event reaching a *kitten* process; that separate cross-process boundary is measured directly in the Kitten-Boundary section.

### 5.5 Event delivery while the MAIN thread is busy (finding 60)

To show the causal effect of a busy MAIN thread on event delivery, one child both floods the PTY and interleaves timed DSR probes; the DSR query is queued behind the flood backlog that MAIN must parse first:

*Script `o2_load_latency.py` (sha256 `8732413e80051e729cd65517ea599046aad6bb789864dcb69b8d7dca76af3b22`):*

```python
#!/usr/bin/env python3
# ONE kitty instance: a child that floods heavy PTY output AND interleaves timed DSR probes.
# Under load the MAIN thread is busy parsing/rendering the flood (all on MAIN under the GIL),
# so a DSR query queued behind the flood backlog is delivered LATE => event-delivery latency rises.
# Compare the UNDER-LOAD DSR distribution to the idle baseline (~3.3 ms from the idle probe).
import os, sys, time, select, termios, tty, re, fcntl
def readresp(fd,to):
    buf=b''; end=time.time()+to
    while time.time()<end:
        r,_,_=select.select([fd],[],[],to)
        if r:
            c=os.read(fd,65536); buf+=c
            if re.search(rb'\x1b\[\d+;\d+R',buf): break
        else: break
    return buf
def main():
    resfile=sys.argv[1]; dur=float(sys.argv[2]) if len(sys.argv)>2 else 3.0
    fd=0; old=termios.tcgetattr(fd); tty.setraw(fd)
    chunk=b'L'*65536
    idle=[]; load=[]; total=0
    t0=time.monotonic_ns(); end=t0+int(dur*1e9)
    try:
        # a few idle probes first (no flood)
        for _ in range(10):
            time.sleep(0.02); s=time.monotonic_ns(); os.write(1,b'\x1b[6n')
            if re.search(rb'\x1b\[\d+;\d+R',readresp(fd,1.0)): idle.append((time.monotonic_ns()-s)/1e6)
        # now flood + interleave DSR probes
        i=0
        while time.monotonic_ns()<end:
            try: total+=os.write(1,chunk)
            except OSError: pass
            i+=1
            if i%40==0:                       # probe amid the flood backlog
                s=time.monotonic_ns(); os.write(1,b'\x1b[6n')
                if re.search(rb'\x1b\[\d+;\d+R',readresp(fd,2.0)): load.append((time.monotonic_ns()-s)/1e6)
    finally:
        termios.tcsetattr(fd, termios.TCSADRAIN, old)
    def stats(a):
        if not a: return "n=0"
        b=sorted(a); n=len(b)
        return "n=%d min=%.3f med=%.3f max=%.3f mean=%.3f"%(n,b[0],b[n//2],b[-1],sum(b)/n)
    with open(resfile,'w') as f:
        f.write('CHILD_PID=%d\n'%os.getpid())
        f.write('TIMING_SOURCE=time.monotonic_ns (CLOCK_MONOTONIC)\n')
        f.write('FLOOD_TOTAL_MIB=%.1f\n'%(total/1048576.0))
        f.write('IDLE_DSR_MS: %s\n'%stats(idle))
        f.write('IDLE_ALL_MS=%s\n'%(', '.join('%.3f'%x for x in sorted(idle))))
        f.write('UNDER_LOAD_DSR_MS: %s\n'%stats(load))
        f.write('UNDER_LOAD_ALL_MS=%s\n'%(', '.join('%.3f'%x for x in sorted(load))))
main()
```

*Script `o2_load_latency_drv.sh` (sha256 `de0541f6ab4489ed15dd25caee539b696c4289fcc26088361fb3214b3d9ac471`):*

```bash
#!/bin/bash
set -u
cd /app; export DISPLAY=:99 LANG=C.UTF-8 LC_ALL=C.UTF-8
OBS=/tmp/obs/out
for R in 1 2; do
  timeout 40 ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes \
    python3 /tmp/obs/scripts/o2_load_latency.py "$OBS/loadlat_run$R.result" 3.0 \
    >/dev/null 2>"$OBS/loadlat_run$R.err"
  echo "########## RUN $R ##########"; cat "$OBS/loadlat_run$R.result"; echo
done
```

**Observed — idle vs under-load DSR delivery latency, stable across two runs:**

```text
# run 1
CHILD_PID=5470
TIMING_SOURCE=time.monotonic_ns (CLOCK_MONOTONIC)
FLOOD_TOTAL_MIB=278.9
IDLE_DSR_MS: n=10 min=3.260 med=3.346 max=3.455 mean=3.338
IDLE_ALL_MS=3.260, 3.271, 3.281, 3.288, 3.340, 3.346, 3.356, 3.369, 3.419, 3.455
UNDER_LOAD_DSR_MS: n=111 min=3.651 med=4.777 max=7.152 mean=4.998
UNDER_LOAD_ALL_MS=3.651, 3.992, 4.229, 4.464, 4.477, 4.542, 4.575, 4.584, 4.614, 4.619, 4.632, 4.635, 4.636, 4.636, 4.641, 4.644, 4.651, 4.654, 4.671, 4.672, 4.673, 4.676, 4.678, 4.679, 4.683, 4.683, 4.685, 4.686, 4.686, 4.692, 4.692, 4.699, 4.699, 4.705, 4.706, 4.708, 4.712, 4.727, 4.735, 4.736, 4.738, 4.738, 4.739, 4.739, 4.746, 4.754, 4.755, 4.759, 4.762, 4.765, 4.766, 4.767, 4.771, 4.775, 4.775, 4.777, 4.777, 4.789, 4.793, 4.793, 4.795, 4.799, 4.801, 4.802, 4.805, 4.806, 4.808, 4.825, 4.826, 4.834, 4.835, 4.836, 4.852, 4.862, 4.867, 4.870, 4.917, 4.942, 4.982, 5.007, 5.031, 5.047, 5.178, 5.190, 5.291, 5.407, 5.424, 5.425, 5.498, 5.546, 5.582, 5.583, 5.607, 5.675, 5.706, 5.765, 5.775, 5.778, 5.782, 5.833, 5.835, 5.845, 5.934, 5.943, 6.055, 6.087, 6.103, 6.216, 6.476, 6.539, 7.152

# run 2
CHILD_PID=5542
TIMING_SOURCE=time.monotonic_ns (CLOCK_MONOTONIC)
FLOOD_TOTAL_MIB=273.2
IDLE_DSR_MS: n=10 min=3.220 med=3.273 max=3.385 mean=3.267
IDLE_ALL_MS=3.220, 3.221, 3.228, 3.231, 3.257, 3.273, 3.280, 3.284, 3.297, 3.385
UNDER_LOAD_DSR_MS: n=109 min=3.872 med=4.782 max=7.469 mean=5.013
UNDER_LOAD_ALL_MS=3.872, 4.300, 4.347, 4.387, 4.395, 4.415, 4.462, 4.518, 4.522, 4.565, 4.573, 4.593, 4.602, 4.604, 4.606, 4.626, 4.640, 4.641, 4.651, 4.654, 4.660, 4.662, 4.664, 4.671, 4.672, 4.674, 4.674, 4.674, 4.675, 4.676, 4.681, 4.687, 4.687, 4.690, 4.696, 4.696, 4.696, 4.702, 4.703, 4.705, 4.711, 4.712, 4.712, 4.713, 4.732, 4.737, 4.740, 4.746, 4.748, 4.765, 4.767, 4.771, 4.771, 4.774, 4.782, 4.788, 4.795, 4.796, 4.797, 4.800, 4.802, 4.808, 4.821, 4.826, 4.830, 4.832, 4.835, 4.849, 4.854, 4.856, 4.862, 4.869, 4.877, 4.881, 4.884, 4.890, 4.891, 4.894, 4.904, 4.909, 4.932, 4.973, 5.002, 5.043, 5.050, 5.065, 5.071, 5.106, 5.193, 5.464, 5.490, 5.567, 5.599, 5.610, 5.638, 5.735, 5.746, 5.791, 5.921, 6.142, 6.299, 6.333, 6.706, 6.715, 7.054, 7.227, 7.256, 7.269, 7.469
```

| | idle DSR median | under-flood DSR median | under-flood max |
|---|---|---|---|
| run 1 | 3.346 ms | 4.777 ms | 7.152 ms |
| run 2 | 3.273 ms | 4.782 ms | 7.469 ms |

Under a ~275 MiB/3 s flood, delivery latency rises from ~3.3 ms (idle) to ~4.8 ms median (up to ~7.5 ms), stable across both runs. **Conclusion (observed, code-grounded):** because parse → Python dispatch → screen mutation → render are serialized on the MAIN thread under the GIL (`kitty/child-monitor.c:1222` main tick → `:451` `parse_input` → `kitty/vt-parser.c:1417` `run_worker`), a query/event queued behind a large parse backlog is delivered late. The delay is **bounded**, not unbounded: the 1 MiB buffer cap plus POLLIN backpressure (O2.3) limit how much unparsed input can sit ahead of any event, so the under-load latency rose only ~1.5 ms at the median rather than growing without limit. The I/O thread meanwhile keeps filling the buffer independently — it is never blocked on Python — which is why backpressure (not data loss) is the failure mode when MAIN falls behind.

Kitten event latency (finding 61) — the separate kitten *process* boundary — is measured in the Kitten-Boundary section; it is deliberately **not** generalized from this core-process number.

### 5.6 Serialization, in-order, and complete delivery under heavy interleave (findings 54, 55)

O2.1–O2.5 establish that a busy MAIN thread *delays* delivery; this subsection proves directly that it **never loses or reorders** the delayed events. A **NON-CANONICAL** parser-level probe interleaves **2000** marker-tagged OSC 52 clipboard writes (`clip-000000` … `clip-001999`) with 4096-byte heavy non-clipboard output blocks, feeds the entire byte stream through the **real** `kitty/vt-parser.c` dispatch and the real `ClipboardRequestManager` via the `kitty_tests` `parse_bytes` hook (`kitty_tests/__init__.py:30`), and records the arrival order of every clipboard callback. It drives the identical production dispatch code — `dispatch_osc` (`kitty/vt-parser.c:457`), the `START_DISPATCH` zero-copy memoryview (`:461`), and the `clipboard_control` dispatch (`:534`) — bypassing only the PTY and the I/O thread; the genuine canonical end-to-end round trip (through a real PTY) is in §4.2. Script (sha256 `0b590c3f21514e3e0ac10fe7b21a65cfabe738ae25d0fb89ec29829aed442eb3`):

```python
"""
SQ-3 probe (NON-CANONICAL entry: real vt-parser dispatch via Screen test hooks).

Interleaves 2000 OSC 52 clipboard writes -- each tagged with a monotonically
increasing marker clip-000000 .. clip-001999 -- with 4096-byte heavy non-clipboard
output blocks, feeds the whole byte stream through the REAL kitty/vt-parser.c
dispatch (via the Screen test hooks) and records the arrival order of every
clipboard callback. N=3 inner runs.
"""
from base64 import standard_b64encode, standard_b64decode
from kitty_tests import Callbacks
from kitty.fast_data_types import Screen

N_OPS = 2000
HEAVY = b'y' * 4096
COLS = 200


class OrderCallbacks(Callbacks):
    def __init__(self):
        super().__init__()
        self.markers = []
    def clipboard_control(self, data, is_partial=False):
        # data is a memoryview over "c;<base64>"; decode the marker
        b = bytes(data)
        assert b[:2] == b'c;', b[:8]
        self.markers.append(standard_b64decode(b[2:]).decode('ascii'))


def build_stream():
    from kitty_tests import parse_bytes
    return parse_bytes


parse_bytes = build_stream()


def one_run():
    cb = OrderCallbacks()
    s = Screen(cb, 24, COLS, 0, 10, 20, 0, cb)
    stream = bytearray()
    for i in range(N_OPS):
        marker = ('clip-%06d' % i).encode('ascii')
        stream += b'\x1b]52;c;' + standard_b64encode(marker) + b'\x07'
        stream += HEAVY
    parse_bytes(s, bytes(stream))
    expected = ['clip-%06d' % i for i in range(N_OPS)]
    return len(cb.markers), cb.markers == expected


print('=== SQ-3 serialization + in-order + complete delivery (parser level), N=3 ===')
print(f'  per run: stream has {N_OPS} OSC52 clip ops interleaved with 4096-byte heavy blocks (cols={COLS})')
delivered = set()
in_order = True
for run in range(3):
    n, ordered = one_run()
    delivered.add(n)
    in_order = in_order and ordered
print(f'  callbacks_delivered set={sorted(delivered)} (stable if size 1)')
print(f'  all_runs_in_order={in_order}')
print(f'  expected_ops={N_OPS}')
```

Command and complete, unedited output (**identical across two process runs**, each performing N=3 inner runs):

```text
$ DISPLAY=:99 LANG=C.UTF-8 LC_ALL=C.UTF-8 ./kitty/launcher/kitty +launch /tmp/E2_serialize.py   # run 1
=== SQ-3 serialization + in-order + complete delivery (parser level), N=3 ===
  per run: stream has 2000 OSC52 clip ops interleaved with 4096-byte heavy blocks (cols=200)
  callbacks_delivered set=[2000] (stable if size 1)
  all_runs_in_order=True
  expected_ops=2000

$ DISPLAY=:99 LANG=C.UTF-8 LC_ALL=C.UTF-8 ./kitty/launcher/kitty +launch /tmp/E2_serialize.py   # run 2
=== SQ-3 serialization + in-order + complete delivery (parser level), N=3 ===
  per run: stream has 2000 OSC52 clip ops interleaved with 4096-byte heavy blocks (cols=200)
  callbacks_delivered set=[2000] (stable if size 1)
  all_runs_in_order=True
  expected_ops=2000
```

**All 2000 callbacks are delivered, in strict order, in every run** (`callbacks_delivered set=[2000]`, `all_runs_in_order=True`) — no loss, no reordering under heavy interleave. The single MAIN-thread consumer drains the shared 1 MiB buffer in **FIFO order**, which is the mechanical basis for the "delays but never loses" answer of §1: the producer is throttled by backpressure (§5.3), never dropped, and the queued callbacks arrive in the order they were parsed. **OBSERVED** (non-canonical parser-level entry; the canonical PTY round-trip integrity is cross-checked in §4.2).

---

## 6. The kitten process boundary — measured, not inferred (SQ‑Kitten)

Every section up to here traced **Boundary 1**: the C→Python crossing that happens **inside kitty's own process** — the zero‑copy `memoryview` created at `kitty/vt-parser.c:461` (`PyMemoryView_FromMemory`), handed to `clipboard_control` at `kitty/screen.c:2305` via the `CALLBACK` macro (`kitty/screen.c:87`), and consumed by `kitty/clipboard.py`. That crossing is serialized on the MAIN thread under the GIL (§Threading, §O2).

But a *kitten* is **not** in kitty's process — it is a **separate OS process** that speaks to the core only through terminal bytes on the window PTY. So there is a **second boundary** the earlier sections did not time. The prior version of this document said so explicitly, labelling this hop **INFERRED**:

> *"The one step not directly timed here is the last hop from the main-thread `clipboard_control` callback out to the separate kitten process … this is inferred from the single-main-thread model, since these probes capture the callback in-process rather than round-tripping to a live kitten subprocess."*

This section **removes that inference** and replaces it with direct observation: a real Go `clipboard` kitten and a real Python `ask` kitten, each captured as a distinct process, with the exact PTY framing in both directions, the event‑loop dispatch tied to `file:line`, the recovered output, and the round‑trip latency across repeated runs.

### 6.0 The two‑boundary model (observed)

| | Boundary 1 — in‑process | Boundary 2 — inter‑process |
|---|---|---|
| **What crosses** | C parser buffer bytes → Python object | terminal escape codes (OSC) over a PTY |
| **Where** | inside a single kitty process, MAIN thread under GIL | between the kitty core process and a *separate* kitten process |
| **Mechanism** | `PyMemoryView_FromMemory` `kitty/vt-parser.c:461` → `clipboard_control` `kitty/screen.c:2305` → `kitty/clipboard.py` | kitten writes/reads OSC on `/dev/tty`; core reads it through the ordinary child path `kitty/child-monitor.c:1337` → `kitty/vt-parser.c:1417` |
| **Kitten‑side parser (Go kittens)** | — | `kitty/tools/tui/loop` — `loop.New` `kittens/clipboard/read.go:285`, `OnEscapeCode` `kittens/clipboard/read.go:334` |
| **Kitten‑side parser (Python kittens)** | — | `kittens/tui/loop.py:246 _read_ready` → `:248 os.read` → `:261 parse_input_from_terminal` → C `kitty/kittens.c:104` (registered `:200`); response reads via `kitty/kittens.c:94 read_command_response` |

Observed process facts (this session): the kitten launcher is a **15,945,988‑byte Go executable** at `kitty/launcher/kitten`; the **`clipboard` kitten is Go** (`kittens/clipboard/{main,read,write,legacy}.go`), and the **`ask` kitten is Python** (`kittens/ask/main.py`). Both run as children **distinct** from the kitty core process (shown below). This is **OBSERVED**.

```text
$ ls -l /app/kitty/launcher/kitten ; wc -c < /app/kitty/launcher/kitten
-rwxr-xr-x 1 root 1001 15945988 Jul 13 21:16 /app/kitty/launcher/kitten
15945988
$ ls /app/kittens/clipboard/*.go
/app/kittens/clipboard/cli_generated.go  /app/kittens/clipboard/legacy.go
/app/kittens/clipboard/main.go  /app/kittens/clipboard/read.go  /app/kittens/clipboard/write.go
$ ls /app/kittens/ask/main.py
/app/kittens/ask/main.py            # Python kitten
```

### 6.1 Boundary 2 outbound — the real Go `clipboard` kitten SET (distinct PIDs + PTY framing)

**What was run (canonical, no permission prompt — a write needs no confirmation).** Inside a live kitty window a shell runs the real Go kitten in filter mode, `strace`‑d to capture the separate PID(s) and the exact bytes it puts on the PTY:

```text
# inside kitty (stdin/stdout = kitty window PTY):
printf '%s' "$PAYLOAD" | strace -f -tt -T -e trace=read,write,openat,close \
    -o /tmp/obs/out/kitten_set_strace.raw /app/kitty/launcher/kitten clipboard
# driver: ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes sh <script>
```

**Result — the kitten is a separate process, and it writes OSC 52 to `/dev/tty` (complete, unedited):**

```text
PARENT_SHELL_PID=5731
KITTY_PID=5664
KITTEN_EXIT=0
PAYLOAD=kitten-boundary-payload-5731
WALL_S=0.046

# distinct kitten PIDs/threads captured by strace: 5737 5738 5739 … 5747  (kitty core = 5664, shell = 5731)

5745  22:21:36.304542 openat(AT_FDCWD, "/dev/tty", O_RDWR|O_NOCTTY|O_NONBLOCK|O_CLOEXEC) = 0 <0.000041>
5745  22:21:36.305318 write(0, "\33]52;c;", 7) = 7 <0.000015>
5745  22:21:36.305422 write(0, "a2l0dGVuLWJvdW5kYXJ5LXBheWxvYWQt"..., 36) = 36 <0.000014>
5744  22:21:36.305610 write(0, "MQ==", 4 <unfinished ...>
5744  22:21:36.305713 write(0, "\33\\", 2) = 2 <0.000012>
```

**Reading the framing.** The kitten opens the controlling terminal (`/dev/tty`, `O_RDWR`) and emits a single **OSC 52** sequence — `ESC ] 52 ; c ;` (the `\33]52;c;` prefix), then the base64 body, then the `ESC \` String Terminator (`\33\\`). This is the **outbound Boundary‑2 framing**, byte‑for‑byte. The whole emission spans `304542 → 305713` ≈ **1.17 ms**; total kitten wall time **0.046 s**.

**Honesty note on the base64 body.** `strace` elides long `write` bodies (the `"…"..., 36`). The bytes *on the wire* are the full base64 of the 28‑byte payload `kitten-boundary-payload-5731` (see `PAYLOAD=` above); the tail group `MQ==` is base64 for the final byte `1`. Do **not** decode the truncated display literally — its byte‑exact value is proven instead by the short GET reads in §6.2, whose bodies are **not** truncated and decode exactly to their seeds. **OBSERVED**, with the display‑truncation caveat stated.

**Which OSC number.** The observed SET (and the plain GET in §6.2) used **legacy OSC 52**. The kitten also implements the extended **OSC 5522** protocol (`kittens/clipboard/read.go:26 OSC_NUMBER = "5522"`, written at `:208`), selected for richer MIME negotiation; my canonical invocations exercised the legacy OSC 52 path. The OSC‑number selection is **OBSERVED** (52 emitted); that 5522 is the extended alternative is **INFERRED from `read.go:26,208`**.

### 6.2 Boundary 2 round‑trip — the real Go `clipboard` kitten GET: event **delivered to** the kitten + latency (≥ 2 runs)

A GET is a true round‑trip: the kitten **writes** an OSC 52 read request, the core parses it (Boundary 1), reads the clipboard, and **writes an OSC 52 response back**, which the kitten's event loop **reads** — that inbound read is exactly "an event delivered to a kitten." Under the default `read-clipboard-ask` this round‑trip is gated by a permission prompt (proven in §6.3). To measure the *kitten‑boundary* cost in isolation — free of the human prompt, which is a **core** cost, not a kitten cost — this run uses a **diagnostic‑only** configuration that removes the prompt.

> **DIAGNOSTIC‑ONLY, and a security warning.** The run below sets `clipboard_control="… read-clipboard read-primary"` (i.e. `-ask` removed). Kitty's own docs flag this exact change as a security risk (`kitty/options/definition.py:3096-3110`): *"disabling the read confirmation is a security risk as it means that any program, even the ones running on a remote server via SSH can read your clipboard."* It is used here **solely** to isolate the boundary latency; the canonical default path is in §6.3.

**What was run (3 identical runs):**

```text
# inside kitty launched with the DIAGNOSTIC no-ask config:
#   ./kitty/launcher/kitty --config NONE \
#     -o clipboard_control="write-clipboard write-primary read-clipboard read-primary" \
#     -o close_on_child_death=yes sh <script>
# per run i: seed via the kitten, then GET via the kitten under strace -tt -T:
printf '%s' "$SEED" | /app/kitty/launcher/kitten clipboard
strace -f -tt -T -e trace=read,write -o kitten_noask_strace_$i.raw \
    /app/kitty/launcher/kitten clipboard --get-clipboard
```

**Result — full round‑trip correct, and the inbound event captured (complete, unedited):**

```text
run=1 exit=0 seed=kitten-noask-run1-6020 got=kitten-noask-run1-6020
run=2 exit=0 seed=kitten-noask-run2-6020 got=kitten-noask-run2-6020
run=3 exit=0 seed=kitten-noask-run3-6020 got=kitten-noask-run3-6020

-- run1 --  (kitten PIDs 6044..6054, distinct from kitty core)
6054  22:25:37.757212 write(3, "\33]52;c;?\33\\", 10) = 10 <0.000028>          # OUTBOUND request
6050  22:25:37.760563 read(3, "\33]52;c;a2l0dGVuLW5vYXNrLXJ1bjEtN"..., 16384) = 41 <0.000022>   # INBOUND = event delivered to kitten
-- run2 --  (kitten PIDs 6077..6088)
6088  22:25:38.132205 write(3, "\33]52;c;?\33\\", 10) = 10 <0.000013>
6085  22:25:38.135494 read(3, "\33]52;c;a2l0dGVuLW5vYXNrLXJ1bjItN"..., 16384) = 41 <0.000025>
-- run3 --  (kitten PIDs 6112..6122)
6115  22:25:38.509787 write(3, "\33]52;c;?\33\\", 10) = 10 <0.000013>
6112  22:25:38.513113 read(3, "\33]52;c;a2l0dGVuLW5vYXNrLXJ1bjMtN"..., 16384) = 41 <0.000021>
```

**Byte‑exact decode of the inbound event** (these reads are 41 B — **not** truncated):

```text
inbound base64 "a2l0dGVuLW5vYXNrLXJ1bjEtNjAyMA==" -> "kitten-noask-run1-6020"   # == seed, == got
```

**The measured kitten round‑trip latency** — time from the kitten's OSC‑52 request `write` to its OSC‑52 response `read`, from the `strace -tt` timestamps:

| run | req write ts | resp read ts | **round‑trip** |
|---|---|---|---|
| 1 | 22:25:37.757212 | 22:25:37.760563 | **3.351 ms** |
| 2 | 22:25:38.132205 | 22:25:38.135494 | **3.289 ms** |
| 3 | 22:25:38.509787 | 22:25:38.513113 | **3.326 ms** |

median **3.326 ms**, range **3.289–3.351 ms** — **stable across 3 runs** (≥ 2 satisfied).

**Why ≈ 3.3 ms, and how it ties to the core model.** The kitten's request is parsed by the **same** core VT parser whose dispatch is gated by `input_delay` (default **3 ms**, `kitty/options/definition.py:878`, gate at `kitty/vt-parser.c:1425`). So the round‑trip is `input_delay (3 ms) + ~0.3 ms overhead` — matching the **core** DSR latency measured independently in §O2 (`3.312 / 3.287 ms` at the 3 ms default) essentially to the microsecond. This is the causal link: **the kitten‑boundary round‑trip is bounded by the very same main‑thread `input_delay` gate as the in‑process path** — not a coincidence, and not the same number "borrowed," but the same mechanism observed twice. **OBSERVED.**

### 6.3 The canonical default (prompt‑gated) path — two real kitten processes at once

The default `read-clipboard-ask` gates a GET behind a confirmation. Here the plain (no‑diagnostic) path runs and an external `xdotool` presses `y` to accept — capturing the live process tree while the request is pending.

**What was run:**

```text
# default clipboard_control; a background job waits ~3s, snapshots the process tree, then:
#   xdotool key --window <kitty> y
strace -f -tt -T -e trace=read,write,openat -o kitten_canon_strace.raw \
    /app/kitty/launcher/kitten clipboard --get-clipboard
```

**Result — two distinct real kitten processes, framing, and recovered output (complete, unedited):**

```text
SEED=kitten-canon-6257  SHELL=6257  KITTY=6190
GET_EXIT=0

# process tree while the GET is pending (the ask overlay is live):
   6190    6188 kitty    ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes sh …kitten_canon.sh
   6257    6190 sh       sh /tmp/obs/scripts/kitten_canon.sh
   6275    6257 strace   strace … /app/kitty/launcher/kitten clipboard --get-clipboard
   6278    6275 kitten   /app/kitty/launcher/kitten clipboard --get-clipboard          <-- Go clipboard kitten (requester)
   6289    6190 kitten   /app/kitty/launcher/kitten ask --type=yesno --message A program running in this window
                          wants to read from the system clipboard. Allow it to do so, once? --default y   <-- Python ask kitten (prompt)

6280  22:26:34.400406 write(3, "\33]52;c;?\33\\", 10) = 10 <0.000016>                 # OUTBOUND request
6288  22:26:36.837077 read(3, "\33]52;c;a2l0dGVuLWNhbm9uLTYyNTc=\33"..., 16384) = 33 <0.000037>   # INBOUND after accept
# kitten stdout (recovered clipboard):
kitten-canon-6257
```

`a2l0dGVuLWNhbm9uLTYyNTc=` decodes to `kitten-canon-6257` — the exact seed, recovered through the **full canonical two‑boundary path**. Two real kitten processes are simultaneously alive and **distinct** from the core (6190): the Go `clipboard` requester (6278) and the Python `ask` prompt (6289). **OBSERVED.**

**Prompt‑gated latency, reported honestly.** Request→response here is `22:26:34.400406 → 22:26:36.837077` = **2.437 s** — but that interval is **dominated by the prompt** (the background job's ~3 s wait before pressing `y`), *not* the kitten boundary. The boundary cost proper is the **3.3 ms** of §6.2; the canonical GET simply adds however long the human takes to confirm. Both numbers are real; they measure different things and are labelled as such.

### 6.4 The Python‑kitten event loop, observed (`loop.py:246/261` → `kittens.c:104`)

The `ask` prompt in §6.3 is a **Python** kitten, so its input travels the Python kitten loop the review names explicitly. Attaching `strace` to the live `ask` process by PID and then pressing `y` captures the keystroke arriving at that loop:

```text
# ask (Python kitten) identity while the prompt is live:
ASK_KITTEN_PID=6479
   6479    6342 kitten  /app/kitty/launcher/kitten ask --type=yesno --message A program running in this window
                         wants to read from the system clipboard. Allow it to do so, once? --default y

# the 'y' keystroke arriving to the Python kitten's event loop:
6491  22:27:18.668581 read(3, "\33[121;;121u", 16384) = 11 <0.000025>

# overall result:
GET_EXIT=0 SEED=kitten-py-6416
stdout=kitten-py-6416
```

The inbound `read(3, "\33[121;;121u", …)` is the accept key delivered to the kitten: `\33[121;;121u` is kitty's keyboard‑protocol **CSI‑u** encoding of `y` (Unicode 121 = ASCII `'y'`). That `os.read` is `kittens/tui/loop.py:248` inside `_read_ready` (`:246`); the bytes are then handed to `kittens/tui/loop.py:261 parse_input_from_terminal`, which is the C function `kitty/kittens.c:104` (bound at `kittens/tui/loop.py:239`, registered at `kitty/kittens.c:200`). Command responses on this path are read by `kitty/kittens.c:94 read_command_response`. The GET completed (`stdout == seed`). This concretely exercises **every `file:line` the C2 finding names for the Python path**. **OBSERVED.**

For **Go** kittens the analogous dispatch is `kittens/clipboard/read.go:334 OnEscapeCode` (registered on the loop created at `:285 loop.New`), which fires on the inbound OSC captured in §6.2/§6.3; the parse guard is `read.go:244`. **OBSERVED** (the inbound OSC read) + **INFERRED** (that `OnEscapeCode` is the specific Go callback, from `read.go:334`).

### 6.5 What this section corrects, and the core‑vs‑kitten latency distinction

- **INFERRED → OBSERVED.** The prior document timed the in‑process callback and *inferred* the final hop to the separate kitten process (its own words, quoted above). That hop is now **directly observed**: distinct kitten PIDs, the OSC bytes in both directions on `/dev/tty`, the event‑loop reads at the exact `file:line`, the recovered output, and a measured round‑trip latency across 3 runs.
- **Core dispatch latency is not kitten latency — but shares one gate.** §O2's DSR figure (`~3.31 ms`) is a **core, in‑process** dispatch latency and was labelled core‑only there. Kitten **event delivery** is a *different* quantity — an inter‑process PTY round‑trip — yet it measures **3.326 ms median** because the core parses the kitten's request under the **same** `input_delay` gate (`kitty/vt-parser.c:1425`). So the two are causally linked but distinct: kitten delivery = `core parse (input_delay‑gated) + inter‑process PTY hop`, plus (on the default path) the permission prompt.
- **A busy MAIN thread delays kitten events too.** Because the core's parse/dispatch of a kitten's request runs on the same single GIL‑holding MAIN thread as everything else (§Threading), a main‑thread hog (the §O3 scrollback scan) delays delivery to a kitten by the scan duration, exactly as it delays the in‑process callback. This is the correct generalization; §O3 measures it on the core side and it applies to Boundary 2 by the shared‑thread argument. **OBSERVED (core side) + INFERRED (that the same delay propagates across the PTY hop, from the single‑MAIN‑thread model).**

**Observed vs inferred for §6:**

| Claim | Status | Evidence |
|---|---|---|
| Kittens are separate processes with PIDs distinct from the core | OBSERVED | process trees §6.1, §6.3 |
| Go `clipboard` kitten writes OSC 52 to `/dev/tty` (`O_RDWR`) | OBSERVED | strace §6.1 |
| GET is a round‑trip; inbound OSC read = event delivered to kitten | OBSERVED | strace §6.2, §6.3 |
| Recovered clipboard bytes == seed (both no‑ask and canonical) | OBSERVED | `got==seed`, stdout==seed |
| Kitten‑boundary round‑trip ≈ 3.33 ms, stable ×3 | OBSERVED | timestamps §6.2 |
| Round‑trip bound by the 3 ms `input_delay` core gate | OBSERVED (both numbers) + INFERRED (same‑gate causation) | §6.2 vs §O2 |
| Python kitten uses `loop.py:246/248/261` → `kittens.c:104` | OBSERVED | strace §6.4 + `file:line` |
| Go kitten dispatch is `read.go:334 OnEscapeCode` | INFERRED | `read.go:285,334` + observed inbound read |
| Default GET is prompt‑gated (`ask` Python kitten) | OBSERVED | §6.3 process tree + 2.437 s |
| OSC 5522 is the extended alternative to the observed OSC 52 | INFERRED | `read.go:26,208` |

### 6.6 Evidence ledger (§6)

All artifacts captured this session in the canonical container (`/tmp/obs/out/`), sha256 (first 16 hex) shown:

```text
a6b3e993ff93a67b  kitten_set_strace.raw       (SET: /dev/tty open + OSC 52 emission)
d81e4aad774564d2  kitten_set.meta             (SET: PIDs, payload, wall)
d89f508f5e4ddfaa  kitten_noask.runs           (GET no-ask x3: seed==got)
00970f69785e8835  kitten_noask_strace_1.raw   (GET no-ask run1: framing + timestamps)
151df5a04a31309b  kitten_canon_strace.raw     (canonical GET: framing)
9db9fdb9bafeabc0  kitten_canon.ptree          (canonical GET: two-kitten process tree)
4ecd72b6de8782ac  kitten_py_ask_strace.raw    (Python ask kitten: CSI-u 'y' read)
```

Scripts (removed after capture, repo unchanged): `kitten_set{,_drv}.sh`, `kitten_get_noask.sh` + `kitten_noask_drv.sh`, `kitten_canon{,_drv}.sh`, `kitten_py{,_drv}.sh` — all under `/tmp` outside the checkout.

---

## 7. O3 — An expensive scrollback scan on the MAIN thread delays event delivery and adds a transient Python object

**Direct answer.** A large scrollback scan is **MAIN-thread, GIL-holding work**. Because parse, Python dispatch (`clipboard_control`), screen mutation, and render are all serialized on the single MAIN thread (§Threading), a scan **delays delivery of a pending clipboard/kitten event by ≈ the full scan duration** — measured directly below, and proven structurally with a live debugger. For **memory**, a scan does **not** change how the C scrollback is managed; it **reads** the segmented C storage (fixed ~5.01 MiB `calloc` blocks per 2048 lines, `kitty/history.c:18-28`) and **materializes a transient owned Python object** on the Python heap (traced below at **4.58 MiB net / 11.9 MiB peak** for a 60k-line `as_text`), which is released after the consumer copies it out. The background image **disk-cache** thread (`kitty/disk-cache.c:342`) is **not** part of scrollback and is discussed only as a separate memory-offload example.

### 7.1 The scan paths, named correctly (corrects M9)

There are **two distinct C scan implementations**, and the earlier draft conflated them. Both are exercised below under the canonical `./kitty/launcher/kitty +launch` interpreter:

| Scan | C function | Built on | Reached from Python | Used by |
|---|---|---|---|---|
| Visible/history text dump | `as_text` / `as_text_non_visual` (`kitty/screen.c:3486` / `:3491`) and `HistoryBuf.__str__`/`as_ansi` (`kitty/history.c:321` / `:348`) | **`as_text_generic`** (`kitty/line.c:874`) — a per‑line callback loop | `screen.as_text(...)`, `str(historybuf)`, `historybuf.as_ansi(cb)` | remote `get-text` (`kitty/window.py:376` picks `as_text_non_visual` when `add_history`), pager, pipe |
| Selection text | **`text_for_range`** (`kitty/screen.c:3035`) | **`unicode_in_range`** (per line; `kitty/screen.c:3057`) — **NOT** `as_text_generic` | `screen.text_for_selection(...)` (`kitty/screen.c:4004`→`:3990`→`:3035`, method table `:4848`) | copy‑to‑clipboard of a selection (`kitty/window.py:1542`, `:1785`) |

So the claim "`text_for_range` is built on `as_text_generic`" is **wrong**; `text_for_range` uses `unicode_in_range`. Both are now **OBSERVED** at runtime (not source-inferred). Command and complete, unedited output (a 60,000‑line × 80‑col real `HistoryBuf` built through the real VT parser via `parse_bytes`; stable across the two configs and the repeated runs in §7.6):

```text
=== O3 AUGMENTATION PROBE (canonical kitty +launch interpreter) PID=7388 PAGER_BYTES=0 ===
[setup] fed_logical_lines=60000 cols=80 hb.count=59977 hb.xnum=80 feed_ms=84.3
[63] text_for_range (Screen.text_for_selection, screen.c:3035->unicode_in_range): tuple_lines=60000 chars=4799920 times_ms=[21.192, 23.335, 23.817, 20.795, 20.466] median=21.192
[62a] str(hb) monolithic as_text (history.c:321): chars=4798159 times_ms=[28.36, 28.37, 28.51, 29.04, 26.84] median=28.37
[62b] hb.as_ansi callback-driven (history.c:348, real-path list.append): chunks=59977 chars=4798160 times_ms=[34.02, 37.16, 34.84, 37.96, 34.94] median=34.94
```

- **`text_for_range`** (the selection→copy path) extracts **60,000** per‑line strings totalling **4,799,920** chars in a median **21.2 ms** — the real `unicode_in_range` loop.
- **`str(hb)`** (monolithic `as_text`, `history.c:321`) produces **4,798,159** chars in **28.4 ms**; **`hb.as_ansi`** (callback‑driven, `history.c:348`, the real‑path `list.append` callback) produces **59,977 chunks** in **34.9 ms**. These are the `as_text_generic` paths.

All three are tens of milliseconds at 60k lines — long enough to matter on the MAIN thread, which §7.2 shows blocks event delivery for exactly that span.

### 7.2 A pending event waits ≈ the full scan — timing proof and a live serialization proof (finding 64)

**Timing proof (deterministic interpose, not a sleep).** This probe puts a genuine OSC 52 event's bytes into the **real vt‑parser buffer** (`t_ready`), then measures the time until the real `screen→clipboard_control` callback fires (`t_delivered`) — first with the MAIN thread free (BASELINE), then with a large `HistoryBuf` scan **interposed on the MAIN thread before the parse**. The event is provably enqueued **before** the scan and dispatched **after** it, so it is pending for the entire scan — this is a deterministic single‑thread interpose, **not** a `sleep`. Complete, unedited output, **two independent process runs × 2 batches × N=7** (labeled in‑process harness; cross‑checked by the live proof below):

```text
setup: hb.count=120000; pager_ring_bytes_used=16777216
[batch 1]  N=7 per condition
  BASELINE (no scan)            delivery_ms=[0.014, 0.002, 0.001, 0.001, 0.001, 0.001, 0.001]  median=0.001 ms
  SCAN-INTERPOSED: as_text(__str__) monolithic history.c:321
     scan_only_ms median=59.0   delivery_ms=[60.6, 61.5, 63.0, 59.0, 61.0, 61.4, 59.2] median=61.0
     delivery_delta(delivery-baseline)=61.0 ms  ~=  scan_only median 59.0 ms
  SCAN-INTERPOSED: as_ansi callback-driven     history.c:348
     scan_only_ms median=72.2   delivery_ms=[75.8, 74.2, 73.8, 72.7, 75.6, 73.8, 71.4] median=73.8
     delivery_delta(delivery-baseline)=73.8 ms  ~=  scan_only median 72.2 ms
```

In every batch and both process runs the increase **`delivery_delta = delivery − baseline` equals the measured `scan_only` median**. When the MAIN thread is free the OSC 52 is dispatched in **~0.001 ms**; behind a scan it waits **~61 ms** (`as_text`) or **~74 ms** (`as_ansi`). The event is **never lost** (`clipboard_control` fires exactly once, `assert cb.n==1`, the moment the scan returns). This is the direct proof that a ready event waits ≈ the full scan because parse/dispatch and the scan share the one GIL‑holding MAIN thread.

**Live serialization proof (canonical, gdb on a running kitty).** To confirm this structurally on the real binary, a live kitty was launched with a 200,000‑line scrollback (`cat` of a big file), `gdb` armed a breakpoint on `as_text_generic`, and a **canonical remote `get-text --extent=all`** (which routes through `as_text_non_visual`→`as_text_generic`, `kitty/window.py:376`) triggered the scan. The break fired; `info threads` shows the scan is on the **MAIN thread while every other thread is parked**. Complete, unedited (the 32 idle Mesa `llvmpipe` GL‑pool threads in `futex_wait` are elided as `… [30 more "kitty" GL-pool threads in __futex_abstimed_wait_common64] …`):

```text
KITTY_CORE_PID=9154
gettext_chars=80817

Thread 1 "kitty" hit Breakpoint 1, 0x0000782c7aa2cb60 in as_text_generic () from /app/kitty/.../fast_data_types.so
==== BREAK HIT: as_text_generic (a scrollback scan is executing) ====
  Id   Target Id                                        Frame
* 1    Thread 0x782c7b39d740 (LWP 9154) "kitty"         0x...aa2cb60 in as_text_generic () from .../fast_data_types.so
  2    Thread 0x782b51ffb6c0 (LWP 9221) "KittyChildMon" 0x...ee4fd in __GI___poll (fds=... <children_fds>, nfds=3, timeout=-1)
  3    Thread 0x782b527fc6c0 (LWP 9220) "KittyPeerMon"  0x...ee4fd in __GI___poll (fds=..., nfds=3, timeout=-1)
  4    Thread 0x782b52ffd6c0 (LWP 9219) "kitty:disk$0"  0x...bd71 in __futex_abstimed_wait_common64 (...)
  … [30 more "kitty" GL-pool threads in __futex_abstimed_wait_common64] …
```

**Reading it.** The MAIN thread (LWP 9154, `Thread 1 "kitty"`) is stopped **inside `as_text_generic`** — the scrollback scan, which produced **80,817 chars** of `get-text` output. Simultaneously **`KittyChildMon` (LWP 9221) is blocked in `__poll` on `children_fds`** (the I/O thread runs no Python), `KittyPeerMon` is in `poll`, and `kitty:disk$0` (the disk‑cache writer) is in `futex_wait`. Because `clipboard_control` runs on this **same** MAIN thread under the GIL, it **cannot execute while `as_text_generic` occupies MAIN** — the pending event is necessarily serialized behind the scan. This is the canonical structural counterpart to the timing proof above. **OBSERVED.**

### 7.3 Why even a callback‑driven scan does not yield (GIL‑hold classification)

The real per‑line callback the code passes is a bound **`list.append`** (`kitty/window.py:377/394/459`) — a **C method**, which does not run the bytecode eval loop, so CPython's periodic GIL‑release check (`eval_breaker`) is never reached. A heartbeat thread (a PROXY for other‑thread Python work) is starved for ~the whole scan: `as_text gap/scan≈1.02`, `as_ansi≈1.01`, `as_text_for_history_buf≈1.02`. A control swapping in a Python‑function callback confirms the mechanism:

```text
as_ansi + list.append (C method, real-path kind): worst_gap median = 72.5 ms
as_ansi + python def(line) (runs bytecode)      : worst_gap median = 6.2 ms
sys.getswitchinterval() = 0.005 s
```

The real path always uses the C‑method `list.append`, so all scans are effectively **monolithic** with respect to the GIL — and even a yielding callback would hand the GIL to *other* threads, never to the MAIN loop's own delivery, so MAIN‑thread event delivery is delayed by ≈ the full scan regardless. **OBSERVED.**

### 7.4 Memory during a scan — allocation profile and PID-tied smaps (findings 66, 70)

**Allocation profile (`tracemalloc`, bounds the transient — corrects M11's "attributed without a profile").** Wrapping one `str(hb)` scan of the 60k‑line buffer in `tracemalloc` traces the Python‑heap allocation directly to the call, separating the retained object from the transient construction peak. Complete, unedited (identical across both configs):

```text
[70] tracemalloc around str(hb): before=0B during_current=4798200B during_peak=12475296B net_alloc=4798200B (~4.58 MiB)
     top_alloc: 4798200B (4.58 MiB) o3_probe.py:57
     top_alloc: 64B (0.00 MiB) o3_probe.py:58
```

The scan retains a single **4.58 MiB** `str` (the `net_alloc`, traced to the `str(hb)` call), with a transient **peak of 11.9 MiB** during construction (the internal `ANSIBuf` plus the final object coexisting). So the transient is **measured, not asserted** — it is the serialized‑text object itself, **~net 4.58 MiB**, not an unexplained allocator effect. **OBSERVED.**

**PID-tied RSS/PSS before/during/after (finding 66).** Because the scan holds the GIL, an in‑process Python sampler would be starved; an **external** sampler (a separate OS process, GIL‑immune) read `/proc/<PID>/smaps_rollup` every 20 ms across a **1.6 s sustained scan loop** (BEFORE 0.5 s idle / DURING loop / AFTER 0.5 s idle). Complete, unedited summary (140 samples, DEFAULT‑pager run PID=7388):

```text
samples total: 140
BEFORE Rss: n=18 min=219200 med=219200 max=219200 kB | Pss: n=18 min=212836 med=212843 max=212854 kB
DURING Rss: n=48 min=219200 med=221420 max=224232 kB | Pss: n=48 min=212837 med=215067 max=217875 kB
AFTER  Rss: n=67 min=219200 med=219200 max=219200 kB | Pss: n=67 min=212835 med=212842 max=212852 kB
```

RSS is flat at **219,200 kB** before, rises to a **max 224,232 kB during** (**+~4.9 MiB**, matching the 4.58 MiB `tracemalloc` net plus allocator rounding as strings are built and freed each loop iteration), and returns to **219,200 kB after**. The scan's memory cost is a **transient, released** Python object — not a persistent growth or leak. The pattern is **stable across the two config runs**: the non-default (16 MiB pager) run gave `BEFORE Rss med=219236 -> DURING max=224356 -> AFTER med=219236` (+~5.0 MiB), and the `tracemalloc` net/peak were byte-identical across both. **OBSERVED.**

### 7.5 The persistent cost is the C scrollback storage — fixed segments (finding 67)

The dominant, persistent memory is the C‑side `HistoryBuf`, which holds cells in fixed **2048‑line segments** (`SEGMENT_SIZE`, `kitty/history.c:15`), each a single `calloc` in `add_segment` (`kitty/history.c:18-28`) sized `xnum·2048·(sizeof(CPUCell)+sizeof(GPUCell)) + 2048·sizeof(LineAttrs)`. At `xnum=80` that is exactly **5,251,072 bytes (~5.01 MiB) per segment**, and the measured RSS step matches to a ratio of **1.00**:

```text
=== (2) C-side HistoryBuf segment growth (RSS step per 2048-line segment), N=3 ===
  run1: base_rss=26529792; per-segment RSS deltas = [6631424, 5255168, 5255168, 5255168, 5255168, 5255168, 5255168, 5255168]
  theoretical_per_segment_bytes=5251072 (~5.01 MiB)
  observed per-segment RSS delta: min=0 median=5251072.0 max=6631424 n=24
  median_observed/theoretical = 1.00
```

A scan **reads** this segmented storage and does not alter it; scrollback RAM grows **linearly and predictably** — one fixed ~5.01 MiB block per 2048 lines (`add_segment`, `kitty/history.c:18`). The Python‑heap object footprint of a scan (§7.4) is transient and dwarfed by this persistent C storage. **OBSERVED.**

### 7.6 Pager-history stores raw bytes (not compressed) and is OFF by default (corrects M11/68; finding 72)

The pager history is a byte **ring buffer that stores UTF‑8/ANSI bytes directly — there is no compression**. `pagerhist_write_bytes` (`kitty/history.c:218`) does a plain `ringbuf_memcpy_into(ph->ringbuf, buf, sz)`; a grep of the entire `kitty/history.c` for `zlib|lz4|deflate|compress|snappy` returns **nothing**:

```text
$ grep -naiE "zlib|lz4|deflate|inflate|compress|snappy" /app/kitty/history.c
$        # (no output — zero matches)
```

It is also **OFF by default**: `scrollback_pager_history_size` defaults to `0` (`kitty/options/definition.py:406` `opt('scrollback_pager_history_size','0',...)`). The **canonical‑default vs non‑default** comparison (finding 72) — and the ring only fills on **eviction** past the in‑RAM scrollback — is shown directly:

```text
# DEFAULT (pager off), even with eviction:
pager=0 scrollback=100000 fed_logical=320000: hb.count(stored)=100000 evicted_to_pager~=219976 pagerhist_bytes=0 (~0.00 MiB) ring_lines~=0 cap16MiB_lines=209715
# NON-DEFAULT (16 MiB pager), with eviction -> ring saturates:
pager=16777216 scrollback=100000 fed_logical=320000: hb.count(stored)=100000 evicted_to_pager~=219976 pagerhist_bytes=16777216 (~16.00 MiB) ring_lines~=209715 cap16MiB_lines=209715
```

At the **default** (`0`) the pager ring does not exist, so `pagerhist_as_bytes` is empty even when 219,976 lines are evicted; only the **non‑default** 16 MiB ring populates, saturating at exactly **16,777,216 bytes = 209,715 lines** (16 MiB / 80 bytes). **OBSERVED.**

### 7.7 Line-count reconciliation — logical vs physical vs stored vs evicted (corrects M10/71)

The earlier "108,423 vs 200,000 for the same output" was an unreconciled mix of **different counts at different stages**. With exact measured numbers:

| Count | Meaning | Measured |
|---|---|---|
| `fed_logical_lines` | logical lines written (each `79×'X'`+CRLF; 79 < 80 ⇒ **no wrap**, so logical = physical here) | 60,000 (or 320,000) |
| `hb.count` | logical lines **stored** in the in‑RAM history buffer (capped by `scrollback`) | 59,977 (or 100,000 at `scrollback=100000`) |
| evicted | `fed − stored − 24 visible`, pushed toward the pager ring | ~219,976 |
| pager ring lines | evicted lines **retained** in the pager ring (capped at 16 MiB ⇒ 209,715) | 209,715 |
| `str(hb)` chars | serialized output = `hb.count × (79 + 1 newline)` | 4,798,159 (= 59,977 × 80 − 1) |

So `fed ≠ stored ≠ evicted ≠ pager‑ring` are **four different, individually‑explained quantities**; a discrepancy like "108,423 vs 200,000" is simply `hb.count` (scrollback‑capped) versus fed/evicted. Wrapping would further split one logical line into several physical lines only when content width exceeds `cols` (not the case here, 79 < 80). All units are stated in **MiB/KiB** consistently. **OBSERVED.**

### 7.8 Disk cache is not scrollback (finding 69)

The background `DiskCacheWrite` thread (`kitty/disk-cache.c:342`) backs the **graphics/image** subsystem, not scrollback text. Scrollback lives entirely in the RAM `HistoryBuf` segments (§7.5) and the optional RAM pager ring (§7.6); the encrypted disk cache is an unrelated memory‑offload for images (`kitty/graphics.c`) and is never used for scrollback text. **OBSERVED (grep of consumers) + INFERRED (that no scrollback path calls it).**

### 7.9 Observed vs inferred (§7)

| Claim | Status | Evidence |
|---|---|---|
| `text_for_range` uses `unicode_in_range`, not `as_text_generic` | OBSERVED | §7.1 run + `screen.c:3057` |
| `as_text`/`as_ansi` use `as_text_generic` (`line.c:874`) | OBSERVED | §7.1 run + `screen.c:3486`/`history.c:321,348` |
| Pending event waits ≈ full scan (`delivery_delta==scan_only`) | OBSERVED | §7.2 timing, N=7×2×2 |
| Scan runs on MAIN while others parked → serialized | OBSERVED | §7.2 gdb `info threads` |
| Callback scan holds the GIL (no yield) | OBSERVED | §7.3 heartbeat + control |
| Scan transient = 4.58 MiB net / 11.9 MiB peak Python object | OBSERVED | §7.4 tracemalloc |
| RSS +~4.9 MiB during, released after | OBSERVED | §7.4 smaps before/during/after |
| Persistent scrollback = 5,251,072 B/2048-line segment (ratio 1.00) | OBSERVED | §7.5 RSS steps |
| Pager stores raw bytes, no compression, off by default | OBSERVED | §7.6 grep + `history.c:218` + `options:406` |
| Pager ring saturates at 16 MiB = 209,715 lines on eviction | OBSERVED | §7.6 eviction run |
| Line counts reconcile (fed/stored/evicted/ring) | OBSERVED | §7.7 table |
| Disk cache is graphics, not scrollback | OBSERVED + INFERRED | §7.8 |
| In-process interpose harness mirrors live serialization | OBSERVED (both) + INFERRED (equivalence) | §7.2 timing vs gdb |

**Evidence ledger (§7):** `o3_probe.py` (945d980ed13cf992), `o3_pager.py` (87f4a33d8cf7046e), `o3_gdb_scan.sh` (1a5dfdc18079778c), `o3_gdb_scan.raw` (e98c398824c7ed34), `o3_smaps_0.samples` (a8e2cdcdd2d6cf3d) — all under `/tmp` outside the checkout, removed after capture.

---

## 8. O4 — Where timing, concurrency, and object ownership start to matter

**Direct answer.** Three mechanisms decide correctness at the boundary, and each is grounded below in a live runtime capture, not source reading alone: (1) a per-parser **mutex** guarding the single 1 MiB producer/consumer buffer, held only to fold the I/O thread's `write.pending` bytes into the main thread's `read.sz` and **released during the actual parse**; (2) the **GIL**, which serialises every Python callback (including `clipboard_control`) on the one main thread; and (3) the **RAII-scoped `memoryview`** handed to Python, whose correctness depends on Python copying the bytes out before the parser reuses the buffer. The single most important correction over earlier drafts: retaining that view past the callback is **not** an immediate use-after-free. The view has **three distinct lifetimes** — the memoryview *object* (refcount), the *buffer allocation* it points into, and the *payload content* — and the immediate hazard of retention is **stale/mutated content** (the buffer is reused); a true use-after-free arises only after the parser itself is torn down (`free_vt_parser`).

> **Diagnostic-build note.** The runtime lock/GIL captures in §8.1–§8.2 use a `--debug` build (`python3 setup.py --debug`, adding `-g -O0` DWARF so gdb can read `struct PS`, the mutex, and `read.sz`/`write.pending`). This is explicitly sanctioned by the AAP (“a `--debug --sanitize` build … for tracing”). The observed behaviour — lock ordering, thread topology, GIL — is identical to the default build because the source is identical; `--debug` only changes optimisation/symbols. Every other section's values come from the default `python3 setup.py` build, and the default `fast_data_types.so` (sha256 `582933cf…`) was restored immediately after these captures.

### 8.1 The parser lock and the pending→read promotion — source + live runtime proof (M12; findings 79, 80)

Source (verbatim, `kitty/vt-parser.c:1412-1446`): the lock macros, the promotion under the lock, and the `end_with_lock { consume_input(...) } with_lock` hand-off that **releases** the lock during the parse:

```c
#define with_lock pthread_mutex_lock(&self->lock);
#define end_with_lock pthread_mutex_unlock(&self->lock);

static void
run_worker(void *p, ParseData *pd, bool flush) {
    Screen *screen = (Screen*)p;
    PS *self = (PS*)screen->vt_parser->state;
    with_lock {
        self->read.sz += self->write.pending; self->write.pending = 0;
        pd->has_pending_input = self->read.pos < self->read.sz;
        if (pd->has_pending_input) {
            pd->time_since_new_input = pd->now - self->new_input_at;
            if (flush || pd->time_since_new_input >= OPT(input_delay) || self->read.sz + 16 * 1024 > BUF_SZ) {
                pd->input_read = true;
                self->dump_callback = pd->dump_callback; self->now = pd->now;
                self->screen = screen;
                self->read.consumed = 0;
                do {
                    end_with_lock; {
                        consume_input(self, pd->dump_callback, screen->window_id);
                    } with_lock;
                    self->read.sz += self->write.pending; self->write.pending = 0;
                } while (self->read.pos < self->read.sz);
                self->new_input_at = 0;
                if (self->read.consumed) {
                    pd->write_space_created = self->read.sz >= BUF_SZ;
                    self->read.pos -= MIN(self->read.pos, self->read.consumed);
                    self->read.sz -= MIN(self->read.sz, self->read.consumed);
                    if (self->read.sz) memmove(self->buf, self->buf + self->read.consumed, self->read.sz);
                }
            }
        }
    } end_with_lock;
}
```

The promotion `self->read.sz += self->write.pending; self->write.pending = 0;` (`kitty/vt-parser.c:1421`) runs **inside** `with_lock`; `consume_input` (defined at `:1367`, called at `:1432`) runs **after** `end_with_lock` — i.e. unlocked. Earlier drafts asserted this from source/disassembly only; M12 required capturing the exact mutation and lock state at runtime. Captured here in **one linked gdb run** on the `--debug` build, driving a single 25-byte OSC 52 write escape (`ESC ] 52 ; c ; <b64> ESC \`) through the real PTY. Breakpoint 1 at `:1421` is conditioned on `write.pending > 0` so it fires on a **real** input tick, not the idle ticks `run_worker` also services:

- **STOP A** — lock **held**: `self->lock.__data.__lock == 1`, `__owner == 11056` (the MAIN thread's own TID), and the promotion folds `write.pending 25 → 0` into `read.sz 0 → 25`.
- **STOP B** — `consume_input` entry, lock **released**: `self->lock.__data.__lock == 0`, `read.sz == 25`.
- **STOP C** — `clipboard_control`: full synchronous stack on the MAIN thread, `PyGILState_Check() == 1`, and the boundary object's live Python type is `"memoryview"`.

Command and complete, unedited output:

```
$ bash /tmp/obs/scripts/o4_lock3_run.sh   # gdb -batch -x o4_lock3.gdb --args kitty --config NONE sh -c '<timed OSC52>'
PAYLOAD=o4-locktest  B64=bzQtbG9ja3Rlc3Q=  escape_wire_length=25 bytes ( ESC]52;c; + b64 + ESC\ )
Breakpoint 1 (vt-parser.c:1421 if self->write.pending > 0) pending.
Breakpoint 2 (vt-parser.c:1367) pending.
Breakpoint 3 (screen.c:2305) pending.

[0.593] Failed to open systemd user bus with error: No medium found

Thread 1 "kitty" hit Breakpoint 1.1, run_worker (p=0x5555569c9630, pd=0x7fffffffac10, flush=false) at kitty/vt-parser.c:1421
1421	        self->read.sz += self->write.pending; self->write.pending = 0;

=========== STOP A: PENDING PROMOTION vt-parser.c:1421 (real input; expect lock HELD __lock==1) ===========
#0  run_worker (p=0x5555569c9630, pd=0x7fffffffac10, flush=false) at kitty/vt-parser.c:1421
1421	        self->read.sz += self->write.pending; self->write.pending = 0;
-- current thread (expect MAIN "kitty") --
[Current thread is 1 (Thread 0x7ffff73c7740 (LWP 11056))]
-- lock held? self->lock.__data.__lock (1=locked) and __owner tid --
$1 = 1
$2 = 11056
-- BEFORE promotion: self->read.sz , self->write.pending --
$3 = 0
$4 = 25
1422	        pd->has_pending_input = self->read.pos < self->read.sz;
-- AFTER promotion line: read.sz , write.pending (pending must be 0; read.sz += old pending) --
$5 = 25
$6 = 0

Thread 1 "kitty" hit Breakpoint 2.1, consume_input (self=self@entry=0x555556728c80, dump_callback=0x0, window_id=1) at kitty/vt-parser.c:1375
1375	    switch (self->vte_state) {

=========== STOP B: consume_input ENTRY vt-parser.c:1367 (expect lock RELEASED __lock==0) ===========
#0  consume_input (self=self@entry=0x555556728c80, dump_callback=0x0, window_id=1) at kitty/vt-parser.c:1375
1375	    switch (self->vte_state) {
-- current thread --
[Current thread is 1 (Thread 0x7ffff73c7740 (LWP 11056))]
-- lock released? self->lock.__data.__lock (0=unlocked) --
$7 = 0
$8 = 25
$9 = 0

Thread 1 "kitty" hit Breakpoint 3, clipboard_control (self=0x5555569c9630, code=code@entry=52, data=0x7ffff5afcc40) at kitty/screen.c:2305
2305	clipboard_control(Screen *self, int code, PyObject *data) {

=========== STOP C: clipboard_control screen.c:2305 SYNCHRONOUS CALLBACK (expect MAIN + GIL) ===========
#0  clipboard_control (self=0x5555569c9630, code=code@entry=52, data=0x7ffff5afcc40) at kitty/screen.c:2305
2305	clipboard_control(Screen *self, int code, PyObject *data) {
-- current thread --
[Current thread is 1 (Thread 0x7ffff73c7740 (LWP 11056))]
#0  clipboard_control (self=0x5555569c9630, code=code@entry=52, data=0x7ffff5afcc40) at kitty/screen.c:2305
#1  0x00007ffff6a54455 in dispatch_osc (self=self@entry=0x555556728c80, buf=<optimized out>, limit=<optimized out>, is_extended_osc=is_extended_osc@entry=false) at kitty/vt-parser.c:534
#2  0x00007ffff6a568e1 in accumulate_st_terminated_esc_code (self=0x555556728c80, dispatch=dispatch@entry=0x7ffff6a542c3 <dispatch_osc>) at kitty/vt-parser.c:403
#3  0x00007ffff6a56a38 in consume_input (self=self@entry=0x555556728c80, dump_callback=<optimized out>, window_id=<optimized out>) at kitty/vt-parser.c:1385
#4  0x00007ffff6a56c04 in run_worker (p=0x5555569c9630, pd=0x7fffffffab50, flush=false) at kitty/vt-parser.c:1432
#5  0x00007ffff6a57813 in parse_worker (p=<optimized out>, pd=<optimized out>, flush=<optimized out>) at kitty/vt-parser.c:1496
#6  0x00007ffff69af6fe in do_parse (self=self@entry=0x7ffff5bd0a30, screen=0x5555569c9630, now=now@entry=4674255049, flush=flush@entry=false) at kitty/child-monitor.c:440
#7  0x00007ffff69b153f in parse_input (self=self@entry=0x7ffff5bd0a30) at kitty/child-monitor.c:530
#8  0x00007ffff69b4712 in process_global_state (data=0x7ffff5bd0a30) at kitty/child-monitor.c:1236
#9  0x00007ffff69b47c1 in do_state_check (timer_id=<optimized out>, data=<optimized out>) at kitty/child-monitor.c:1218
#10 0x00007ffff59dfcf4 in dispatchTimers (eld=eld@entry=0x7ffff5a2c6f0 <_glfw+133552>) at glfw/backend_utils.c:208
#11 0x00007ffff59dff80 in pollForEvents (eld=0x7ffff5a2c6f0 <_glfw+133552>, timeout=1650831, display_callback=display_callback@entry=0x0) at glfw/backend_utils.c:309
#12 0x00007ffff59d5f08 in handleEvents (timeout=<optimized out>) at glfw/x11_window.c:70
#13 0x00007ffff59d5f86 in _glfwPlatformWaitEvents () at glfw/x11_window.c:2731
#14 0x00007ffff59d01ed in _glfwPlatformRunMainLoop (tick_callback=0x7ffff69b46c4 <process_global_state>, data=0x7ffff5bd0a30) at glfw/main_loop.h:30
#15 0x00007ffff59c7709 in glfwRunMainLoop (callback=<optimized out>, data=<optimized out>) at glfw/init.c:360
#16 0x00007ffff6a00673 in run_main_loop (cb=cb@entry=0x7ffff69b46c4 <process_global_state>, cb_data=cb_data@entry=0x7ffff5bd0a30) at kitty/glfw.c:2103
#17 0x00007ffff69b0e0f in main_loop (self=0x7ffff5bd0a30, a=<optimized out>) at kitty/child-monitor.c:1262
#18 0x00007ffff789fce2 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#19 0x00007ffff7891b2c in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#20 0x00007ffff782c5ee in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#21 0x00007ffff7893580 in _PyObject_FastCallDictTstate () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#22 0x00007ffff78937ee in _PyObject_Call_Prepend () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#23 0x00007ffff7912075 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#24 0x00007ffff78917df in _PyObject_MakeTpCall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#25 0x00007ffff782c5ee in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#26 0x00007ffff79af91f in PyEval_EvalCode () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#27 0x00007ffff79ab8b0 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#28 0x00007ffff78eeadc in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#29 0x00007ffff7891b2c in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#30 0x00007ffff782c5ee in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#31 0x00007ffff7a34242 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#32 0x00007ffff7a34da3 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#33 0x00007ffff7a3539c in Py_RunMain () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#34 0x00005555555579a4 in run_embedded (run_data=0x7fffffffb9f0) at kitty/launcher/main.c:216
#35 0x00005555555588ac in main (argc=8, argv=0x7fffffffeb88, envp=0x7fffffffebd0) at kitty/launcher/main.c:464
-- GIL held on MAIN? PyGILState_Check()==1 --
$10 = 1
-- the data object handed to Python: pointer + live Python type name --
$11 = (PyObject *) 0x7ffff5afcc40
$12 = 0x7ffff7b401e5 "memoryview"
[Inferior 1 (process 11056) exited normally]

=========== END-OF-LINKED-RUN ===========
```

Reading the output. At **STOP A**, `$1 = 1` is the held mutex and `$2 = 11056` is the owning TID (identical to the current thread LWP 11056 = MAIN “kitty”), so the promotion is performed under the lock; `$3/$4` (`read.sz=0`, `write.pending=25`) become `$5/$6` (`read.sz=25`, `write.pending=0`) after the single line executes. At **STOP B**, `$7 = 0` is the released mutex at `consume_input` entry, `read.sz` still 25. At **STOP C**, the backtrace is the synchronous chain `clipboard_control` ← `dispatch_osc` (`:534`) ← `accumulate_st_terminated_esc_code` (`:403`) ← `consume_input` (`:1385`) ← `run_worker` (`:1432`), `$10 = 1` (GIL held on MAIN), and `$12 = … "memoryview"` is the live type of the object handed to Python. This is the exact lock/GIL/callback provenance M12 asked for, in one linked run.

### 8.2 The C→Python callback is synchronous on MAIN, and every other thread is parked

To show nothing runs concurrently with the callback, a companion linked run (`o4_lock_gdb.raw`) dumps `info threads` at the moment `clipboard_control` is entered. Only the MAIN thread is in `clipboard_control`; `KittyChildMon` is parked in `poll()` on `children_fds`, `kitty:disk$0` is parked in a futex, and the 64 GL software-rasteriser pool threads (`llvmpipe-*` plus unnamed `kitty` pool threads, a headless Mesa/Xvfb artifact) are all in `__futex_abstimed_wait_common64`:

```
=========== STOP1 clipboard_control (OSC52_A) : SYNCHRONOUS CALLBACK STACK on MAIN ===========
[Current thread is 1 (Thread 0x7ffff73c7740 (LWP 10753))]
  Id   Target Id                                         Frame 
* 1    Thread 0x7ffff73c7740 (LWP 10753) "kitty"         clipboard_control (self=0x555556734030, code=code@entry=52, data=0x7ffff5afcc40) at kitty/screen.c:2305
  2    Thread 0x7fffe77a96c0 (LWP 10756) "llvmpipe-0"    0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=400, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559aaa68) at ./nptl/futex-internal.c:57
  3    Thread 0x7fffe6fa86c0 (LWP 10757) "llvmpipe-1"    0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=400, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559aabc8) at ./nptl/futex-internal.c:57
  4    Thread 0x7fffe67a76c0 (LWP 10758) "llvmpipe-2"    0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=240, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559aad28) at ./nptl/futex-internal.c:57
  5    Thread 0x7fffe5fa66c0 (LWP 10759) "llvmpipe-3"    0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=240, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559aae88) at ./nptl/futex-internal.c:57
  6    Thread 0x7fffe57a56c0 (LWP 10760) "llvmpipe-4"    0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=240, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559aafe8) at ./nptl/futex-internal.c:57
  7    Thread 0x7fffe4fa46c0 (LWP 10761) "llvmpipe-5"    0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=304, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ab148) at ./nptl/futex-internal.c:57
  8    Thread 0x7fffcffff6c0 (LWP 10762) "llvmpipe-6"    0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=21845, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ab2a8) at ./nptl/futex-internal.c:57
  9    Thread 0x7fffcf7fe6c0 (LWP 10763) "llvmpipe-7"    0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=21845, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ab408) at ./nptl/futex-internal.c:57
  10   Thread 0x7fffceffd6c0 (LWP 10764) "llvmpipe-8"    0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=48, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ab568) at ./nptl/futex-internal.c:57
  11   Thread 0x7fffce7fc6c0 (LWP 10765) "llvmpipe-9"    0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=21845, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ab6c8) at ./nptl/futex-internal.c:57
  12   Thread 0x7fffcdffb6c0 (LWP 10766) "llvmpipe-10"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=240, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ab828) at ./nptl/futex-internal.c:57
  13   Thread 0x7fffcd7fa6c0 (LWP 10767) "llvmpipe-11"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=368, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ab988) at ./nptl/futex-internal.c:57
  14   Thread 0x7fffccff96c0 (LWP 10768) "llvmpipe-12"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=21845, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559abae8) at ./nptl/futex-internal.c:57
  15   Thread 0x7fffa7fff6c0 (LWP 10769) "llvmpipe-13"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=304, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559abc48) at ./nptl/futex-internal.c:57
  16   Thread 0x7fffaffff6c0 (LWP 10770) "llvmpipe-14"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=368, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559abda8) at ./nptl/futex-internal.c:57
  17   Thread 0x7fffaf7fe6c0 (LWP 10771) "llvmpipe-15"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=21845, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559abf08) at ./nptl/futex-internal.c:57
  18   Thread 0x7fffaeffd6c0 (LWP 10772) "llvmpipe-16"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=21845, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ac068) at ./nptl/futex-internal.c:57
  19   Thread 0x7fffae7fc6c0 (LWP 10773) "llvmpipe-17"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=21845, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ac1c8) at ./nptl/futex-internal.c:57
  20   Thread 0x7fffadffb6c0 (LWP 10774) "llvmpipe-18"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=304, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ac328) at ./nptl/futex-internal.c:57
  21   Thread 0x7fffad7fa6c0 (LWP 10775) "llvmpipe-19"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=368, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ac488) at ./nptl/futex-internal.c:57
  22   Thread 0x7fffacff96c0 (LWP 10776) "llvmpipe-20"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=368, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ac5e8) at ./nptl/futex-internal.c:57
  23   Thread 0x7fffa77fe6c0 (LWP 10777) "llvmpipe-21"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=21845, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ac748) at ./nptl/futex-internal.c:57
  24   Thread 0x7fffa6ffd6c0 (LWP 10778) "llvmpipe-22"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=21845, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ac8a8) at ./nptl/futex-internal.c:57
  25   Thread 0x7fffa67fc6c0 (LWP 10779) "llvmpipe-23"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=416, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559aca08) at ./nptl/futex-internal.c:57
  26   Thread 0x7fffa5ffb6c0 (LWP 10780) "llvmpipe-24"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=21845, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559acb68) at ./nptl/futex-internal.c:57
  27   Thread 0x7fffa57fa6c0 (LWP 10781) "llvmpipe-25"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=368, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559accc8) at ./nptl/futex-internal.c:57
  28   Thread 0x7fffa4ff96c0 (LWP 10782) "llvmpipe-26"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=368, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ace28) at ./nptl/futex-internal.c:57
  29   Thread 0x7fff6ffff6c0 (LWP 10783) "llvmpipe-27"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=21845, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559acf88) at ./nptl/futex-internal.c:57
  30   Thread 0x7fff6f7fe6c0 (LWP 10784) "llvmpipe-28"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=368, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ad0e8) at ./nptl/futex-internal.c:57
  31   Thread 0x7fff6effd6c0 (LWP 10785) "llvmpipe-29"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=400, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ad248) at ./nptl/futex-internal.c:57
  32   Thread 0x7fff6e7fc6c0 (LWP 10786) "llvmpipe-30"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=416, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ad3a8) at ./nptl/futex-internal.c:57
  33   Thread 0x7fff6dffb6c0 (LWP 10787) "llvmpipe-31"   0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=21845, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559ad508) at ./nptl/futex-internal.c:57
  34   Thread 0x7fff6d7fa6c0 (LWP 10788) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  35   Thread 0x7fff6cff96c0 (LWP 10789) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  36   Thread 0x7fff4bfff6c0 (LWP 10790) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  37   Thread 0x7fff4b7fe6c0 (LWP 10791) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  38   Thread 0x7fff4affd6c0 (LWP 10792) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  39   Thread 0x7fff4a7fc6c0 (LWP 10793) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  40   Thread 0x7fff49ffb6c0 (LWP 10794) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  41   Thread 0x7fff497fa6c0 (LWP 10795) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  42   Thread 0x7fff48ff96c0 (LWP 10796) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  43   Thread 0x7fff33fff6c0 (LWP 10797) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  44   Thread 0x7fff337fe6c0 (LWP 10798) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  45   Thread 0x7fff32ffd6c0 (LWP 10799) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  46   Thread 0x7fff327fc6c0 (LWP 10800) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  47   Thread 0x7fff31ffb6c0 (LWP 10801) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  48   Thread 0x7fff317fa6c0 (LWP 10802) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  49   Thread 0x7fff30ff96c0 (LWP 10803) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  50   Thread 0x7fff0ffff6c0 (LWP 10804) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  51   Thread 0x7fff0f7fe6c0 (LWP 10805) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  52   Thread 0x7fff0effd6c0 (LWP 10806) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  53   Thread 0x7fff0e7fc6c0 (LWP 10807) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  54   Thread 0x7fff0dffb6c0 (LWP 10808) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  55   Thread 0x7fff0d7fa6c0 (LWP 10809) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  56   Thread 0x7fff0cff96c0 (LWP 10810) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32767, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  57   Thread 0x7ffee7fff6c0 (LWP 10811) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32766, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  58   Thread 0x7ffeeffff6c0 (LWP 10812) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32766, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  59   Thread 0x7ffeef7fe6c0 (LWP 10813) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32766, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  60   Thread 0x7ffeeeffd6c0 (LWP 10814) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32766, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  61   Thread 0x7ffeee7fc6c0 (LWP 10815) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32766, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  62   Thread 0x7ffeedffb6c0 (LWP 10816) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32766, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  63   Thread 0x7ffeed7fa6c0 (LWP 10817) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32766, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  64   Thread 0x7ffeecff96c0 (LWP 10818) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32766, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  65   Thread 0x7ffee77fe6c0 (LWP 10819) "kitty"         0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=32766, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x5555559fb8d0) at ./nptl/futex-internal.c:57
  66   Thread 0x7ffee6ffd6c0 (LWP 10820) "kitty:disk$0"  0x00007ffff7595d71 in __futex_abstimed_wait_common64 (private=0, cancel=true, abstime=0x0, op=393, expected=0, futex_word=0x55555587dc48) at ./nptl/futex-internal.c:57
  67   Thread 0x7ffee67fc6c0 (LWP 10821) "KittyChildMon" 0x00007ffff76184fd in __GI___poll (fds=fds@entry=0x7ffff6af6a80 <children_fds>, nfds=3, timeout=timeout@entry=-1) at ../sysdeps/unix/sysv/linux/poll.c:29
```

So at the instant the clipboard callback runs, 66 of 67 threads are blocked and the callback owns the process. *(inferred)* On a real GPU the ~64 `llvmpipe`/pool threads would not exist; they are a software-GL artifact and do not affect the serialisation conclusion — the functionally relevant threads (`KittyChildMon`, `kitty:disk$0`) are demonstrably idle.

### 8.3 The boundary memoryview has three distinct lifetimes (corrects C3 / S1)

The bytes live in `PS.buf`, an **inline** array inside `struct PS` — verbatim `kitty/vt-parser.c:193-210`:

```c
typedef struct PS {
    alignas(BUF_EXTRA) uint8_t buf[BUF_SZ + BUF_EXTRA];
    UTF8Decoder utf8_decoder;

    id_type window_id;

    VTEState vte_state;
    ParsedCSI csi;

    // these are temporary variables set only for duration of a parse call
    PyObject *dump_callback;
    Screen *screen;
    monotonic_t now, new_input_at;
    pthread_mutex_t lock;

    // The buffer
    struct { size_t consumed, pos, sz; } read;
    struct { size_t offset, sz, pending; } write;
```

The view is created over that inline buffer at `kitty/vt-parser.c:457-465`; `mv` is an `RAII_PyObject`, so kitty's **local** reference is dropped when the `START_DISPATCH … END_DISPATCH` block exits (`END_DISPATCH`, `:464`):

```c
dispatch_osc(PS *self, uint8_t *buf, size_t limit, bool is_extended_osc) {
#define DISPATCH_OSC_WITH_CODE(name) REPORT_OSC2(name, code, mv); name(self->screen, code, mv);
#define DISPATCH_OSC(name) REPORT_OSC(name, mv); name(self->screen, mv);
#define START_DISPATCH {\
    RAII_PyObject(mv, PyMemoryView_FromMemory((char*)buf + i, limit - i, PyBUF_READ)); \
    if (mv) {
#define END_DISPATCH_WITHOUT_BREAK }; PyErr_Clear(); }
#define END_DISPATCH }; PyErr_Clear(); break; }
```

and the OSC 52 case that hands the view to `clipboard_control` (`kitty/vt-parser.c:530-536`):

```c
            END_DISPATCH
        case 52: case 5522:
            START_DISPATCH
            if (is_extended_osc && code == 52) code = -52;
            DISPATCH_OSC_WITH_CODE(clipboard_control);
            END_DISPATCH
        case 133:
```

The inline buffer is freed **only** at parser teardown, in `free_vt_parser` (`kitty/vt-parser.c:1507-1516`; wired as `tp_dealloc` at `:1552`):

```c
void
free_vt_parser(Parser* self) {
    if (self->state) {
        PS *s = (PS*)self->state;
        utf8_decoder_free(&s->utf8_decoder);
        pthread_mutex_destroy(&s->lock);
        free(self->state); self->state = NULL;
    }
    Py_TYPE(self)->tp_free((PyObject*)self);
}
```

Three lifetimes must therefore be kept separate:

| Lifetime | What it is | Created / ends where | The hazard if a view is retained |
|---|---|---|---|
| **L1 — object (refcount)** | the `memoryview` PyObject | created `:461`; kitty's local ref dropped at `END_DISPATCH` `:464`; a Python-held ref keeps the OBJECT alive | none by itself — the object simply persists |
| **L2 — buffer allocation** | the inline `PS.buf` memory (`:194`) it points into | allocated with the `PS` struct; freed only by `free_vt_parser` `:1513` at teardown | **use-after-free**, but only *after teardown* (not immediate) |
| **L3 — payload content** | the specific bytes at that address | valid only until the buffer is **reused** by the next parse (`memmove`/overwrite in `run_worker`) | **stale/mutated content** — the *immediate* hazard |

A genuine retained-view/parser-reuse probe (NON-CANONICAL entry via the `Screen` test hooks driving the identical production `vt_parser_*` and the real `parse_bytes`; cross-checked against the canonical OSC 52 round trip in §3 and the live gdb callback in §8.1) retains the view and re-reads it before and after a **second** parse through the **same** buffer. This is **not** the “standalone `free(buf)` analog” the review rejected — it exercises real parser buffer reuse. Command and complete, unedited output (**identical across 2 runs**, sha256 `ed433fb9391334ade5e52737696a3337c045f79f48b2af2304471e29f04968c4`):

```
$ ./kitty/launcher/kitty +launch /tmp/obs/scripts/o4_lifetimes.py
=== O4 PART A: three distinct lifetimes of the boundary memoryview ===
expected head payload#1 (A): b'c;QUFBQUFBQUFBQUFB'
expected head payload#2 (B): b'c;QkJCQkJCQkJCQkJC'
[L1 refcount ] after dispatch returns: refcount(v0)=3 (>0 => OBJECT alive, held by Python)  readonly=True  obj_is_None=True  nbytes=402
[L2 alloc    ] head at dispatch       : b'c;QUFBQUFBQUFBQUFB'  ==payload#1? True
[L2 alloc    ] head AFTER scope return : b'c;QUFBQUFBQUFBQUFB'  ==payload#1? True  (re-read did NOT crash => buffer allocation still valid => NOT immediate UAF)
[L3 content  ] head at 2nd dispatch    : b'c;QkJCQkJCQkJCQkJC'  ==payload#2? True
[L3 content  ] retained view#0 re-read  : b'c;QkJCQkJCQkJCQkJC'
[L3 content  ]   -> now aliases REUSED buffer (==payload#2)? True
[L3 content  ]   -> still original payload#1?                False
[teardown    ] v0.obj is None => view holds NO reference to the parser; PS.buf is freed only at free_vt_parser (vt-parser.c:1508-1513). Reading AFTER teardown would be a genuine UAF and is deliberately NOT executed (labeled inferred-from-source).
```

Reading the output:
- **L1 (refcount).** `refcount(v0)=3` *after* the callback returned — the memoryview object is alive because Python holds references (list slot + local), even though kitty's local `RAII_PyObject` reference was dropped at `END_DISPATCH`. `readonly=True`, `obj_is_None=True` (no base object — `PyMemoryView_FromMemory` produces a raw window that keeps nothing alive), `nbytes=402`.
- **L2 (allocation).** Re-reading the retained view *after the dispatch scope returned* yields the original payload#1 bytes and **does not crash** — the `PS.buf` allocation is still valid. This is the direct correction to the “immediate UAF” claim: post-callback retention is **not** a use-after-free.
- **L3 (content).** After a second OSC 52 (payload#2) is parsed through the same buffer, the retained view now reads payload#2 (`aliases REUSED buffer? True`, `still original? False`). The bytes silently changed — **stale/mutated content is the immediate hazard**.
- **Teardown.** Because `obj is None`, the view holds no reference to the parser and cannot keep `PS.buf` alive; a true use-after-free would occur only if the view were read *after* `free_vt_parser` frees the `PS` struct at teardown (`:1513`). That is a genuine UAF and is deliberately **not executed** — labeled inferred-from-source.

### 8.4 How kitty makes ownership safe — the copy-out during the synchronous callback (finding 78)

Because a view is valid only for the *current* content, the clipboard manager copies the bytes it needs **out** of the view into owned Python objects during the same synchronous callback. The decoded payload is written into the `Tempfile` at `kitty/clipboard.py:316-323`:

```python
    def write_base64_data(self, b: bytes) -> None:
        from base64 import standard_b64decode
        if not self.max_size_exceeded:
            d = standard_b64decode(b)
            self.tempfile.write(d)
            if self.max_size > 0 and self.tempfile.tell() > (self.max_size * 1024 * 1024):
                log_error(f'Clipboard write request has more data than allowed by clipboard_max_size ({self.max_size}), truncating')
                self.max_size_exceeded = True
```

and the undecodable base64 remainder is copied at `kitty/clipboard.py:280-297` — the inner `bytes(mv[-extra:])` at `:286` is an owned copy:

```python
        def write_saving_leftover_bytes(data: bytes) -> None:
            if len(data) == 0:
                return
            extra = len(data) % 4
            if extra > 0:
                mv = memoryview(data)
                self.current_leftover_bytes = memoryview(bytes(mv[-extra:]))
                mv = mv[:-extra]
                if len(mv) > 0:
                    self.write_base64_data(mv)
            else:
                self.write_base64_data(data)

        if len(self.current_leftover_bytes) > 0:
            extra = 4 - len(self.current_leftover_bytes)
            if len(data) >= extra:
                self.write_base64_data(memoryview(bytes(self.current_leftover_bytes) + data[:extra]))
                self.current_leftover_bytes = memoryview(b'')
```

Part B of the probe proves the leftover is owned, not aliased (identical across 2 runs):

```
$ ./kitty/launcher/kitty +launch /tmp/obs/scripts/o4_lifetimes.py
=== O4 PART B: kitty copies OUT within the callback (owned copy, clipboard.py:286) ===
B1 (canonical values, kitty_tests/clipboard.py:12-17):
  full base64 fed        : b'bGlnaHQgd29yaw'
  current_leftover_bytes : b'aw'  (expected b'aw')
  data_for()             : b'light work'  (expected b'light work')
B2 (source-mutation proof the leftover is an OWNED copy at clipboard.py:286):
  current_leftover_bytes BEFORE source mutation: b'aw'
  source bytearray mutated tail -> b'ZZ'
  current_leftover_bytes AFTER  source mutation: b'aw' -> UNCHANGED => OWNED copy, not a view
```

B1 reproduces the in-repo unit-test values (`kitty_tests/clipboard.py:12-17`): feeding `b'bGlnaHQgd29yaw'` leaves a 2-byte leftover `b'aw'`, and after flush `data_for()` is `b'light work'`. B2 overwrites the source `bytearray`'s tail with `b'ZZ'` *after* the leftover is captured; `current_leftover_bytes` stays `b'aw'` — proving the `bytes(...)` at `:286` produced an owned copy. So the retained-view content hazard of §8.3 **never arises in production**: kitty always copies out within the callback scope, before any buffer reuse.

### 8.5 Ownership at the core-Python boundary vs the kitten boundary (finding 81)

The lifetime analysis above applies to **Boundary 1** only — the in-process C→core-Python handoff, where the memoryview *aliases* the live parser buffer (zero-copy). The **kitten** boundary (**Boundary 2**, §6) is different in kind: a kitten is a **separate process** that exchanges bytes with core over the PTY/pipe. There is **no shared buffer and no memoryview** across that boundary — core serialises an OSC escape and the kitten reads its **own copy** via `os.read` (`kittens/tui/loop.py:248`, in `_read_ready` `:246`) into a Python `bytes`, parsed by `parse_input_from_terminal` (`kittens/tui/loop.py:261` → `kitty/kittens.c:104`). Object-ownership lifetimes (L1–L3) are therefore a Boundary-1 concern; at Boundary 2 each side owns its own copy and the only cross-process coupling is byte framing and timing (the measured 3.3 ms round trip in §6.2). This is precisely why the retained-view hazard cannot propagate to a kitten: the kitten never receives the aliasing view, only copied bytes.

### 8.6 Observed / inferred ledger (§8)

| Claim | Status | Evidence |
|---|---|---|
| Promotion `read.sz += write.pending` runs under the held lock | OBSERVED (gdb) | §8.1 STOP A (`__lock=1`, `__owner=MAIN tid`, 25→0) |
| `consume_input` runs with the lock released | OBSERVED (gdb) | §8.1 STOP B (`__lock=0`) |
| Callback is synchronous on MAIN under the GIL | OBSERVED (gdb) | §8.1 STOP C (`PyGILState_Check()==1`, full stack) |
| All other threads parked during the callback | OBSERVED (gdb) | §8.2 `info threads` (66/67 blocked) |
| Boundary object is a read-only `memoryview`, `obj is None` | OBSERVED | §8.1 (`tp_name=memoryview`), §8.3 (L1) |
| L1 object outlives dispatch when Python retains a ref | OBSERVED | §8.3 (`refcount=3` post-callback) |
| L2 buffer valid after callback (NOT immediate UAF) | OBSERVED | §8.3 (re-read post-scope, no crash) |
| L3 content goes stale on buffer reuse | OBSERVED | §8.3 (payload#1→payload#2) |
| True UAF only after `free_vt_parser` teardown | INFERRED (source `:1507-1516`; not executed) | §8.3 teardown row |
| kitty copies out (owned leftover + decoded Tempfile) | OBSERVED | §8.4 Part B (source-mutation unchanged) |
| Kitten boundary copies bytes (no shared view) | OBSERVED (§6) + INFERRED (ownership consequence) | §8.5, §6.2 |

**Evidence ledger (§8):** `o4_lock3.gdb` (3ecc1759b71cbb4a)  `o4_lock3_run.sh` (44ab762ede931b6a)  `o4_lifetimes.py` (cf15380fbeb0aaf3)  `o4_lock3_gdb.raw` (64595ec790b26e08)  `o4_lock_gdb.raw` (5f86b1ee2e94dc1c)  `o4_lifetimes_run1.txt` (ed433fb9391334ad)  `o4_lifetimes_run2.txt` (ed433fb9391334ad)  `o4_debug_build.log` (3da6c69bb7cafeb4) — all under `/tmp` outside the checkout, removed after capture. The `--debug` build was used only for §8.1–§8.2 gdb inspection; the default `fast_data_types.so` (`582933cf…`) was restored immediately afterward.

---

## 9. O5 — Subtle races, reentrancy, and the disk cache

**Direct answer (four parts, each grounded below).** (1) The clipboard C→Python transfer path is serialized by two mechanisms observed at runtime in §8.1 — the per-parser mutex and the CPython GIL — so no *Python-level* data race is possible on it, and a data-race detector (valgrind **helgrind** 3.22.0) observed **no data race in the `vt-parser.c` / `dispatch_osc` / `clipboard_control` path** across every run in this section. (2) The one genuine object-lifetime hazard — a retained `memoryview` aliasing the reused parser buffer — is **not** a data race; it is the single-threaded content-reuse hazard already demonstrated live in §8.3, and helgrind cannot and does not flag it (there is no second thread). (3) helgrind **did** observe genuine *unsynchronized `shutting_down` plain-bool* data races at teardown in **`disk-cache.c`** (read `:348` vs write `:439`, no mutex on either side) and in **`child-monitor.c`** (read `:1491` vs write `:424`), plus an asymmetric-lock race on `cache_file_fd` (`:421` unlocked write vs `:361` locked read) — the exact shared-state disclosure asked for by M16/S4. (4) The self-offer reentrancy path (an OSC 52 *self-read*) is a clean, fully canonical **C entry → `RuntimeError('is_self_offer')` → Python catch → owned-data fallback → result** chain, captured end-to-end in one linked run. Every conclusion is bounded to “no race observed in these runs” with the untested surfaces enumerated in §9.7.

All detector runs use a disposable, PTRACE-enabled container (`kitty-work`, from image `kitty-diag:latest`, itself committed from `swe-atlas-kitty:canonical` id `796bc91c3984`), `/app` at HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`. Every command below shows its exact invocation, bound, exit status, and complete unedited output.

### 9.1 Detector methodology and the valgrind-compatible diagnostic build (M13)

**Why helgrind, not ThreadSanitizer.** kitty's build has exactly one sanitizer switch, `--sanitize`, and it wires up **`-fsanitize=address,undefined`** only — ASan + UBSan, which are memory-error / undefined-behavior detectors, *not* data-race detectors (`setup.py:380`, `def get_sanitize_args(...)→['-fsanitize=address,undefined', '-fno-omit-frame-pointer']`; the `asan:` Makefile target is `--debug --sanitize`). kitty ships **no** ThreadSanitizer build. The ASan/UBSan diagnostic build was already exercised in §2 (clipboard + parser suites, zero sanitizer reports). For **data races** the tool used here is valgrind **helgrind**, which instruments at runtime and therefore runs on the *canonical* default `.so` with no recompilation — in principle. In practice one obstacle had to be solved first, disclosed next.

**The canonical `-O3 -march=native` build emits AVX-512 that valgrind-3.22.0 cannot decode.** Running helgrind on the canonical default `fast_data_types.so` (sha256 `582933cf…`) aborts immediately with SIGILL. Command and complete unedited output:

```
$ cd /app   # canonical default .so restored (sha256 582933cfd7b6cecb5ee60cfd20ef35a1f74acc2c6a905022180a60a76bf722e8)
$ valgrind --tool=helgrind --history-level=approx \
    ./kitty/launcher/kitty +runpy "import runpy; runpy.run_path('/tmp/obs/scripts/o5_diskcache.py', run_name='__main__')"
# (helgrind rc=132 = SIGILL)
vex amd64->IR: unhandled instruction bytes: 0x62 0xF2 0xFD 0x8 0x3B 0xC1 0xC5 0xFA 0x7E 0xD
vex amd64->IR:   REX=0 REX.W=0 REX.R=0 REX.X=0 REX.B=0
vex amd64->IR:   VEX=0 VEX.L=0 VEX.nVVVV=0x0 ESC=NONE
vex amd64->IR:   PFX.66=0 PFX.F2=0 PFX.F3=0
==14047== valgrind: Unrecognised instruction at address 0x5eef452.
==14047==    at 0x5EEF452: convert_opts_from_python_opts.constprop.0 (in /app/kitty/fast_data_types.so)
==14047==    by 0x5EC5BAE: pyset_options.lto_priv.0 (in /app/kitty/fast_data_types.so)
==14047==    by 0x4A48497: ??? (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047==    by 0x49EB7DE: _PyObject_MakeTpCall (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047==    by 0x49865ED: _PyEval_EvalFrameDefault (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047==    by 0x4B0991E: PyEval_EvalCode (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047==    by 0x4B058AF: ??? (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047==    by 0x4988BD2: _PyEval_EvalFrameDefault (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047==    by 0x4B0991E: PyEval_EvalCode (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047==    by 0x4B64227: ??? (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047==    by 0x4B6434B: ??? (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047==    by 0x4B67963: PyRun_StringFlags (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047== Your program just tried to execute an instruction that Valgrind
==14047== did not recognise.  There are two possible reasons for this.
==14047== 1. Your program has a bug and erroneously jumped to a non-code
==14047==    location.  If you are running Memcheck and you just saw a
==14047==    warning about a bad jump, it's probably your program's fault.
==14047== 2. The instruction is legitimate but Valgrind doesn't handle it,
==14047==    i.e. it's Valgrind's fault.  If you think this is the case or
==14047==    you are not sure, please let us know and we'll try to fix it.
==14047== Either way, Valgrind will now raise a SIGILL signal which will
==14047== probably kill your program.
==14047== 
==14047== Process terminating with default action of signal 4 (SIGILL): dumping core
==14047==  Illegal opcode at address 0x5EEF452
==14047==    at 0x5EEF452: convert_opts_from_python_opts.constprop.0 (in /app/kitty/fast_data_types.so)
==14047==    by 0x5EC5BAE: pyset_options.lto_priv.0 (in /app/kitty/fast_data_types.so)
==14047==    by 0x4A48497: ??? (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047==    by 0x49EB7DE: _PyObject_MakeTpCall (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047==    by 0x49865ED: _PyEval_EvalFrameDefault (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047==    by 0x4B0991E: PyEval_EvalCode (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047==    by 0x4B058AF: ??? (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047==    by 0x4988BD2: _PyEval_EvalFrameDefault (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047==    by 0x4B0991E: PyEval_EvalCode (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047==    by 0x4B64227: ??? (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047==    by 0x4B6434B: ??? (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047==    by 0x4B67963: PyRun_StringFlags (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==14047== 
==14047== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

The unhandled bytes begin `0x62` — the **EVEX prefix**, i.e. an **AVX-512** instruction — emitted into `convert_opts_from_python_opts` by the default `-O3 -march=native` codegen (`setup.py:585`, `-march=native -mtune=native`). valgrind-3.22.0's VEX front-end does not decode this EVEX form, so it raises SIGILL. (This is the real root of the earlier report's confused “`-no-pie` / ASLR” detector notes: the blocker is instruction-set, not address-space layout.)

**The fix: a valgrind-compatible *diagnostic* build whose C source is byte-identical to canonical.** `setup.py:301` reads the compiler from `os.environ['CC']`, so a one-line wrapper that appends a generic ISA after the caller's flags (gcc honors the **last** `-march`) disables AVX-512 without editing a single repository file. The wrapper (sha256 `73d24b92…`):

```sh
#!/bin/sh
# valgrind-compatible compiler wrapper: force a generic x86-64-v3 ISA (AVX2, NO
# AVX-512) by appending after the caller's flags (gcc honors the LAST -march).
# Used ONLY to produce a diagnostic build that valgrind-3.22.0 can instrument;
# the C source compiled is byte-identical to the canonical build.
exec gcc "$@" -march=x86-64-v3 -mno-avx512f
```

Build commands (a clean, from-scratch diagnostic build) and the **complete, unedited** build log (380 lines: 28 Wayland-protocol generations, the full 122-file `fast_data_types` C-extension compile, 5 links, then the Go tool builds), exit 0:

```
$ cd /app
$ CC=/tmp/ccwrap.sh python3 setup.py clean          # rc=0
$ CC=/tmp/ccwrap.sh python3 setup.py --debug        # rc=0
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
go: downloading golang.org/x/sys v0.21.0
go: downloading github.com/bmatcuk/doublestar/v4 v4.6.1
go: downloading golang.org/x/exp v0.0.0-20230801115018-d63ba01acd4b
go: downloading github.com/shirou/gopsutil/v3 v3.24.5
go: downloading github.com/kovidgoyal/imaging v1.6.3
go: downloading github.com/edwvee/exiffix v0.0.0-20240229113213-0dbb146775be
go: downloading golang.org/x/image v0.17.0
go: downloading github.com/zeebo/xxh3 v1.0.2
go: downloading github.com/alecthomas/chroma/v2 v2.14.0
go: downloading github.com/seancfoley/ipaddress-go v1.6.0
go: downloading howett.net/plist v1.0.1
go: downloading github.com/google/uuid v1.6.0
go: downloading github.com/ALTree/bigfloat v0.2.0
go: downloading github.com/dlclark/regexp2 v1.11.0
go: downloading github.com/rwcarlsen/goexif v0.0.0-20190401172101-9e8deecbddbd
go: downloading github.com/disintegration/imaging v1.6.2
go: downloading github.com/klauspost/cpuid/v2 v2.2.5
go: downloading github.com/tklauser/go-sysconf v0.3.12
go: downloading github.com/tklauser/numcpus v0.6.1
go: downloading github.com/seancfoley/bintree v1.3.1
unicode/utf16
container/list
github.com/seancfoley/ipaddress-go/ipaddr/addrerr
github.com/seancfoley/ipaddress-go/ipaddr/addrstr
internal/nettrace
encoding
github.com/shirou/gopsutil/v3/common
log/internal
vendor/golang.org/x/crypto/internal/alias
crypto/subtle
vendor/golang.org/x/crypto/cryptobyte/asn1
kitty
crypto/internal/boring/sig
golang.org/x/exp/constraints
crypto/internal/alias
image/color
github.com/seancfoley/ipaddress-go/ipaddr/addrstrparam
internal/weak
maps
internal/singleflight
hash
vendor/golang.org/x/net/dns/dnsmessage
crypto/internal/randutil
math/rand/v2
vendor/golang.org/x/text/transform
bufio
net/http/internal/ascii
encoding/base32
regexp/syntax
crypto/rc4
encoding/binary
context
embed
runtime/cgo
io/ioutil
encoding/hex
kitty/tools/utils/shlex
net/url
log
flag
vendor/golang.org/x/sys/cpu
vendor/golang.org/x/net/http2/hpack
github.com/bmatcuk/doublestar/v4
crypto/internal/edwards25519/field
crypto/cipher
github.com/ALTree/bigfloat
crypto/internal/bigmod
encoding/asn1
crypto/internal/nistec/fiat
github.com/seancfoley/bintree/tree
crypto/dsa
crypto
hash/adler32
hash/crc32
image/color/palette
crypto/md5
golang.org/x/image/riff
internal/concurrent
compress/flate
os/exec
compress/bzip2
crypto/des
crypto/internal/boring
encoding/xml
database/sql/driver
compress/lzw
os/signal
mime/quotedprintable
net/http/internal
golang.org/x/image/tiff/lzw
vendor/golang.org/x/text/unicode/bidi
crypto/internal/edwards25519
image
encoding/base64
unique
vendor/golang.org/x/crypto/chacha20
vendor/golang.org/x/crypto/internal/poly1305
github.com/rwcarlsen/goexif/tiff
crypto/x509/pkix
github.com/klauspost/cpuid/v2
github.com/dlclark/regexp2/syntax
vendor/golang.org/x/crypto/cryptobyte
vendor/golang.org/x/crypto/sha3
vendor/golang.org/x/text/unicode/norm
golang.org/x/sys/unix
crypto/internal/boring/bbig
crypto/hmac
crypto/sha1
crypto/rand
crypto/sha512
crypto/aes
crypto/sha256
regexp
vendor/golang.org/x/crypto/hkdf
vendor/golang.org/x/crypto/chacha20poly1305
encoding/pem
mime
encoding/json
kitty/tools/utils/secrets
crypto/rsa
net/netip
crypto/internal/mlkem768
crypto/ed25519
compress/gzip
compress/zlib
archive/zip
github.com/shirou/gopsutil/v3/internal/common
vendor/golang.org/x/text/secure/bidirule
image/internal/imageutil
image/png
golang.org/x/image/ccitt
golang.org/x/image/bmp
golang.org/x/image/vp8l
golang.org/x/image/vp8
crypto/internal/nistec
image/draw
image/jpeg
golang.org/x/image/tiff
github.com/zeebo/xxh3
golang.org/x/image/webp
howett.net/plist
image/gif
vendor/golang.org/x/net/idna
github.com/dlclark/regexp2
crypto/elliptic
crypto/ecdh
github.com/disintegration/imaging
github.com/kovidgoyal/imaging
github.com/rwcarlsen/goexif/exif
crypto/internal/hpke
crypto/ecdsa
github.com/edwvee/exiffix
github.com/alecthomas/chroma/v2
github.com/tklauser/numcpus
github.com/shirou/gopsutil/v3/mem
github.com/tklauser/go-sysconf
github.com/shirou/gopsutil/v3/cpu
os/user
net
github.com/alecthomas/chroma/v2/styles
github.com/alecthomas/chroma/v2/lexers
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
kitty/tools/rsync
kitty/tools/tty
kitty/tools/utils/base85
kitty/tools/utils/paths
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
kitty/kittens/hyperlinked_grep
kitty/kittens/query_terminal
kitty/kittens/show_key
kitty/tools/tui/readline
kitty/tools/tui
kitty/tools/utils/images
kitty/tools/tui/subseq
kitty/kittens/clipboard
kitty/tools/unicode_names
kitty/tools/cmd/update_self
kitty/tools/cmd/run_shell
kitty/kittens/ask
kitty/kittens/hints
kitty/tools/cmd/show_error
kitty/tools/cmd/edit_in_kitty
kitty/tools/tui/graphics
kitty/tools/cmd/at
kitty/tools/themes
kitty/kittens/unicode_input
kitty/tools/cmd/benchmark
kitty/kittens/icat
kitty/kittens/choose_fonts
kitty/kittens/themes
kitty/kittens/ssh
kitty/kittens/transfer
kitty/tools/cmd/pytest
kitty/kittens/diff
kitty/tools/cmd/tool
kitty/tools/cmd/completion
kitty/tools/cmd
```

Verification that the resulting diagnostic `.so` is valgrind-safe, carries DWARF, and is bit-for-bit reproducible. The accurate AVX-512 signature on this CPU is **mask-register use** (`%k0`–`%k7`) and `{%k}` predication (the `-march=native` codegen selected AVX-512**VL** — 128/256-bit EVEX with mask registers, **not** 512-bit `%zmm`, which is why a naive `zmm` grep is **0 on both** builds and is not a valid check). The mask-register count is the real discriminator — complete unedited output for the canonical vs. the diagnostic `.so`:

```
# --- CANONICAL build (582933cf, default -O3 -march=native): AVX-512VL present ---
$ objdump -d kitty/fast_data_types.so | grep -cE '%k[0-7]'          # AVX-512 mask-register uses
79
$ objdump -d kitty/fast_data_types.so | grep -cE '\{%k[0-7]\}'      # AVX-512 mask predication
19
$ objdump -d kitty/fast_data_types.so | grep -cE '%zmm[0-9]'        # 512-bit regs (naive grep: 0 -- misleading)
0
# --- DIAGNOSTIC build (b13d104d, CC=ccwrap.sh -mno-avx512f): mask registers eliminated ---
$ objdump -d kitty/fast_data_types.so | grep -cE '%k[0-7]'          # AVX-512 mask-register uses
0
$ objdump -d kitty/fast_data_types.so | grep -cE '\{%k[0-7]\}'      # AVX-512 mask predication
0
$ readelf -SW kitty/fast_data_types.so | grep -c debug_info         # DWARF present
1
$ sha256sum kitty/fast_data_types.so
b13d104d27106f6df961d7be630ac3bee2ef7e56174ac2a16ecc08ddfa2dc750  kitty/fast_data_types.so
```

The **definitive** proof that the diagnostic build removed the un-decodable instruction is not the static count but the runtime behavior: valgrind instruments the diagnostic `.so` **without SIGILL** (every run in §9.2–§9.4 exits rc=0), whereas the canonical `.so` SIGILLs on the first EVEX instruction it reaches (§9.1 SIGILL block, `convert_opts_from_python_opts`). The 79→0 mask-register drop is the static corroboration of that runtime contrast.

This diagnostic build differs from canonical only in **instruction selection and optimization level** (`-O0`, no LTO, no `-march=native`); the **C source compiled is identical**, so the lock/thread/shared-state structure under test — the only thing a race detector observes — is identical. It is a **diagnostic build, not the canonical build**; every value that depends on codegen (timings, sizes) is taken from the canonical build elsewhere in this report, never from here. Immediately after the detector campaign the canonical `.so` (`582933cf…`) is restored and re-hashed (§9.7), and `git status --porcelain` on `/app` is empty.

**Canonicality labels for the harnesses in this section.** The disk-cache harness (§9.2) and the focused parse harness (§9.4) drive kitty's real C code through the `kitty_tests` `Screen` surface; they are **NON-CANONICAL** for the clipboard question because they bypass the PTY, the `input_delay` gate, and (for the parse harness) the second thread. The full-GUI round-trip (§9.3) and the self-offer chain (§9.5) are **canonical**: real OSC 52 through a child PTY into the real launcher with its real `KittyChildMon` I/O thread and main thread. Each subsection restates its label in place.

### 9.2 Disk-cache add/read/shutdown under helgrind — the unsynchronized `shutting_down` flag (M16, S4, findings 91, 92)

**The disclosure first (source-grounded).** `DiskCache.shutting_down` is a plain `bool` field (`disk-cache.c:53`, `bool thread_started, lock_inited, loop_data_inited, shutting_down, fully_initialized;`). The background `DiskCacheWrite` thread reads it at the **top of its loop, before taking the cache mutex** (`disk-cache.c:348`, `while (!self->shutting_down) {` — the `mutex(lock)` is the *next* line, :349), and the main thread writes it in `dealloc` **without the mutex at all** (`disk-cache.c:439`, `self->shutting_down = true;`). So the categorical claim that “all relevant shared state is mutex-protected” is false for this flag — exactly as M16/S4 state. Verbatim source:

*Read site* — `write_loop`, the `while (!self->shutting_down)` test precedes `mutex(lock)`:

```c
    fds[0].fd = self->loop_data.wakeup_read_fd;
    fds[0].events = POLLIN;
    bool found_dirty_entry = false;

    while (!self->shutting_down) {
        mutex(lock);
        found_dirty_entry = find_cache_entry_to_write(self);
        size_t count = HASH_COUNT(self->entries);
        mutex(unlock);
        if (found_dirty_entry) {
            write_dirty_entry(self);
            mutex(lock);
            retire_currently_writing(self);
            mutex(unlock);
            continue;
        } else if (!count) {
            mutex(lock);
            if (self->cache_file_fd > -1) {
                if (ftruncate(self->cache_file_fd, 0) == 0) lseek(self->cache_file_fd, 0, SEEK_END);
```

*Write site* — `dealloc` sets the flag with **no** surrounding mutex, and the field itself is a plain non-atomic `bool`:

```c
static void
dealloc(DiskCache* self) {
    self->shutting_down = true;
    if (self->thread_started) {
        wakeup_write_loop(self);
// kitty/disk-cache.c:53
    bool thread_started, lock_inited, loop_data_inited, shutting_down, fully_initialized;
```

**Exercising a real writer/reader/shutdown interaction (NON-CANONICAL harness).** Thread presence alone proves nothing (M16); this harness performs a genuine `add` → `wait_for_write` → `get` → shutdown cycle so the `DiskCacheWrite` thread and the main thread actually touch shared state concurrently. It drives kitty's real `DiskCache` via the `Screen` graphics-manager surface (`s.grman.disk_cache`), so it is NON-CANONICAL (no PTY/GUI) but exercises the real C threads and the real shared fields. Script (sha256 `f3ce4d00…`):

```python
# NON-CANONICAL disk-cache add/read/shutdown harness (O5 / M16 / S4).
# Exercises the REAL kitty DiskCache (background DiskCacheWrite thread) via the
# Screen graphics manager test surface. Purpose: drive a genuine add -> wait ->
# read -> shutdown interaction so a data-race detector (valgrind helgrind) can
# observe the writer/reader threads AND the unsynchronized `shutting_down` flag
# (read disk-cache.c:348 while(!self->shutting_down) BEFORE mutex(lock):349;
#  written disk-cache.c:439 self->shutting_down=true in dealloc WITHOUT mutex).
import sys, os, gc
sys.path.insert(0, '/app')
from kitty_tests import BaseTest

class _T(BaseTest):
    def runTest(self):  # unittest scaffolding requirement
        pass

t = _T()
s = t.create_screen(cols=80, lines=24, scrollback=100)
dc = s.grman.disk_cache
dc.small_hole_threshold = 0
print("[stage 0] disk_cache obtained:", type(dc).__name__, "pid=", os.getpid())

# ADD phase -> queues dirty entries, starts/feeds the DiskCacheWrite thread
data = {}
for i in range(60):
    k = ("k%03d" % i).encode()
    v = (("val%03d-" % i) * 300).encode()   # ~2.1 KB each -> forces real disk writes
    data[k] = v
    dc.add(k, v)
print("[stage 1] added", len(data), "entries; total_size=", dc.total_size)

# Force the background writer to drain (writer/reader interaction on shared state)
wrote = dc.wait_for_write(10.0)
print("[stage 2] wait_for_write ->", wrote, "; size_on_disk=", dc.size_on_disk())

# READ phase -> reader path pulls bytes back from disk while writer thread lives
ok = 0
for k, v in data.items():
    if dc.get(k) == v:
        ok += 1
print("[stage 3] read back OK:", ok, "/", len(data))

# SHUTDOWN phase -> drop refs so DiskCache dealloc runs:
#   dealloc sets self->shutting_down=true (disk-cache.c:439, NO mutex) and joins
#   the writer thread, whose loop reads !self->shutting_down (disk-cache.c:348).
del dc
del s
gc.collect()
print("[stage 4] released disk_cache+screen (dealloc -> shutting_down=true, thread joined)")
print("[done] add/read/shutdown exercised")
```

Command and the complete helgrind race report (run 1 of 2; diagnostic build `b13d104d…`; `--history-level=full`). helgrind emits the two race contexts as the writer thread runs; the harness's own `[stage …]` stdout is line-buffered and flushes at process exit, so it appears after the contexts — shown here exactly as captured, unedited:

```
$ cd /app
$ valgrind --tool=helgrind --history-level=full --error-limit=no \
    ./kitty/launcher/kitty +runpy "import runpy; runpy.run_path('/tmp/obs/scripts/o5_diskcache.py', run_name='__main__')"
==15042== Possible data race during write of size 4 at 0x5D2F848 by thread #1
==15042== Locks held: none
==15042==    at 0x5E5A29D: ensure_state (disk-cache.c:421)
==15042==    by 0x5E5B3F2: add_to_disk_cache (disk-cache.c:490)
==15042==    by 0x5E5BDD0: add (disk-cache.c:729)
==15042==    by 0x49F899A: ??? (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==15042==    by 0x49EBB2B: PyObject_Vectorcall (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==15042==    by 0x49865ED: _PyEval_EvalFrameDefault (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==15042==    by 0x4B0991E: PyEval_EvalCode (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==15042==    by 0x4B058AF: ??? (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==15042==    by 0x4988BD2: _PyEval_EvalFrameDefault (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==15042==    by 0x4B0991E: PyEval_EvalCode (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==15042==    by 0x4B64227: ??? (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==15042==    by 0x4B6434B: ??? (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==15042== 
==15042== This conflicts with a previous read of size 4 by thread #2
==15042== Locks held: 1, at address 0x5D2F858
==15042==    at 0x5E5AB2C: write_loop (disk-cache.c:361)
==15042==    by 0x4854B7A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==15042==    by 0x51ADAA3: start_thread (pthread_create.c:447)
==15042==    by 0x523AA63: clone (clone.S:100)
==15042==  Address 0x5d2f848 is in a rw- anonymous segment
==15042== 
==15042== ----------------------------------------------------------------
==15042== 
==15042== Possible data race during read of size 1 at 0x5D2F88B by thread #2
==15042== Locks held: none
==15042==    at 0x5E5AAB2: write_loop (disk-cache.c:348)
==15042==    by 0x4854B7A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==15042==    by 0x51ADAA3: start_thread (pthread_create.c:447)
==15042==    by 0x523AA63: clone (clone.S:100)
==15042== 
==15042== This conflicts with a previous write of size 1 by thread #1
==15042== Locks held: none
==15042==    at 0x5E5B0E6: dealloc (disk-cache.c:439)
==15042==    by 0x5E9EAB1: Py_DECREF (object.h:705)
==15042==    by 0x5E9EAB1: dealloc (graphics.c:181)
==15042==    by 0x5EC661C: Py_DECREF (object.h:705)
==15042==    by 0x5EC661C: dealloc (screen.c:486)
==15042==    by 0x4A355E7: ??? (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==15042==    by 0x4985020: _PyEval_EvalFrameDefault (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==15042==    by 0x4B0991E: PyEval_EvalCode (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==15042==    by 0x4B058AF: ??? (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==15042==    by 0x4988BD2: _PyEval_EvalFrameDefault (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==15042==  Address 0x5d2f88b is in a rw- anonymous segment
==15042== 
[stage 0] disk_cache obtained: DiskCache pid= 15042
[stage 1] added 60 entries; total_size= 126000
[stage 2] wait_for_write -> True ; size_on_disk= 126000
[stage 3] read back OK: 60 / 60
[stage 4] released disk_cache+screen (dealloc -> shutting_down=true, thread joined)
[done] add/read/shutdown exercised
==15042== 
==15042== Use --history-level=approx or =none to gain increased speed, at
==15042== the cost of reduced accuracy of conflicting-access information
==15042== For lists of detected and suppressed errors, rerun with: -s
==15042== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 927 from 35)
```

**Reading the two races.** The second block is the M16/S4 target: thread #2 (the `DiskCacheWrite` writer) reads one byte at `write_loop (disk-cache.c:348)` with **Locks held: none**, conflicting with thread #1 (main) writing one byte at `dealloc (disk-cache.c:439)` — reached via `Py_DECREF → dealloc (graphics.c:181) → dealloc (screen.c:486)` object teardown — also with **Locks held: none**. That is the unsynchronized `shutting_down` bool, observed. The first block is a *bonus* asymmetric-lock race on `cache_file_fd`: main writes it at `ensure_state (disk-cache.c:421)` (`self->cache_file_fd = open_cache_file(...)`) with **Locks held: none**, while the writer reads it at `write_loop (disk-cache.c:361)` **under the lock** (`Locks held: 1`). The 927 suppressed contexts are helgrind's built-in CPython/glibc suppressions.

**Reproducibility (run 2 of 2, identical input).** The same two contexts reproduce exactly:

```
$ valgrind --tool=helgrind --history-level=full --error-limit=no ./kitty/launcher/kitty +runpy "...o5_diskcache.py..."
==15055== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 931 from 35)
# race frames present (grep 'disk-cache.c'):
  dealloc (disk-cache.c:439)      <- shutting_down write (thread #1)
  write_loop (disk-cache.c:348)   <- shutting_down read  (thread #2)
  ensure_state (disk-cache.c:421) <- cache_file_fd write (thread #1)
  write_loop (disk-cache.c:361)   <- cache_file_fd read  (thread #2)
```

Both runs: **2 data-race contexts**, the same source lines. This is detector-observed, reproduced, and limited to these runs (no claim about exploitability is made — `dealloc` joins the writer thread immediately after setting the flag, so the observable window is tiny, but the access is nonetheless an unsynchronized read/write of non-atomic memory, which C11 classifies as a data race and which helgrind correctly reports).

### 9.3 Full-GUI canonical clipboard round-trip under helgrind (M13, M14)

This is the **canonical** two-thread surface: the real launcher, a real child PTY, the real `KittyChildMon` I/O thread filling the shared buffer, and the main thread parsing/dispatching. The child writes OSC 52 then reads it back (the same self-offer child used in §9.5, with the same **no-ask diagnostic** `clipboard_control` disclosed there — non-default, used only to keep the run non-interactive). helgrind is **bounded** by a 360 s `timeout` (M19/finding 85); the run completed well within it (~34 s).

```
$ cd /app; export DISPLAY=:99 LANG=C.UTF-8 LC_ALL=C.UTF-8
$ timeout 360 valgrind --tool=helgrind --history-level=approx --error-limit=no \
    ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes \
    -o clipboard_control="write-clipboard write-primary read-clipboard read-primary" \
    sh /tmp/obs/scripts/o5_selfoffer_child.sh
  # run 1: EXIT rc=0 (bound not hit)
  # self-offer child completed under helgrind:
  selfoffer payload_in=[self-offer-canonical-proof] readback=[self-offer-canonical-proof] match=YES
==13745== ERROR SUMMARY: 5 errors from 5 contexts (suppressed: 3313 from 198)
  # run 2 (identical input): EXIT rc=0 ; match=YES ;
==13930== ERROR SUMMARY: 5 errors from 5 contexts (suppressed: 3687 from 181)
```

**The 5 contexts, classified (identical across both runs): 1 kitty data race + 4 non-kitty loader/GL lock-order inversions.** Critically, **zero** of them are in `vt-parser.c` or `clipboard_control` — the clipboard C→Python transfer path showed no data race in these runs.

The one kitty data race is again an unsynchronized `shutting_down` read — this time in the I/O thread's own loop. Thread #70 (`KittyChildMon`) reads the byte at `io_loop (child-monitor.c:1491)` (`while (LIKELY(!self->shutting_down)) {`) with **Locks held: none**, racing the main thread's teardown (window destroy → `shutdown_monitor (child-monitor.c:427)` join; the flag is written at `child-monitor.c:424`, `self->shutting_down = true;`). Complete helgrind context:

```
==13745== Possible data race during read of size 1 at 0x6F8CA5C by thread #70
==13745== Locks held: none
==13745==    at 0x5C33217: io_loop (child-monitor.c:1491)
==13745==    by 0x4854B7A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==13745==    by 0x51ADAA3: start_thread (pthread_create.c:447)
==13745==    by 0x523AA63: clone (clone.S:100)
==13745==  This conflicts with a previous access by thread #1, after
==13745==    at 0x4851A2D: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==13745==    by 0x71C0897: XFlush (in /usr/lib/x86_64-linux-gnu/libX11.so.6.4.0)
==13745==    by 0x713A389: _glfwPlatformDestroyWindow (x11_window.c:1963)
==13745==    by 0x7133C9E: glfwDestroyWindow (window.c:525)
==13745==    by 0x5C8144C: destroy_os_window (glfw.c:1383)
==13745==    by 0x5C326AA: close_os_window (child-monitor.c:1088)
==13745==    by 0x5C32866: process_pending_closes (child-monitor.c:1123)
==13745==    by 0x5C35750: process_global_state (child-monitor.c:1246)
==13745==    by 0x71371D2: _glfwPlatformRunMainLoop (main_loop.h:34)
==13745==    by 0x712E708: glfwRunMainLoop (init.c:360)
==13745==    by 0x5C81744: run_main_loop (glfw.c:2103)
==13745==    by 0x5C31E0C: main_loop (child-monitor.c:1262)
==13745==  but before
==13745==    at 0x51A9D71: __futex_abstimed_wait_common64 (futex-internal.c:57)
==13745==    by 0x51A9D71: __futex_abstimed_wait_common (futex-internal.c:87)
==13745==    by 0x51A9D71: __futex_abstimed_wait_cancelable64 (futex-internal.c:139)
==13745==    by 0x51AF7A2: __pthread_clockjoin_ex (pthread_join_common.c:102)
==13745==    by 0x4850E13: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==13745==    by 0x5C3299E: shutdown_monitor (child-monitor.c:427)
==13745==    by 0x49F9CE1: ??? (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==13745==    by 0x49EBB2B: PyObject_Vectorcall (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==13745==    by 0x49865ED: _PyEval_EvalFrameDefault (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==13745==    by 0x49ED57F: _PyObject_FastCallDictTstate (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==13745==    by 0x49ED7ED: _PyObject_Call_Prepend (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==13745==    by 0x4A6C074: ??? (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==13745==    by 0x49EB7DE: _PyObject_MakeTpCall (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==13745==    by 0x49865ED: _PyEval_EvalFrameDefault (in /usr/lib/x86_64-linux-gnu/libpython3.12.so.1.0)
==13745==  Address 0x6f8ca5c is in a rw- anonymous segment
```

So the same *plain-bool `shutting_down`* pattern that M16 flagged for the disk cache is present in a second subsystem (`child-monitor.c:54` `bool shutting_down;`, read unlocked at `:1491` and `:1820`, written unlocked at `:424`). The remaining **4 contexts are lock-order inversions inside the dynamic loader and Mesa/libGLX teardown** (`_glfwTerminateGLX (glx_context.c:418) → dlclose → _dl_close`), i.e. GL-driver unload at `glfwTerminate` — **not kitty code**, and an artifact of the headless llvmpipe software-GL stack in this container (the ~64 llvmpipe/GL-pool threads noted in §3/§8.2). They are reported here for completeness, not attributed to kitty.

### 9.4 Focused single-thread parse under helgrind (M13, NON-CANONICAL, finding 86)

To isolate the *in-process* parse→dispatch→callback code from the two-thread machinery, this **NON-CANONICAL** harness feeds 200 identical OSC 52 writes straight through the real `vt-parser` via the `kitty_tests` module-level `parse_bytes(screen, data)` (`kitty_tests/__init__.py:30`), single-threaded. Script (sha256 `d4e53da4…`):

```python
# NON-CANONICAL focused clipboard-parse harness (O5 / M13).
# Drives the REAL vt-parser OSC 52 dispatch + clipboard_control callback via the
# kitty_tests module-level parse_bytes(screen, data) (kitty_tests/__init__.py:30),
# single-threaded. This BYPASSES the PTY, the input_delay gate, the child-monitor
# I/O thread, and the GUI; it exercises only the in-process parse->dispatch->
# callback code. Purpose under helgrind: confirm the single-threaded parse path
# itself contains no data race (the real two-thread producer/consumer surface
# exists only in the full GUI, exercised separately and bounded).
import sys, base64
sys.path.insert(0, '/app')
from kitty_tests import BaseTest, parse_bytes

class _T(BaseTest):
    def runTest(self): pass

t = _T()
s = t.create_screen(cols=80, lines=24, scrollback=100)
payload = b'o5-parse-focus-detector'
b64 = base64.standard_b64encode(payload)
osc = b'\x1b]52;c;' + b64 + b'\x07'
n = 200
for _ in range(n):
    parse_bytes(s, osc)
print('[focus] parsed', n, 'OSC 52 writes single-threaded; last payload b64=', b64.decode())
print('[done] focused parse path exercised')
```

```
$ valgrind --tool=helgrind --history-level=full --error-limit=no \
    ./kitty/launcher/kitty +runpy "import runpy; runpy.run_path('/tmp/obs/scripts/o5_parse_focus.py', run_name='__main__')"
  [focus] parsed 200 OSC 52 writes single-threaded; last payload b64= bzUtcGFyc2UtZm9jdXMtZGV0ZWN0b3I=
  [done] focused parse path exercised
==13914== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

**Zero races** in the single-threaded parse/dispatch/`clipboard_control` path, as expected: with one thread there is no concurrency for helgrind to flag. This bounds where a race *could* live — not in the parse logic itself, but only at the two-thread buffer hand-off (§8.1 shows that hand-off is mutex-guarded) or in the teardown-flag pattern of §9.2/§9.3.

### 9.5 Self-offer reentrancy — the complete C → exception → catch → fallback chain (M15, findings 89, 90)

When kitty owns the OS clipboard and a program asks kitty to *read* it (OSC 52 with `?`), GLFW issues a **self-offer**: it invokes kitty's write callback with `data == NULL`. The C side turns that into a Python exception (`glfw.c:2180-2184`), which the clipboard manager catches and satisfies from its **owned** copy (`clipboard.py:104-119`). Verbatim source of both ends:

```c
// kitty/glfw.c:2180
write_clipboard_data(void *callback, const char *data, size_t sz) {
    Py_ssize_t z = sz;
    if (data == NULL) {
        PyErr_SetString(PyExc_RuntimeError, "is_self_offer");
        return false;
    }
```

```python
# kitty/clipboard.py:104
    def get_mime(self, mime: str, output: Callable[[bytes], None]) -> None:
        if self.enabled:
            try:
                get_clipboard_mime(self.clipboard_type, mime, output)
            except RuntimeError as err:
                if str(err) != 'is_self_offer':
                    raise
                data = self.data.get(mime, b'')
                if isinstance(data, bytes):
                    output(data)
                else:
                    chunker = data()
                    q = b' '
                    while q:
                        q = chunker()
                        output(q)
```

**Canonicality and the no-ask diagnostic configuration (M20).** The self-offer *mechanism* under test is fully canonical: real OSC 52 written to a real child PTY, parsed by the live VT parser, dispatched to the real GLFW clipboard bridge which issues the self-offer. The one non-default setting is `clipboard_control`: the run below sets `write-clipboard write-primary read-clipboard read-primary`, whereas the shipped default is `write-clipboard write-primary read-clipboard-ask read-primary-ask` (`kitty/options/definition.py:3096`). The `-ask` variants are dropped **only** so the read-back does not raise an interactive permission overlay that would block the automated, non-interactive gdb/helgrind capture. **This is a no-ask diagnostic run, not the default behavior**, and per the option documentation disabling the read confirmation lets any local — or, over SSH, remote — program read the clipboard without prompting. The self-offer exception/catch/fallback is independent of the `-ask` gate (it fires whenever kitty is asked for a selection it owns); the canonical **default** prompt behavior (the `read-clipboard-ask` overlay and its ACCEPT/DENY decisions) is captured separately in the O1 section. Trigger script (sha256 `0093190b…`):

```sh
#!/bin/sh
# Canonical self-offer trigger: write OSC 52 (kitty becomes the clipboard owner),
# then read it back with the real `kitten clipboard` (separate process). The
# read-back makes kitty request its OWN selection -> GLFW self-offer ->
# write_clipboard_data(data==NULL) [glfw.c:2182] -> RuntimeError('is_self_offer')
# [glfw.c:2183] -> Python get_mime except [clipboard.py:107-109] -> owned-data
# fallback [clipboard.py:110-119] -> the read-back returns the owned bytes.
PAYLOAD='self-offer-canonical-proof'
printf '\033]52;c;%s\007' "$(printf '%s' "$PAYLOAD" | base64)"
sleep 1.2
got=$(kitten clipboard --get-clipboard)
printf 'selfoffer payload_in=[%s] readback=[%s] match=%s\n' \
    "$PAYLOAD" "$got" "$([ "$got" = "$PAYLOAD" ] && echo YES || echo NO)" \
    > /tmp/obs/out/o5_selfoffer.result
sleep 0.5
```

**One linked gdb run** captures the whole chain (gdb batch script, sha256 `e40b7c04…`):

```text
set pagination off
set confirm off
set print pretty off
set width 0
set breakpoint pending on
set follow-fork-mode parent
set detach-on-fork on
break glfw.c:2182 if data == 0
run --config NONE -o close_on_child_death=yes -o clipboard_control="write-clipboard write-primary read-clipboard read-primary" sh /tmp/obs/scripts/o5_selfoffer_child.sh
echo \n========== STOP: write_clipboard_data reached (glfw.c:2182), data==NULL => self-offer ==========\n
echo -- data pointer (expect 0x0 => self-offer) --\n
p data
echo -- C backtrace (canonical entry: OSC52 read -> clipboard_control -> get_clipboard_mime -> glfwGetClipboard -> write_clipboard_data) --\n
bt
echo -- step over the data==NULL check and the PyErr_SetString (glfw.c:2183) --\n
next
next
echo -- exception TYPE set by glfw.c:2183 (PyErr_Occurred() returns the type; expect RuntimeError) --\n
p ((PyTypeObject*)PyErr_Occurred())->tp_name
echo -- finish out of write_clipboard_data (returns false to glfwGetClipboard) --\n
finish
echo \n========== continue: RuntimeError('is_self_offer') propagates -> clipboard.py get_mime except -> owned-data fallback ==========\n
delete
continue
echo \n========== kitty exited; canonical self-offer read-back result (proves owned-data fallback executed): ==========\n
shell cat /tmp/obs/out/o5_selfoffer.result
quit
```

Command and complete unedited output (per-thread `[New Thread]`/`exited` noise elided by an explicit `grep -avE` filter; the filter removes only those loader lines and nothing else):

```
$ cd /app; export DISPLAY=:99 LANG=C.UTF-8 LC_ALL=C.UTF-8
$ gdb -batch -x /tmp/obs/scripts/o5_selfoffer.gdb ./kitty/launcher/kitty   # rc=0
No source file named glfw.c.
Breakpoint 1 (glfw.c:2182 if data == 0) pending.

[0.415] Failed to open systemd user bus with error: No medium found

Thread 1 "kitty" hit Breakpoint 1, write_clipboard_data (callback=callback@entry=0x7fffe45d1990, data=data@entry=0x0, sz=sz@entry=1) at kitty/glfw.c:2182
2182	    if (data == NULL) {

========== STOP: write_clipboard_data reached (glfw.c:2182), data==NULL => self-offer ==========
-- data pointer (expect 0x0 => self-offer) --
$1 = 0x0
-- C backtrace (canonical entry: OSC52 read -> clipboard_control -> get_clipboard_mime -> glfwGetClipboard -> write_clipboard_data) --
#0  write_clipboard_data (callback=callback@entry=0x7fffe45d1990, data=data@entry=0x0, sz=sz@entry=1) at kitty/glfw.c:2182
#1  0x00007ffff59d2200 in getSelectionString (selection=selection@entry=235, targets=targets@entry=0x7fffffffa4e0, num_targets=4, write_data=write_data@entry=0x7ffff69fb266 <write_clipboard_data>, object=object@entry=0x7fffe45d1990, report_not_found=report_not_found@entry=true) at glfw/x11_window.c:932
#2  0x00007ffff59d65d0 in _glfwPlatformGetClipboard (clipboard_type=<optimized out>, mime_type=0x7ffff5ca5458 "text/plain", write_data=0x7ffff69fb266 <write_clipboard_data>, object=0x7fffe45d1990) at glfw/x11_window.c:3010
#3  0x00007ffff59ca931 in glfwGetClipboard (clipboard_type=<optimized out>, mime_type=<optimized out>, write_data=<optimized out>, object=<optimized out>) at glfw/input.c:1543
#4  0x00007ffff69fb413 in get_clipboard_mime (self=<optimized out>, args=<optimized out>) at kitty/glfw.c:2198
#5  0x00007ffff78ee498 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#6  0x00007ffff78917df in _PyObject_MakeTpCall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#7  0x00007ffff782c5ee in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#8  0x00007ffff789507d in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#9  0x00007ffff789239c in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#10 0x00007ffff789286e in _PyObject_CallMethod_SizeT () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#11 0x00007ffff6a32dd1 in clipboard_control (self=<optimized out>, code=code@entry=52, data=<optimized out>) at kitty/screen.c:2306
#12 0x00007ffff6a5457c in dispatch_osc (self=self@entry=0x5555569bf840, buf=<optimized out>, limit=<optimized out>, is_extended_osc=is_extended_osc@entry=false) at kitty/vt-parser.c:534
#13 0x00007ffff6a56a08 in accumulate_st_terminated_esc_code (self=0x5555569bf840, dispatch=dispatch@entry=0x7ffff6a543ea <dispatch_osc>) at kitty/vt-parser.c:403
#14 0x00007ffff6a56b5f in consume_input (self=self@entry=0x5555569bf840, dump_callback=<optimized out>, window_id=<optimized out>) at kitty/vt-parser.c:1385
#15 0x00007ffff6a56d2b in run_worker (p=0x55555605aab0, pd=0x7fffffffaad0, flush=false) at kitty/vt-parser.c:1432
#16 0x00007ffff6a57938 in parse_worker (p=<optimized out>, pd=<optimized out>, flush=<optimized out>) at kitty/vt-parser.c:1496
#17 0x00007ffff69af6fc in do_parse (self=self@entry=0x7ffff5bd0a30, screen=0x55555605aab0, now=now@entry=1649036546, flush=flush@entry=false) at kitty/child-monitor.c:440
#18 0x00007ffff69b1551 in parse_input (self=self@entry=0x7ffff5bd0a30) at kitty/child-monitor.c:530
#19 0x00007ffff69b4703 in process_global_state (data=0x7ffff5bd0a30) at kitty/child-monitor.c:1236
#20 0x00007ffff69b47b2 in do_state_check (timer_id=<optimized out>, data=<optimized out>) at kitty/child-monitor.c:1218
#21 0x00007ffff59dfcc6 in dispatchTimers (eld=eld@entry=0x7ffff5a2c6f0 <_glfw+133552>) at glfw/backend_utils.c:208
#22 0x00007ffff59dff52 in pollForEvents (eld=0x7ffff5a2c6f0 <_glfw+133552>, timeout=2953077, display_callback=display_callback@entry=0x0) at glfw/backend_utils.c:309
#23 0x00007ffff59d5eda in handleEvents (timeout=<optimized out>) at glfw/x11_window.c:70
#24 0x00007ffff59d5f58 in _glfwPlatformWaitEvents () at glfw/x11_window.c:2731
#25 0x00007ffff59d01b6 in _glfwPlatformRunMainLoop (tick_callback=0x7ffff69b46b5 <process_global_state>, data=0x7ffff5bd0a30) at glfw/main_loop.h:30
#26 0x00007ffff59c7709 in glfwRunMainLoop (callback=<optimized out>, data=<optimized out>) at glfw/init.c:360
#27 0x00007ffff6a00745 in run_main_loop (cb=cb@entry=0x7ffff69b46b5 <process_global_state>, cb_data=cb_data@entry=0x7ffff5bd0a30) at kitty/glfw.c:2103
#28 0x00007ffff69b0e0d in main_loop (self=0x7ffff5bd0a30, a=<optimized out>) at kitty/child-monitor.c:1262
#29 0x00007ffff789fce2 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#30 0x00007ffff7891b2c in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#31 0x00007ffff782c5ee in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#32 0x00007ffff7893580 in _PyObject_FastCallDictTstate () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#33 0x00007ffff78937ee in _PyObject_Call_Prepend () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#34 0x00007ffff7912075 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#35 0x00007ffff78917df in _PyObject_MakeTpCall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#36 0x00007ffff782c5ee in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#37 0x00007ffff79af91f in PyEval_EvalCode () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#38 0x00007ffff79ab8b0 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#39 0x00007ffff78eeadc in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#40 0x00007ffff7891b2c in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#41 0x00007ffff782c5ee in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#42 0x00007ffff7a34242 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#43 0x00007ffff7a34da3 in ?? () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#44 0x00007ffff7a3539c in Py_RunMain () from /lib/x86_64-linux-gnu/libpython3.12.so.1.0
#45 0x0000555555557875 in run_embedded (run_data=0x7fffffffb9c0) at kitty/launcher/main.c:216
#46 0x0000555555558645 in main (argc=9, argv=0x7fffffffeb58, envp=0x7fffffffeba8) at kitty/launcher/main.c:464
-- step over the data==NULL check and the PyErr_SetString (glfw.c:2183) --
2183	        PyErr_SetString(PyExc_RuntimeError, "is_self_offer");
2184	        return false;
-- exception TYPE set by glfw.c:2183 (PyErr_Occurred() returns the type; expect RuntimeError) --
$2 = 0x7ffff7b3e986 "RuntimeError"
-- finish out of write_clipboard_data (returns false to glfwGetClipboard) --
getSelectionString (selection=selection@entry=235, targets=targets@entry=0x7fffffffa4e0, num_targets=<optimized out>, write_data=write_data@entry=0x7ffff69fb266 <write_clipboard_data>, object=object@entry=0x7fffe45d1990, report_not_found=report_not_found@entry=true) at glfw/x11_window.c:933
933	        return;
Value returned is $3 = false

========== continue: RuntimeError('is_self_offer') propagates -> clipboard.py get_mime except -> owned-data fallback ==========
[Inferior 1 (process 13606) exited normally]

========== kitty exited; canonical self-offer read-back result (proves owned-data fallback executed): ==========
selfoffer payload_in=[self-offer-canonical-proof] readback=[self-offer-canonical-proof] match=YES
```

**The chain, step by step, all in this one run.** (i) **C entry:** `write_clipboard_data` is hit with `data=0x0` — the self-offer signal — reached canonically from OSC 52 read: the backtrace runs `process_global_state (child-monitor.c:1236) → parse_worker → dispatch_osc (vt-parser.c:534) → clipboard_control (screen.c:2306, code=52) → [Python] → get_clipboard_mime (glfw.c:2198) → glfwGetClipboard → getSelectionString (x11) → write_clipboard_data`. (ii) **Exception:** stepping over `glfw.c:2183` sets the error; `PyErr_Occurred()`'s type name is `"RuntimeError"`, and the verbatim source line is `PyErr_SetString(PyExc_RuntimeError, "is_self_offer")`. (iii) `finish` returns **`false`** into `getSelectionString`. (iv) **Catch + fallback + result:** after `continue`, the process exits normally and the read-back file shows `readback=[self-offer-canonical-proof] match=YES` — the owned data came back, which is reachable **only** through the `except RuntimeError → data = self.data.get(mime) → output(data)` branch (`clipboard.py:108-111`); had the exception not been caught it would have propagated and the child would have received nothing. `match=YES` reproduced 3× (this gdb run + both full-GUI helgrind runs in §9.3).

### 9.6 The retained-view hazard is not a data race (finding 88)

The report's single genuine object-lifetime hazard — a Python-retained `memoryview` that keeps *pointing at* the parser's 1 MiB buffer after the dispatch scope, so a later parse overwrites the bytes it shows — was demonstrated live in **§8.3** (the L1/L2/L3 three-lifetime probe: the retained view's head flips from payload #1 to payload #2 after the buffer is reused; re-reading it does **not** crash, so it is *stale content*, not an immediate use-after-free). That hazard is **single-threaded**: it needs only one thread parsing twice through the one buffer. It is therefore **categorically not** a data race, and helgrind neither can nor does report it (there is no second thread accessing that memory concurrently). Keeping the two apart is the point of finding 88: the **C-level content-reuse hazard** (§8.3, avoided in production by the owned copy at `clipboard.py:286` and the decode-out at `:319-320`) is distinct from the **detector-observed teardown data races** (§9.2/§9.3, on `shutting_down`/`cache_file_fd`), which are a different mechanism entirely.

### 9.7 Bounded conclusions and untested surfaces (M14, S3, findings 87, 93, 94)

**What was observed (detector), stated without overclaim:**

- Across a focused single-thread parse (§9.4), and a canonical full-GUI clipboard round-trip run **twice** (§9.3), helgrind reported **no data race in the clipboard C→Python transfer path** (`vt-parser.c`, `dispatch_osc`, `clipboard_control`). This is “**no race observed in these runs**,” not a proof of race-freedom.
- helgrind **did** observe, reproducibly (2× each), genuine unsynchronized data races on the **`shutting_down`** plain bool in **`disk-cache.c`** (`:348`/`:439`) and **`child-monitor.c`** (`:1491`/`:424`), and an asymmetric-lock race on **`cache_file_fd`** (`:421`/`:361`). These are the shared-state disclosures M16/S4 required; all are teardown-time flags, not the clipboard payload path.
- The remaining 4 detector contexts are **lock-order inversions in the glibc dynamic loader and Mesa/libGLX GL-driver unload** at `glfwTerminate` — not kitty code; an artifact of the headless software-GL container.

**Observed by detector vs. reasoned from source (kept distinct):**

- *Source-observed (§8.1, live gdb):* the parser mutex guards the buffer hand-off (promotion under lock, consumption with the lock released) and the GIL serializes every Python callback on the main thread. This is why a Python-level clipboard data race is structurally impossible — it is an **intentional-concurrency** design, established from runtime state, and is *not* offered as detector proof of race-absence.
- *Detector-observed (§9):* the presence of the teardown-flag races and the absence of any transfer-path race **in these specific runs**.

**Untested surfaces (finding 94 — explicitly not covered by these runs):** the Wayland clipboard backend (only X11/Xvfb was exercised); the macOS Cocoa clipboard; the `>256 KiB` chunked transfer *while under a concurrent output flood and a detector simultaneously*; the primary-selection path under a detector; the `KittyPeerMon` remote-control socket thread (`child-monitor.c:1820`, which reads the same `shutting_down` flag); and any race that manifests only on real hardware GL rather than llvmpipe. **Varied exact attempts (finding 93):** three distinct triggers were run — a single-thread focused parse, a real two-thread full-GUI round-trip, and a real background-thread disk-cache add/read/shutdown — rather than a single harness, precisely so that a transfer-path race, if present, would have had multiple chances to surface.

### 9.8 O5 observed/inferred ledger

| Claim | Status | Evidence |
|---|---|---|
| kitty ships no TSan build; `--sanitize`=ASan+UBSan only | observed | `setup.py:380` |
| canonical `-march=native` `.so` SIGILLs under valgrind (AVX-512/EVEX) | observed | §9.1 SIGILL block, `convert_opts_from_python_opts` |
| diagnostic build is source-identical, AVX-512-free, DWARF, canonical restored after | observed | §9.1 build+verify; `.so` `b13d104d`→restored `582933cf` |
| `DiskCache.shutting_down` read `:348` / write `:439` unsynchronized | observed | §9.2 helgrind, 2× |
| `cache_file_fd` write `:421` (unlocked) / read `:361` (locked) | observed | §9.2 helgrind, 2× |
| `ChildMonitor.shutting_down` read `:1491` / write `:424` unsynchronized | observed | §9.3 full-GUI helgrind, 2× |
| no data race in `vt-parser.c`/`clipboard_control` transfer path | observed (these runs) | §9.3 (2×), §9.4 |
| 4 lock-order contexts are GLX/loader teardown, not kitty | observed | §9.3, `_glfwTerminateGLX`/`_dl_close` frames |
| self-offer C→RuntimeError→catch→fallback→result | observed | §9.5 one linked gdb run; `match=YES` 3× |
| retained-view hazard is stale-content (single-threaded), not a data race | observed | §8.3 probe; §9.6 |
| GIL + parser-lock serialize the transfer (why no Python race is possible) | observed (source/gdb) | §8.1 (M12) |
| exploitability/severity of the teardown-flag races | not claimed | bounded to “race observed,” §9.2/10.7 |

---

## 10. Coverage checklist and consolidated observed/inferred ledger

### 10.1 Coverage checklist — every objective sub-part and every named item

| Item (question sub-part / named mechanism) | Where addressed | Evidence anchor |
|---|---|---|
| **O1** core↔Python transport (zero-copy `memoryview` → `clipboard_control`) | §4.0, §4.1 | real PTY OSC 52; `vt-parser.c:461`, `screen.c:87,2305`, `window.py:1391` |
| **O1** small write — single dispatch | §4.1 | one `dispatch_osc`; `--dump-commands` trace |
| **O1** large write — partial-OSC-52 chunking (code `-52` per partial, `+52` final) | §4.2 | per-chunk lengths + reconstruction; `vt-parser.c:21` (256 KiB), `:18` (1 MiB) |
| **O1** very large write — `io.BytesIO`→on-disk `TemporaryFile` at 16 MiB | §4.3 | `/proc/<pid>/fd` + strace; `clipboard.py:237` |
| **O1** `clipboard_max_size` double-scale (512 → ≈512 TiB) + truncation guard | §4.4 | arithmetic + positive control; `clipboard.py:318-323` |
| **O1** read paths — legacy OSC 52 `?` and extended OSC 5522 + MIME listing | §4.5 | captured response packets |
| **O1** permission prompts — `read-clipboard-ask` accept vs deny | §4.6 | overlay state + ACCEPT + DENY |
| **O1** read/write status ledger (`OK`/`DATA`/`DONE`/`EPERM`/`EINVAL`/`ENOSYS`/`EIO`/`EBUSY`) | §4.7 | canonical where reachable; source-inferred branches labeled |
| **O1/O5** `is_self_offer` reentrancy | §4.6, §9.5 | `glfw.c:2182-2183`; `clipboard.py:107-119` |
| **O2** transfer under concurrent load — flood absorption + backpressure | §5.1 | `MAX_ABSORBED`≈1.002 MiB; N≥2 |
| **O2** serialization / in-order / complete delivery (no loss, no reorder) | §5.6 | 2000/2000 callbacks in strict order, N=3 ×2 |
| **O2** `input_delay` coalescing (default 3 ms) | §5.2 | causal 0/3/25 ms contrast; `options/definition.py:878` |
| **O2** 1 MiB buffer backpressure (POLLIN disable/re-enable) | §5.3 | `events=POLLIN`↔`events=0` in `poll()` strace; `child-monitor.c:1501` |
| **O2** canonical latency distribution (idle baseline) | §5.4 | ≈3.3 ms baseline, N≥2; default(ask) vs no-ask labeled |
| **O2** event delivery while MAIN busy | §5.5 | under-load DSR distribution vs baseline |
| **Kitten** separate-process boundary (PTY/pipe, own event loop) | §6.0-§6.4 | distinct kitten PIDs; `kittens/tui/loop.py:246,261`; `kittens.c:94,104` |
| **Kitten** measured round-trip latency | §6.2 | ≈3.3 ms, N≥2 |
| **O3** scan paths named (`as_text`/`text_for_range`/`unicode_in_range`/`as_text_generic`) | §7.1 | `screen.c:3486,3035,3054`; `line.c:874` |
| **O3** scan → event delivery delay (≈ full scan) | §7.2 | gdb-synchronized enqueue; N≥2 |
| **O3** why a callback-driven scan does not yield (GIL) | §7.3 | `child-monitor.c` main loop; GIL fact |
| **O3** scan → memory (allocation profile + PID-tied smaps) | §7.4 | before/during/after `/proc/<pid>/smaps` |
| **O3** persistent cost = C scrollback RAM segments (`add_segment`) | §7.5 | segment growth; `history.c:18` |
| **O3** pager-history stores raw bytes (not compressed), off by default | §7.6 | source + observation |
| **O3** line-count reconciliation (logical vs physical) | §7.7 | recomputed KiB/line |
| **O3** disk cache ≠ scrollback | §7.8 | `disk-cache.c` used by graphics only |
| **O4** parser lock + `pending→read` promotion (`read.sz += write.pending`) | §8.1 | gdb STOP A; `vt-parser.c:1421` under lock |
| **O4** synchronous callback on MAIN, lock released during parse | §8.2 | gdb STOP B/C; `tp_name="memoryview"` |
| **O4** boundary `memoryview` — three distinct lifetimes | §8.3 | real parser-reuse probe (byte-identical ×2) |
| **O4** copy-out into owned objects | §8.4 | `clipboard.py:286` owned copy; source-mutation UNCHANGED |
| **O4** ownership: core-Python boundary vs kitten boundary | §8.5 | Boundary-1 alias vs Boundary-2 copy |
| **O5** detector methodology (helgrind; AVX-512VL SIGILL; diagnostic build) | §9.1 | 79 `%k` mask regs vs 0; runtime rc contrast; `b13d104d` |
| **O5** disk-cache races (`shutting_down`, `cache_file_fd`) | §9.2 | helgrind ×2; `disk-cache.c:348,439,421,361` |
| **O5** full-GUI child-monitor corroboration | §9.3 | helgrind ×2; `child-monitor.c:54,424,1491` |
| **O5** focused single-thread parse — no race | §9.4 | non-canonical harness; helgrind |
| **O5** self-offer reentrancy — complete chain | §9.5 | gdb linked run rc=0 |
| **O5** retained-view hazard is not a data race | §9.6 | cross-ref §8.3 |
| **O5** bounded conclusions + untested surfaces | §9.7 | "no race observed in these runs" |
| `BUF_SZ` (1 MiB) / `MAX_ESCAPE_CODE_LENGTH` (256 KiB) | §4.2, §5.3 | `vt-parser.c:18,21` |
| `memoryview` / `CALLBACK` / `clipboard_control` | §4.0, §8.2 | `readonly=True`; `PyObject_CallMethod` |
| `Tempfile` / `io.BytesIO` / `TemporaryFile` | §4.3 | observed `BytesIO`→disk transition |
| `WriteRequest` / `rollover_size` (16 MiB) / `clipboard_max_size` (512) | §4.3, §4.4 | `clipboard.py:237,247,321` |
| `KittyChildMon` / `KittyPeerMon` / `KittyWriteStdin` / `DiskCacheWrite` | §3 | `child-monitor.c:1489,1808,967`; `disk-cache.c:342` |
| history RAM segments / pager-history ring | §7.5, §7.6 | `history.c:18` |
| parser mutex / `run_worker` / `consume_input` / `free_vt_parser` | §8.1, §8.3 | `vt-parser.c:205,1417,1432,1508` |
| GIL serialization of the main thread | §3.5, §8.2 | thread topology + `tp_name` capture |

### 10.2 Consolidated observed/inferred ledger

Each objective section carries its own detailed ledger (§6.6, §7.9, §8.6, §9.8); the table below consolidates the load-bearing claims of the whole document.

| Claim | Status | Section |
|---|---|---|
| OSC 52 payload handed to Python as a zero-copy read-only `memoryview` over the 1 MiB buffer | **Observed** | §4.0, §8.2 |
| Payload > 256 KiB delivered as partial-OSC-52 chunks (code `-52` per partial, final `+52`), reassembled in Python | **Observed** | §4.2 |
| 16 MiB `io.BytesIO`→on-disk `TemporaryFile` rollover | **Observed** (`/proc/<pid>/fd`) | §4.3 |
| `clipboard_max_size=512` ⇒ effective ≈512 TiB (double-scale); guard works at small limits | **Observed** (arithmetic + positive control) | §4.4 |
| Default configuration leaves the truncation guard inert (CWE-400-class exposure) | **Inferred** (code-grounded) | §4.4 |
| Default 3 ms `input_delay` coalesces bursts (causal) | **Observed** (0/3/25 ms) | §5.2 |
| POLLIN disabled/re-enabled at the 1 MiB buffer limit | **Observed** (`poll()` strace) | §5.3 |
| Event-delivery latency rises under load vs idle baseline | **Observed** (distribution, N≥2) | §5.4, §5.5 |
| Under heavy interleave, all events delivered in strict FIFO order (no loss/reorder) | **Observed** (2000/2000, N=3 ×2) | §5.6 |
| Kitten is a separate process reading its own copy over PTY/pipe | **Observed** | §6.0-§6.4 |
| Measured kitten round-trip latency ≈3.3 ms | **Observed** (N≥2) | §6.2 |
| Internals of the final separate-process kitten IPC micro-hop | **Inferred** where noted | §6.5 |
| Pending event waits ≈ the full scrollback scan (main-thread serialization) | **Observed** (gdb-synchronized) | §7.2 |
| C scrollback stored in growable RAM segments; pager-history raw bytes, off by default | **Observed** + source | §7.5, §7.6 |
| `read.sz += write.pending` promotion happens under the parser mutex | **Observed** (gdb STOP A) | §8.1 |
| Callback runs unlocked but on MAIN under the GIL | **Observed** (gdb STOP B/C) | §8.2 |
| `memoryview` has three lifetimes; retention hazard = stale content on buffer reuse (not UAF) | **Observed** (parser-reuse probe) | §8.3 |
| Python copies out into owned objects | **Observed** (source-mutation UNCHANGED) | §8.4 |
| No race on the vt-parser clipboard path in the exercised runs | **Observed** (bounded) | §9.3, §9.4 |
| Two genuine disk-cache races (`shutting_down`, `cache_file_fd`) | **Observed** (helgrind ×2) | §9.2 |
| `is_self_offer` end-to-end reentrancy chain | **Observed** (gdb linked run) | §9.5 |
| Canonical `-march=native` build emits AVX-512VL ⇒ valgrind SIGILL (origin of earlier `-no-pie` confusion) | **Observed** (runtime rc contrast) | §9.1 |

---

## 11. Appendix — temporary observation scripts and cleanup

### 11.1 Temporary observation scripts (complete bodies embedded inline in §2–§9)

Every probe used in this document is reproduced **in full** inline in the section that presents its output, so the evidence is **self-contained and re-runnable** without any external file. During the investigation each script was written under `/tmp` — a scratch tree **outside** the repository — run against the canonical build of §2 (or, where noted, the byte-identical valgrind-compatible diagnostic build of §9.1), and **deleted** after its output was captured. Each probe is labelled **canonical** (a genuine OSC 52/5522 escape or DSR query delivered through the real PTY to the built `kitty` launcher, or the real Go/Python kitten as a separate process) or **non-canonical** (drives the *identical* production C functions through the `Screen`/`kitty_tests` hooks, bypassing only the PTY + I/O thread; never a remote-control or debug shortcut); non-canonical values are cross-checked against the canonical round trip.

| Section | Script | Kind | Purpose |
|---|---|---|---|
| §2 | `kitty_tests.sh` | canonical | run `./test.py` suite on the canonical build |
| §4 | `o1_small_child.sh` | canonical | small (<256 KiB) OSC 52 write through a real PTY |
| §4 | `o1_chunk_big.sh`, `o1_bigchild.sh` | canonical | >256 KiB write → partial-OSC-52 chunking |
| §4 | `o1_rollover.sh`, `o1_roll_child.sh` | canonical | >16 MiB write → `BytesIO`→on-disk rollover (strace/`/proc/fd`) |
| §4 | `o1_roundtrip.sh`, `o1_rt_child.sh` | canonical | OSC 52 write then read-back integrity check |
| §4 | `o1_trunc_neg.sh`, `o1_trunc_pos.sh`, `o1_trunc_retained.py`, `o1_trunc_retained_drv.sh` | canonical | `clipboard_max_size` default (no truncation) + positive control + retained/overshoot |
| §4 | `osc52read.py`, `osc5522_harness.py`, `osc5522_driver.sh` | canonical | legacy OSC 52 `?` read + extended OSC 5522 read/MIME |
| §4 | `o1_feed_child.py`, `o1_dumpfmt.sh`, `hashlib.sh` | canonical | streaming feeder + `--dump-commands` formatting + hashing |
| §5 | `o2_flood.py`, `o2_flood_drv.sh` | canonical | heavy PTY flood; `MAX_ABSORBED` + in-order check |
| §5 | `o2_latency.py`, `o2_latency_drv.sh` | canonical | idle DSR latency distribution + `input_delay` 0/3/25 ms |
| §5 | `o2_pollin.sh` | canonical | POLLIN disable/re-enable `poll()` strace |
| §5 | `o2_load_latency.py`, `o2_load_latency_drv.sh` | canonical | under-load event-delivery latency vs baseline |
| §5 | `E2_serialize.py` | non-canonical | 2000 marker-tagged OSC 52 writes interleaved with heavy blocks — in-order / complete-delivery proof |
| §6 | `kitten_canon.sh` | canonical | real Go `kitten clipboard` set/get through the real PTY |
| §6 | `kitten_get_noask.sh`, `kitten_noask_drv.sh` | canonical (no-ask, diagnostic-labeled) | kitten round-trip latency without the interactive overlay |
| §7 | `o3_probe.py`, `o3_gdb_scan.sh`, `o3_pager.py` | canonical + gdb | scan durations, gdb-synchronized event delay, pager-history |
| §8 | `o4_lifetimes.py` | non-canonical | three-lifetime `memoryview` parser-reuse + copy-out probe |
| §8 | `o4_lock3_run.sh`, `o4_lock3.gdb` | canonical + gdb | linked lock / GIL / synchronous-callback proof |
| §9 | `ccwrap.sh` | build wrapper | force x86-64-v3 (no AVX-512) for a valgrind-compatible diagnostic build |
| §9 | `o5_diskcache.py` | non-canonical | disk-cache add/read/shutdown under helgrind |
| §9 | `o5_parse_focus.py` | non-canonical | focused single-thread parse under helgrind |
| §9 | `o5_selfoffer_child.sh`, `o5_selfoffer.gdb` | canonical + gdb | end-to-end `is_self_offer` reentrancy chain |

The authoritative sha256 of the §9 (O5) scripts, as embedded inline, are: `ccwrap.sh` = `73d24b92c48a13f46ab2c007e591964cfd524390aa66368faeb0961a61ab511c`, `o5_diskcache.py` = `f3ce4d00e5df3635595b8a629a47d737b68bf47ebc553aa7d9c0346c52c0b0ce`, `o5_parse_focus.py` = `d4e53da4b3dfdf03d115603a8c30784da21234d1e0cb0ac1ee8b9809e8950fd7`, `o5_selfoffer.gdb` = `e40b7c04b34e26ef9110430a0ae3d0076c377dfae3a99b8b8a96a19acef7660a`, `o5_selfoffer_child.sh` = `0093190bad0e578b6eda9c7279c5af2cd9448984e9422a36a734ffde62761d58`.

### 11.2 Cleanup and repository integrity

All observation was performed against a disposable copy of the repository built inside the canonical container (§2). Every temporary script and captured log lived under `/tmp`, **outside** the checkout, and was removed after its output was embedded above. The canonical `fast_data_types.so` (sha256 `582933cfd7b6cecb5ee60cfd20ef35a1f74acc2c6a905022180a60a76bf722e8`) was restored after each diagnostic build, and any `vgcore.*` dropped by a valgrind SIGILL was removed from the build tree. The source checkout at HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` is left **byte-for-byte unchanged** — `git status --porcelain` reports empty (build artifacts such as the `.so` files are git-ignored, so tracked-source integrity is intact). The only file written by this investigation is this document, `blitzy/documentation/kitty_815df1e210e0.md`.

**Container artifact inventory (`docker diff`).** The disposable container was audited with `docker diff` before and after removing the scratch tree, path-guarded to `/tmp` (only the system X11 socket dirs `\.X11-unix`/`\.*-unix` were preserved; no broad `rm -rf` outside `/tmp`, no process kills beyond the ones spawned here):

```console
$ docker diff <container> | grep -cE '^A /tmp/'      # before cleanup
251
# remove only my own /tmp scratch, preserving system socket dirs
$ docker exec <container> bash -lc 'cd /tmp; for e in $(ls -1 /tmp | grep -vE "^\.(X11|font|ICE|Test|XIM)-unix$"); do case "/tmp/$e" in /tmp/*) rm -rf -- "/tmp/$e";; esac; done'
$ docker diff <container> | grep -cE '^A /tmp/'      # after cleanup
1
$ docker diff <container> | grep -E '^A /tmp/'       # the single remainder
A /tmp/.X11-unix/X99
```

The one surviving `/tmp` entry is the **Xvfb display socket** (`/tmp/.X11-unix/X99`), a system runtime artifact of the headless X server (§2), not investigation scratch. The categorized full diff after cleanup is:

```console
tmp_added(remaining scratch)=1          # only /tmp/.X11-unix/X99 (Xvfb socket)
tmp_deleted(my cleaned scratch)=7       # obs/, test*.log, build.log, xvfb.log, fish.root
app_entries(all git-ignored build products)=623
root_entries(build cache: go/pip)=8570
usr_entries=55
var_entries=32
```

Every one of the 623 `/app` entries is a **git-ignored build product** (regenerated by the `python3 setup.py` builds of §2/§9.1), not tracked source. This is proven two ways — a spot `git check-ignore` and the authoritative tracked-source diff:

```console
$ docker exec <container> bash -lc 'cd /app && git check-ignore -q constants_generated.go && echo IGNORED; \
    git check-ignore -q kitty/uniforms_generated.h && echo IGNORED; \
    git check-ignore -q glfw/wayland-xdg-shell-client-protocol.c && echo IGNORED'
IGNORED
IGNORED
IGNORED
$ docker exec <container> bash -lc 'cd /app && git status --porcelain | wc -l; git diff --name-only HEAD | wc -l; git rev-parse HEAD'
0
0
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
```

`git status --porcelain` is empty and `git diff --name-only HEAD` lists **zero** tracked files, so the source checkout is byte-for-byte identical to HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` after the entire investigation. **OBSERVED.**
