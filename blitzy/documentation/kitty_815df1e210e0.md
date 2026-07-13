# kitty's keyboard-protocol flag stack across the main/alternate screen buffers — a runtime-observed investigation

> **Objective.** Answer, with real runtime-observed evidence, how kitty's keyboard-protocol
> progressive-enhancement flag stack behaves when the terminal switches between its **main** and
> **alternate** screen buffers. Every behavioral claim below is paired with the **complete, unedited**
> output that demonstrates it and the exact command that produced it, captured from the **compiled**
> terminal (not from code-reading alone).
>
> **Source under study.** `kovidgoyal/kitty` at the pinned commit
> `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (subject *"Wire up applying of font config"*). That
> commit is the logical source branch `kitty_815df1e210e0`, from which this document's file name
> derives. The investigation and this deliverable are committed on the **destination** branch
> `blitzy-03f51806-b79e-4da1-b1c1-47a97745c4c7`. Because this deliverable was refined over several
> documentation commits on that branch, the branch **HEAD is *ahead of* the pinned source commit**,
> and its exact position moves with each revision — so the branch HEAD (and any relative reference
> such as `HEAD~1`) is **not** a stable anchor. The immutable anchor is the pinned hash itself:
> `git merge-base --is-ancestor 815df1e210e0 HEAD` confirms the pinned commit is an ancestor of HEAD,
> and `git diff --name-status 815df1e210e0 HEAD` shows the only difference between the pinned commit
> and HEAD is the addition of this one document — i.e. **no kitty source file differs** between the
> pinned commit and HEAD (both invariants are verified in [§1.1](#11-provenance-and-toolchain)). The
> code exercised here is therefore the pinned commit `815df1e210e0…`, built in its default
> configuration.
>
> **Canonical build image.**
> `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
> (from `ghcr.io/scaleapi/swe-atlas`).
>
> **Observed vs. inferred.** Two kinds of evidence appear below and are labelled at each use:
> **[observed]** — values printed by the compiled code at runtime (the real VT parser, the real
> per-buffer stack, and the real C key-encoder); and **[inferred]** — statements derived by reading
> the source (with `file:line` anchors) where a runtime channel is unavailable in this headless
> environment (notably the live GUI keypress path, which needs a display). No inferred value is
> presented as if it were observed.

## TL;DR — direct answers

