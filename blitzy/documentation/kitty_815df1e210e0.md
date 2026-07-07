# How Kitty's diff kitten works internally (commit `815df1e210e0`)

This document explains — grounded in **both source code and observed runtime behavior** — how the
[kitty](https://github.com/kovidgoyal/kitty) terminal emulator's **"diff kitten"** (`kittens/diff/`)
works internally. It answers eight behavioral questions (Q1–Q8). Every behavioral claim follows the
same pattern:

> **mechanism → `file:line` → the exact command run → the complete, unedited observed output → the causal reason (cause → effect).**

Statements that are read from the source but *not* directly demonstrated at runtime are explicitly
labelled **(inferred)**. Everything else was produced by building kitty and running the real
`kitten diff` Go entry point in the canonical container, and the output blocks below are the actual,
unedited captures.

All `file:line` citations correspond to the pinned commit
`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.

---

## 0. Environment & Build

### 0.1 Canonical environment and pinned commit

The investigation was performed inside the rule-mandated container image
`andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
(from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`), checked out at the pinned
commit.

```console
$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1

$ git rev-parse --abbrev-ref HEAD
blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7

$ go version
go version go1.22.12 linux/amd64

$ python3 --version
Python 3.13.7
```

`go1.22.12` satisfies the `go 1.22` directive in `go.mod:3`; Python 3.13.7 satisfies the kitten
shim and the `setup.py` build driver.

### 0.2 Canonical build

The canonical build is `python3 setup.py` (aliased by `make all`). On this toolchain the global
option `--ignore-compiler-warnings` is required: gcc 15's default `-Werror=switch` makes
`glfw/wl_window.c:668` fatal because wayland-protocols 1.45 adds `XDG_TOPLEVEL_STATE_CONSTRAINED_*`
enum values the mid-2024 switch statement does not handle. This is a toolchain/dependency-version
incompatibility, not a code bug, and `--ignore-compiler-warnings` is a documented `setup.py` global
option (`setup.py:2002-2008`) — a build-invocation choice, not a source edit.

To capture a genuine, **complete** build transcript (rather than an incremental no-op), the
gitignored launcher binaries were removed and the Go build cache was cleared before rebuilding. The
transcript below is the **complete, unedited** output of the canonical build (198 lines): line 1
relinks the launcher (the C object files were already compiled and cached), and lines 3–198 are the
freshly recompiled Go packages — including `kitty/kittens/diff` (line 195 of the output), the package
under study:

```console
$ rm -f kitty/launcher/kitten kitty/launcher/kitty && go clean -cache
$ python3 setup.py --ignore-compiler-warnings
[1/1] Linking launcher ...
 done
internal/nettrace
vendor/golang.org/x/crypto/cryptobyte/asn1
github.com/seancfoley/ipaddress-go/ipaddr/addrerr
image/color
github.com/shirou/gopsutil/v3/common
log/internal
crypto/subtle
github.com/seancfoley/ipaddress-go/ipaddr/addrstr
github.com/seancfoley/ipaddress-go/ipaddr/addrstrparam
kitty
maps
unicode/utf16
container/list
crypto/internal/alias
crypto/internal/boring/sig
golang.org/x/exp/constraints
encoding
vendor/golang.org/x/crypto/internal/alias
encoding/base32
internal/singleflight
crypto/internal/randutil
vendor/golang.org/x/text/transform
hash
vendor/golang.org/x/net/dns/dnsmessage
internal/intern
math/rand/v2
net/http/internal/ascii
bufio
encoding/base64
regexp/syntax
embed
context
io/ioutil
golang.org/x/sys/unix
vendor/golang.org/x/sys/cpu
encoding/hex
log
net/url
vendor/golang.org/x/net/http2/hpack
kitty/tools/utils/shlex
runtime/cgo
github.com/ALTree/bigfloat
flag
github.com/bmatcuk/doublestar/v4
crypto/internal/bigmod
github.com/seancfoley/bintree/tree
github.com/dlclark/regexp2/syntax
crypto/rc4
crypto/internal/edwards25519/field
crypto/cipher
image/color/palette
crypto/internal/nistec/fiat
vendor/golang.org/x/crypto/internal/poly1305
encoding/asn1
hash/crc32
golang.org/x/image/riff
crypto/dsa
hash/adler32
net/netip
compress/flate
encoding/pem
crypto
github.com/rwcarlsen/goexif/tiff
encoding/json
vendor/golang.org/x/text/unicode/norm
vendor/golang.org/x/crypto/chacha20
os/exec
database/sql/driver
crypto/internal/edwards25519
crypto/internal/boring
compress/bzip2
os/signal
compress/lzw
net/http/internal
crypto/md5
golang.org/x/image/tiff/lzw
image
crypto/des
mime
github.com/klauspost/cpuid/v2
encoding/xml
mime/quotedprintable
vendor/golang.org/x/text/unicode/bidi
vendor/golang.org/x/crypto/cryptobyte
crypto/x509/pkix
regexp
crypto/rand
crypto/internal/boring/bbig
crypto/sha1
crypto/aes
crypto/sha512
crypto/sha256
crypto/hmac
vendor/golang.org/x/crypto/hkdf
vendor/golang.org/x/crypto/chacha20poly1305
compress/zlib
compress/gzip
archive/zip
kitty/tools/utils/secrets
crypto/rsa
github.com/shirou/gopsutil/v3/internal/common
crypto/ed25519
github.com/dlclark/regexp2
vendor/golang.org/x/text/secure/bidirule
crypto/internal/nistec
golang.org/x/image/bmp
image/internal/imageutil
golang.org/x/image/ccitt
image/png
golang.org/x/image/vp8l
golang.org/x/image/vp8
image/draw
image/jpeg
golang.org/x/image/tiff
vendor/golang.org/x/net/idna
github.com/rwcarlsen/goexif/exif
golang.org/x/image/webp
github.com/zeebo/xxh3
howett.net/plist
image/gif
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
kitty/tools/utils/base85
kitty/tools/tty
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
kitty/tools/cmd/show_error
kitty/tools/cmd/run_shell
kitty/tools/cmd/edit_in_kitty
kitty/tools/tui/graphics
kitty/kittens/ask
kitty/tools/cmd/update_self
kitty/kittens/hints
kitty/tools/cmd/at
kitty/tools/themes
kitty/kittens/unicode_input
kitty/kittens/transfer
kitty/kittens/themes
kitty/kittens/ssh
kitty/tools/cmd/benchmark
kitty/kittens/icat
kitty/kittens/choose_fonts
kitty/tools/cmd/pytest
kitty/kittens/diff
kitty/tools/cmd/tool
kitty/tools/cmd/completion
kitty/tools/cmd
```

The build produced the launcher binary (gitignored — verified untracked, so rebuilding leaves the
working tree clean). This is the actual, unedited verification output:

```console
$ ls -l kitty/launcher/kitten
-rwxr-xr-x 1 root root 15765764 Jul  7 00:25 kitty/launcher/kitten

$ git check-ignore kitty/launcher/kitten kitty/launcher/kitty
kitty/launcher/kitten
kitty/launcher/kitty

$ git status --porcelain      # after the full rebuild
                              # (empty — the rebuild dirtied nothing)
```

### 0.2.1 Debug build

The debug build is `python3 setup.py build --debug` (`--debug` is the documented `setup.py` global
option at `setup.py:1848-1851`, which builds the C extension modules with debugging symbols `-g3`,
`setup.py:1246`, and disables `-O3`, `setup.py:482`). On this toolchain it also requires
`--ignore-compiler-warnings` for the same gcc-15/wayland-protocols reason as the canonical build.
Because the debug flags differ from the cached release objects, this recompiles all 122 C modules in
debug mode. The following is the **complete, unedited** debug-build transcript (130 lines):

```console
$ python3 setup.py build --debug --ignore-compiler-warnings
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
kitty/tools/cmd
```

After the debug build, the canonical **release** build was re-run (`python3 setup.py
--ignore-compiler-warnings`) so that all runtime observations below come from the canonical release
binary; that restore likewise left the git working tree clean (the launcher binary is gitignored).

Throughout the investigation the launcher path `kitty/launcher/kitten` is referred to as `$KIT`.

### 0.3 Real entry point (Go), not the Python shim

The diff kitten is a dual-language component: a thin Python shim (`kittens/diff/main.py`) declares
configuration options, while the runtime behavior is implemented in Go. The **canonical** entry point
is `kitten diff <left> <right>` (equivalently `kitty +kitten diff <left> <right>`), which reaches the
Go `main` at `kittens/diff/main.go:102` (`EntryPoint` at `main.go:177`):

```console
$ $KIT diff --help
Usage: kitten diff [options] file_or_directory_left file_or_directory_right

Show a side-by-side diff of the specified files/directories. You can also use
ssh:hostname:remote-file-path to diff remote files.

Options:
  --context [=-1]
    Number of lines of context to show between changes. Negative values use the
    number set in diff.conf.

  --config
    Specify a path to the configuration file(s) to use. All configuration files
    are merged onto the builtin diff.conf, overriding the builtin values. This
    option can be specified multiple times to read multiple configuration files
    in sequence, which are merged. Use the special value NONE to not load any
    config file.

    If this option is not specified, config files are searched for in the order:
    $XDG_CONFIG_HOME/kitty/diff.conf, ~/.config/kitty/diff.conf,
    $XDG_CONFIG_DIRS/kitty/diff.conf. The first one that exists is used as the
    config file.

    If the environment variable KITTY_CONFIG_DIRECTORY is specified, that
    directory is always used and the above searching does not happen.

    If /etc/xdg/kitty/diff.conf exists, it is merged before (i.e. with lower
    priority) than any user config files. It can be used to specify system-wide
    defaults for all users. You can use either - or /dev/stdin to read the
    config from STDIN.

  --override, -o
    Override individual configuration options, can be specified multiple times.
    Syntax: name=value. For example: -o background=gray

  --help, -h
    Show help for this command

kitten diff 0.35.2 created by Kovid Goyal
```

The Python `main()` is **not** a valid observation path — it deliberately errors. This is the
**forbidden / non-canonical** path, shown here only to prove it errors (`kittens/diff/main.py:13-14`):

```console
$ python3 -c "import kittens.diff.main as m; m.main([])"
Must be run as kitten diff
```

`kittens/diff/main.py:13-14`:

```python
def main(args: List[str]) -> None:
    raise SystemExit('Must be run as kitten diff')
```

All observations below therefore come from `$KIT diff …` (the Go path).

### 0.4 How full-screen TUI output was captured

`kitten diff` renders a full-screen, side-by-side TUI on the alternate screen, so its rendered frame
must be captured from a PTY rather than read from stdout. `tmux` is not installed in the container,
so a small Python `pty` harness (`/tmp/tui_capture.py`, outside the repository) was used. It:

1. `pty.fork()`s and sets the child window size with `TIOCSWINSZ` before exec'ing `$KIT diff …`;
2. answers the terminal's Primary Device Attributes (`ESC[?62;1;6c`) and Cursor-Position-Report
   queries so the kitty TUI event loop does not block;
3. accumulates the raw byte stream, records the offset at which `q` (quit) is sent;
4. feeds the captured bytes (up to the quit offset, avoiding the alt-screen-exit blank) to a fresh
   [`pyte`](https://pypi.org/project/pyte/) VT emulator and dumps the character grid.

Truecolor SGR written by kitty uses the colon form `ESC[38:2:R:G:Bm`, which `pyte` 0.8.x cannot
parse, so those SGR runs are stripped before the grid is dumped (this only removes color, never
text). A `--raw-out <path>` flag additionally dumps the raw bytes so that graphics-protocol escapes
and pre-render/transitional states can be inspected with `strings`/`grep`. The harness was validated
on a single-file diff before use. Where a captured pyte grid occasionally garbles a transient
mid-render frame (a harness artifact, not kitten behavior), the raw byte stream is grepped instead
and that is called out inline.

### 0.5 Selecting the diff engine and options

Config options are declared in `kittens/diff/main.py`: `diff_cmd` (default `'auto'`, `main.py:41`),
`ignore_name` (default empty `''`, `main.py:56`), `num_context_lines` (default `'3'`, `main.py:37`),
`syntax_aliases` (`main.py:29`), `replace_tab_by` (`main.py:52`). The CLI accepts `--config <file>`
and repeatable `--override`/`-o key=value`; `kittens/diff/main.go:24-27` loads them via
`config.ConfigParser.LoadConfig("diff.conf", opts.Config, opts.Override)`. Both override forms were
confirmed working in the container. To force a specific engine:

```sh
$KIT diff -o diff_cmd=git     L R    # external git diff --no-index
$KIT diff -o diff_cmd=diff    L R    # external diff
$KIT diff -o diff_cmd=builtin L R    # in-process anchored diff (diff.go)
```

An isolated config directory was used via `KITTY_CONFIG_DIRECTORY=/tmp/kdiff/kittyconf` so the host
config never interferes.

---

## 1. Q1 — Directory pairing ("what belongs together")

### 1.1 Direct answer

When two directories are compared, the diff kitten decides which file on the left corresponds to which
file on the right **purely by identical relative path** — not by content, not by position in the
listing. A file that exists at the same relative path on both sides is paired (and shown as a change
or omitted if identical); a path present on only one side is a removal or an addition.

### 1.2 Mechanism & code anchors

The work is done by `collect_files` (`kittens/diff/collect.go:296`). It walks both trees with `walk`
(calls at `collect.go:299` and `collect.go:303`), accumulating the relative paths into two sets
`left_names, right_names` (`collect.go:297`). Pairing is the set intersection:

- `common_names := left_names.Intersect(right_names)` (`collect.go:306`) — the paired files;
- `removed := left_names.Subtract(common_names)` (`collect.go:332`) — left-only paths;
- `added := right_names.Subtract(common_names)` (`collect.go:333`) — right-only paths.

For each common name the file data is compared: `if ld != rd` (`collect.go:317`) →
`add_change` (`collect.go:319`, definition at `collect.go:167`); if the bytes are equal the file is
skipped (no entry) unless the file **mode** differs (see §1.6). The relative path (not the absolute
path) is the map key, which is why a file nested in a subdirectory pairs with the same relative path
on the other side. This is corroborated by the user-facing docs: `docs/kittens/diff.rst:20` — "Does
recursive directory diffing".

### 1.3 Command(s) run

The left tree has `{only_left.txt, same.txt, sub/a.txt}`; the right tree has
`{only_right.txt, same.txt, sub/a.txt}`. `same.txt` is byte-identical on both sides; `sub/a.txt`
differs in one line; `only_left.txt`/`only_right.txt` exist on one side only.

```sh
$KIT diff /tmp/kdiff/q1left /tmp/kdiff/q1right
```

### 1.4 Observed output

```text
   only_left.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1  only on the left side                                                 This file was removed

   only_right.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   This file was added                                                1  only on the right side

   sub/a.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   @@ -1,3 +1,3 @@
1  alpha                                                              1  alpha
2  beta                                                               2  BETA-changed
3  gamma                                                              3  gamma
```

Three entries are shown, sorted by path: `only_left.txt` ("This file was removed"), `only_right.txt`
("This file was added"), and `sub/a.txt` (a change, `beta` → `BETA-changed`). `same.txt` — identical
on both sides — is **omitted entirely**.

The directory walk and pairing machinery is also exercised by the package's own unit test:

```console
$ go test ./kittens/diff/ -run TestDiffCollectWalk -v
=== RUN   TestDiffCollectWalk
--- PASS: TestDiffCollectWalk (0.00s)
PASS
ok  	kitty/kittens/diff	0.021s
```

(`kittens/diff/collect_test.go:19-54` walks a tree and asserts the resulting relative-name set.)

### 1.5 Causal reason

Pairing is set intersection over **relative paths** (`Intersect` at `collect.go:306`). `sub/a.txt`
pairs across the nested subdirectory because its relative path is identical on both sides; `same.txt`
produces no entry because its bytes are equal (`ld != rd` is false at `collect.go:317`); the two
single-sided files fall into `removed`/`added` because they are absent from `common_names`
(`Subtract` at `collect.go:332-333`). Nothing about content similarity or list position affects
pairing.

### 1.6 Sibling variants / edge cases

**Argument validation** — the kitten requires exactly two paths (`main.go:108-110`,
`if len(args) != 2`); 0, 1, or 3 arguments all produce the same error and exit code 1:

```console
$ $KIT diff            ; echo "[exit=$?]"
Error: You must specify exactly two files/directories to compare
[exit=1]
$ $KIT diff onlyone    ; echo "[exit=$?]"
Error: You must specify exactly two files/directories to compare
[exit=1]
$ $KIT diff a b c      ; echo "[exit=$?]"
Error: You must specify exactly two files/directories to compare
[exit=1]
```

**Identical inputs** — diffing two byte-identical files shows the "identical" banner (this is the
`Diff() == nil` path described in §8; captured here from `-o diff_cmd=builtin` on an identical pair):

```text
   /tmp/kdiff/q8id_left/f.txt                                  /tmp/kdiff/q8id_right/f.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   The files are identical
```

**Mode-only change** — if the bytes are identical but the file mode differs, the else-branch at
`collect.go:321-329` runs `os.Stat` on both sides and, when `lstat.Mode() != rstat.Mode()`
(`collect.go:323`), calls `add_change` (`collect.go:326`). The rendered banner comes from
`render.go:577`. Fixture: identical content (`md5 = 3fe37309da949d82291eb060c6dc69e3` both sides),
mode `0644` vs `0755`:

```console
$ md5sum /tmp/kdiff/mo_left/script.sh /tmp/kdiff/mo_right/script.sh
3fe37309da949d82291eb060c6dc69e3  /tmp/kdiff/mo_left/script.sh
3fe37309da949d82291eb060c6dc69e3  /tmp/kdiff/mo_right/script.sh
$ $KIT diff /tmp/kdiff/mo_left /tmp/kdiff/mo_right
```

```text
   script.sh
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Mode changed: -rw-r--r-- to -rwxr-xr-x
```

**Ignore-glob filtering** — the walk drops entries whose basename matches any `ignore_name` glob via
`allowed` (`collect.go:230`), which tests each pattern against the file's basename with Go's
standard-library `filepath.Match` (`collect.go:233`) — the diff kitten does **not** use the
`doublestar` package for this filter (`collect.go:5-14` imports only `path/filepath`; a `grep` for
`doublestar` in `kittens/diff/` returns nothing). The default `ignore_name` is **empty** (`main.py:56`),
so by default nothing is ignored (the `.git`/`*~`/`*.pyc` values in the `main.py:63-65` help text are
documentation examples, not active defaults). With a fixture containing
`{#draft#, keep.txt, mod.pyc, notes.txt~}` on each side, the default run shows all four; supplying the
globs filters them:

```console
$ $KIT diff /tmp/kdiff/ig_left /tmp/kdiff/ig_right          # default: all four shown
   #draft#
   keep.txt
   mod.pyc
   notes.txt~

$ $KIT diff -o 'ignore_name=*~' -o 'ignore_name=#*#' -o 'ignore_name=*.pyc' \
      /tmp/kdiff/ig_left /tmp/kdiff/ig_right                # only keep.txt survives
   keep.txt
```

The same filtering is asserted by `TestDiffCollectWalk` (`collect_test.go:19-54`), which walks
`{a/b/c, b, d, e, #d#, e~, f/g, h space}` with globs `["*~", "#*#", "b"]` and expects
`{d, e, f/g, h space}`.

---


## 2. Q2 — Rename recognition (rename vs deletion + addition)

### 2.1 Direct answer

A change is classified as a **rename** only when a left-only (removed) file and a right-only (added)
file have **the same MD5 hash AND byte-identical content**. If a single byte differs, the MD5
pre-filter (and the exact-bytes check) fails and the pair degrades to a separate **removal + addition**
— there is no rename.

### 2.2 Mechanism & code anchors

Inside `collect_files`, after computing `removed`/`added` (§1.2), the kitten builds hash maps for the
added files (`ahash`, `collect.go:335-340`) and the removed files (`rhash`, `collect.go:341-346`) using
`hash_for_path` (`collect.go:106`), which computes `md5.Sum(...)` (`collect.go:112`; `import
crypto/md5` at `collect.go:6`). It then, for each removed hash, scans the added hashes
(`for name, rh := range rhash`, `collect.go:347`):

- `if ah == rh` (`collect.go:350`) — the MD5 pre-filter matches; then
- it reads both files' data (`ld, rd`, `collect.go:351-352`) and requires `if ld == rd`
  (`collect.go:353`) — **exact byte equality** — before calling `add_rename` (`collect.go:354`,
  definition `collect.go:175`) and discarding the added entry (`Discard`, `collect.go:355`);
- otherwise the removed file becomes `add_removal` (`collect.go:362`, definition `collect.go:192`), and
  any added files with no rename match become `add_add` (`collect.go:365-366`, definition
  `collect.go:181`).

The rename is surfaced in the UI by the **two-name title**: `title_lines` (`render.go:208`) reads
`left_name, right_name` from `path_name_map` (`render.go:209`); when the right name differs from the
left name (only true for a rename) it renders **both** names.

### 2.3 Command(s) run — both transitional states

```sh
# (a) byte-identical -> RENAME
$KIT diff /tmp/kdiff/q2a_left /tmp/kdiff/q2a_right
# (b) one byte changed -> REMOVE + ADD
$KIT diff /tmp/kdiff/q2b_left /tmp/kdiff/q2b_right
```

### 2.4 Observed output

**(a) byte-identical → rename.** `old_name.txt` (left) and `new_name.txt` (right) have the same MD5:

```console
$ md5sum /tmp/kdiff/q2a_left/old_name.txt /tmp/kdiff/q2a_right/new_name.txt
315108a035552eb328252565386e663a  /tmp/kdiff/q2a_left/old_name.txt
315108a035552eb328252565386e663a  /tmp/kdiff/q2a_right/new_name.txt
```

```text
   old_name.txt                                                new_name.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

A **single entry** appears, titled with **both** names side by side (`old_name.txt` on the left,
`new_name.txt` on the right) — the rename.

**(b) one byte changed → removal + addition.** Changing the last word (`dog` → `doX`) changes the MD5:

```console
$ md5sum /tmp/kdiff/q2b_left/old_name.txt /tmp/kdiff/q2b_right/new_name.txt
315108a035552eb328252565386e663a  /tmp/kdiff/q2b_left/old_name.txt
6c2d1fe9341479db3eedff36ddaefb6e  /tmp/kdiff/q2b_right/new_name.txt
```

```text
   new_name.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   This file was added                                      1  the quick brown fox
                                                            2  jumps over
                                                            3  the lazy doX

   old_name.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1  the quick brown fox                                         This file was removed
2  jumps over
3  the lazy dog
```

Two **separate** entries: `new_name.txt` ("This file was added") and `old_name.txt`
("This file was removed"). No rename.

### 2.5 Causal reason

The MD5 hash is a cheap pre-filter (`ah == rh`, `collect.go:350`); the decisive condition is the
exact-bytes comparison (`ld == rd`, `collect.go:353`). In (a) both are equal → `add_rename`. In (b)
the changed byte makes both the hash and the byte comparison fail → the removed and added files are
emitted independently via `add_removal` + `add_add`. Thus a rename is, by construction, a
content-preserving move; any content edit converts it to remove + add.

### 2.6 Sibling / honest finding — the rename **body** message is never rendered

`rename_lines` (`render.go:684`) *intends* to print `"The file %s was renamed to %s"`
(`render.go:688`), but that message is **not** rendered. Grepping the raw byte stream of the rename
capture for "renamed" returns nothing:

```console
$ strings /tmp/kdiff/q2a_raw.bin | grep -i renamed
$        # (no matches)
```

**Cause (grounded in source):** `rename_lines` sets `is_full_width: true` (`render.go:687`) and writes
the message to `sl.right.marked_up_text` (`render.go:690`). But `render_screen_line`
(`render.go:70`), for a full-width line, computes `available_cols = columns - margin_size`
(`render.go:76-78`), renders only `sl.left.marked_up_text` (`render.go:80`), and then
`if self.is_full_width { return }` (`render.go:99-101`) **returns before the right half is drawn**. The
message lives in the right half, so it is skipped. Consequently the rename is observable **only** via
the two-name title (§2.4a), not via any body text. This is reported as an observed latent bug; the
answer does not claim the message appears.

---


## 3. Q3 — Caching pipeline (raw bytes → highlighted lines)

### 3.1 Direct answer

Each file's raw bytes are read from disk **exactly once**, by `data_for_path`, and cached. Every
later stage — hashing, text/binary detection, line splitting, and syntax highlighting — reuses those
cached bytes rather than re-reading the file. Seven `LRUCache` instances (capacity **4096** each) hold
the intermediate results, keyed by absolute path, so repeated lookups are O(1) and memory is bounded
by LRU eviction.

### 3.2 Mechanism & code anchors

The seven caches are declared at `collect.go:20-24` and created in `init_caches()`
(`collect.go:26`) with `const sz = 4096` (`collect.go:29`):

| Cache | Value type | Purpose |
|-------|------------|---------|
| `size_cache` | `int64` | file size (via `os.Stat`) |
| `mimetypes_cache` | `string` | MIME type (by extension) |
| `data_cache` | `string` | **raw file bytes (the single content read)** |
| `is_text_cache` | `bool` | text-vs-binary decision |
| `lines_cache` | `[]string` | raw content split into lines |
| `highlighted_lines_cache` | `[]string` | Chroma-highlighted lines |
| `hash_cache` | `string` | MD5 hash |

The pipeline is layered: `data_for_path` (`collect.go:65-70`) → `sanitize` (`collect.go:128-130`) →
`lines_for_path` (`collect.go:138-146`) → `highlighted_lines_for_path` (`collect.go:148-157`). The
single content read is `data_cache.GetOrCreate(path, …os.ReadFile…)` (`collect.go:66-69`).
`hash_for_path` (`collect.go:106`) and `is_path_text` (`collect.go:86`) both call `data_for_path`, so
they consume the cached bytes rather than reopening the file. `LRUCache` itself is defined at
**`tools/utils/cache.go:13`** (`Get` at `:25`, `Set` at `:32`, `GetOrCreate` at `:39`,
`MustGetOrCreate` at `:60`) — note there is **no** `kittens/diff/cache.go`.

### 3.3 Command(s) run

`strace` (system tool, `apt`-installed; the repository is untouched) traced the `openat` syscalls the
kitten makes while diffing a 3-file-per-side directory (`q3_left`/`q3_right`, files `f1.txt`–`f3.txt`,
each changed on one line). Because `kitten diff` is a full-screen TUI and `tmux` is absent, the run is
driven through the PTY harness (§0.4). The kitten's own content read uses Go's `os.ReadFile`, which
always opens with `O_CLOEXEC`; counting `O_CLOEXEC` opens per path isolates the cache's single content
read from any external-subprocess opens. The exact trace command (builtin shown; for auto, drop the
`-o diff_cmd=builtin`) is:

```sh
$ strace -f -e trace=openat -o out.txt \
    python3 /tmp/tui_capture.py --cols 120 --rows 24 --quit-after 1.5 -- \
    $KIT diff -o diff_cmd=builtin /tmp/kdiff/q3_left /tmp/kdiff/q3_right
```

The exact per-path counting script (`/tmp/count_openat.sh`, outside the repository) and its use:

```sh
$ cat /tmp/count_openat.sh
#!/bin/sh
# Usage: count_openat.sh <strace_output_file>
# For each distinct Q3 content path, print total openat calls and O_CLOEXEC opens.
f="$1"
for p in q3_left/f1.txt q3_left/f2.txt q3_left/f3.txt q3_right/f1.txt q3_right/f2.txt q3_right/f3.txt; do
  total=$(grep -c "openat(.*\"/tmp/kdiff/$p\"" "$f")
  cloexec=$(grep "openat(.*\"/tmp/kdiff/$p\"" "$f" | grep -c 'O_CLOEXEC')
  printf '  %-16s total_openat=%s  O_CLOEXEC=%s\n' "$p:" "$total" "$cloexec"
done
$ /tmp/count_openat.sh out.txt
```

### 3.4 Observed output (stable across ≥2 runs)

**Builtin engine** (`-o diff_cmd=builtin`, no external subprocess) — each distinct path is opened
**exactly once**, and that open is the `O_CLOEXEC` content read:

```text
--- RUN1 ---                                --- RUN2 ---
  q3_left/f1.txt:  total_openat=1  O_CLOEXEC=1    q3_left/f1.txt:  total_openat=1  O_CLOEXEC=1
  q3_left/f2.txt:  total_openat=1  O_CLOEXEC=1    q3_left/f2.txt:  total_openat=1  O_CLOEXEC=1
  q3_left/f3.txt:  total_openat=1  O_CLOEXEC=1    q3_left/f3.txt:  total_openat=1  O_CLOEXEC=1
  q3_right/f1.txt: total_openat=1  O_CLOEXEC=1    q3_right/f1.txt: total_openat=1  O_CLOEXEC=1
  q3_right/f2.txt: total_openat=1  O_CLOEXEC=1    q3_right/f2.txt: total_openat=1  O_CLOEXEC=1
  q3_right/f3.txt: total_openat=1  O_CLOEXEC=1    q3_right/f3.txt: total_openat=1  O_CLOEXEC=1
```

**Auto engine** (default → git, see §8) — each path shows `total_openat=3` but still exactly **one**
`O_CLOEXEC` content read; the two extra non-`O_CLOEXEC` opens belong to the external `git` subprocess
(a different PID) reading the file itself:

Complete per-path output for both runs (`/tmp/count_openat.sh` on each run's `out.txt`):

```text
=== AUTO RUN1 (default engine → git) ===
  q3_left/f1.txt:  total_openat=3  O_CLOEXEC=1
  q3_left/f2.txt:  total_openat=3  O_CLOEXEC=1
  q3_left/f3.txt:  total_openat=3  O_CLOEXEC=1
  q3_right/f1.txt: total_openat=3  O_CLOEXEC=1
  q3_right/f2.txt: total_openat=3  O_CLOEXEC=1
  q3_right/f3.txt: total_openat=3  O_CLOEXEC=1
=== AUTO RUN2 ===
  q3_left/f1.txt:  total_openat=3  O_CLOEXEC=1
  q3_left/f2.txt:  total_openat=3  O_CLOEXEC=1
  q3_left/f3.txt:  total_openat=3  O_CLOEXEC=1
  q3_right/f1.txt: total_openat=3  O_CLOEXEC=1
  q3_right/f2.txt: total_openat=3  O_CLOEXEC=1
  q3_right/f3.txt: total_openat=3  O_CLOEXEC=1
```

All six paths are identical within a run and stable across both runs (`total_openat=3`,
`O_CLOEXEC=1`). The actual syscall lines for one path make the split explicit — one `O_CLOEXEC` open
from the kitten (pid 169139) and two plain opens from the external `git` subprocess (pid 169146). This
is the verbatim `grep 'openat(.*"/tmp/kdiff/q3_left/f1.txt"' out.txt` from AUTO RUN1 (strace's
`<unfinished ...>` markers, emitted when `-f` interleaves the two processes, are preserved exactly as
printed):

```text
169139 openat(AT_FDCWD, "/tmp/kdiff/q3_left/f1.txt", O_RDONLY|O_CLOEXEC) = 11
169146 openat(AT_FDCWD, "/tmp/kdiff/q3_left/f1.txt", O_RDONLY) = 3
169146 openat(AT_FDCWD, "/tmp/kdiff/q3_left/f1.txt", O_RDONLY <unfinished ...>
```

### 3.5 Causal reason

The kitten reads each file's content exactly once because `data_for_path` funnels every read through
`data_cache.GetOrCreate` (`collect.go:66-69`): the first miss runs `os.ReadFile` (one `O_CLOEXEC`
open); every subsequent consumer — `hash_for_path`, `is_path_text`, `lines_for_path`,
`highlighted_lines_for_path` — retrieves the cached string with no further open. That is why the
`O_CLOEXEC` count is exactly 1 per path in **both** engines, and why the builtin engine's total is
also 1 (it does everything in-process). The extra opens in auto mode are not the cache pipeline at
all; they are the external git process, which reads the files independently (this also feeds the
cache-interplay discussion in §8). The cache key is the absolute path and capacity 4096 bounds memory
via LRU eviction (`NewLRUCache(sz)`).

---


## 4. Q4 — Concurrency across many files

### 4.1 Direct answer

When many files must be processed, the diff kitten builds a job list — **one job per changed
text-file pair** (`ui.go:147`) — and runs it through a worker pool whose size is
`min(runtime.NumCPU(), number-of-jobs)` (`utils.go:35,37-38`). Diffing and syntax-highlighting are
dispatched **at the same moment** (`generate_diff()` then `highlight_all()`, `ui.go:249-250`), each
with its own `NumCPU`-sized pool. Constraining the CPU set to 1, 2, or 4 CPUs makes the peak number of
concurrent `git diff` subprocesses **exactly 1, 2, or 4** — the direct, deterministic signature of the
`NumCPU`-sized pool.

At scale on this 128-CPU machine the picture is more nuanced, and is reported honestly below. A single
CPU processes all N files to completion (400 files → 400 `git diff` subprocesses, peak 1). But an
**unconstrained** run does **not** complete: the concurrently-running highlight pool trips a data race
(the very same `LRUCache.Set` race examined in Q5) and the process aborts with
`fatal error: concurrent map writes` after a **variable** number of files. That is precisely why the
amount of work completed is **run-to-run inconsistent** at high concurrency — same input, different
truncation point every time. The worker-pool magnitude itself (peak concurrency = `NumCPU`) is proven
cleanly at 1/2/4 CPUs where the race does not fire.

### 4.2 Mechanism & code anchors

The worker-pool primitive is `Context.Parallel(start, stop, fn)` at
**`tools/utils/images/utils.go:27-56`**:

- `count := stop - start` (`utils.go:28`);
- `procs := self.NumberOfThreads()` (`utils.go:33`); when unset this is 0, so
  `if procs <= 0 { procs = runtime.NumCPU() }` (`utils.go:35`) — the pool size;
- `if procs > count { procs = count }` (`utils.go:37-38`) — never more workers than jobs;
- all indices are pushed onto a buffered channel `make(chan int, count)` (`utils.go:41`), then closed
  (`utils.go:45`);
- `procs` goroutines each drain the channel and call `fn(c)` (`utils.go:48-54`), joined by
  `wg.Wait()` (`utils.go:55`).

The diff jobs are driven by `diff(jobs, context_count)` (`patch.go:352`), which creates a zero-value
`images.Context{}` (so `NumberOfThreads() == 0` → the `NumCPU` fallback fires), calls
`ctx.Parallel(0, len(jobs), …)` (`patch.go:361`), and each worker runs `do_diff(job.file1,
job.file2, context_count)` (`patch.go:365`). One `do_diff` job corresponds to one file pair. The same
primitive drives highlighting (Q5).

Two details of the dispatch matter for what follows:

- **Only text pairs become diff jobs.** `generate_diff()` appends a `diff_job` only
  `if is_path_text(path) && is_path_text(changed_path)` (`ui.go:147`); binary pairs produce **no**
  `git` subprocess (they are rendered directly — Q6). So *N text pairs → exactly N `git diff`
  subprocesses*.
- **Diffing and highlighting run concurrently.** When the collection arrives, the handler fires
  `self.generate_diff()` **and** `self.highlight_all()` back to back (`ui.go:249-250`), each launching
  its own `NumCPU`-sized `Context.Parallel` pool over the *same* set of text files. On a multi-core
  machine both pools are live at once — which is why the highlight pool's data race (Q5) can abort the
  process mid-diff.

### 4.3 Command(s) run

**(a) Pool-size basis** (a **non-canonical stand-in** — the canonical fact is the code at
`utils.go:35`; this standalone probe merely reports what `runtime.NumCPU()` returns in this
container, and how `taskset` lowers it):

```console
$ cat /tmp/ncpu.go
package main
import ("fmt";"runtime")
func main(){ fmt.Println("runtime.NumCPU()=", runtime.NumCPU()) }
$ go run /tmp/ncpu.go
runtime.NumCPU()= 128
$ nproc --all
128
$ taskset -c 0   go run /tmp/ncpu.go
runtime.NumCPU()= 1
$ taskset -c 0-1 go run /tmp/ncpu.go
runtime.NumCPU()= 2
$ taskset -c 0-3 go run /tmp/ncpu.go
runtime.NumCPU()= 4
```

**(b) Effective parallelism**, observed **canonically** through the real `kitten diff` entry point by
tracing the `git` subprocesses the kitten spawns (`execve`/`exit_group` with microsecond timestamps)
and computing the peak number alive at the same instant. The kitten is a full-screen TUI, so it runs
under the PTY harness from §0.4; `taskset` pins the CPU set (which is what `runtime.NumCPU()` reads),
and `-o diff_cmd=git` selects the engine that `auto` resolves to (§0.5).

Fixture (80 identical-structure text pairs, line 2 changed — every pair is a highlightable text file,
so each becomes one `git diff` job **and** one highlight job):

```sh
mkdir -p /tmp/kdiff/q4_80_left /tmp/kdiff/q4_80_right
for i in $(seq 1 80); do
  printf 'file %s line 1\nfile %s line 2 original\nfile %s line 3\n' "$i" "$i" "$i" > /tmp/kdiff/q4_80_left/f$i.txt
  printf 'file %s line 1\nfile %s line 2 CHANGED\nfile %s line 3\n'  "$i" "$i" "$i" > /tmp/kdiff/q4_80_right/f$i.txt
done
```

Controlled sweep — the exact command at each CPU count (N=80), run twice each:

```sh
KIT=kitty/launcher/kitten
L=/tmp/kdiff/q4_80_left ; R=/tmp/kdiff/q4_80_right

taskset -c 0   strace -f -ttt -e trace=execve,exit_group -o /tmp/sweep1.strace \
  python3 /tmp/tui_capture.py --quit-after 5 -- "$KIT" diff -o diff_cmd=git "$L" "$R"
python3 /tmp/parse_concurrency.py /tmp/sweep1.strace          # NumCPU=1

taskset -c 0-1 strace -f -ttt -e trace=execve,exit_group -o /tmp/sweep2.strace \
  python3 /tmp/tui_capture.py --quit-after 5 -- "$KIT" diff -o diff_cmd=git "$L" "$R"
python3 /tmp/parse_concurrency.py /tmp/sweep2.strace          # NumCPU=2

taskset -c 0-3 strace -f -ttt -e trace=execve,exit_group -o /tmp/sweep4.strace \
  python3 /tmp/tui_capture.py --quit-after 5 -- "$KIT" diff -o diff_cmd=git "$L" "$R"
python3 /tmp/parse_concurrency.py /tmp/sweep4.strace          # NumCPU=4
```

The concurrency parser (`/tmp/parse_concurrency.py`), shown in full — it counts one `git diff` per
`execve` of `git … --no-index`, then sweeps start/end events to find the peak overlap:

```python
#!/usr/bin/env python3
import sys, re
fn = sys.argv[1]
start = {}   # pid -> execve timestamp of a git-diff
end = {}     # pid -> exit timestamp
line_re = re.compile(r'^(\d+)\s+(\d+\.\d+)\s+(.*)$')
for line in open(fn, errors='replace'):
    m = line_re.match(line)
    if not m:
        continue
    pid, ts, rest = int(m.group(1)), float(m.group(2)), m.group(3)
    if rest.startswith('execve(') and '"git"' in rest and '"--no-index"' in rest:
        start[pid] = ts
    elif rest.startswith('exit_group(') or rest.startswith('+++ exited'):
        end[pid] = ts          # last exit wins (a pid is reused only after it exits)
events = []
for pid, t0 in start.items():
    t1 = end.get(pid, t0)
    events.append((t0, +1)); events.append((t1, -1))
events.sort(key=lambda e: (e[0], e[1]))   # end before start at equal ts
cur = mx = 0
for _, d in events:
    cur += d
    if cur > mx: mx = cur
total = len(start)
span = (max(end.get(p, start[p]) for p in start) - min(start.values())) if start else 0.0
print(f"git-diff subprocesses={total}  MAX_CONCURRENT={mx}  span={span:.2f}s")
```

Scale run (N=400) and the crash capture use the same harness; the full-machine variant simply omits
`taskset`, and the crash is captured by redirecting the kitten's stderr with `/tmp/tui_err.py`:

```sh
L=/tmp/kdiff/q4_400_left ; R=/tmp/kdiff/q4_400_right
# 1-CPU baseline (completes all 400)
taskset -c 0 strace -f -ttt -e trace=execve,exit_group -o /tmp/s1.strace \
  python3 /tmp/tui_capture.py --quit-after 12 -- "$KIT" diff -o diff_cmd=git "$L" "$R"
python3 /tmp/parse_concurrency.py /tmp/s1.strace
# Full machine (unconstrained NumCPU=128), SAME input repeated
strace -f -ttt -e trace=execve,exit_group -o /tmp/sf.strace \
  python3 /tmp/tui_capture.py --quit-after 6 -- "$KIT" diff -o diff_cmd=git "$L" "$R"
python3 /tmp/parse_concurrency.py /tmp/sf.strace
# Capture the kitten's own stderr on the full machine
python3 /tmp/tui_err.py --err /tmp/crash.err -- "$KIT" diff -o diff_cmd=git "$L" "$R"
head -15 /tmp/crash.err
```

### 4.4 Observed output — controlled proof (stable across ≥2 runs)

When the run completes, the total number of `git diff` subprocesses is **exactly N (=80)** and the
peak concurrency **equals the CPU count exactly**:

```text
taskset -c 0     (NumCPU=1) RUN1: git-diff subprocesses=80  MAX_CONCURRENT=1  span=0.37s
taskset -c 0     (NumCPU=1) RUN2: git-diff subprocesses=80  MAX_CONCURRENT=1  span=0.37s
taskset -c 0-1   (NumCPU=2) RUN1: git-diff subprocesses=80  MAX_CONCURRENT=2  span=0.60s
taskset -c 0-1   (NumCPU=2) RUN2: git-diff subprocesses=80  MAX_CONCURRENT=2  span=0.49s
taskset -c 0-3   (NumCPU=4) RUN1: git-diff subprocesses=80  MAX_CONCURRENT=4  span=0.31s
taskset -c 0-3   (NumCPU=4) RUN2: git-diff subprocesses=80  MAX_CONCURRENT=4  span=0.33s
```

`MAX_CONCURRENT` tracks the CPU count 1 → 2 → 4 with no exceptions, and `total == 80 == N` confirms
one `git diff` per text pair. At `NumCPU=1` and `NumCPU=2` this is fully stable (6/6 and 2/2 runs). At
`NumCPU=4` it is stable in the large majority of runs (7 of 8 observed) but **borderline** — in one run
of eight the co-scheduled highlight pool tripped the Q5 race and truncated the diff early; that failure
mode is the subject of §4.5. The clean measurement above is what the pool produces whenever it is
allowed to finish.

### 4.5 Observed output — scale, and the honest run-to-run inconsistency

At `NumCPU=1` the pool is structurally safe (a single highlight worker → no concurrent
`LRUCache.Set`), so a 400-pair diff **always completes**, with peak concurrency 1:

```text
1-CPU RUN1: git-diff subprocesses=400  MAX_CONCURRENT=1  span=1.87s
1-CPU RUN2: git-diff subprocesses=400  MAX_CONCURRENT=1  span=2.06s
```

On the **full machine** (unconstrained, `NumCPU=128`), the *same* 400-pair input does **not** complete.
The co-running highlight pool (dispatched alongside the diff pool at `ui.go:249-250`) hits the
`LRUCache.Set` data race and the whole process aborts with `fatal error: concurrent map writes` after
a **variable** number of files. Running the identical input eight times gives a wide distribution of
how far it got before dying (this is reported exactly as observed — not smoothed):

```text
RUN1: git-diff subprocesses=332  MAX_CONCURRENT=64  span=1.41s
RUN2: git-diff subprocesses=43   MAX_CONCURRENT=15  span=0.35s
RUN3: git-diff subprocesses=16   MAX_CONCURRENT=9   span=0.10s
RUN4: git-diff subprocesses=86   MAX_CONCURRENT=47  span=0.43s
RUN5: git-diff subprocesses=4    MAX_CONCURRENT=3   span=0.11s
RUN6: git-diff subprocesses=8    MAX_CONCURRENT=6   span=0.04s
RUN7: git-diff subprocesses=191  MAX_CONCURRENT=54  span=0.80s
RUN8: git-diff subprocesses=39   MAX_CONCURRENT=24  span=0.27s
```

The diff pool genuinely reaches **dozens concurrent** before the crash (peak `MAX_CONCURRENT=64` in
RUN1) — so "many files run at once" is real — but the number that *complete* is dominated by *when* the
highlight race fires. Capturing the kitten's own stderr on the full machine confirms the cause; the
first stderr line is `fatal error: concurrent map writes` on **3 of 3** runs, and the stack is
unambiguous (this is the exact `head -15 /tmp/crash.err`):

```text
fatal error: concurrent map writes

goroutine 328 [running]:
kitty/tools/utils.(*LRUCache[...]).Set(0x39, {0xc0005025e0?, 0x0?}, {0xc001055408?, 0x3, 0x200})
	/tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a/tools/utils/cache.go:34 +0x85
kitty/kittens/diff.highlight_all.func1(0xc000742000)
	/tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a/kittens/diff/highlight.go:224 +0xae
kitty/tools/utils/images.(*Context).Parallel.func1()
	/tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a/tools/utils/images/utils.go:52 +0x54
created by kitty/tools/utils/images.(*Context).Parallel in goroutine 139
	/tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a/tools/utils/images/utils.go:50 +0xe5
```

The crash is a **timing-dependent race, not a hard threshold**. Repeating the N=400 run at fixed CPU
counts shows it firing probabilistically — never at 1 CPU, and intermittently once ≥2 workers can
`Set` at the same time:

```text
taskset -c 0     (NumCPU=1)  : crashed=no   crashed=no    (never — single highlight worker)
taskset -c 0-3   (NumCPU=4)  : crashed=YES  crashed=no
taskset -c 0-7   (NumCPU=8)  : crashed=no   crashed=no
taskset -c 0-15  (NumCPU=16) : crashed=no   crashed=YES
```

This is the canonical, real-entry-point manifestation of the latent `LRUCache.Set` race that Q5
examines with the race detector; see §5 for the read/write-lock root cause (`cache.go:33-35`).

### 4.6 Causal reason

Each pool goroutine runs one `do_diff` at a time, and with `diff_cmd=git` each `do_diff` spawns one
`git` subprocess; therefore the peak number of concurrent `git` processes equals the number of
goroutines, which is `procs = min(runtime.NumCPU(), jobs)` (`utils.go:35,37-38`). Pinning to *k* CPUs
makes `runtime.NumCPU()` return *k*, so the peak is exactly *k* — precisely what the 1/2/4 sweep shows,
and `total == N` confirms one `git` per text pair. The reason the **unconstrained** run does not simply
show "peak≈128, total=400" is that the diff pool does not run alone: `highlight_all()` is dispatched in
the same breath (`ui.go:249-250`) and its `NumCPU` workers concurrently mutate a shared map through
`LRUCache.Set`, which is not write-safe (Q5). On a multi-core machine that race is eventually taken and
the Go runtime aborts the entire process, cutting the diff pool off at a random point — hence the
variable completed-count distribution above. At one CPU the highlight pool has a single worker, the
race cannot occur, and the run completes deterministically. So the plain answer to "what happens when
many files are processed at once" is: *a `NumCPU`-sized worker pool runs up to `NumCPU` diffs
concurrently (proven at 1/2/4), but on this multi-core machine the concurrently-running highlighter's
data race makes a large unconstrained run abort partway with `concurrent map writes` — the honest,
observed reason the completed work is inconsistent run-to-run.*

---


## 5. Q5 — Parallel highlighting without "stepping on itself"

### 5.1 Direct answer

Syntax highlighting runs in parallel — **one path per worker** — and each worker writes its result
under a **distinct per-path cache key** (`highlight.go:224`). Because no two workers write the *same*
key, there is no *logical* collision: whenever a run completes, the highlighted output is
byte-for-byte identical across runs (§5.4). In that sense highlighting does not "step on itself."

However — and this is the honest, observed nuance — that safety is **not lock-enforced**.
`LRUCache.Set` mutates its Go map while holding only a **read** lock (`cache.go:33-35`), and Go maps
are unsafe for concurrent writes **even to different keys**. On this multi-core (128-CPU) machine the
parallel highlight pool therefore trips a genuine data race, and the kitten **aborts with**
`fatal error: concurrent map writes`. This is not rare: it is the very crash that truncates the large
Q4 diff runs (§4.5). So the precise answer is: *highlighting does not corrupt output by writing the
same key twice, but the shared-map write is not concurrency-safe, and on a multi-core machine the race
is taken often enough to crash the process rather than to produce wrong output.* (Per the read-only
scope this is **reported, not fixed**.)

### 5.2 Mechanism & code anchors

`highlight_all(paths)` (`highlight.go:217`) creates a zero-value `images.Context{}` and dispatches one
path per worker via `ctx.Parallel(0, len(paths), …)` (`highlight.go:219`). Each worker highlights with
`highlight_file` (`highlight.go:161`, which uses the cached `data_for_path` bytes plus Chroma lexer
matching) and, on success, writes `highlighted_lines_cache.Set(path, text_to_lines(raw))`
(`highlight.go:224`) — a **distinct key per path**. Corroborated by `docs/kittens/diff.rst:15-16`
("asynchronously, for maximum speed").

The nuance to verify (not assume): `LRUCache.Set` (`tools/utils/cache.go:32`) takes `RLock`
(`:33`), assigns `self.data[key] = val` (`:34`), then `RUnlock` (`:35`) — i.e. it **mutates the map
under a read lock**. `GetOrCreate` (`:39`) reads under `RLock`, runs the create function **outside**
any lock (no single-flight guard), then writes under `Lock` (`:48`). A read lock permits multiple
concurrent holders, so two `Set` calls can assign into the same map simultaneously — exactly the
unsafe pattern Go's runtime and race detector flag below.

### 5.3 Command(s) run and observed output — race detector (verbatim)

**(a) CANONICAL — the real `kitten diff` entry point aborts on the race.** This is the primary,
non-synthetic evidence: driving the many-file diff through the real Go entry point (the Q4 fixture,
§4.3) on the unconstrained 128-CPU machine, the kitten's own stderr shows the highlight worker pool
racing on `LRUCache.Set`. The stack is unambiguous — `highlight_all.func1` → `LRUCache.Set`
(`cache.go:34`) dispatched by `Context.Parallel` (`utils.go:50,52`), with the main goroutine in the
TUI event loop (`loop.(*Loop).run`). This is the exact `head -15` of the captured stderr:

```text
fatal error: concurrent map writes

goroutine 328 [running]:
kitty/tools/utils.(*LRUCache[...]).Set(0x39, {0xc0005025e0?, 0x0?}, {0xc001055408?, 0x3, 0x200})
	/tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a/tools/utils/cache.go:34 +0x85
kitty/kittens/diff.highlight_all.func1(0xc000742000)
	/tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a/kittens/diff/highlight.go:224 +0xae
kitty/tools/utils/images.(*Context).Parallel.func1()
	/tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a/tools/utils/images/utils.go:52 +0x54
created by kitty/tools/utils/images.(*Context).Parallel in goroutine 139
	/tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a/tools/utils/images/utils.go:50 +0xe5

goroutine 1 [select]:
kitty/tools/tui/loop.(*Loop).run(0xc0001c86c8)
	/tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a/tools/tui/loop/run.go:479 +0x10d9
```

This canonical crash is the same one characterized quantitatively in §4.5 (first stderr line
`fatal error: concurrent map writes` on 3/3 full-machine runs). It proves the race is reached on the
**real** code path — not merely in a synthetic probe.

**(b) The diff kitten's shipped tests report no race** — because they never drive concurrent `Set`:

```console
$ go test -race ./kittens/diff/...
ok  	kitty/kittens/diff	1.099s
```

**(c) The `tools/utils` tests also report no `DATA RACE`** (complete, unedited output). The only
failure is the unrelated `TestFileLock`, an environment issue — it fails **identically without**
`-race`, so it is not a race:

```console
$ go test -race ./tools/utils/...
?   	kitty/tools/utils/images	[no test files]
?   	kitty/tools/utils/paths	[no test files]
?   	kitty/tools/utils/random	[no test files]
?   	kitty/tools/utils/secrets	[no test files]
--- FAIL: TestFileLock (0.00s)
    filelock_test.go:41: Lock test process failed with error: exec: no command and output:
FAIL
FAIL	kitty/tools/utils	0.067s
ok  	kitty/tools/utils/base85	1.026s
ok  	kitty/tools/utils/humanize	1.026s
ok  	kitty/tools/utils/shlex	1.017s
ok  	kitty/tools/utils/shm	1.018s
ok  	kitty/tools/utils/style	1.019s
FAIL
```

```console
$ go test ./tools/utils/ -run TestFileLock        # WITHOUT -race: same failure
--- FAIL: TestFileLock (0.00s)
    filelock_test.go:41: Lock test process failed with error: exec: no command and output:
FAIL
FAIL	kitty/tools/utils	0.006s
FAIL
```

Neither test suite exercises *concurrent* `Set`, so neither surfaces the latent race. To isolate it
deterministically, a small **external diagnostic probe (NON-CANONICAL)** — living outside the repo
tree, so the repository stays unchanged — drives the **real** `kitty/tools/utils.LRUCache.Set`
concurrently with **distinct** keys, exactly mirroring the `highlight_all` pattern (one distinct key
per worker, one shared cache). It imports the real package via a `replace` directive; it is a
diagnostic probe, **not** the kitten's own path (that is (a) above).

Full probe source (`/tmp/racetest/go.mod` and `/tmp/racetest/main.go`):

```go
// go.mod
module racetest

go 1.22

require kitty v0.0.0

require (
	github.com/ALTree/bigfloat v0.2.0 // indirect
	github.com/google/uuid v1.6.0 // indirect
	github.com/seancfoley/bintree v1.3.1 // indirect
	github.com/seancfoley/ipaddress-go v1.6.0 // indirect
	github.com/shirou/gopsutil/v3 v3.24.5 // indirect
	github.com/tklauser/go-sysconf v0.3.12 // indirect
	github.com/tklauser/numcpus v0.6.1 // indirect
	golang.org/x/exp v0.0.0-20230801115018-d63ba01acd4b // indirect
	golang.org/x/sys v0.21.0 // indirect
	howett.net/plist v1.0.1 // indirect
)

replace kitty => /tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a
```

```go
// main.go
// External diagnostic probe (NON-CANONICAL): lives outside the kitty repo and
// drives the REAL kitty/tools/utils.LRUCache.Set concurrently, mirroring the
// highlight_all pattern (one DISTINCT key per worker, one shared cache) at
// kittens/diff/highlight.go:224. It exists only to surface the latent map-write
// race in cache.go:34 in isolation; it is not the kitten's own code path.
package main

import (
	"fmt"
	"sync"

	"kitty/tools/utils"
)

func main() {
	cache := utils.NewLRUCache[string, int](4096)
	const workers = 8
	const perWorker = 2000
	var wg sync.WaitGroup
	for g := 0; g < workers; g++ {
		wg.Add(1)
		go func(g int) {
			defer wg.Done()
			for i := 0; i < perWorker; i++ {
				// DISTINCT key per write — exactly like one path per highlight worker.
				cache.Set(fmt.Sprintf("g%d-k%d", g, i), i)
			}
		}(g)
	}
	wg.Wait()
	fmt.Println("done", workers*perWorker)
}
```

Exact command and **complete, unedited** output (`go run -race .` from `/tmp/racetest`):

```console
$ cd /tmp/racetest && GOFLAGS=-mod=mod go run -race .
==================
WARNING: DATA RACE
Write at 0x00c000418e70 by goroutine 8:
  runtime.mapassign_faststr()
      /usr/local/go/src/runtime/map_faststr.go:203 +0x0
  kitty/tools/utils.(*LRUCache[go.shape.string,go.shape.int]).Set()
      /tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a/tools/utils/cache.go:34 +0x98
  main.main.func1()
      /tmp/racetest/main.go:26 +0x184
  main.main.gowrap1()
      /tmp/racetest/main.go:28 +0x41

Previous write at 0x00c000418e70 by goroutine 12:
  runtime.mapassign_faststr()
      /usr/local/go/src/runtime/map_faststr.go:203 +0x0
  kitty/tools/utils.(*LRUCache[go.shape.string,go.shape.int]).Set()
      /tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a/tools/utils/cache.go:34 +0x98
  main.main.func1()
      /tmp/racetest/main.go:26 +0x184
  main.main.gowrap1()
      /tmp/racetest/main.go:28 +0x41

Goroutine 8 (running) created at:
  main.main()
      /tmp/racetest/main.go:22 +0x1b7

Goroutine 12 (running) created at:
  main.main()
      /tmp/racetest/main.go:22 +0x1b7
==================
fatal error: concurrent map writes
fatal error: concurrent map writes
fatal error: concurrent map writes
fatal error: concurrent map writes

goroutine 38 [running]:
kitty/tools/utils.(*LRUCache[...]).Set(0x87ac20, {0xc0000141cb, 0x5}, 0x8)
	/tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a/tools/utils/cache.go:34 +0x99
main.main.func1(0x3)
	/tmp/racetest/main.go:26 +0x185
created by main.main in goroutine 1
	/tmp/racetest/main.go:22 +0x1b8

goroutine 1 [semacquire]:
sync.runtime_Semacquire(0xc000226e08?)
	/usr/local/go/src/runtime/sema.go:62 +0x25
sync.(*WaitGroup).Wait(0xc000226e00)
	/usr/local/go/src/sync/waitgroup.go:116 +0xa5
main.main()
	/tmp/racetest/main.go:30 +0x30a

goroutine 35 [running]:
	goroutine running on other thread; stack unavailable
created by main.main in goroutine 1
	/tmp/racetest/main.go:22 +0x1b8

goroutine 36 [running]:
	goroutine running on other thread; stack unavailable
created by main.main in goroutine 1
	/tmp/racetest/main.go:22 +0x1b8

goroutine 37 [runnable]:
kitty/tools/utils.(*LRUCache[...]).Set(0x87ac20, {0xc00071003b, 0x5}, 0x2)
	/tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a/tools/utils/cache.go:34 +0x99
main.main.func1(0x2)
	/tmp/racetest/main.go:26 +0x185
created by main.main in goroutine 1
	/tmp/racetest/main.go:22 +0x1b8

goroutine 39 [runnable]:
fmt.(*pp).doPrintf(0xc00060e270, {0x7e9a83, 0x7}, {0xc0001c4f78, 0x2, 0x2})
	/usr/local/go/src/fmt/print.go:1019 +0x1db9
fmt.Sprintf({0x7e9a83, 0x7}, {0xc0001c4f78, 0x2, 0x2})
	/usr/local/go/src/fmt/print.go:239 +0x5d
main.main.func1(0x4)
	/tmp/racetest/main.go:26 +0x165
created by main.main in goroutine 1
	/tmp/racetest/main.go:22 +0x1b8

goroutine 40 [running]:
	goroutine running on other thread; stack unavailable
created by main.main in goroutine 1
	/tmp/racetest/main.go:22 +0x1b8

goroutine 41 [runnable]:
sync/atomic.(*Int32).Add(0xc000418e28, 0xffffffff)
	/usr/local/go/src/sync/atomic/type.go:88 +0x56
sync.(*RWMutex).RUnlock(0xc000418e18)
	/usr/local/go/src/sync/rwmutex.go:116 +0x4e
kitty/tools/utils.(*LRUCache[...]).Set(0x87ac20, {0xc00069803b, 0x5}, 0x2)
	/tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a/tools/utils/cache.go:35 +0xbc
main.main.func1(0x6)
	/tmp/racetest/main.go:26 +0x185
created by main.main in goroutine 1
	/tmp/racetest/main.go:22 +0x1b8

goroutine 42 [runnable]:
main.main.func1(0x7)
	/tmp/racetest/main.go:26 +0xcf
created by main.main in goroutine 1
	/tmp/racetest/main.go:22 +0x1b8

goroutine 36 [running]:
kitty/tools/utils.(*LRUCache[...]).Set(0x87ac20, {0xc0001ca18a, 0x6}, 0x12)
	/tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a/tools/utils/cache.go:34 +0x99
main.main.func1(0x1)
	/tmp/racetest/main.go:26 +0x185
created by main.main in goroutine 1
	/tmp/racetest/main.go:22 +0x1b8

goroutine 40 [running]:
kitty/tools/utils.(*LRUCache[...]).Set(0x87ac20, {0xc0002270ea, 0x6}, 0x25)
	/tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a/tools/utils/cache.go:34 +0x99
main.main.func1(0x5)
	/tmp/racetest/main.go:26 +0x185
created by main.main in goroutine 1
	/tmp/racetest/main.go:22 +0x1b8

goroutine 35 [running]:
kitty/tools/utils.(*LRUCache[...]).Set(0x87ac20, {0xc00049020a, 0x6}, 0x18)
	/tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a/tools/utils/cache.go:34 +0x99
main.main.func1(0x0)
	/tmp/racetest/main.go:26 +0x185
created by main.main in goroutine 1
	/tmp/racetest/main.go:22 +0x1b8
exit status 2
```

The probe pinpoints the race at **`cache.go:34`** inside `LRUCache.Set` (`runtime.mapassign_faststr`
called from two goroutines writing distinct keys), then the Go runtime aborts with
`fatal error: concurrent map writes`. The `WARNING: DATA RACE` + `cache.go:34` + `concurrent map`
`writes` invariants reproduced on **3/3** runs of the probe. One goroutine in the dump is even caught
at `cache.go:35` (`RUnlock`), confirming the mutation happens under the read lock.

### 5.4 Determinism check (same input ×2) — and the honest failure mode

To test whether parallel highlighting *corrupts* output, the same 12-file, highlightable directory
(`src_1.py … src_12.py`) is diffed twice and the rendered text compared. Pinned to a single CPU
(`taskset -c 0` → `runtime.NumCPU()==1` → a single highlight worker → the race **cannot** occur), the
output is a stable, race-free baseline and is **identical across runs**:

```console
$ taskset -c 0 python3 /tmp/tui_capture.py --raw-out --cols 160 --rows 60 --quit-after 6 -- \
      "$KIT" diff -o diff_cmd=git /tmp/kdiff/q5_left /tmp/kdiff/q5_right > /tmp/q5_runA.txt
$ taskset -c 0 python3 /tmp/tui_capture.py --raw-out --cols 160 --rows 60 --quit-after 6 -- \
      "$KIT" diff -o diff_cmd=git /tmp/kdiff/q5_left /tmp/kdiff/q5_right > /tmp/q5_runB.txt
$ md5sum /tmp/q5_runA.txt /tmp/q5_runB.txt
d71a09172b1eba66a2cd30264fa3cb36  /tmp/q5_runA.txt
d71a09172b1eba66a2cd30264fa3cb36  /tmp/q5_runB.txt
$ diff -q /tmp/q5_runA.txt /tmp/q5_runB.txt && echo "IDENTICAL TEXT CONTENT ACROSS RUNS"
IDENTICAL TEXT CONTENT ACROSS RUNS
```

All 12 files pair correctly (`src_1.py` … `src_12.py`), each `return x * N  # left` →
`return x + N  # right` (rendered rows, run A):

```text
   src_1.py
4      return x * 1  # left                          4      return x + 1  # right
   src_10.py
4      return x * 10  # left                         4      return x + 10  # right
   src_11.py
4      return x * 11  # left                         4      return x + 11  # right
   src_12.py
4      return x * 12  # left                         4      return x + 12  # right
```

**The honest failure mode on the multi-core machine:** the *same* 12-file input run **unconstrained**
(`runtime.NumCPU()==128`) does not reliably complete — it **crashes probabilistically** with the same
`concurrent map writes` race. Across two batches (7 runs total) it aborted on **3** and completed on
**4**:

```text
full-machine 12-file, capturing kitten stderr:
  batch A: RUN1 CRASHED  RUN2 CRASHED                       (2/2 concurrent map writes)
  batch B: RUN1 ok  RUN2 ok  RUN3 CRASHED  RUN4 ok  RUN5 ok (1/5 concurrent map writes)
  => 3 crashes / 7 runs — probabilistic, same race as §4.5 and §5.3(a)
```

Crucially, the failure mode is a **crash (no output)**, never *corrupted* output — consistent with the
distinct-key design (§5.5). When the run does complete, the text is identical to the race-free
baseline above.

### 5.5 Causal reason

Highlighting does not "step on itself" **logically** because each worker writes a *different* cache key
(`highlighted_lines_cache.Set(path, …)`, `highlight.go:224`); there is no shared value two workers
fight over, which is why a completed run is byte-for-byte identical across runs (§5.4). But that key
disjointness does **not** make the operation thread-safe: `LRUCache.Set` writes into a single shared Go
map while holding only a **read** lock (`cache.go:33-35`), and Go maps are unsafe for concurrent
writes regardless of key. When `Context.Parallel` runs `runtime.NumCPU()` highlight workers
(`utils.go:50,52`), two of them can execute `self.data[key] = val` (`cache.go:34`) at the same
instant. The Go runtime detects the concurrent map write and aborts the whole process — observed
**canonically** through the real `kitten diff` entry point (§5.3(a), §4.5) and **isolated** by the
external race probe (§5.3, `WARNING: DATA RACE … cache.go:34` → `fatal error: concurrent map writes`).
At one CPU there is a single highlight worker, the concurrent write cannot happen, and the run is
deterministic (§5.4). So the plain answer — *"parallel highlighting is safe because each worker writes
a distinct key"* — is only half true: it avoids logical corruption, but the unsynchronized shared-map
write is a real race that, on this multi-core machine, crashes the kitten rather than corrupting data.
(Read-only scope: reported, not fixed.)

---


## 6. Q6 — Binary files and images

### 6.1 Direct answer

When a file is not text, the kitten does not attempt a line diff. A **non-text, non-image** file is
rendered as a single banner `Binary file: <human-readable size>`. A **non-text file that is an image**
is rendered via the kitty graphics protocol (its pixels are transmitted to the terminal), with a
`Dimensions: WxH Size: …` header. Plain UTF-8 text takes the normal syntax-highlighted line diff.

### 6.2 Mechanism & code anchors

In `render()` (`render.go:696`) the classification is:

- `is_binary := !is_path_text(path)` (`render.go:706`), where `is_path_text` (`collect.go:86`) tests
  the image MIME prefix and UTF-8 validity of the cached bytes;
- refined for diffs: `if !is_binary && item_type == "diff" && !is_path_text(changed_path) { is_binary
  = true }` (`render.go:707-709`);
- `is_img := is_binary && is_image(path) || (item_type == "diff" && is_image(changed_path))`
  (`render.go:710`).

The dispatch `switch item_type` (`render.go:712`) routes each case — `"diff"` (`render.go:713-714`),
`"add"` (`render.go:726-727`), `"removal"` (`render.go:739-740`) — to `image_lines`
(definition `render.go:333`) when `is_img`, else to `binary_lines` (definition `render.go:446`), else
to the normal text path. `binary_lines` emits `fmt.Sprintf("Binary file: %s", human_readable(sz))`
(`render.go:452`). `image_lines` emits a `Size: %s` line (`render.go:341`) and, once the resolution is
known (`res.Width > -1`, `render.go:343`), prepends `Dimensions: %dx%d` (`render.go:344`); the pixel
payload is only reserved once `GetSizeIfAvailable` succeeds (`render.go:358`), otherwise the placeholder
`Loading image...` is shown (`render.go:364`). Corroborated by `docs/kittens/diff.rst:18` ("Displays
images as well as text diffs, even over SSH").

### 6.3 Command(s) run and observed output — all three branches

All three run through the **real** `kitten diff` entry point under the PTY harness (§0.4), pinned to
one CPU (`taskset -c 0`) so the render is stable. Each fixture holds one file per side.

**(a) UTF-8 text** — `file` reports UTF-8; the diff is a normal highlighted line diff (note the
non-ASCII `café` is preserved on both sides):

```console
$ file /tmp/kdiff/q6_text_l/doc.txt
/tmp/kdiff/q6_text_l/doc.txt: Unicode text, UTF-8 text
$ taskset -c 0 python3 /tmp/tui_capture.py --cols 120 --rows 20 --quit-after 4 -- \
      "$KIT" diff -o diff_cmd=git /tmp/kdiff/q6_text_l /tmp/kdiff/q6_text_r
```

```text
   doc.txt
   @@ -1,3 +1,3 @@
1  plain UTF-8 text                                         1  plain UTF-8 text
2  second line café                                         2  second line CHANGED café
3  third                                                    3  third
```

**(b) non-UTF-8 binary** — 4096 bytes each side that are **not** valid UTF-8. `file` reports `data`,
and a UTF-8 decode raises `UnicodeDecodeError` (the **complete, unedited** traceback — the offending
byte is `0x85` at position 19):

```console
$ file /tmp/kdiff/q6_bin_l/data.bin
/tmp/kdiff/q6_bin_l/data.bin: data
$ python3 -c "open('/tmp/kdiff/q6_bin_l/data.bin','rb').read().decode('utf-8')"
Traceback (most recent call last):
  File "<string>", line 1, in <module>
    open('/tmp/kdiff/q6_bin_l/data.bin','rb').read().decode('utf-8')
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^
UnicodeDecodeError: 'utf-8' codec can't decode byte 0x85 in position 19: invalid start byte
```

Rendered through the kitten, both sides are the literal `Binary file: 4 KB` banner (no line diff):

```console
$ taskset -c 0 python3 /tmp/tui_capture.py --cols 120 --rows 20 --quit-after 4 -- \
      "$KIT" diff -o diff_cmd=git /tmp/kdiff/q6_bin_l /tmp/kdiff/q6_bin_r
```

```text
   data.bin
   Binary file: 4 KB                                           Binary file: 4 KB
```

**(c) image** — a 48×32 PNG each side (left 118 B, right 119 B). `file` confirms the PNG; the kitten
routes it to the image branch, showing the `Dimensions: 48x32 Size: <N> B` header and the
`Loading image...` placeholder (reproduced **identically across 2 runs**):

```console
$ file /tmp/kdiff/q6_img_l/pic.png
/tmp/kdiff/q6_img_l/pic.png: PNG image data, 48 x 32, 8-bit/color RGB, non-interlaced
$ taskset -c 0 python3 /tmp/tui_capture.py --cols 120 --rows 20 --quit-after 4 -- \
      "$KIT" diff -o diff_cmd=git /tmp/kdiff/q6_img_l /tmp/kdiff/q6_img_r
```

```text
   pic.png
   Dimensions: 48x32 Size: 118 B                              Dimensions: 48x32 Size: 119 B
   Loading image...                                           Loading image...
```

**Honest note on pixel transmission (reported exactly as observed).** In this pyte-based PTY harness
the kitten transmits **no image pixels**. The faithful metric is *not* the raw total of kitty
graphics-protocol APC chunks (`ESC _ G …`): that total is entirely control-plane traffic — capability
probes and cleanup deletes — whose count depends on the terminal lifecycle and so varies between
harnesses. The reproducible invariant is the number of **pixel transmit/display** chunks (graphics
action `a=T`, `a=t`, or the default-`a` transmit, and display `a=p`), which is **0**. Capturing the
raw byte stream (`--raw-out`) and classifying every `ESC _ G` chunk by its `a=` action confirms this,
stably across 3 identical runs and across quit-windows of 2 s – 8 s:

```console
$ taskset -c 0 python3 /tmp/tui_capture.py --raw-out /tmp/q6c_img_raw.bin --cols 120 --rows 20 --quit-after 6 -- \
      "$KIT" diff -o diff_cmd=git /tmp/kdiff/q6_img_l /tmp/kdiff/q6_img_r > /tmp/q6c_grid.txt
$ python3 -c "import re,collections; d=open('/tmp/q6c_img_raw.bin','rb').read(); ch=re.findall(rb'\x1b_G([^;\x1b]*)', d); act=lambda c: dict(p.split(b'=',1) for p in c.split(b',') if b'=' in p).get(b'a',b't').decode(); a=[act(c) for c in ch]; C=collections.Counter(a); print('total ESC_G =', len(a), '| by action =', dict(sorted(C.items())), '| pixel transmit/display (a=t/T/p) =', sum(v for k,v in C.items() if k in ('t','T','p')))"
total ESC_G = 4 | by action = {'d': 2, 'q': 2} | pixel transmit/display (a=t/T/p) = 0
```

The four chunks observed here are all control-plane and none carries pixels: two capability
**queries** (`a=q`) from `ImageCollection.Initialize` — the tempfile probe (`a=q,f=24,t=t,…,i=1`)
and the shared-memory probe (`a=q,f=24,t=s,…,i=2`) (`tools/tui/graphics/collection.go:183-211`) —
and two **deletes** (`a=d`), one per allocated image id, from `Finalize`
(`tools/tui/graphics/collection.go:223`). The *total* is therefore harness-dependent: an independent
harness that redraws a different number of times observes a different total (e.g. `5 = a=q×2 + a=d×3`),
because each `draw_screen` redraw can additionally emit a `DeleteAllVisiblePlacements` delete
(`collection.go:149-151`, `ui.go:345`). What is invariant — and what actually answers the question —
is that **zero pixel-transmit/display chunks are emitted**.

The cause is visible in `image_lines`: pixels are only reserved when
`image_collection.GetSizeIfAvailable(path, image_size)` **succeeds** (`render.go:358`); when it
returns `graphics.ErrNotFound` the branch instead emits `Loading image...` (`render.go:364`). The
harness is not a graphics-capable terminal — it answers the DA/CPR queries but never the graphics
`a=q` capability probe — so `GetSizeIfAvailable` never succeeds and the render stays on the
placeholder, hence **no pixel-transmit chunks**. **The actual on-screen pixel display therefore
could not be exercised in this harness**; that it *does* display in a real terminal ("even over SSH")
is corroborated by `docs/kittens/diff.rst:18` and by the `render.go:358` success branch — this is
*inferred / documentation-corroborated*, not observed here.

**State before/during/after (Rule R8).** The classification header has two states in the code:
before resolution, `res.Width == -1`, so only `Size: <N> B` is shown; after resolution
(`res.Width > -1`, `render.go:343`) the `Dimensions: 48x32` prefix is prepended (`render.go:344`).
In practice the tiny local PNG resolves **before the first captured frame**: even at a 0.2 s quit
window the only observed state is the final `Dimensions: 48x32 Size: <N> B`. The bare `Size:`-only
pre-resolution state is therefore *inferred from the code path*, not captured. The observable state
progression here is: `Loading image...` placeholder (throughout, since pixels never load) alongside
the resolved `Dimensions: 48x32` header.

### 6.4 Causal reason

`is_path_text` (`collect.go:86`) drives the split: it returns false for the random-bytes file
(invalid UTF-8, byte `0x85` above) and for the PNG (image MIME), setting `is_binary`
(`render.go:706`). `is_image` (`collect.go:82`, MIME prefix `image/`) then separates the two — the PNG
makes `is_img` true (`render.go:710`) → `image_lines` (`render.go:333`) → the `Dimensions`/`Size`
header and, in a graphics terminal, the transmitted pixels; the random bytes stay `is_img == false`
→ `binary_lines` (`render.go:446`) → the `Binary file: 4 KB` banner (`render.go:452`, size from
`human_readable(4096)`). Text passes both checks and takes the normal highlighted diff. Within the
image branch, the placeholder-vs-pixels decision is made by whether `GetSizeIfAvailable` finds the
image already registered with the terminal (`render.go:358`); in this harness it never does, so
`Loading image...` (`render.go:364`) is shown and no pixel-transmit chunks are emitted — exactly the
observed **pixel transmit/display (`a=t`/`T`/`p`) = 0** (the only `ESC _ G` chunks seen are the
control-plane `a=q` capability probes and `a=d` deletes).

---


## 7. Q7 — End-to-end runtime trace

### 7.1 Direct answer

From "two directories compared" to a fully resolved screen, the flow is:
`main` validates the two arguments and sets up the engine + caches → a background goroutine builds the
**collection** (pairing/classification) → the main thread is woken and, on the collection result,
runs **generate_diff**, **highlight_all**, and **load_all_images** → the diff jobs complete and the
screen renders every change, rename, addition, and removal together. A single fixture containing all
four kinds resolves them all in one screen.

### 7.2 Mechanism & code anchors

Entry and setup (`kittens/diff/main.go`): `main` (`main.go:102`) → `load_config` (`main.go:104`) →
argument check `if len(args) != 2` returning the error `"You must specify exactly two
files/directories to compare"` (`main.go:108-109`) → `set_diff_command(conf.Diff_cmd)`
(`main.go:111`) → `init_caches()` (`main.go:114`) → `create_formatters()` (`main.go:115`); the exported
`EntryPoint` is at `main.go:177`.

UI orchestration (`kittens/diff/ui.go`): the `Handler` struct (`ui.go:55`) owns
`async_results chan AsyncResult` (`ui.go:56`), created with buffer 32 (`ui.go:132`). A startup
goroutine (`ui.go:133`) runs `create_collection(self.left, self.right)` (`ui.go:135`), sends the
result (`ui.go:136`), and calls `self.lp.WakeupMainThread()` (`ui.go:137`). `handle_async_result`
(`ui.go:245`) routes results: on `COLLECTION` (`ui.go:247`) it stores the collection (`ui.go:248`) and
calls `generate_diff()` (`ui.go:249`), `highlight_all()` (`ui.go:250`), `load_all_images()`
(`ui.go:251`); on `DIFF` (`ui.go:252`) it stores the diff map (`ui.go:253`), computes statistics
(`ui.go:254`), and renders (`ui.go:256`). `generate_diff` (`ui.go:142`) builds jobs from
`collection.Apply` (`ui.go:145`) and, in a goroutine, calls `diff(jobs, …)` (`ui.go:155`) then wakes
the main thread (`ui.go:157`). `mouse.go` and `search.go` are peripheral (referenced only for context).

### 7.3 Command(s) run

A single fixture contains all four change kinds at once, verified by hash: `changed.txt` differs;
`oldname.txt` (left) and `newname.txt` (right) are byte-identical (a rename); `removed.txt` is
left-only; `added.txt` is right-only.

```console
$ (cd /tmp/kdiff/q7_left  && for f in $(find . -type f|sort); do echo "$(md5sum "$f"|cut -d' ' -f1)  $f"; done)
4ac5d7a447c4194d9a380c0965bf8adf  ./changed.txt
1ab8c70ed77446bd72b7be2d3782072c  ./oldname.txt
068cd2ad65bdd9e65c1006a0c17cb881  ./removed.txt
$ (cd /tmp/kdiff/q7_right && for f in $(find . -type f|sort); do echo "$(md5sum "$f"|cut -d' ' -f1)  $f"; done)
ecf5812c8deede9a38dac7fc56213ea1  ./added.txt
6d2532a81558cc3f2856c1b4a277aeb1  ./changed.txt
1ab8c70ed77446bd72b7be2d3782072c  ./newname.txt
$ $KIT diff /tmp/kdiff/q7_left /tmp/kdiff/q7_right
```

### 7.4 Observed output — all four resolved in one screen

```text
   added.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   This file was added                                                     1  this file exists only on the right and is brand new

   changed.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   @@ -1,2 +1,2 @@
1  config value = 1                                                        1  config value = 2
2  keep this                                                               2  keep this

   oldname.txt                                                                newname.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


   removed.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1  this file exists only on the left and is deleted                           This file was removed
```

All four are present, sorted by path: `added.txt` (addition), `changed.txt` (change,
`config value = 1` → `2`), `oldname.txt│newname.txt` (rename — two-name title, empty body per §2.6),
and `removed.txt` (removal).

### 7.5 Causal reason and observed-vs-inferred

The screen proves the pipeline resolved every classification: `create_collection` paired and
categorized the files (Q1/Q2 mechanisms), `generate_diff` produced the line hunks for the change, and
rendering placed additions/removals/renames using the branches in `render.go`. The following are
**observed** at runtime: the four resolved classifications (above), the parallel `git` subprocesses
(§4), the transient "Calculating diff…" state during `generate_diff`, and "Loading image…" from
`load_all_images` (§6). The goroutine / `async_results` channel / `WakeupMainThread` routing internals
are read from source and so are labelled **(inferred)** — the trace hops
`main → create_collection → handle_async_result → generate_diff → highlight_all → load_all_images` are
grounded in the cited `file:line` locations but were not individually instrumented at runtime.

---


## 8. Q8 — Diff algorithm + cache interplay

### 8.1 Direct answer (lead with the plain reading)

With the **default** configuration (`diff_cmd = 'auto'`), the builtin diff algorithm in `diff.go`
**does not run** on this machine — `auto` resolves to external `git diff --no-index`, because git is
installed. The builtin "anchored diff" engine only runs when explicitly selected (`-o
diff_cmd=builtin`) or when neither `git` nor `diff` exists. All three engines were exercised
explicitly; they produce the same side-by-side rendering, differing only in the raw unified-diff
header they emit. The builtin engine matches regions using a longest-common-subsequence of **unique**
lines ("anchored"/"patience" diff), and it consumes the once-read cached bytes, so switching engines
re-runs only the diff step — not the file reads or highlighting.

### 8.2 Mechanism & code anchors

**Engine resolution** (`kittens/diff/patch.go`): the command templates are
`GIT_DIFF = "git diff --no-color --no-ext-diff --exit-code -U_CONTEXT_ --no-index --"`
(`patch.go:21`) and `DIFF_DIFF = "diff -p -U _CONTEXT_ --"` (`patch.go:22`). `find_differ`
(`patch.go:34`) prefers git (`patch.go:36`), then `diff` (`patch.go:38`), then the builtin empty
command (`patch.go:40`). `set_diff_command` (`patch.go:44`) maps the config value: `"auto"` →
`find_differ` (`patch.go:47`); `"builtin"`/`""` → `diff_cmd = []string{}` (`patch.go:48-49`); `"diff"`
(`patch.go:50-51`); `"git"` (`patch.go:52-53`); a custom string via `shlex` (`patch.go:54-59`).
`run_diff` (`patch.go:282`) branches on `if len(diff_cmd) == 0` (`patch.go:294`): builtin path reads
both files via `data_for_path` and calls `Diff(path1, data1, path2, data2, num_of_context_lines)`
(`patch.go:303`), returning nil→no-difference (`patch.go:304`); otherwise it substitutes `_CONTEXT_`
with the context count (`patch.go:309-311`) and runs the external command via `exec.Command`
(`patch.go:315`), treating exit code 1 as "differences found" (`patch.go:321-323`).

**Builtin anchored diff** (`kittens/diff/diff.go`): the file header states it is *"Copied from the Go
stdlib, with modifications."* (`diff.go:1-2`), referencing `internal/diff`. `Diff(...)` (`diff.go:49`,
**exported**) returns `nil` immediately when `old == new` (`diff.go:50-52`), prints the
`diff`/`---`/`+++` header (`diff.go:56-60`), and iterates the anchor pairs returned by `tgs(x, y)`
(`diff.go:192`) — the longest common subsequence of lines that appear exactly once in both inputs —
emitting `@@ … @@` unified hunks with context (`diff.go:74-164`). `lines` (`diff.go:172`) appends
`"\ No newline at end of file"` (`diff.go:179`) when an input lacks a trailing newline.

*Background (framing):* the Go `internal/diff` documentation describes this as an "anchored diff": it
seeks the smallest set of unique lines to insert/remove, where unique means a line appearing exactly
once in both old and new, using those unique lines as anchors; it is sometimes called a "patience
diff." This framing is corroborated by directly running the builtin engine below.

### 8.3 Command(s) run — all three engines on the same fixture

The fixture is a 5-line file changed on two lines (`beta`→`BETA`, `epsilon`→`EPSILON`). Each engine is
exercised **canonically** through the real `kitten diff` entry point (§0.4), selecting the engine with
`-o diff_cmd=<engine>`:

```sh
KIT=kitty/launcher/kitten
for eng in git diff builtin; do
  taskset -c 0 python3 /tmp/tui_capture.py --cols 116 --rows 16 --quit-after 4 -- \
      "$KIT" diff -o diff_cmd=$eng /tmp/kdiff/q8_left /tmp/kdiff/q8_right
done
```

To expose the *raw* unified-diff each engine emits (which the TUI parses but never prints verbatim),
the two external engines are also run via their exact command templates, and the builtin engine via a
small **non-canonical** probe that calls the exported `Diff()` directly (full source in §8.4).

### 8.4 Observed output — rendering is identical across engines

**Canonical (real entry point).** All three engines render the *same* side-by-side grid. The loop
above prints byte-identical visible rows for `git`, `diff`, and `builtin`:

```text
engine=git     : @@ -1,5 +1,5 @@ | 2  beta -> 2  BETA | 5  epsilon -> 5  EPSILON
engine=diff    : @@ -1,5 +1,5 @@ | 2  beta -> 2  BETA | 5  epsilon -> 5  EPSILON
engine=builtin : @@ -1,5 +1,5 @@ | 2  beta -> 2  BETA | 5  epsilon -> 5  EPSILON
```

The full builtin frame (captured via `-o diff_cmd=builtin` through the real entry point):

```text
   /tmp/kdiff/q8_left/f.txt                                    /tmp/kdiff/q8_right/f.txt
   @@ -1,5 +1,5 @@
1  alpha                                                    1  alpha
2  beta                                                     2  BETA
3  gamma                                                    3  gamma
4  delta                                                    4  delta
5  epsilon                                                  5  EPSILON
```

**The engines differ only in the raw unified diff** they feed the parser. The two external engines,
run via their exact templates (these ARE the commands the kitten spawns — `patch.go:21-22`):

**git** (`GIT_DIFF`, `_CONTEXT_`=3) — adds a `diff --git`/`index` header; exit 1 = different:

```console
$ git diff --no-color --no-ext-diff --exit-code -U3 --no-index -- /tmp/kdiff/q8_left/f.txt /tmp/kdiff/q8_right/f.txt ; echo "[exit=$?]"
diff --git a/tmp/kdiff/q8_left/f.txt b/tmp/kdiff/q8_right/f.txt
index 600d48ac7..1663761a9 100644
--- a/tmp/kdiff/q8_left/f.txt
+++ b/tmp/kdiff/q8_right/f.txt
@@ -1,5 +1,5 @@
 alpha
-beta
+BETA
 gamma
 delta
-epsilon
+EPSILON
[exit=1]
```

**diff** (`DIFF_DIFF`) — adds timestamp lines; exit 1 = different:

```console
$ diff -p -U 3 -- /tmp/kdiff/q8_left/f.txt /tmp/kdiff/q8_right/f.txt ; echo "[exit=$?]"
--- /tmp/kdiff/q8_left/f.txt	2026-07-07 01:39:16.520276708 +0000
+++ /tmp/kdiff/q8_right/f.txt	2026-07-07 01:39:16.520276708 +0000
@@ -1,5 +1,5 @@
 alpha
-beta
+BETA
 gamma
 delta
-epsilon
+EPSILON
[exit=1]
```

**builtin** — the exported `Diff()` (`diff.go:49`) is the exact function `run_diff` invokes
(`patch.go:303`), but calling it directly is **NON-CANONICAL**: it bypasses the `kitten diff` entry
point. It is shown only to reveal the raw bytes the TUI does not print (the canonical *rendering* is
the identical grid above). The complete external probe (outside the repo tree) is:

```go
// /tmp/difftest/go.mod
module difftest

go 1.22

require kitty v0.0.0

replace kitty => /tmp/blitzy/kitty/blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7_a7903a
```

```go
// /tmp/difftest/main.go  (NON-CANONICAL diagnostic; not the kitten entry point)
package main

import (
	"fmt"

	diff "kitty/kittens/diff"
)

func show(tag, p1, a, p2, b string) {
	out := diff.Diff(p1, a, p2, b, 3)   // diff.go:49
	fmt.Printf("=== %s ===\n", tag)
	fmt.Printf("[len=%d nil?=%v]\n", len(out), out == nil)
	if out != nil {
		fmt.Print(string(out))
	}
	fmt.Println("--- end ---")
}

func main() {
	show("5-line change", "/tmp/kdiff/q8_left/f.txt", "alpha\nbeta\ngamma\ndelta\nepsilon\n",
		"/tmp/kdiff/q8_right/f.txt", "alpha\nBETA\ngamma\ndelta\nEPSILON\n")
	show("no-newline (right only)", "a.txt", "one\ntwo\nthree\n", "b.txt", "one\ntwo\nthree")
	show("no-newline (both, last differs)", "a.txt", "one\ntwo\nthree", "b.txt", "one\ntwo\nTHREE")
	show("identical", "a.txt", "same\ncontent\n", "b.txt", "same\ncontent\n")
	show("both empty", "a.txt", "", "b.txt", "")
}
```

Its raw output for the 5-line fixture — the cleanest header form, bare `diff <old> <new>` with no
`index` line and no timestamps (182 bytes):

```console
$ cd /tmp/difftest && GOFLAGS=-mod=mod go run .
=== 5-line change ===
[len=182 nil?=false]
diff /tmp/kdiff/q8_left/f.txt /tmp/kdiff/q8_right/f.txt
--- /tmp/kdiff/q8_left/f.txt
+++ /tmp/kdiff/q8_right/f.txt
@@ -1,5 +1,5 @@
 alpha
-beta
+BETA
 gamma
 delta
-epsilon
+EPSILON
--- end ---
```

(The remaining `show(...)` cases print in §8.5 and §8.6.)

### 8.5 Edge — "No newline at end of file" (`diff.go:179`)

Fixture: left ends with `\n` (14 bytes), right omits it (13 bytes), last line changed
(`three`→`THREE`); `od -c` confirms:

```console
$ od -c /tmp/kdiff/q8nl_left/f.txt  | tail -2
0000000   o   n   e  \n   t   w   o  \n   t   h   r   e   e  \n
0000016
$ od -c /tmp/kdiff/q8nl_right/f.txt | tail -2
0000000   o   n   e  \n   t   w   o  \n   T   H   R   E   E
0000015
```

**Canonical (real entry point).** `-o diff_cmd=builtin` renders the changed line; the full-screen
side-by-side TUI does **not** surface the `\ No newline at end of file` marker as its own visible
row (honest nuance — the marker lives in the engine byte stream, not the grid):

```text
   f.txt
   @@ -1,3 +1,3 @@
1  one                                                    1  one
2  two                                                    2  two
3  three                                                  3  THREE
```

**NON-CANONICAL probe** — the direct `Diff()` output exposes the marker. Right-only missing newline
(len=105):

```console
=== no-newline (right only) ===
[len=105 nil?=false]
diff a.txt b.txt
--- a.txt
+++ b.txt
@@ -1,3 +1,3 @@
 one
 two
-three
+three
\ No newline at end of file
--- end ---
```

When **both** sides omit the trailing newline (and the last line differs), the marker appears on both
the `-` and `+` lines (len=133):

```console
=== no-newline (both, last differs) ===
[len=133 nil?=false]
diff a.txt b.txt
--- a.txt
+++ b.txt
@@ -1,3 +1,3 @@
 one
 two
-three
\ No newline at end of file
+THREE
\ No newline at end of file
--- end ---
```

The external `git` engine emits the identical `\ No newline at end of file` text for the same fixture
(`diff.go:179` mirrors git's marker).

### 8.6 Edge — identical / empty inputs return `nil` (`diff.go:50-52`)

**Canonical (real entry point).** Diffing two byte-identical **files** with `-o diff_cmd=builtin`
renders the `The files are identical` banner (the `patch.Len() == 0` branch, `render.go:572-573`):

```text
   /tmp/kdiff/q8id_left/f.txt                                  /tmp/kdiff/q8id_right/f.txt
   The files are identical
```

(Observed nuance: two identical files placed *inside two directories* are instead classified
`unchanged` during collection, so the directory diff shows an empty body — the banner is the
file-to-file case, also shown in §1.6.)

**NON-CANONICAL probe** — the direct `Diff()` returns `nil` (zero-length) for identical and for
both-empty inputs, the `old == new` short-circuit at `diff.go:50-52`:

```console
=== identical ===
[len=0 nil?=true]
--- end ---
=== both empty ===
[len=0 nil?=true]
--- end ---
```

### 8.7 Causal reason and cache interplay

`find_differ` (`patch.go:34-42`) checks for `git` first, so on a git-equipped machine `auto` never
reaches the builtin engine — hence the plain-reading "builtin does not run by default." The three
engines produce the same *rendering* because the TUI parses whichever unified diff it receives into the
same side-by-side grid; only the raw header differs (git's `index` line, diff's timestamps, builtin's
bare `diff <old> <new>`). The builtin engine finds matching regions via `tgs` (`diff.go:192`), the
LCS-of-unique-lines that anchors the match, which is why it produces clean, minimal hunks.

**Cache interplay:** the builtin path calls `data_for_path` for both files (`patch.go:295-301`) and so
reuses the once-read cached bytes (§3) before running `Diff()` in-process; this is why the builtin
`openat` count is exactly 1 per file (§3.4). External `git`/`diff` re-read the files themselves in a
subprocess (the two extra non-`O_CLOEXEC` opens in §3.4). Either way, switching the engine re-runs only
the diff step — the file reads (`data_cache`) and syntax highlighting (`highlighted_lines_cache`)
remain cached — so the cache keeps everything fast regardless of which engine is selected.

---


## 9. Coverage checklist (every named item)

Each named mechanism, condition, config option, dependency, and doc corroboration, mapped to the
section that answers it, its `file:line`, and whether it was **observed** at runtime or **(inferred)**
from source.

### 9.1 Functions / methods / structs

| Item | `file:line` | Section | Observed? |
|------|-------------|---------|-----------|
| `collect_files` | collect.go:296 | §1 | observed |
| `walk` | collect.go:260 (calls 299,303) | §1 | observed |
| `data_for_path` (single content read) | collect.go:65-70 | §3 | observed (strace) |
| `hash_for_path` / `md5.Sum` | collect.go:106 / 112 | §2 | observed |
| `sanitize` | collect.go:128-130 | §3 | inferred |
| `lines_for_path` | collect.go:138-146 | §3 | inferred |
| `highlighted_lines_for_path` | collect.go:148-157 | §3 | inferred |
| `add_change` | collect.go:167 (call 319) | §1 | observed |
| `add_rename` | collect.go:175 (call 354) | §2 | observed |
| `add_add` | collect.go:181 (call 366) | §2 | observed |
| `add_removal` | collect.go:192 (call 362) | §2 | observed |
| `allowed` (ignore-glob) | collect.go:230 | §1.6 | observed |
| `init_caches` / `const sz = 4096` | collect.go:26 / 29 | §3 | observed |
| `Context.Parallel` | tools/utils/images/utils.go:27 | §4 | observed |
| `diff` (worker pool) | patch.go:352 (dispatch 361) | §4 | observed |
| `do_diff` | patch.go:330 (call 365) | §4 | observed |
| `run_diff` | patch.go:282 (branch 294) | §8 | observed |
| `parse_patch` | patch.go:245 | §8 | inferred |
| `find_differ` | patch.go:34-42 | §8 | observed |
| `set_diff_command` | patch.go:44 | §8 | observed |
| `highlight_all` | highlight.go:217 (dispatch 219) | §5 | observed |
| `highlight_file` | highlight.go:161 | §5 | inferred |
| `render` | render.go:696 | §6 | observed |
| `binary_lines` / `"Binary file: %s"` | render.go:446 / 452 | §6 | observed |
| `image_lines` / `Dimensions` | render.go:333 / 344 | §6 | observed |
| `render_screen_line` (rename full-width return) | render.go:70,99-101 | §2.6 | observed |
| `rename_lines` | render.go:684 (msg 688) | §2.6 | observed (as latent bug) |
| `title_lines` (two-name title) | render.go:208-209 | §2 | observed |
| `main` / `EntryPoint` | main.go:102 / 177 | §0,§7 | observed |
| `handle_async_result` | ui.go:245 (COLLECTION 247, DIFF 252) | §7 | inferred |
| `generate_diff` | ui.go:142 (diff() 155) | §7 | inferred |
| `create_collection` (call) | ui.go:135 | §7 | observed (result) |
| `LRUCache` (Get/Set/GetOrCreate/MustGetOrCreate) | tools/utils/cache.go:13 (25/32/39/60) | §3,§5 | observed |
| `Diff` / `tgs` / `lines` | diff.go:49 / 192 / 172 | §8 | observed |

### 9.2 Structs / consts / vars

| Item | `file:line` | Section |
|------|-------------|---------|
| seven caches (size,mimetypes,data,is_text,lines,highlighted_lines,hash) | collect.go:20-24 | §3 |
| `const sz = 4096` | collect.go:29 | §3 |
| `LRUCache[K,V]` | tools/utils/cache.go:13 | §3,§5 |
| `async_results chan AsyncResult` + buffer 32 | ui.go:56 / 132 | §7 |
| `GIT_DIFF` / `DIFF_DIFF` / `diff_cmd` var | patch.go:21 / 22 / 24 | §8 |

### 9.3 Conditions / branches

| Condition | `file:line` | Section | Observed value |
|-----------|-------------|---------|----------------|
| relative-path pairing (`Intersect`) | collect.go:306 | §1 | 3 entries, same.txt omitted |
| rename = MD5 `ah==rh` AND `ld==rd` | collect.go:350 / 353 | §2 | identical→rename; 1 byte→remove+add |
| mode-only change (`lstat.Mode()!=rstat.Mode()`) | collect.go:321-329 (323) | §1.6 | `Mode changed: -rw-r--r-- to -rwxr-xr-x` |
| `is_binary` / `is_img` | render.go:706 / 710 | §6 | binary→banner, image→graphics |
| `"Binary file: %s"` | render.go:452 | §6 | "Binary file: 4 KB" |
| builtin vs external dispatch `len(diff_cmd)==0` | patch.go:294 (Diff 303) | §8 | builtin openat=1; auto=3 |
| no-newline marker | diff.go:179 | §8.5 | "\ No newline at end of file" |
| identical → nil | diff.go:50-52 | §8.6 | len=0 nil?=true |
| arg count `len(args)!=2` | main.go:108-109 | §1.6 | error, exit 1 |

### 9.4 Config options (by name)

| Option | Default | `file:line` | Section |
|--------|---------|-------------|---------|
| `diff_cmd` | `auto` | main.py:41 | §8 |
| `ignore_name` | empty `''` | main.py:56 | §1.6 |
| `num_context_lines` | `3` | main.py:37 | §8 (`-U3`) |
| `syntax_aliases` | `pyj:py pyi:py recipe:py` | main.py:29 | §0.5 |
| `replace_tab_by` | 4 spaces | main.py:52 | §0.5 |

### 9.5 Dependency versions (`go.mod`) and doc corroboration

| Dependency | Version | `go.mod` | Role |
|------------|---------|----------|------|
| Go runtime | 1.22 | go.mod:3 | build |
| `github.com/alecthomas/chroma/v2` | v2.14.0 | go.mod:7 | syntax highlighting (§5) |
| `github.com/bmatcuk/doublestar/v4` | v4.6.1 | go.mod:8 | present in `go.mod` but **not** imported by the diff kitten; `ignore_name` matching uses stdlib `filepath.Match` (`collect.go:233`, §1.6) |
| `github.com/kovidgoyal/imaging` | v1.6.3 | go.mod:13 | image decode/scale (§6) |
| `github.com/edwvee/exiffix` | v0.0.0-20240229113213 | go.mod:10 | EXIF-aware image load (§6) |
| `golang.org/x/image` | v0.17.0 | go.mod:18 | image formats (§6) |
| `github.com/zeebo/xxh3` | v1.0.2 | go.mod:16 | fast hashing |

Doc corroboration (`docs/kittens/diff.rst`): syntax highlighting "asynchronously, for maximum speed"
(L15-16, §5); "Displays images as well as text diffs, even over SSH" (L18, §6); "Does recursive
directory diffing" (L20, §1). All three phrases were confirmed verbatim in the docs file at the cited
lines.

**Note on git (coverage-map correction).** `docs/kittens/diff.rst:92-116` ("Integrating with git")
documents a **user-facing** setup — adding a `[difftool "kitty"]` block with `cmd = kitten diff $LOCAL
$REMOTE` to `~/.gitconfig` and then running `git difftool --no-symlinks --dir-diff` — so that **git
invokes the diff kitten**. That topic lies outside the eight behavioral questions and is therefore
deliberately **not** part of the answer body, so this docs section is **not** mapped to §8. It is the
opposite-direction, distinct mechanism from the one §8 examines, where the diff kitten **internally
invokes `git diff --no-index`** as its default `auto` engine (`GIT_DIFF`, `patch.go:21`; selected by
`find_differ`, `patch.go:34-42`). The two are different features and are intentionally kept separate
here, so the coverage map now lists only docs content actually discussed in the answer body.

---


## 10. Cleanup verification

This investigation was strictly read-only: no existing repository file was modified, and the only
artifact added is this document (plus the new `blitzy/`/`blitzy/documentation/` directories). All
temporary fixtures and observation scripts lived outside the repository tree (under `/tmp`) and were
removed on completion. The gitignored launcher binaries produced by the build are untracked and do not
appear as changes.

The working tree after the investigation (and after all `/tmp` fixtures and scripts were removed)
shows only the new document and its parent directories, with **no existing tracked file modified or
deleted**. This is the actual, unedited output:

```console
$ git status --porcelain
?? blitzy/

$ git status --porcelain --untracked-files=all
?? blitzy/documentation/kitty_815df1e210e0.md

$ git diff --stat
                                 # (empty — zero changes to any existing tracked file)

$ git status --porcelain --untracked-files=all | grep -E '^ ?[MDR]' \
    && echo "TRACKED CHANGES" || echo "OK: zero tracked-file modifications/deletions"
OK: zero tracked-file modifications/deletions
```

The single new untracked path `blitzy/documentation/kitty_815df1e210e0.md` is this document; the
gitignored launcher binaries produced by the build do not appear (they are untracked and ignored).
After committing this document the working tree is clean.

---

*End of document. All `file:line` citations correspond to commit
`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`; every behavioral claim above is paired with the exact
command run and its complete, unedited observed output, or is explicitly labelled `(inferred)`.*