1. **Round trip (main → push → alt → push → back to main).** After pushing *disambiguate* (flag `1`)
   on the main buffer, switching to the alternate buffer, pushing *report-all-keys* (flag `8`) there,
   and switching back, the **active flags on the main buffer are `1` again** — the main-buffer stack
   survives the round trip intact because switching buffers only re-points an active pointer; it never
   copies flags between the two arrays. **[observed]**, see [§2](#2-obj-1--round-trip-stack-survival).

2. **Exhaustion.** The stack holds **8 entries**. A push onto a full stack **silently evicts the
   oldest entry** (no error). Exhausting one buffer's stack does **not** affect the other's.
   **[observed]**, see [§3](#3-obj-2--stack-exhaustion-on-both-buffers-and-cross-buffer-isolation).

3. **Pop-to-empty.** A pop that empties the stack **resets all flags to `0`**; over-popping does not
   underflow or error. **[observed]**, see [§4](#4-obj-3--pop-to-empty-reset-first-push-from-empty-and-over-pop).

4. **Ctrl+Shift+a in the four requested states.** All four states emit the identical byte sequence
   **`b'\x1b[97;6u'`** (`ESC [ 97 ; 6 u`): codepoint `97` = `a`, modifier field `6` = the Ctrl+Shift
   bitmask `5` plus `1`. The *plain* and *Ctrl+a* keys, by contrast, differ per state and expose the
   flag semantics. **[observed]**, see [§5](#5-obj-4--the-four-ctrlshifta-byte-captures-plus-contrast-and-flag-sensitivity).

5. **Independence & leakage.** The differing per-state byte captures, plus a 10-cycle rapid-switching
   probe, show the two buffers keep **independent** stacks with **no leakage**.
   **[observed]**, see [§6](#6-obj-5--independence-proof-and-leakage-probe).

6. **Mode-dependent edge cases.** Isolation holds across all three alternate-screen mode constants
   `47`/`1047`/`1049`; `DECCKM` interacts with the flags for cursor keys; and a bare `CSI u` is
   *restore-cursor* (SCORC), not a keyboard-stack operation.
   **[observed]**, see [§7](#7-obj-6--mode-dependent-edge-cases-4710471049-decckm-and-scorc).

## 1. Environment, build, and the canonical observation path

### 1.1 Provenance and toolchain

The following was captured at the repository root on the destination branch. It records the exact
commit under study by its **immutable pinned hash** `815df1e210e0…` — proving, in a way that does not
depend on where the branch HEAD currently sits, that the pinned commit is an ancestor of HEAD and that
the only difference between the two is this document — together with the toolchain versions and the
`pkg-config` state that governs which GLFW backend is built (see [§1.3](#13-wayland-auto-disable-and-why-it-is-irrelevant-here)).

Command:

```bash
set -o pipefail
pinned=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1   # immutable pinned source commit under study
{
  echo "=== repo root (pwd) ==="; pwd
  echo; echo "=== git provenance (anchored to the immutable pinned commit) ==="
  echo "destination/review branch:        $(git rev-parse --abbrev-ref HEAD)"
  echo "current branch HEAD (varies):     $(git rev-parse HEAD)"
  echo "pinned source commit under study: $pinned"
  echo "pinned commit subject:            $(git log -1 --format='%s' "$pinned")"
  if git merge-base --is-ancestor "$pinned" HEAD; then
    echo "pinned is an ancestor of HEAD:    YES (invariant)"
  else
    echo "pinned is an ancestor of HEAD:    NO"
  fi
  echo "diff pinned..HEAD (invariant — only the deliverable appears):"
  git diff --name-status "$pinned" HEAD
  echo; echo "=== toolchain versions ==="
  python3 --version; cc --version | head -1; go version; pkg-config --version
  echo; echo "=== Wayland pkg-config probe (governs GLFW backend selection) ==="
  pkg-config --exists wayland-protocols && echo "wayland-protocols: PRESENT" || echo "wayland-protocols: ABSENT (rc=$?)"
  pkg-config --exists wayland-client && echo "wayland-client: PRESENT" || echo "wayland-client: ABSENT (rc=$?)"
  echo; echo "=== built GLFW backend shared objects present in kitty/ ==="
  ls -1 kitty/glfw-*.so 2>/dev/null || echo "(none yet)"
} 2>&1 | tee /tmp/env.txt
echo "EXIT=${PIPESTATUS[0]}"
```

Output **[observed]**:

```text
=== repo root (pwd) ===
/tmp/blitzy/kitty/blitzy-03f51806-b79e-4da1-b1c1-47a97745c4c7_c64464

=== git provenance (anchored to the immutable pinned commit) ===
destination/review branch:        blitzy-03f51806-b79e-4da1-b1c1-47a97745c4c7
current branch HEAD (varies):     db9bbddeb0de105a7586119924309ab2c8df48b7
pinned source commit under study: 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
pinned commit subject:            Wire up applying of font config
pinned is an ancestor of HEAD:    YES (invariant)
diff pinned..HEAD (invariant — only the deliverable appears):
A	blitzy/documentation/kitty_815df1e210e0.md

=== toolchain versions ===
Python 3.13.7
cc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
go version go1.24.4 linux/amd64
1.8.1

=== Wayland pkg-config probe (governs GLFW backend selection) ===
wayland-protocols: ABSENT (rc=1)
wayland-client: ABSENT (rc=1)

=== built GLFW backend shared objects present in kitty/ ===
kitty/glfw-x11.so
EXIT=0
```

Two provenance facts here are **invariant** — they hold regardless of where the branch HEAD currently
sits, so they survive every future revision of this document: (i) the pinned commit
`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` **is an ancestor of HEAD**, and (ii) the only difference
between the pinned commit and HEAD is the addition of this document (**no kitty source file differs**).
The `current branch HEAD` line above is expected to differ on re-run as the deliverable is revised and
is *not* a reproducibility anchor. With provenance thus pinned, the two facts that matter downstream
are: the code under study is the pinned commit `815df1e210e0…`, and `wayland-protocols` is **absent**
from `pkg-config`, so only the X11 GLFW backend (`kitty/glfw-x11.so`) is built.

### 1.2 Building the C extension (canonical, default configuration)

kitty's behavior lives in a compiled C extension, `kitty/fast_data_types.so`, which is not present in
a fresh checkout and must be built before anything can be observed. The canonical build command is the
default `build` action of `setup.py` ([setup.py:L175]); it applies a strict warning/error set by
default — `-Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes`
together with `-std=c11 -pedantic-errors -Werror` ([setup.py:L491], [setup.py:L500]) — so a clean
build is also a zero-warning guarantee.

To produce a genuine, complete compile transcript, the git-ignored C object cache (`build/`) and the
extension (`kitty/fast_data_types.so`) were removed first, forcing a full recompile. (`setup.py clean`
was deliberately *not* used, because it also wipes the offline Go module cache.)

Command (run at the repository root):

```bash
set -o pipefail
rm -rf build kitty/fast_data_types.so          # remove ONLY git-ignored C build artifacts
CI=true python3 setup.py build --verbose > /tmp/build.txt 2>&1
echo "BUILD_EXIT=$?"
echo "lines: $(wc -l < /tmp/build.txt)  bytes: $(wc -c < /tmp/build.txt)"
echo "sha256: $(sha256sum /tmp/build.txt | cut -d' ' -f1)"
grep -ni 'warning:\|error:' /tmp/build.txt || echo 'NONE (zero warnings/errors)'
```

The complete, unedited transcript follows. All 85 compile lines carry the identical strict
warning/error flag set shown above; the preamble (lines 1–6) records the Wayland auto-disable, and the
final lines link `fast_data_types.so`, `glfw-x11.so`, the `kitty` launcher, and the Go `kitten`
launcher.

Output **[observed]**:

```text
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
CC: ['gcc'] (15, 0)
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
Copyright (C) 2025 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Detected: CompilerType.gcc
gcc -MMD -DNDEBUG -DPRIMARY_VERSION=4000 -DSECONDARY_VERSION=35 -DXT_VERSION="0.35.2" -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/screen.c -o build/fast_data_types-kitty-screen.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/unicode-data.c -o build/fast_data_types-kitty-unicode-data.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/x11_window.c -o build/glfw-x11-glfw-x11_window.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/glfw.c -o build/fast_data_types-kitty-glfw.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/graphics.c -o build/fast_data_types-kitty-graphics.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/child-monitor.c -o build/fast_data_types-kitty-child-monitor.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/fonts.c -o build/fast_data_types-kitty-fonts.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/shaders.c -o build/fast_data_types-kitty-shaders.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/vt-parser.c -o build/fast_data_types-kitty-vt-parser.c.o
gcc -MMD -DNDEBUG -DDUMP_COMMANDS -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/vt-parser.c -o build/fast_data_types-kitty-vt-parser-dump.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/state.c -o build/fast_data_types-kitty-state.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/input.c -o build/glfw-x11-glfw-input.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/mouse.c -o build/fast_data_types-kitty-mouse.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/xkb_glfw.c -o build/glfw-x11-glfw-xkb_glfw.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/freetype.c -o build/fast_data_types-kitty-freetype.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/window.c -o build/glfw-x11-glfw-window.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/line.c -o build/fast_data_types-kitty-line.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/glfw-wrapper.c -o build/fast_data_types-kitty-glfw-wrapper.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -Ikitty -I/usr/include/python3.13 -c kittens/transfer/algorithm.c -o build/rsync-kittens-transfer-algorithm.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/x11_init.c -o build/glfw-x11-glfw-x11_init.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/freetype_render_ui_text.c -o build/fast_data_types-kitty-freetype_render_ui_text.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/egl_context.c -o build/glfw-x11-glfw-egl_context.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/disk-cache.c -o build/fast_data_types-kitty-disk-cache.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/glx_context.c -o build/glfw-x11-glfw-glx_context.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/line-buf.c -o build/fast_data_types-kitty-line-buf.c.o
gcc -MMD -DNDEBUG -DKITTY_VCS_REV="3e8e1ae1b1ec026e8a27f83bf6474817ade3a93f" -DWRAPPED_KITTENS="ask clipboard diff hints hyperlinked_grep icat query_terminal show_key ssh themes transfer unicode_input" -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/data-types.c -o build/fast_data_types-kitty-data-types.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/colors.c -o build/fast_data_types-kitty-colors.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/history.c -o build/fast_data_types-kitty-history.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/keys.c -o build/fast_data_types-kitty-keys.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/x11_monitor.c -o build/glfw-x11-glfw-x11_monitor.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/fontconfig.c -o build/fast_data_types-kitty-fontconfig.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/context.c -o build/glfw-x11-glfw-context.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/crypto.c -o build/fast_data_types-kitty-crypto.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/ibus_glfw.c -o build/glfw-x11-glfw-ibus_glfw.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/key_encoding.c -o build/fast_data_types-kitty-key_encoding.c.o
gcc -DWRAPPED_KITTENS=" ask clipboard diff hints hyperlinked_grep icat query_terminal show_key ssh themes transfer unicode_input " -DFROM_SOURCE -DKITTY_LIB_PATH="../.." -DKITTY_CLI_BOOL_OPTIONS=" detach hold single-instance 1 wait-for-single-instance-window-close version v dump-commands debug-rendering debug-gl debug-input debug-keyboard debug-font-fallback execute e " -DKITTY_VERSION="0.35.2" -Wall -pedantic-errors -Werror -fpie -O3 -I/usr/include/python3.13 -c kitty/launcher/main.c -o build/kitty-launcher-main.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/monitor.c -o build/glfw-x11-glfw-monitor.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/font-names.c -o build/fast_data_types-kitty-font-names.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/backend_utils.c -o build/glfw-x11-glfw-backend_utils.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/charsets.c -o build/fast_data_types-kitty-charsets.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/linux_joystick.c -o build/glfw-x11-glfw-linux_joystick.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/init.c -o build/glfw-x11-glfw-init.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/dbus_glfw.c -o build/glfw-x11-glfw-dbus_glfw.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/gl.c -o build/fast_data_types-kitty-gl.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/vulkan.c -o build/glfw-x11-glfw-vulkan.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/osmesa_context.c -o build/glfw-x11-glfw-osmesa_context.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/cursor.c -o build/fast_data_types-kitty-cursor.c.o
gcc -DWRAPPED_KITTENS=" ask clipboard diff hints hyperlinked_grep icat query_terminal show_key ssh themes transfer unicode_input " -DFROM_SOURCE -DKITTY_LIB_PATH="../.." -DKITTY_CLI_BOOL_OPTIONS=" detach hold single-instance 1 wait-for-single-instance-window-close version v dump-commands debug-rendering debug-gl debug-input debug-keyboard debug-font-fallback execute e " -DKITTY_VERSION="0.35.2" -Wall -pedantic-errors -Werror -fpie -O3 -I/usr/include/python3.13 -c kitty/launcher/single-instance.c -o build/kitty-launcher-single-instance.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/desktop.c -o build/fast_data_types-kitty-desktop.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/loop-utils.c -o build/fast_data_types-kitty-loop-utils.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c 3rdparty/ringbuf/ringbuf.c -o build/fast_data_types-3rdparty-ringbuf-ringbuf.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/simd-string.c -o build/fast_data_types-kitty-simd-string.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/systemd.c -o build/fast_data_types-kitty-systemd.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/shlex.c -o build/fast_data_types-kitty-shlex.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/child.c -o build/fast_data_types-kitty-child.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/kittens.c -o build/fast_data_types-kitty-kittens.c.o
gcc -MMD -DNDEBUG -DHAVE_AVX512=0 -DHAVE_AVX2=1 -DHAVE_NEON32=0 -DHAVE_NEON64=0 -DHAVE_SSSE3=0 -DHAVE_SSE41=1 -DHAVE_SSE42=1 -DHAVE_AVX=1 -DHAVE_SSE3=1 -I3rdparty/base64 -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c 3rdparty/base64/lib/codec_choose.c -o build/fast_data_types-3rdparty-base64-lib-codec_choose.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/png-reader.c -o build/fast_data_types-kitty-png-reader.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/linux_notify.c -o build/glfw-x11-glfw-linux_notify.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/rowcolumn-diacritics.c -o build/fast_data_types-kitty-rowcolumn-diacritics.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/hyperlink.c -o build/fast_data_types-kitty-hyperlink.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/wcswidth.c -o build/fast_data_types-kitty-wcswidth.c.o
gcc -MMD -DNDEBUG -DHAS_COPY_FILE_RANGE -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/fast-file-copy.c -o build/fast_data_types-kitty-fast-file-copy.c.o
gcc -MMD -DNDEBUG -DHAVE_AVX512=0 -DHAVE_AVX2=1 -DHAVE_NEON32=0 -DHAVE_NEON64=0 -DHAVE_SSSE3=0 -DHAVE_SSE41=1 -DHAVE_SSE42=1 -DHAVE_AVX=1 -DHAVE_SSE3=1 -I3rdparty/base64 -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c 3rdparty/base64/lib/lib.c -o build/fast_data_types-3rdparty-base64-lib-lib.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/posix_thread.c -o build/glfw-x11-glfw-posix_thread.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/window_logo.c -o build/fast_data_types-kitty-window_logo.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/glyph-cache.c -o build/fast_data_types-kitty-glyph-cache.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/logging.c -o build/fast_data_types-kitty-logging.c.o
gcc -MMD -DNDEBUG -DHAVE_AVX512=0 -DHAVE_AVX2=1 -DHAVE_NEON32=0 -DHAVE_NEON64=0 -DHAVE_SSSE3=0 -DHAVE_SSE41=1 -DHAVE_SSE42=1 -DHAVE_AVX=1 -DHAVE_SSE3=1 -I3rdparty/base64 -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c 3rdparty/base64/lib/arch/neon64/codec.c -o build/fast_data_types-3rdparty-base64-lib-arch-neon64-codec.c.o
gcc -MMD -DNDEBUG -DHAVE_AVX512=0 -DHAVE_AVX2=1 -DHAVE_NEON32=0 -DHAVE_NEON64=0 -DHAVE_SSSE3=0 -DHAVE_SSE41=1 -DHAVE_SSE42=1 -DHAVE_AVX=1 -DHAVE_SSE3=1 -I3rdparty/base64 -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c 3rdparty/base64/lib/tables/tables.c -o build/fast_data_types-3rdparty-base64-lib-tables-tables.c.o
gcc -MMD -DNDEBUG -DHAVE_AVX512=0 -DHAVE_AVX2=1 -DHAVE_NEON32=0 -DHAVE_NEON64=0 -DHAVE_SSSE3=0 -DHAVE_SSE41=1 -DHAVE_SSE42=1 -DHAVE_AVX=1 -DHAVE_SSE3=1 -I3rdparty/base64 -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c 3rdparty/base64/lib/arch/neon32/codec.c -o build/fast_data_types-3rdparty-base64-lib-arch-neon32-codec.c.o
gcc -MMD -DNDEBUG -DHAVE_AVX512=0 -DHAVE_AVX2=1 -DHAVE_NEON32=0 -DHAVE_NEON64=0 -DHAVE_SSSE3=0 -DHAVE_SSE41=1 -DHAVE_SSE42=1 -DHAVE_AVX=1 -DHAVE_SSE3=1 -I3rdparty/base64 -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -mavx -c 3rdparty/base64/lib/arch/avx/codec.c -o build/fast_data_types-3rdparty-base64-lib-arch-avx-codec.c.o
gcc -MMD -DNDEBUG -DHAVE_AVX512=0 -DHAVE_AVX2=1 -DHAVE_NEON32=0 -DHAVE_NEON64=0 -DHAVE_SSSE3=0 -DHAVE_SSE41=1 -DHAVE_SSE42=1 -DHAVE_AVX=1 -DHAVE_SSE3=1 -I3rdparty/base64 -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c 3rdparty/base64/lib/arch/ssse3/codec.c -o build/fast_data_types-3rdparty-base64-lib-arch-ssse3-codec.c.o
gcc -MMD -DNDEBUG -DHAVE_AVX512=0 -DHAVE_AVX2=1 -DHAVE_NEON32=0 -DHAVE_NEON64=0 -DHAVE_SSSE3=0 -DHAVE_SSE41=1 -DHAVE_SSE42=1 -DHAVE_AVX=1 -DHAVE_SSE3=1 -I3rdparty/base64 -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -msse4.2 -c 3rdparty/base64/lib/arch/sse42/codec.c -o build/fast_data_types-3rdparty-base64-lib-arch-sse42-codec.c.o
gcc -MMD -DNDEBUG -DHAVE_AVX512=0 -DHAVE_AVX2=1 -DHAVE_NEON32=0 -DHAVE_NEON64=0 -DHAVE_SSSE3=0 -DHAVE_SSE41=1 -DHAVE_SSE42=1 -DHAVE_AVX=1 -DHAVE_SSE3=1 -I3rdparty/base64 -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -msse4.1 -c 3rdparty/base64/lib/arch/sse41/codec.c -o build/fast_data_types-3rdparty-base64-lib-arch-sse41-codec.c.o
gcc -MMD -DNDEBUG -DHAVE_AVX512=0 -DHAVE_AVX2=1 -DHAVE_NEON32=0 -DHAVE_NEON64=0 -DHAVE_SSSE3=0 -DHAVE_SSE41=1 -DHAVE_SSE42=1 -DHAVE_AVX=1 -DHAVE_SSE3=1 -I3rdparty/base64 -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -mavx2 -c 3rdparty/base64/lib/arch/avx2/codec.c -o build/fast_data_types-3rdparty-base64-lib-arch-avx2-codec.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/utmp.c -o build/fast_data_types-kitty-utmp.c.o
gcc -MMD -DNDEBUG -DHAVE_AVX512=0 -DHAVE_AVX2=1 -DHAVE_NEON32=0 -DHAVE_NEON64=0 -DHAVE_SSSE3=0 -DHAVE_SSE41=1 -DHAVE_SSE42=1 -DHAVE_AVX=1 -DHAVE_SSE3=1 -I3rdparty/base64 -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c 3rdparty/base64/lib/arch/avx512/codec.c -o build/fast_data_types-3rdparty-base64-lib-arch-avx512-codec.c.o
gcc -MMD -DNDEBUG -DHAVE_AVX512=0 -DHAVE_AVX2=1 -DHAVE_NEON32=0 -DHAVE_NEON64=0 -DHAVE_SSSE3=0 -DHAVE_SSE41=1 -DHAVE_SSE42=1 -DHAVE_AVX=1 -DHAVE_SSE3=1 -I3rdparty/base64 -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c 3rdparty/base64/lib/arch/generic/codec.c -o build/fast_data_types-3rdparty-base64-lib-arch-generic-codec.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/cleanup.c -o build/fast_data_types-kitty-cleanup.c.o
gcc -MMD -DNDEBUG -D_GLFW_X11 -D_GLFW_BUILD_DLL -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/monotonic.c -o build/glfw-x11-glfw-monotonic.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/monotonic.c -o build/fast_data_types-kitty-monotonic.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -fopenmp-simd -DSIMDE_ENABLE_OPENMP -msse4.2 -c kitty/simd-string-128.c -o build/fast_data_types-kitty-simd-string-128.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -fopenmp-simd -DSIMDE_ENABLE_OPENMP -mavx2 -mno-vzeroupper -c kitty/simd-string-256.c -o build/fast_data_types-kitty-simd-string-256.c.o
gcc -MMD -DNDEBUG -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -pthread -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -c kitty/gl-wrapper.c -o build/fast_data_types-kitty-gl-wrapper.c.o
gcc -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -Wall -O3 -shared -flto build/fast_data_types-kitty-charsets.c.o build/fast_data_types-kitty-child-monitor.c.o build/fast_data_types-kitty-child.c.o build/fast_data_types-kitty-cleanup.c.o build/fast_data_types-kitty-colors.c.o build/fast_data_types-kitty-crypto.c.o build/fast_data_types-kitty-cursor.c.o build/fast_data_types-kitty-data-types.c.o build/fast_data_types-kitty-desktop.c.o build/fast_data_types-kitty-disk-cache.c.o build/fast_data_types-kitty-fast-file-copy.c.o build/fast_data_types-kitty-font-names.c.o build/fast_data_types-kitty-fontconfig.c.o build/fast_data_types-kitty-fonts.c.o build/fast_data_types-kitty-freetype.c.o build/fast_data_types-kitty-freetype_render_ui_text.c.o build/fast_data_types-kitty-gl-wrapper.c.o build/fast_data_types-kitty-gl.c.o build/fast_data_types-kitty-glfw-wrapper.c.o build/fast_data_types-kitty-glfw.c.o build/fast_data_types-kitty-glyph-cache.c.o build/fast_data_types-kitty-graphics.c.o build/fast_data_types-kitty-history.c.o build/fast_data_types-kitty-hyperlink.c.o build/fast_data_types-kitty-key_encoding.c.o build/fast_data_types-kitty-keys.c.o build/fast_data_types-kitty-kittens.c.o build/fast_data_types-kitty-line-buf.c.o build/fast_data_types-kitty-line.c.o build/fast_data_types-kitty-logging.c.o build/fast_data_types-kitty-loop-utils.c.o build/fast_data_types-kitty-monotonic.c.o build/fast_data_types-kitty-mouse.c.o build/fast_data_types-kitty-png-reader.c.o build/fast_data_types-kitty-rowcolumn-diacritics.c.o build/fast_data_types-kitty-screen.c.o build/fast_data_types-kitty-shaders.c.o build/fast_data_types-kitty-shlex.c.o build/fast_data_types-kitty-simd-string-128.c.o build/fast_data_types-kitty-simd-string-256.c.o build/fast_data_types-kitty-simd-string.c.o build/fast_data_types-kitty-state.c.o build/fast_data_types-kitty-systemd.c.o build/fast_data_types-kitty-unicode-data.c.o build/fast_data_types-kitty-utmp.c.o build/fast_data_types-kitty-vt-parser.c.o build/fast_data_types-kitty-wcswidth.c.o build/fast_data_types-kitty-window_logo.c.o build/fast_data_types-kitty-vt-parser-dump.c.o build/fast_data_types-3rdparty-ringbuf-ringbuf.c.o build/fast_data_types-3rdparty-base64-lib-arch-neon32-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-sse42-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-ssse3-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-sse41-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-generic-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-avx2-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-avx512-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-avx-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-neon64-codec.c.o build/fast_data_types-3rdparty-base64-lib-tables-tables.c.o build/fast_data_types-3rdparty-base64-lib-codec_choose.c.o build/fast_data_types-3rdparty-base64-lib-lib.c.o -ldl -lm -L/usr/lib/x86_64-linux-gnu -lpython3.13 -Xlinker -export-dynamic -Wl,-O1 -Wl,-Bsymbolic-functions -lharfbuzz -lGL -lpng16 -llcms2 -llcms2_fast_float -llcms2_threaded -pthread -lm -lcrypto -lrt -lz -o build/kitty/fast_data_types.so
gcc -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -Wall -O3 -shared -flto build/glfw-x11-glfw-context.c.o build/glfw-x11-glfw-init.c.o build/glfw-x11-glfw-input.c.o build/glfw-x11-glfw-monitor.c.o build/glfw-x11-glfw-vulkan.c.o build/glfw-x11-glfw-monotonic.c.o build/glfw-x11-glfw-window.c.o build/glfw-x11-glfw-x11_init.c.o build/glfw-x11-glfw-x11_monitor.c.o build/glfw-x11-glfw-x11_window.c.o build/glfw-x11-glfw-xkb_glfw.c.o build/glfw-x11-glfw-dbus_glfw.c.o build/glfw-x11-glfw-ibus_glfw.c.o build/glfw-x11-glfw-posix_thread.c.o build/glfw-x11-glfw-glx_context.c.o build/glfw-x11-glfw-egl_context.c.o build/glfw-x11-glfw-osmesa_context.c.o build/glfw-x11-glfw-backend_utils.c.o build/glfw-x11-glfw-linux_joystick.c.o build/glfw-x11-glfw-linux_notify.c.o -pthread -lm -lrt -ldl -lX11 -lXrandr -lXinerama -lXcursor -lxkbcommon -lxkbcommon-x11 -lxkbcommon -lX11-xcb -lX11 -lxcb -ldbus-1 -o build/kitty/glfw-x11.so
gcc -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -Ikitty -I/usr/include/python3.13 -Wall -O3 -shared -flto build/rsync-kittens-transfer-algorithm.c.o -lxxhash -ldl -lm -L/usr/lib/x86_64-linux-gnu -lpython3.13 -Xlinker -export-dynamic -Wl,-O1 -Wl,-Bsymbolic-functions -o build/kittens/transfer/rsync.so
gcc build/kitty-launcher-main.o build/kitty-launcher-single-instance.o -ldl -lm -L/usr/lib/x86_64-linux-gnu -lpython3.13 -Xlinker -export-dynamic -Wl,-O1 -Wl,-Bsymbolic-functions -o kitty/launcher/kitty
Updating Go generated files...
kitty/tools/cmd
/usr/bin/go build -v -ldflags '-X kitty.VCSRevision=3e8e1ae1b1ec026e8a27f83bf6474817ade3a93f -s -w' -o kitty/launcher/kitten /tmp/blitzy/kitty/blitzy-03f51806-b79e-4da1-b1c1-47a97745c4c7_c64464/tools/cmd
```

Build summary **[observed]**: `BUILD_EXIT=0`, and the `grep` for `warning:`/`error:` returned
**NONE (zero warnings/errors)** — with the strict `-pedantic-errors -Werror` set applied by default,
a clean build is a zero-warning guarantee.

The transcript above is a genuine capture, but **two of its lines are run/commit-specific and are
deliberately *not* used as a reproducibility anchor**:

- the `KITTY_VCS_REV="…"` field on the `kitty/data-types.c` compile line and the matching
  `-X kitty.VCSRevision=…` field on the Go `kitten` build line both record the **checkout's git HEAD
  at build time**. (This transcript was captured at an earlier documentation commit on the branch, so
  those fields show that build-time HEAD — `3e8e1ae1b…` — not the current branch HEAD.)
- the `go build -v` progress line(s) printed under `Updating Go generated files…` depend on the **Go
  build-cache** warmth: a warm cache prints fewer package lines than a cold one.

Because of the embedded `KITTY_VCS_REV`, the raw `sha256` of the **whole transcript** *and* the raw
`sha256` of `kitty/fast_data_types.so` are only reproducible **at a fixed checkout**, not across
commits — rebuilding at `815df1e21…` versus a later documentation commit changes only the embedded
40-hex revision. (For the record, the whole-transcript `sha256` of the capture above was
`841d09404c5d0e29f39ee985401d19cc49ad7a22e2fe1f36b9d75c235bc7ca11` at 104 lines *at capture time*; a
fresh clean rebuild on a warm Go cache yields a different whole-transcript hash and 103 lines — as
expected, and precisely why that hash is not used as an anchor.) The **stable, reproducible anchors** —
the values a normal re-runner should actually check — are:

1. `BUILD_EXIT=0`;
2. zero `warning:`/`error:` lines (the `-Werror` guarantee);
3. `kitty/fast_data_types.so` links, imports, and passes the behavior sanity check
   (`current_key_encoding_flags() == 0`, `toggle_alt_screen` present — see the *Post-build
   verification* block below);
4. the artifact **size `1253792` bytes** (stable across commits because `KITTY_VCS_REV` is always a
   fixed-width 40-hex string, so swapping one revision for another does not change the byte count); and
5. a **revision-masked hash of the invariant compile + link commands**, which *is* byte-for-byte
   reproducible across runs and across commits.

Anchor (5) keeps only the `gcc` compile/link lines and masks the sole per-commit token among them
(the 40-hex `KITTY_VCS_REV`); none of these lines contains a host-specific path, so the result is
portable:

```bash
grep '^gcc ' /tmp/build.txt | sed 's/[0-9a-f]\{40\}/<REV>/g' | sha256sum
```

Output **[observed]** (identical on two consecutive clean rebuilds, and identical to the transcript
above once masked):

```text
6921c2615358aa6551b95feeb8b48241c19a662f883847216dc4369579bc1a06  -
```

That normalized hash matches byte-for-byte between the transcript shown above (captured at an earlier
commit) and a fresh clean rebuild at the current HEAD, confirming the **90 `gcc` compile/link commands
are invariant**; only the embedded revision and the Go-cache progress line differ between runs.

Post-build verification **[observed]**:

```text
$ ls -l kitty/fast_data_types.so
-rwxr-xr-x 1 root root 1253792 kitty/fast_data_types.so
$ PYTHONPATH="$(pwd)" python3 -c "import kitty.fast_data_types as f; from kitty_tests import Callbacks; c=Callbacks(); s=f.Screen(c,5,40,5,10,20,0,c); print('import OK; flags =', s.current_key_encoding_flags(), '; toggle_alt_screen present =', hasattr(s,'toggle_alt_screen'))"
import OK; flags = 0 ; toggle_alt_screen present = True
$ git status --porcelain
(empty — all build outputs are git-ignored, so the working tree stays clean)
```

### 1.3 Wayland auto-disable and why it is irrelevant here

The build transcript opens with:

```text
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
```

`wayland-protocols` is not installed in this image (the `pkg-config` probe in [§1.1](#11-provenance-and-toolchain)
reports it **ABSENT**), so `setup.py` follows its default behavior and **disables the Wayland GLFW
backend**, building only the X11 backend `kitty/glfw-x11.so`. This is a property of the *default*
configuration as a normal user would build it in this environment — nothing was forced or overridden.

This has **no bearing on the investigation**. The keyboard-protocol stack and the key encoder live
entirely in `kitty/fast_data_types.so`; the GLFW backends (`glfw-x11.so`, and the absent
`glfw-wayland.so`) are the windowing/display layer. The headless observation path used below
([§1.4](#14-the-canonical-observation-path-headless-and-the-observedinferred-boundary)) imports only
`fast_data_types.so` and never opens a window, so whether the Wayland backend is present or absent
cannot change any byte reported here.

### 1.4 The canonical observation path (headless) and the observed/inferred boundary

An application controls kitty's keyboard protocol by **writing escape codes**, which kitty's VT parser
parses and dispatches into the screen's per-buffer stack routines. The sanctioned headless harness in
`kitty_tests/__init__.py` drives *that same real parser*:

- `parse_bytes(screen, data)` ([kitty_tests/__init__.py:L30]) feeds bytes through the real VT parser
  (`test_create_write_buffer` / `test_commit_write_buffer` / `test_parse_written_data`), exactly as
  bytes arriving from an application would be parsed.
- `Callbacks.write` accumulates everything kitty writes **to the child** into `wtcbuf`
  ([kitty_tests/__init__.py:L51]); `Callbacks.clear()` resets it ([kitty_tests/__init__.py:L95]). This
  captures the real child-bound bytes (e.g. the reply to a flags *query*).
- `create_screen` ([kitty_tests/__init__.py:L237]) builds the `Screen` with
  `Screen(callbacks, lines, cols, scrollback, cell_width, cell_height, 0, test_child)`; the scripts
  below use that exact construction.

Two distinct channels of evidence, with an explicit boundary between what is observed and what is
inferred:

- **Flag state — fully canonical & [observed].** The active flags are read from the real per-buffer
  stack via `Screen.current_key_encoding_flags()` (C: `screen_current_key_encoding_flags`,
  [kitty/screen.c:L1204]), and the *query reply* is the real child-bound write produced by
  `screen_report_key_encoding_flags` ([kitty/screen.c:L1212-L1215], which formats `?%uu`). Both are
  produced by the compiled code at runtime.
- **Key bytes — real encoder, [observed] output; live-GUI equivalence [inferred].** The bytes a key
  press produces are computed by the C function `encode_glfw_key_event`
  ([kitty/key_encoding.c:L414]). On the live path this is called from `on_key_input`
  with the *active stack flags*:
  `encode_glfw_key_event(ev, screen->modes.mDECCKM, screen_current_key_encoding_flags(screen), encoded_key)`
  ([kitty/keys.c:L251]). Headlessly there is no GLFW key event object, so the scripts invoke the very
  same encoder through its Python export, `encode_key_for_tty`, **passing the flags they just read
  from the real stack**. The returned string is converted with `.encode('ascii')` — exactly as the
  live GUI does in `Window.encoded_key`, which calls
  `encode_key_for_tty(..., key_encoding_flags=self.screen.current_key_encoding_flags(), cursor_key_mode=self.screen.cursor_key_mode).encode('ascii')` ([kitty/window.py:L1795-L1801]).
  The encoder output printed below is therefore **[observed]**; the claim that a *live* Ctrl+Shift+a
  keypress in a GUI window would write these identical bytes to the child is **[inferred]** from
  [kitty/keys.c:L251] and [kitty/window.py:L1795-L1801] (this headless environment has no display, so
  a live keypress→child capture via `kitty --debug-keyboard` [kitty/keys.c:L260-L269] or
  `kitten show-key -m kitty` [kittens/show_key/main.py:L12-L15] was not performed).

Because the *encoder* is the production function and the *flags fed to it are read from the real
stack immediately before each call*, every encoder capture below is a **stack-confirmed,
production-equivalent cross-check** — not a value conjured from an arbitrary flags argument. The one
exception is the deliberately labelled *encoder-characterization* block in
[§5.3](#53-flag-sensitivity-explicit-non-canonical-encoder-characterization), which feeds **explicit
flag literals** (not stack-derived) purely to isolate the effect of one flag bit; it proves encoder
behavior, not stack state, and is marked **non-canonical** accordingly.

### 1.5 The `CSI u` escape-code grammar (and the SCORC confusion)

All stack operations share the final byte `u` and are disambiguated by the **leading** modifier byte.
The dispatch is in the VT parser ([kitty/vt-parser.c:L1217-L1240]):

| Escape code | Meaning | Dispatch |
|-------------|---------|----------|
| `CSI u` (no modifier, no params) | **Restore cursor (SCORC)** — *not* a keyboard op | `screen_restore_cursor` [kitty/vt-parser.c:L1218-L1221] |
| `CSI ? u` | Query current flags (writes reply to child) | `screen_report_key_encoding_flags` [kitty/vt-parser.c:L1223-L1226] |
| `CSI = flags ; mode u` | Set flags (mode 1=set, 2=or, 3=and-not) | `screen_set_key_encoding_flags` [kitty/vt-parser.c:L1228-L1230], [kitty/screen.c:L1220] |
| `CSI > flags u` | Push flags onto the active stack | `screen_push_key_encoding_flags` [kitty/vt-parser.c:L1232-L1234], [kitty/screen.c:L1234] |
| `CSI < number u` | Pop `number` entries from the active stack | `screen_pop_key_encoding_flags` [kitty/vt-parser.c:L1236-L1238], [kitty/screen.c:L1248] |

The first row is a well-known confusion source: a bare `CSI u` with no modifier is the ancient
*restore cursor* operation, entirely unrelated to the keyboard protocol. This is demonstrated at
runtime in [§7.3](#73-scorc-a-bare-csi-u-restores-the-cursor-not-the-keyboard-stack).

## 2. (OBJ-1) Round-trip stack survival

**Question.** Starting on the main buffer, push keyboard flags, toggle to the alternate screen, push
different flags, then toggle back to the main buffer: which mode is active at the end, does the
main-buffer stack survive intact, and what does a flags *query* return in each intermediate state?

**Mechanism (source).** kitty stores **two** 8-slot flag arrays plus one active pointer —
`uint8_t main_key_encoding_flags[8], alt_key_encoding_flags[8], *key_encoding_flags;`
([kitty/screen.h:L128]). Switching buffers calls `screen_toggle_screen_buffer`, which only **re-points**
that active pointer — to the alternate array on entry ([kitty/screen.c:L1079]) and back to the main
array on exit ([kitty/screen.c:L1086]) — and never copies bytes between the two arrays. So the
main-buffer stack is expected to be exactly as it was left.

The script below is fully self-contained and safe: it creates a private temp dir with `mktemp -d`
under `umask 077`, writes its observation program there, runs only that program, and removes the dir
via a `trap` on exit.

Script (reproducible):

```bash
#!/usr/bin/env bash
# OBJ-1 + OBJ-5(query-byte independence): main -> push flags -> alt -> push flags -> back to main.
# Flag state is read CANONICALLY from the real per-buffer stack after driving the REAL VT parser.
set -o pipefail
umask 077
repo="${1:-$(git rev-parse --show-toplevel)}"   # portable: repo root from $1, else auto-detect via git
tmp="$(mktemp -d "${TMPDIR:-/tmp}/kbdobs.XXXXXXXX")"
trap 'rm -rf "$tmp"' EXIT
cat > "$tmp/obs.py" <<'PYEOF'
import kitty.fast_data_types as f
from kitty_tests import parse_bytes, Callbacks

def new_screen():
    c = Callbacks()
    return f.Screen(c, 5, 40, 5, 10, 20, 0, c), c

def active(s):                 # canonical: low-7-bits of highest occupied slot of the ACTIVE stack
    return s.current_key_encoding_flags()

def query_to_child(s, c):      # canonical: real parser emits flags report; capture child-bound bytes
    c.clear(); parse_bytes(s, b'\x1b[?u'); return bytes(c.wtcbuf)

s, c = new_screen()
print("step 0  start on MAIN, nothing pushed        active=%d  query->child=%r" % (active(s), query_to_child(s, c)))
parse_bytes(s, b'\x1b[>1u')    # CSI > 1 u  == push 'disambiguate' (flag 1) onto the active(main) stack
print("step 1  MAIN push disambiguate (CSI >1u)     active=%d  query->child=%r" % (active(s), query_to_child(s, c)))
parse_bytes(s, b'\x1b[?1049h') # DECSET 1049 == switch to ALTERNATE screen (re-points active stack ptr)
print("step 2  toggle to ALT (CSI ?1049h)           active=%d  query->child=%r" % (active(s), query_to_child(s, c)))
parse_bytes(s, b'\x1b[>8u')    # CSI > 8 u  == push 'report all keys' (flag 8) onto the active(alt) stack
print("step 3  ALT push report-all-keys (CSI >8u)   active=%d  query->child=%r" % (active(s), query_to_child(s, c)))
parse_bytes(s, b'\x1b[?1049l') # DECRST 1049 == switch back to MAIN screen (re-points active stack ptr)
print("step 4  toggle back to MAIN (CSI ?1049l)     active=%d  query->child=%r" % (active(s), query_to_child(s, c)))
print("RESULT  final active keyboard-encoding flags on MAIN = %d" % active(s))
PYEOF
PYTHONPATH="$repo" python3 "$tmp/obs.py"
rc=$?
echo "EXIT=$rc"
exit $rc
```

Output **[observed]**:

```text
step 0  start on MAIN, nothing pushed        active=0  query->child=b'\x1b[?0u'
step 1  MAIN push disambiguate (CSI >1u)     active=1  query->child=b'\x1b[?1u'
step 2  toggle to ALT (CSI ?1049h)           active=0  query->child=b'\x1b[?0u'
step 3  ALT push report-all-keys (CSI >8u)   active=8  query->child=b'\x1b[?8u'
step 4  toggle back to MAIN (CSI ?1049l)     active=1  query->child=b'\x1b[?1u'
RESULT  final active keyboard-encoding flags on MAIN = 1
EXIT=0
```

**Interpretation.** The active flags trace `0 → 1` (main push) `→ 0` (fresh alt) `→ 8` (alt push)
`→ 1` (back on main). The final active keyboard-encoding mode on the main buffer is **`1`
(disambiguate)** — the main-buffer stack **survived the round trip intact**, because the toggle only
re-pointed the active pointer ([kitty/screen.c:L1079], [kitty/screen.c:L1086]) and the alternate
buffer's push (flag `8`) landed in the *separate* `alt_key_encoding_flags` array. The `query->child`
bytes are the real replies written to the child by `screen_report_key_encoding_flags`
([kitty/screen.c:L1212-L1215]); they read `b'\x1b[?1u'`, `b'\x1b[?0u'`, `b'\x1b[?8u'`, `b'\x1b[?1u'`
respectively — a direct, child-visible confirmation of the same trace. **[observed]**

## 3. (OBJ-2) Stack exhaustion on both buffers, and cross-buffer isolation

**Question.** What happens when pushes exceed the stack limit — silent drop, error, or otherwise — and
does exhausting one buffer's stack affect the other's?

**Mechanism (source).** Each array is 8 slots ([kitty/screen.h:L128]); a slot's high bit `0x80` marks
it *occupied* and the current flags are the low 7 bits (`& 0x7f`) of the **highest** occupied slot
(`screen_current_key_encoding_flags`, [kitty/screen.c:L1204-L1206]). `screen_push_key_encoding_flags`
([kitty/screen.c:L1234]) writes the new value at the next slot; **when the top slot is already the
last one (index 7), it `memmove`s the whole array down by one, silently discarding the oldest entry**
([kitty/screen.c:L1241]) — no error is raised. All of this acts through the *active* pointer, so it
targets whichever buffer is current.

The script exercises exhaustion on **both** buffers (push values `1..12` — four more than capacity —
then pop 12 times), and then a dedicated isolation check.

Script (reproducible):

```bash
#!/usr/bin/env bash
# OBJ-2: 8-slot stack exhaustion (oldest silently evicted, no error) on BOTH buffers, plus isolation.
set -o pipefail
umask 077
repo="${1:-$(git rev-parse --show-toplevel)}"   # portable: repo root from $1, else auto-detect via git
tmp="$(mktemp -d "${TMPDIR:-/tmp}/kbdobs.XXXXXXXX")"
trap 'rm -rf "$tmp"' EXIT
cat > "$tmp/obs.py" <<'PYEOF'
import kitty.fast_data_types as f
from kitty_tests import parse_bytes, Callbacks

def new_screen():
    c = Callbacks()
    return f.Screen(c, 5, 40, 5, 10, 20, 0, c), c

def active(s):
    return s.current_key_encoding_flags()

def exhaust(label, s):
    pushed = []
    for v in range(1, 13):                 # push flag values 1..12 (12 > 8-slot capacity)
        parse_bytes(s, b'\x1b[>%du' % v)   # CSI > v u  == push value v onto active stack
        pushed.append(active(s))
    popped = []
    for _ in range(12):                    # pop 12 times (more than were retained)
        parse_bytes(s, b'\x1b[<1u')        # CSI < 1 u  == pop one entry from active stack
        popped.append(active(s))
    print("%s active-after-each-push (v=1..12): %s" % (label, pushed))
    print("%s active-after-each-pop  (12 pops): %s" % (label, popped))

# --- exhaustion on the MAIN buffer ---
s, c = new_screen()
exhaust("MAIN", s)

# --- exhaustion on the ALT buffer (toggle first, so the ACTIVE stack is the alt array) ---
s, c = new_screen()
parse_bytes(s, b'\x1b[?1049h')             # switch to ALT; active stack ptr -> alt array
print("ALT  entered alternate screen; active=%d" % active(s))
exhaust("ALT ", s)

# --- cross-buffer isolation: exhausting one buffer must not disturb the other ---
s, c = new_screen()
parse_bytes(s, b'\x1b[>1u')                # MAIN: push 1
print("ISO  MAIN after push1                 active=%d" % active(s))
parse_bytes(s, b'\x1b[?1049h')             # -> ALT (independent, empty)
print("ISO  entered ALT                      active=%d" % active(s))
for v in range(1, 13):
    parse_bytes(s, b'\x1b[>%du' % v)       # ALT: push 1..12 (exhaust the alt stack)
print("ISO  ALT after pushing 1..12          active=%d" % active(s))
parse_bytes(s, b'\x1b[?1049l')             # -> back to MAIN
print("ISO  back on MAIN (unchanged?)        active=%d" % active(s))
PYEOF
PYTHONPATH="$repo" python3 "$tmp/obs.py"
rc=$?
echo "EXIT=$rc"
exit $rc
```

Output **[observed]**:

```text
MAIN active-after-each-push (v=1..12): [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]
MAIN active-after-each-pop  (12 pops): [11, 10, 9, 8, 7, 6, 5, 0, 0, 0, 0, 0]
ALT  entered alternate screen; active=0
ALT  active-after-each-push (v=1..12): [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]
ALT  active-after-each-pop  (12 pops): [11, 10, 9, 8, 7, 6, 5, 0, 0, 0, 0, 0]
ISO  MAIN after push1                 active=1
ISO  entered ALT                      active=0
ISO  ALT after pushing 1..12          active=12
ISO  back on MAIN (unchanged?)        active=1
EXIT=0
```

**Interpretation.**

- *Exhaustion is silent and identical on both buffers.* On the MAIN buffer the pushes `1..12` make the
  active value climb `1..12`, and the 12 pops read back `[11, 10, 9, 8, 7, 6, 5, 0, 0, 0, 0, 0]`. The
  ALT buffer, exercised the same way after `CSI ?1049h`, produces the **identical** push and pop
  sequences. Only the most-recent **8** values (`5..12`) were retained; the older values (`1..4`, and
  the base-0 seed discussed in [§4](#4-obj-3--pop-to-empty-reset-first-push-from-empty-and-over-pop))
  were evicted by the `memmove` at [kitty/screen.c:L1241], and **no error was raised** at any point
  (the program ran to `EXIT=0`).

- *Why the first pop reads `11`, not `12`.* This is ordinary LIFO behavior, **not** an artifact of the
  base-0 seeding: `screen_pop_key_encoding_flags` clears the highest occupied slot
  ([kitty/screen.c:L1248-L1252]), so the first pop removes the just-pushed top value `12`, revealing
  the next entry `11`. Popping continues to reveal `10, 9, 8, 7, 6, 5`; the eighth pop clears the last
  occupied slot, at which point `screen_current_key_encoding_flags` finds no occupied slot and returns
  `0` — that drain-to-`0` (not the first pop) is where the reset/seed behavior shows up.

- *Isolation holds.* In the `ISO` block, the main buffer is left holding `1`; entering the alternate
  buffer shows a fresh `0`; pushing `1..12` on the alternate buffer drives its active value to `12`;
  and returning to the main buffer shows **`1` again**, wholly unaffected by the alternate buffer's
  exhaustion. Exhausting one buffer's stack therefore does not touch the other's, exactly as the two
  separate arrays ([kitty/screen.h:L128]) predict.

## 4. (OBJ-3) Pop-to-empty reset, first-push-from-empty, and over-pop

**Question.** Does a pop that empties the stack reset all flags, as the specification mandates?

**Mechanism (source).** The specification requires that *"if a pop request is received that empties the
stack, all flags are reset"* ([docs/keyboard-protocol.rst:L301-L302]).
`screen_pop_key_encoding_flags` implements this by clearing (setting to `0`) each occupied slot it
pops ([kitty/screen.c:L1248-L1252]); once every slot is clear, `screen_current_key_encoding_flags`
finds nothing occupied and returns `0` ([kitty/screen.c:L1204-L1206]). Separately, the very first push
onto an **empty** stack seeds a base entry: in `screen_push_key_encoding_flags`
([kitty/screen.c:L1234-L1243]) the scan for the highest occupied slot leaves `current_idx = 0` when
none is occupied, so the `else` branch marks slot 0 occupied *with value 0* and advances, and the
pushed value is written at slot 1. That base-0 seed is why a stack that is pushed once and then popped
once returns to `0` rather than underflowing.

Script (reproducible):

```bash
#!/usr/bin/env bash
# OBJ-3: a pop that empties the stack resets all flags; first-push-from-empty seeds a base-0 entry;
#         over-popping does not underflow/error (stays at 0).
set -o pipefail
umask 077
repo="${1:-$(git rev-parse --show-toplevel)}"   # portable: repo root from $1, else auto-detect via git
tmp="$(mktemp -d "${TMPDIR:-/tmp}/kbdobs.XXXXXXXX")"
trap 'rm -rf "$tmp"' EXIT
cat > "$tmp/obs.py" <<'PYEOF'
import kitty.fast_data_types as f
from kitty_tests import parse_bytes, Callbacks

def new_screen():
    c = Callbacks()
    return f.Screen(c, 5, 40, 5, 10, 20, 0, c), c

def active(s):
    return s.current_key_encoding_flags()

# (A) first-push-from-empty, then pop back to empty
s, c = new_screen()
print("A0 empty stack                 active=%d" % active(s))
parse_bytes(s, b'\x1b[>5u')            # push flag value 5 onto the empty stack
print("A1 after push(5)               active=%d" % active(s))
parse_bytes(s, b'\x1b[<1u')            # pop once
print("A2 after 1 pop                 active=%d" % active(s))
parse_bytes(s, b'\x1b[<1u')            # pop again (already empty)
print("A3 after 2 pops                active=%d" % active(s))

# (B) over-pop after several pushes: push 1,2,4 then pop 6 times
s, c = new_screen()
for v in (1, 2, 4):
    parse_bytes(s, b'\x1b[>%du' % v)
print("B0 after push 1,2,4            active=%d" % active(s))
seq = []
for _ in range(6):
    parse_bytes(s, b'\x1b[<1u')
    seq.append(active(s))
print("B1 active-after-each-of-6-pops %s" % seq)

# (C) pop-to-empty via SET then explicit reset semantics: set flags=7, then pop to empty
s, c = new_screen()
parse_bytes(s, b'\x1b[=7;1u')          # CSI = 7 ; 1 u  == SET flags to 7 (mode 1 = set all bits in value)
print("C0 after set flags=7 (CSI =7;1u) active=%d" % active(s))
parse_bytes(s, b'\x1b[<3u')            # pop 3 (more than present) -> empties -> reset
print("C1 after pop 3 (CSI <3u)         active=%d" % active(s))
PYEOF
PYTHONPATH="$repo" python3 "$tmp/obs.py"
rc=$?
echo "EXIT=$rc"
exit $rc
```

Output **[observed]**:

```text
A0 empty stack                 active=0
A1 after push(5)               active=5
A2 after 1 pop                 active=0
A3 after 2 pops                active=0
B0 after push 1,2,4            active=4
B1 active-after-each-of-6-pops [2, 1, 0, 0, 0, 0]
C0 after set flags=7 (CSI =7;1u) active=7
C1 after pop 3 (CSI <3u)         active=0
EXIT=0
```

**Interpretation.**

- *(A) first-push-from-empty + pop-to-empty.* The empty stack reads `0`; after `CSI >5u` the active
  value is `5`; a single pop returns to **`0`** (revealing the seeded base-0 entry), and a second pop
  stays `0`. This is the pop-to-empty reset mandated by [docs/keyboard-protocol.rst:L301-L302].

- *(B) over-pop does not underflow.* After pushing `1, 2, 4` the active value is `4`; six successive
  pops read `[2, 1, 0, 0, 0, 0]` — the third pop reveals the seeded base-0, and every pop past empty
  simply stays at `0`. No underflow, no error (`EXIT=0`).

- *(C) set-then-drain also resets.* `CSI =7;1u` sets the current flags to `7` in place
  (`screen_set_key_encoding_flags`, mode 1 = set, [kitty/screen.c:L1220-L1229]); popping past the
  bottom (`CSI <3u`) empties the stack and the active value resets to **`0`**.

## 5. (OBJ-4) The four Ctrl+Shift+a byte captures, plus contrast and flag sensitivity

**Question (user's verbatim example).** Press **Ctrl+Shift+a** — (a) on the main buffer with no flags
pushed; (b) after pushing disambiguate mode; (c) after switching to the alternate buffer and pushing
report-all-keys mode; (d) back on the main buffer. Show the actual bytes sent to the child for each.

Per [§1.4](#14-the-canonical-observation-path-headless-and-the-observedinferred-boundary): the active
flags in each state are read **canonically** from the real per-buffer stack, and the bytes are then
produced by the production encoder `encode_glfw_key_event` ([kitty/key_encoding.c:L414]) via its Python
export `encode_key_for_tty`, fed those exact stack flags and converted with `.encode('ascii')` just as
`Window.encoded_key` does on the live path ([kitty/window.py:L1795-L1801]).

### 5.1 The four requested states

Script (reproducible):

```bash
#!/usr/bin/env bash
# OBJ-4: exact bytes for Ctrl+Shift+a in the four requested states.
# Flag state is CANONICAL (real parser -> real stack). The bytes are produced by the SAME C encoder
# the live path uses (encode_glfw_key_event, key_encoding.c:L414), invoked here via its Python export
# encode_key_for_tty with the flags READ FROM THE REAL STACK immediately before each call, and
# converted with .encode('ascii') exactly as kitty/window.py:L1801 does on the live path.
set -o pipefail
umask 077
repo="${1:-$(git rev-parse --show-toplevel)}"   # portable: repo root from $1, else auto-detect via git
tmp="$(mktemp -d "${TMPDIR:-/tmp}/kbdobs.XXXXXXXX")"
trap 'rm -rf "$tmp"' EXIT
cat > "$tmp/obs.py" <<'PYEOF'
import kitty.fast_data_types as f
from kitty_tests import parse_bytes, Callbacks

SHIFT, CTRL = f.GLFW_MOD_SHIFT, f.GLFW_MOD_CONTROL     # 1 and 4
print("GLFW_MOD_SHIFT=%d GLFW_MOD_CONTROL=%d  -> Ctrl+Shift bitmask=%d" % (SHIFT, CTRL, SHIFT | CTRL))

def new_screen():
    c = Callbacks()
    return f.Screen(c, 5, 40, 5, 10, 20, 0, c), c

def active(s):
    return s.current_key_encoding_flags()

def query(s, c):
    c.clear(); parse_bytes(s, b'\x1b[?u'); return bytes(c.wtcbuf)

def ctrl_shift_a(stack_flags):
    # key 'a' = codepoint 97; Shift's shifted form 'A' = 65; no text; mods = Shift|Ctrl.
    out = f.encode_key_for_tty(97, shifted_key=65, mods=SHIFT | CTRL, key_encoding_flags=stack_flags)
    return out.encode('ascii')            # mirrors Window.encoded_key() .encode('ascii') at window.py:L1801

s, c = new_screen()
fl = active(s)
print("(a) MAIN / no flags pushed      stack active=%d  query->child=%r  Ctrl+Shift+a -> encoder(flags=%d)=%r" % (fl, query(s,c), fl, ctrl_shift_a(fl)))
parse_bytes(s, b'\x1b[>1u')
fl = active(s)
print("(b) MAIN / pushed disambiguate  stack active=%d  query->child=%r  Ctrl+Shift+a -> encoder(flags=%d)=%r" % (fl, query(s,c), fl, ctrl_shift_a(fl)))
parse_bytes(s, b'\x1b[?1049h'); parse_bytes(s, b'\x1b[>8u')
fl = active(s)
print("(c) ALT  / pushed report-all    stack active=%d  query->child=%r  Ctrl+Shift+a -> encoder(flags=%d)=%r" % (fl, query(s,c), fl, ctrl_shift_a(fl)))
parse_bytes(s, b'\x1b[?1049l')
fl = active(s)
print("(d) MAIN / back after round trip stack active=%d  query->child=%r  Ctrl+Shift+a -> encoder(flags=%d)=%r" % (fl, query(s,c), fl, ctrl_shift_a(fl)))
PYEOF
PYTHONPATH="$repo" python3 "$tmp/obs.py"
rc=$?
echo "EXIT=$rc"
exit $rc
```

Output **[observed]**:

```text
GLFW_MOD_SHIFT=1 GLFW_MOD_CONTROL=4  -> Ctrl+Shift bitmask=5
(a) MAIN / no flags pushed      stack active=0  query->child=b'\x1b[?0u'  Ctrl+Shift+a -> encoder(flags=0)=b'\x1b[97;6u'
(b) MAIN / pushed disambiguate  stack active=1  query->child=b'\x1b[?1u'  Ctrl+Shift+a -> encoder(flags=1)=b'\x1b[97;6u'
(c) ALT  / pushed report-all    stack active=8  query->child=b'\x1b[?8u'  Ctrl+Shift+a -> encoder(flags=8)=b'\x1b[97;6u'
(d) MAIN / back after round trip stack active=1  query->child=b'\x1b[?1u'  Ctrl+Shift+a -> encoder(flags=1)=b'\x1b[97;6u'
EXIT=0
```

**Interpretation.** In all four states, Ctrl+Shift+a emits the identical byte sequence
**`b'\x1b[97;6u'`** = `ESC [ 97 ; 6 u`. The emitted bytes are themselves the confirmation of the
encoding: the `97` field is the key's Unicode codepoint (`ord('a') == 97`), and the `6` field is the
modifier, computed by the encoder as `1 + bitmask` where the Ctrl+Shift bitmask is
`GLFW_MOD_SHIFT(1) | GLFW_MOD_CONTROL(4) = 5` (the script's first line prints
`Ctrl+Shift bitmask=5`, which is setup, not the result). Each line also shows the exact stack flag that
was read and handed to the encoder — `encoder(flags=0)`, `encoder(flags=1)`, `encoder(flags=8)`,
`encoder(flags=1)` — so the byte value is tied to a canonically-observed stack state, and the
`query->child` column independently corroborates that state from the child's point of view.

The reason the bytes are identical across the four states is that Ctrl+Shift+a is an *otherwise
ambiguous* modified key: it is emitted as a `CSI ... u` sequence under legacy mode (flags `0`),
disambiguate (`1`), and report-all-keys (`8`) alike. The flags *do* change behavior for other keys —
which is what the contrast in [§5.2](#52-contrast-keys-canonical-stack-flags) shows. The live-GUI
equivalence of these bytes is **[inferred]** from [kitty/keys.c:L251] and [kitty/window.py:L1795-L1801],
as explained in [§1.4](#14-the-canonical-observation-path-headless-and-the-observedinferred-boundary).

### 5.2 Contrast keys (canonical stack flags)

While Ctrl+Shift+a is invariant, the *plain* `a` key and the *Ctrl+a* key expose the flag semantics.
The script below encodes both across the same four states, again reading the flags canonically from the
real stack before each encode. It also includes, as a clearly separate Part 2, a **non-canonical**
encoder-characterization block (see [§5.3](#53-flag-sensitivity-explicit-non-canonical-encoder-characterization)).

Script (reproducible):

```bash
#!/usr/bin/env bash
# OBJ-4 (contrast + flag sensitivity):
#  Part 1 = CANONICAL stack flags: plain 'a' and Ctrl+a across the four states (shows disambiguate vs
#           report-all-keys behaviour on printable/ctrl keys).
#  Part 2 = NON-CANONICAL encoder characterization: Ctrl+Shift+a with EXPLICIT flag literals 0/1/4/8/5
#           to isolate bit-4 (report_alternate_key). These flags are literals, not read from a stack,
#           so they are a pure encoder cross-check, NOT proof of stack state.
set -o pipefail
umask 077
repo="${1:-$(git rev-parse --show-toplevel)}"   # portable: repo root from $1, else auto-detect via git
tmp="$(mktemp -d "${TMPDIR:-/tmp}/kbdobs.XXXXXXXX")"
trap 'rm -rf "$tmp"' EXIT
cat > "$tmp/obs.py" <<'PYEOF'
import kitty.fast_data_types as f
from kitty_tests import parse_bytes, Callbacks
SHIFT, CTRL = f.GLFW_MOD_SHIFT, f.GLFW_MOD_CONTROL

def new_screen():
    c = Callbacks()
    return f.Screen(c, 5, 40, 5, 10, 20, 0, c), c
def active(s):
    return s.current_key_encoding_flags()
def enc(key, mods, flags, shifted_key=0):
    return f.encode_key_for_tty(key, shifted_key=shifted_key, mods=mods, key_encoding_flags=flags).encode('ascii')

print("== Part 1: CANONICAL stack flags — plain 'a' (mods=0) and Ctrl+a (mods=CTRL) ==")
s, c = new_screen()
def row(tag):
    fl = active(s)
    print("%s stack active=%d  plain-a=%r  Ctrl+a=%r" % (tag, fl, enc(97, 0, fl), enc(97, CTRL, fl)))
row("(a) MAIN/no-flags   ")
parse_bytes(s, b'\x1b[>1u');                       row("(b) MAIN/disambig(1)")
parse_bytes(s, b'\x1b[?1049h'); parse_bytes(s, b'\x1b[>8u'); row("(c) ALT /report-all(8)")
parse_bytes(s, b'\x1b[?1049l');                    row("(d) MAIN/return(1)  ")

print()
print("== Part 2: NON-CANONICAL encoder characterization — Ctrl+Shift+a with EXPLICIT flag literals ==")
for fl in (0, 1, 4, 8, 5):
    print("explicit flags=%-2d  Ctrl+Shift+a=%r" % (fl, enc(97, SHIFT | CTRL, fl, shifted_key=65)))
PYEOF
PYTHONPATH="$repo" python3 "$tmp/obs.py"
rc=$?
echo "EXIT=$rc"
exit $rc
```

Output **[observed]**:

```text
== Part 1: CANONICAL stack flags — plain 'a' (mods=0) and Ctrl+a (mods=CTRL) ==
(a) MAIN/no-flags    stack active=0  plain-a=b'a'  Ctrl+a=b'\x01'
(b) MAIN/disambig(1) stack active=1  plain-a=b'a'  Ctrl+a=b'\x1b[97;5u'
(c) ALT /report-all(8) stack active=8  plain-a=b'\x1b[97u'  Ctrl+a=b'\x1b[97;5u'
(d) MAIN/return(1)   stack active=1  plain-a=b'a'  Ctrl+a=b'\x1b[97;5u'

== Part 2: NON-CANONICAL encoder characterization — Ctrl+Shift+a with EXPLICIT flag literals ==
explicit flags=0   Ctrl+Shift+a=b'\x1b[97;6u'
explicit flags=1   Ctrl+Shift+a=b'\x1b[97;6u'
explicit flags=4   Ctrl+Shift+a=b'\x1b[97:65;6u'
explicit flags=8   Ctrl+Shift+a=b'\x1b[97;6u'
explicit flags=5   Ctrl+Shift+a=b'\x1b[97:65;6u'
EXIT=0
```

**Interpretation of Part 1 (canonical).** The contrast confirms the flag semantics against the
authoritative flag table ([docs/keyboard-protocol.rst:L275-L283]) and the encoder's bit decode
([kitty/key_encoding.c:L419-L423]):

- With **no flags** (state a, active `0`): plain `a` sends the literal byte `b'a'`; Ctrl+a sends the
  legacy control byte `b'\x01'`.
- With **disambiguate** (states b/d, active `1`, `.disambiguate = flags & 1`
  [kitty/key_encoding.c:L419]): plain printable `a` *still* sends `b'a'` (only ambiguous keys change),
  but Ctrl+a now disambiguates to `b'\x1b[97;5u'` (modifier `5` = Ctrl bitmask `4` + `1`).
- With **report-all-keys** (state c, active `8`, `.report_text = flags & 8`
  [kitty/key_encoding.c:L422]): *every* key becomes a `CSI ... u` sequence, so even plain `a` reports
  as `b'\x1b[97u'`; Ctrl+a remains `b'\x1b[97;5u'`.

This is the crux of the independence proof in [§6](#6-obj-5--independence-proof-and-leakage-probe):
the plain-`a` byte differs by buffer state (`b'a'` on the main buffer vs `b'\x1b[97u'` on the
alternate buffer), which can only happen if the two buffers carry different active flags.

### 5.3 Flag sensitivity: explicit (non-canonical) encoder characterization

Part 2 of the output above feeds the encoder **explicit flag literals** `0, 1, 4, 8, 5` rather than
values read from a stack. This is a pure characterization of the encoder and is labelled
**non-canonical** because the flags are not proven to be any buffer's real stack state — it isolates
the effect of one bit:

- flags `0`, `1`, `8` → `b'\x1b[97;6u'` (no alternate key reported);
- flags `4` (`.report_alternate_key = flags & 4`, [kitty/key_encoding.c:L421]) → `b'\x1b[97:65;6u'`,
  which adds the shifted key `65` (`ord('A')`) after a colon;
- flags `5` (= `1 | 4`) → `b'\x1b[97:65;6u'` as well.

So bit 4 is what would add the shifted-key sub-field; it is *not* set in any of the four canonical
states (whose stack flags were `0`, `1`, `8`, `1`), which is the deeper reason all four canonical
captures were the plain `b'\x1b[97;6u'`.

## 6. (OBJ-5) Independence proof and leakage probe

**Question.** Do the captured byte sequences prove the two buffers maintain independent stacks, and is
there any state leakage during rapid buffer switching while the keyboard mode is manipulated?

**Proof from the captures already shown.** Independence is established two ways:

1. *Query replies* (canonical, child-visible): in [§2](#2-obj-1--round-trip-stack-survival) the flags
   query returns `b'\x1b[?1u'` on the main buffer but `b'\x1b[?8u'` on the alternate buffer at the same
   point in the round trip, and returns to `b'\x1b[?1u'` on the main buffer afterward.
2. *Key bytes* (contrast, [§5.2](#52-contrast-keys-canonical-stack-flags)): the plain-`a` byte is
   `b'a'` on the main buffer (flags `1`) but `b'\x1b[97u'` on the alternate buffer (flags `8`). A
   single shared stack could not yield two different bytes for the same key at the same moment.

**Leakage probe.** The script drives 10 rapid main↔alt round trips while the two stacks hold distinct
values (main `1`, alt `8`) and checks the active value on each side every cycle.

Script (reproducible):

```bash
#!/usr/bin/env bash
# OBJ-5: rapid main<->alt switching while both stacks hold distinct flags; probe for any leakage.
set -o pipefail
umask 077
repo="${1:-$(git rev-parse --show-toplevel)}"   # portable: repo root from $1, else auto-detect via git
tmp="$(mktemp -d "${TMPDIR:-/tmp}/kbdobs.XXXXXXXX")"
trap 'rm -rf "$tmp"' EXIT
cat > "$tmp/obs.py" <<'PYEOF'
import kitty.fast_data_types as f
from kitty_tests import parse_bytes, Callbacks

c = Callbacks()
s = f.Screen(c, 5, 40, 5, 10, 20, 0, c)

def active(s):
    return s.current_key_encoding_flags()

parse_bytes(s, b'\x1b[>1u')                 # MAIN stack: push disambiguate (1)
parse_bytes(s, b'\x1b[?1049h')              # -> ALT
parse_bytes(s, b'\x1b[>8u')                 # ALT stack: push report-all (8)
parse_bytes(s, b'\x1b[?1049l')              # -> back to MAIN
print("setup: MAIN=1, ALT=8;  MAIN active now=%d" % active(s))

leaked = False
for i in range(1, 11):                      # 10 rapid round-trip cycles
    parse_bytes(s, b'\x1b[?1049h'); a = active(s)   # into ALT
    parse_bytes(s, b'\x1b[?1049l'); m = active(s)   # back to MAIN
    ok = (a == 8 and m == 1)
    leaked = leaked or not ok
    print("cycle %2d: ALT active=%d  MAIN active=%d  %s" % (i, a, m, "OK" if ok else "LEAK!"))
print("RESULT: leakage detected = %s (MAIN stayed 1 and ALT stayed 8 across all cycles = no leak)" % leaked)
PYEOF
PYTHONPATH="$repo" python3 "$tmp/obs.py"
rc=$?
echo "EXIT=$rc"
exit $rc
```

Output **[observed]**:

```text
setup: MAIN=1, ALT=8;  MAIN active now=1
cycle  1: ALT active=8  MAIN active=1  OK
cycle  2: ALT active=8  MAIN active=1  OK
cycle  3: ALT active=8  MAIN active=1  OK
cycle  4: ALT active=8  MAIN active=1  OK
cycle  5: ALT active=8  MAIN active=1  OK
cycle  6: ALT active=8  MAIN active=1  OK
cycle  7: ALT active=8  MAIN active=1  OK
cycle  8: ALT active=8  MAIN active=1  OK
cycle  9: ALT active=8  MAIN active=1  OK
cycle 10: ALT active=8  MAIN active=1  OK
RESULT: leakage detected = False (MAIN stayed 1 and ALT stayed 8 across all cycles = no leak)
EXIT=0
```

**Interpretation.** Across all 10 cycles the alternate buffer always reads `8` and the main buffer
always reads `1`; `leakage detected = False`. No value bled from one buffer's stack into the other's,
consistent with the two physically separate arrays ([kitty/screen.h:L128]) and the pointer-only toggle
([kitty/screen.c:L1079], [kitty/screen.c:L1086]). **[observed]**

## 7. (OBJ-6) Mode-dependent edge cases: 47/1047/1049, DECCKM, and SCORC

**Question.** Enumerate and exercise the terminal-mode variants under which stack isolation might
behave unexpectedly: the three alternate-screen mode constants `47`/`1047`/`1049`, the cursor-key mode
`DECCKM`, and the first-push-from-empty behavior (covered in [§4](#4-obj-3--pop-to-empty-reset-first-push-from-empty-and-over-pop)).

**Mechanism (source).** The three constants are `TOGGLE_ALT_SCREEN_1` (47), `TOGGLE_ALT_SCREEN_2`
(1047), and `ALTERNATE_SCREEN` (1049) ([kitty/modes.h:L75-L77]). All three are dispatched to
`screen_toggle_screen_buffer` ([kitty/screen.c:L1165-L1169]); only `1049` additionally saves the
cursor and clears the alternate screen (its two boolean arguments are true only for
`mode == ALTERNATE_SCREEN`). Crucially, **all three go through the same toggle**, which re-points the
keyboard-flag pointer ([kitty/screen.c:L1079], [kitty/screen.c:L1086]), so isolation is expected to
hold for all three. (`terminfo`'s `rmcup` is `\E[?1049l`, [kitty/terminfo.py:L207].) `DECCKM` is read
by the encoder as its `cursor_key_mode` argument on the live path ([kitty/keys.c:L251]).

Script (reproducible):

```bash
#!/usr/bin/env bash
# OBJ-6: (A) isolation across the 3 alt-screen mode constants 47/1047/1049;
#         (B) DECCKM x flags cross-product for the Up arrow;
#         (C) SCORC confusion: bare CSI u = restore-cursor, NOT a keyboard-stack op.
set -o pipefail
umask 077
repo="${1:-$(git rev-parse --show-toplevel)}"   # portable: repo root from $1, else auto-detect via git
tmp="$(mktemp -d "${TMPDIR:-/tmp}/kbdobs.XXXXXXXX")"
trap 'rm -rf "$tmp"' EXIT
cat > "$tmp/obs.py" <<'PYEOF'
import kitty.fast_data_types as f
from kitty_tests import parse_bytes, Callbacks

def new_screen():
    c = Callbacks()
    return f.Screen(c, 5, 40, 5, 10, 20, 0, c), c
def active(s):
    return s.current_key_encoding_flags()

print("== Part A: stack isolation across alternate-screen mode constants 47 / 1047 / 1049 ==")
for mode in (47, 1047, 1049):
    s, c = new_screen()
    parse_bytes(s, b'\x1b[>1u')                       # MAIN push disambiguate (1)
    m0 = active(s)
    parse_bytes(s, b'\x1b[?%dh' % mode)               # enter ALT via this mode
    a0 = active(s)
    parse_bytes(s, b'\x1b[>8u')                       # ALT push report-all (8)
    a1 = active(s)
    parse_bytes(s, b'\x1b[?%dl' % mode)               # leave ALT via same mode
    m1 = active(s)
    print("mode %-4d: MAIN=%d  enter-ALT=%d  ALT-after-push=%d  back-MAIN=%d  isolation %s"
          % (mode, m0, a0, a1, m1, "HOLDS" if (m0 == 1 and a1 == 8 and m1 == 1) else "BROKEN"))

print()
print("== Part B: DECCKM x flags cross-product for the Up arrow (GLFW_FKEY_UP=%d) ==" % f.GLFW_FKEY_UP)
print("(DECCKM set canonically via CSI ?1h / ?1l and read back from screen.cursor_key_mode;")
print(" flags set canonically via the stack; Up encoded by the real encoder with both real inputs)")
for fl_target in (0, 1, 8):
    for deckm_on in (False, True):
        s, c = new_screen()
        parse_bytes(s, b'\x1b[?1h' if deckm_on else b'\x1b[?1l')   # DECCKM set/reset
        ckm = s.cursor_key_mode
        if fl_target:
            parse_bytes(s, b'\x1b[>%du' % fl_target)               # push target flags
        fl = active(s)
        up = f.encode_key_for_tty(f.GLFW_FKEY_UP, mods=0, key_encoding_flags=fl, cursor_key_mode=ckm).encode('ascii')
        print("DECCKM %-3s (cursor_key_mode=%d)  flags=%d  Up-> %r" % ("on" if deckm_on else "off", ckm, fl, up))

print()
print("== Part C: SCORC confusion — bare CSI u restores the cursor, leaving the keyboard stack alone ==")
s, c = new_screen()
parse_bytes(s, b'\x1b[>1u')                            # push disambiguate so we can watch it survive
parse_bytes(s, b'\x1b[3;10H')                          # move cursor to row3 col10 (1-based)
parse_bytes(s, b'\x1b[s')                              # SCOSC: save cursor (CSI s)
saved = (s.cursor.x, s.cursor.y)
parse_bytes(s, b'\x1b[1;1H')                           # move cursor to home
moved = (s.cursor.x, s.cursor.y)
c.clear()
parse_bytes(s, b'\x1b[u')                              # bare CSI u == SCORC (restore cursor)
restored = (s.cursor.x, s.cursor.y)
print("cursor saved(x,y)=%s  after-move(x,y)=%s  after 'CSI u'(x,y)=%s" % (saved, moved, restored))
print("bytes written to child by 'CSI u' = %r (empty => NOT a query; it is restore-cursor)" % bytes(c.wtcbuf))
print("keyboard flags after 'CSI u' = %d (unchanged; SCORC does not touch the stack)" % active(s))
c.clear()
parse_bytes(s, b'\x1b[?u')                              # contrast: CSI ? u IS the flags query
print("for contrast, 'CSI ? u' writes to child = %r (this is the keyboard-flags query)" % bytes(c.wtcbuf))
PYEOF
PYTHONPATH="$repo" python3 "$tmp/obs.py"
rc=$?
echo "EXIT=$rc"
exit $rc
```

Output **[observed]**:

```text
== Part A: stack isolation across alternate-screen mode constants 47 / 1047 / 1049 ==
mode 47  : MAIN=1  enter-ALT=0  ALT-after-push=8  back-MAIN=1  isolation HOLDS
mode 1047: MAIN=1  enter-ALT=0  ALT-after-push=8  back-MAIN=1  isolation HOLDS
mode 1049: MAIN=1  enter-ALT=0  ALT-after-push=8  back-MAIN=1  isolation HOLDS

== Part B: DECCKM x flags cross-product for the Up arrow (GLFW_FKEY_UP=57352) ==
(DECCKM set canonically via CSI ?1h / ?1l and read back from screen.cursor_key_mode;
 flags set canonically via the stack; Up encoded by the real encoder with both real inputs)
DECCKM off (cursor_key_mode=0)  flags=0  Up-> b'\x1b[A'
DECCKM on  (cursor_key_mode=1)  flags=0  Up-> b'\x1bOA'
DECCKM off (cursor_key_mode=0)  flags=1  Up-> b'\x1b[A'
DECCKM on  (cursor_key_mode=1)  flags=1  Up-> b'\x1b[A'
DECCKM off (cursor_key_mode=0)  flags=8  Up-> b'\x1b[A'
DECCKM on  (cursor_key_mode=1)  flags=8  Up-> b'\x1bOA'

== Part C: SCORC confusion — bare CSI u restores the cursor, leaving the keyboard stack alone ==
cursor saved(x,y)=(9, 2)  after-move(x,y)=(0, 0)  after 'CSI u'(x,y)=(9, 2)
bytes written to child by 'CSI u' = b'' (empty => NOT a query; it is restore-cursor)
keyboard flags after 'CSI u' = 1 (unchanged; SCORC does not touch the stack)
for contrast, 'CSI ? u' writes to child = b'\x1b[?1u' (this is the keyboard-flags query)
EXIT=0
```

### 7.1 Isolation holds across 47 / 1047 / 1049

For every one of the three constants the trace is identical: main `1`, fresh alt `0`, alt after push
`8`, back to main `1` — printed as `isolation HOLDS` on all three rows. The differing side effects of
`1049` (cursor save + alt-screen clear) do not affect the keyboard stacks, because isolation comes from
the shared pointer re-point ([kitty/screen.c:L1079], [kitty/screen.c:L1086]) that all three constants
trigger. **[observed]**

### 7.2 DECCKM × flags cross-product for the Up arrow

`DECCKM` is set canonically via `CSI ?1h`/`CSI ?1l` and read back from `screen.cursor_key_mode`; the
flags are set canonically on the stack; the Up arrow (`GLFW_FKEY_UP = 57352`) is then encoded by the
real encoder with both real inputs. The observed cross-product:

| flags | DECCKM off | DECCKM on |
|-------|------------|-----------|
| `0` (legacy) | `b'\x1b[A'` | `b'\x1bOA'` |
| `1` (disambiguate) | `b'\x1b[A'` | `b'\x1b[A'` |
| `8` (report-all-keys) | `b'\x1b[A'` | `b'\x1bOA'` |

With legacy or report-all-keys flags, `DECCKM on` switches the Up arrow from the CSI form `b'\x1b[A'`
to the SS3 form `b'\x1bOA'`, the classic application-cursor-keys behavior. With **disambiguate** flags
(`1`), the kitty encoder emits `b'\x1b[A'` regardless of `DECCKM` — the progressive-enhancement mode
takes precedence over the legacy cursor-key toggle for this key. **[observed]**

### 7.3 SCORC: a bare `CSI u` restores the cursor, not the keyboard stack

Part C sets the cursor to row 3 / col 10, saves it with `CSI s`, moves it home, then issues a bare
`CSI u`. The output shows the cursor position `saved=(9, 2)` (0-based x/y for col 10, row 3),
`after-move=(0, 0)`, and `after 'CSI u'=(9, 2)` — i.e. the cursor was **restored**. The bare `CSI u`
wrote `b''` to the child (it is *not* a query) and left the keyboard flags unchanged at `1`. For
contrast, `CSI ? u` wrote `b'\x1b[?1u'` — the keyboard-flags query. This confirms the grammar caveat
from [§1.5](#15-the-csi-u-escape-code-grammar-and-the-scorc-confusion): a modifier-less `CSI u` is
SCORC ([kitty/vt-parser.c:L1218-L1221]), unrelated to the keyboard stack. **[observed]**

## 8. How the mechanism works (source grounding)

The behaviors above all reduce to one structural fact and a handful of routines:

- **Storage — two arrays, one pointer** ([kitty/screen.h:L128]):
  `uint8_t main_key_encoding_flags[8], alt_key_encoding_flags[8], *key_encoding_flags;`. The two
  8-slot arrays are the per-buffer stacks; `key_encoding_flags` is the active pointer.
- **Buffer toggle — pointer re-point only** (`screen_toggle_screen_buffer`): on entering the alternate
  screen the pointer is set to the alt array ([kitty/screen.c:L1079]); on leaving, back to the main
  array ([kitty/screen.c:L1086]). No bytes are copied between arrays — the source of independence.
- **Current flags** (`screen_current_key_encoding_flags`, [kitty/screen.c:L1204-L1206]): the low 7 bits
  of the highest slot whose `0x80` "occupied" bit is set.
- **Push with silent eviction** (`screen_push_key_encoding_flags`, [kitty/screen.c:L1234-L1243]):
  writes `0x80 | (val & 0x7f)` at the next slot; if the top slot is index 7, `memmove`s the array down
  one and drops the oldest entry ([kitty/screen.c:L1241]).
- **Pop with reset** (`screen_pop_key_encoding_flags`, [kitty/screen.c:L1248-L1252]): clears the popped
  occupied slots to `0`; when all are clear the current value is `0`.
- **Encoder** (`encode_glfw_key_event`, [kitty/key_encoding.c:L414]) reads the flag bits
  ([kitty/key_encoding.c:L419-L423]) and the cursor-key mode to produce the emitted bytes; the live
  path calls it with the active stack flags ([kitty/keys.c:L251]).

The round trip that motivated OBJ-1 is exactly this pointer dance:

```mermaid
graph TD
    A["Main buffer active<br/>active ptr = main_key_encoding_flags"] --> B["Push disambiguate (1) on main<br/>main array: base-0, then 1"]
    B --> C["Toggle to alt (CSI ?1049h)<br/>active ptr = alt_key_encoding_flags"]
    C --> D["Push report-all-keys (8) on alt<br/>alt array: base-0, then 8"]
    D --> E["Toggle back to main (CSI ?1049l)<br/>active ptr = main_key_encoding_flags"]
    E --> F["Active flags = 1 again<br/>main array untouched, survived intact"]
    D -. no copy between arrays .-> F
```

## 9. Specification cross-check and the doc-vs-code naming skew

**Stack semantics match the specification.** The authoritative specification in the repository states
that terminals *"must maintain separate stacks for the main and alternate screens"*, that *"if a pop
request is received that empties the stack, all flags are reset"*, and that *"if a push request is
received and the stack is full, the oldest entry from the stack must be evicted"*
([docs/keyboard-protocol.rst:L300-L303]). Every one of these was observed above: separate stacks
([§2](#2-obj-1--round-trip-stack-survival), [§6](#6-obj-5--independence-proof-and-leakage-probe)),
pop-to-empty reset ([§4](#4-obj-3--pop-to-empty-reset-first-push-from-empty-and-over-pop)), and
oldest-entry eviction ([§3](#3-obj-2--stack-exhaustion-on-both-buffers-and-cross-buffer-isolation)).

**Stack depth is kitty's implementation choice, not a spec minimum.** The specification does not
mandate a minimum depth; it only says terminals *"should limit the size of the stack as appropriate,
to prevent Denial-of-Service attacks"* ([docs/keyboard-protocol.rst:L299-L300]). The depth of **8**
reported throughout this document is the size of kitty's own fixed arrays as observed in the source —
`main_key_encoding_flags[8]` / `alt_key_encoding_flags[8]` ([kitty/screen.h:L128]) — and confirmed at
runtime by the exhaustion test retaining exactly 8 values
([§3](#3-obj-2--stack-exhaustion-on-both-buffers-and-cross-buffer-isolation)). It is not attributed to
any external "minimum depth" requirement.

**Doc-vs-code naming skew (stated so the reader is not misled).** The user's "disambiguate mode" is
flag value `1`, decoded by the encoder as `.disambiguate = flags & 1` ([kitty/key_encoding.c:L419]).
The user's "report-all-keys mode" is the specification's flag `0b1000` (value `8`), *"report all keys
as escape codes"* ([docs/keyboard-protocol.rst:L275-L283]); internally, however, the C source names
that bit's field `report_text` (`.report_text = flags & 8`, [kitty/key_encoding.c:L422]), and it names
the next bit `0b10000` (value `16`) `embed_text` (`.embed_text = flags & 16`,
[kitty/key_encoding.c:L423]). In other words the spec label "report all keys as escape codes" and the
C field name `report_text` denote the *same* bit (value `8`); the code's `report_text`/`embed_text`
names do not line up verbatim with the specification's prose, which can mislead a reader comparing the
two side by side.

## 10. Coverage pass

Every distinct part of the question and every named item, with the section that answers it from
observed evidence:

| Question part / named item | Section | Observed result |
|----------------------------|---------|-----------------|
| Round trip: which mode active at end | [§2](#2-obj-1--round-trip-stack-survival) | main active = `1` (disambiguate) |
| Round trip: does main stack survive intact | [§2](#2-obj-1--round-trip-stack-survival) | yes, intact |
| Round trip: escape produced in each intermediate state | [§2](#2-obj-1--round-trip-stack-survival) | query bytes `?0u`,`?1u`,`?0u`,`?8u`,`?1u` |
| Exhaustion: silent drop / error / otherwise | [§3](#3-obj-2--stack-exhaustion-on-both-buffers-and-cross-buffer-isolation) | silent eviction of oldest, no error |
| Exhaustion: does one buffer affect the other | [§3](#3-obj-2--stack-exhaustion-on-both-buffers-and-cross-buffer-isolation) | no; isolated |
| Exhaustion exercised on the alternate buffer too | [§3](#3-obj-2--stack-exhaustion-on-both-buffers-and-cross-buffer-isolation) | identical push/pop sequences |
| Pop-to-empty resets all flags | [§4](#4-obj-3--pop-to-empty-reset-first-push-from-empty-and-over-pop) | resets to `0` |
| First-push-from-empty behavior | [§4](#4-obj-3--pop-to-empty-reset-first-push-from-empty-and-over-pop) | seeds base-0 entry |
| Over-pop behavior | [§4](#4-obj-3--pop-to-empty-reset-first-push-from-empty-and-over-pop) | stays `0`, no underflow |
| Ctrl+Shift+a (a) main / no flags | [§5.1](#51-the-four-requested-states) | `b'\x1b[97;6u'` (stack `0`) |
| Ctrl+Shift+a (b) main / disambiguate | [§5.1](#51-the-four-requested-states) | `b'\x1b[97;6u'` (stack `1`) |
| Ctrl+Shift+a (c) alt / report-all-keys | [§5.1](#51-the-four-requested-states) | `b'\x1b[97;6u'` (stack `8`) |
| Ctrl+Shift+a (d) back on main | [§5.1](#51-the-four-requested-states) | `b'\x1b[97;6u'` (stack `1`) |
| Contrast keys (plain `a`, Ctrl+a) | [§5.2](#52-contrast-keys-canonical-stack-flags) | differ per state |
| Flag sensitivity (bit 4) | [§5.3](#53-flag-sensitivity-explicit-non-canonical-encoder-characterization) | `b'\x1b[97:65;6u'` for flags 4/5 |
| Independence proof | [§6](#6-obj-5--independence-proof-and-leakage-probe) | query + key bytes differ by buffer |
| Leakage during rapid switching | [§6](#6-obj-5--independence-proof-and-leakage-probe) | none across 10 cycles |
| Mode constant `47` | [§7.1](#71-isolation-holds-across-47--1047--1049) | isolation holds |
| Mode constant `1047` | [§7.1](#71-isolation-holds-across-47--1047--1049) | isolation holds |
| Mode constant `1049` | [§7.1](#71-isolation-holds-across-47--1047--1049) | isolation holds |
| `DECCKM` cursor-key mode | [§7.2](#72-decckm--flags-cross-product-for-the-up-arrow) | full flags×DECCKM matrix |
| Bare `CSI u` = SCORC caveat | [§7.3](#73-scorc-a-bare-csi-u-restores-the-cursor-not-the-keyboard-stack) | restores cursor, not a keyboard op |
| Doc-vs-code naming skew | [§9](#9-specification-cross-check-and-the-doc-vs-code-naming-skew) | flag 8 = `report_text` |

## 11. Reproducibility: two-run stability

The seven observation scripts from [§2](#2-obj-1--round-trip-stack-survival)–[§7](#7-obj-6--mode-dependent-edge-cases-4710471049-decckm-and-scorc)
are deterministic. To confirm the reported values are stable, the full set was run **twice**
unchanged, each run's concatenated output hashed, and the two runs compared byte-for-byte.

Command (reproducible; `$R` is the directory holding the seven scripts shown above, and `$repo` is
the checkout root — pass it as the first argument, or let it auto-detect via
`git rev-parse --show-toplevel` when the runner is invoked from within the tree. The runner forwards
`$repo` to each script as its first argument, matching the scripts' own `repo="${1:-$(git rev-parse --show-toplevel)}"`
convention, so the whole set is portable and does not depend on the current working directory):

```bash
set -o pipefail
repo="${1:-$(git rev-parse --show-toplevel)}"   # portable: repo root from $1, else auto-detect via git
SCRIPTS="obj1_roundtrip obj2_exhaustion obj3_pop_reset obj4_ctrlshifta obj4b_contrast obj5_leakage obj6_modes"
run_all() { for s in $SCRIPTS; do bash "$R/$s.sh" "$repo"; done; }
run_all > run1.txt 2>&1; echo "run1 rc=$?"
run_all > run2.txt 2>&1; echo "run2 rc=$?"
echo "run1 sha256: $(sha256sum run1.txt | cut -d' ' -f1)"
echo "run2 sha256: $(sha256sum run2.txt | cut -d' ' -f1)"
cmp -s run1.txt run2.txt && echo "cmp: IDENTICAL (byte-for-byte)" || { echo "cmp: DIFFER"; cmp run1.txt run2.txt; }
echo "run1 final-main: $(grep 'flags on MAIN' run1.txt)"
echo "run2 final-main: $(grep 'flags on MAIN' run2.txt)"
```

Output **[observed]**:

```text
run1 rc=0
run2 rc=0
run1 sha256: 0c87b9bef253086742e37abdb5e69e6fc30be5570ca9692b071b5de150a594db
run2 sha256: 0c87b9bef253086742e37abdb5e69e6fc30be5570ca9692b071b5de150a594db
cmp: IDENTICAL (byte-for-byte)
run1 final-main: RESULT  final active keyboard-encoding flags on MAIN = 1
run2 final-main: RESULT  final active keyboard-encoding flags on MAIN = 1
```

Both runs produce the identical sha256
`0c87b9bef253086742e37abdb5e69e6fc30be5570ca9692b071b5de150a594db` and `cmp` confirms them
byte-for-byte identical; the round-trip result (main active flags = `1`) is stable across both runs.
**[observed]**

## 12. Cleanup and final repository state

This investigation is read-only with respect to the source tree; the sole repository change is this
document. The cleanup discipline was built into every observation:

- Each script created its program inside a private `mktemp -d` directory under `umask 077` and removed
  that directory via a `trap '... EXIT'` on exit — so no observation script was ever left behind, and
  none was ever written inside the repository working tree.
- The build artifacts (`kitty/fast_data_types.so`, `kitty/glfw-x11.so`, `build/`, the launchers) are
  all git-ignored, so rebuilding them leaves the tracked tree untouched.

Cleanup and final-state verification:

```bash
set -o pipefail
rm -rf /tmp/kbd_evidence                 # remove the out-of-tree capture directory
ls -d /tmp/kbdobs.* 2>/dev/null || echo "no private mktemp dirs remain (trap-removed)"
find . -path ./.git -prune -o \( -name 'blitzy_adhoc_test_*' -o -name 'kbdobs.*' \) -print
echo "--- git working-tree state ---"
git status --porcelain
git diff --name-status
```

Output **[observed]**:

```text
no private mktemp dirs remain (trap-removed)
--- git working-tree state ---
 M blitzy/documentation/kitty_815df1e210e0.md
M	blitzy/documentation/kitty_815df1e210e0.md
```

The `find` prints nothing (no temporary script exists anywhere in the repository), and
`git status --porcelain` lists exactly one changed file — the deliverable. The net repository change
is this single Markdown document; everything else (temporary scripts, capture files, build outputs)
is either out-of-tree or git-ignored.
