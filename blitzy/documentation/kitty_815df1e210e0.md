# kitty — Compiled C Extensions and the Test Suite: An Evidence-Grounded Analysis

> **Source branch investigated:** `kitty_815df1e210e0` · **kitty version (built):** `0.35.2`
> **Task type:** read-only, run-first investigation. The only file written by this task is this document; no **tracked** file in the kitty source tree was created, modified, or deleted. The only on-disk changes made while building and observing were **git-ignored** build outputs and the object cache — the four `.so` extensions, the `kitty`/`kitten` binaries, and the `build/` directory (all matched by `[.gitignore:L1-L20]`) — which the build produces and which are removed again, leaving tracked version-control state unchanged (verified in *Read-only verification*).
> **Methodology (SWE-AtlasQnA):** every behavioral claim below is placed next to the **actual, complete, unedited output** that produced it, together with the exact command and its exit code. For each condition the **primary run (RUN #1) is shown complete and unedited**. **Stability across two runs** is then demonstrated according to the size of the output: for the short import-cascade conditions (Q4 Tier 1 and Tier 2) the *entire* second log is compared against the first with an actual `diff` whose output is shown — empty for Tier 1, and reducing to zero once the single legitimately-varying line is accounted for in Tier 2 (see below); for the long full-suite conditions (Q1.c, Q4 Tier 3 and Tier 4) RUN #1 is complete and **RUN #2 is a clearly-labelled excerpt** reproducing the terminal counts, the exact failing test IDs, the during-state, and any new tracebacks, with the 145 per-test progress lines — byte-identical *in kind* to RUN #1 — collapsed behind an explicit bracketed elision marker (a labelled elision, never a silent truncation). Values that legitimately vary run-to-run are identified and **excluded** from the stability comparison: **wall-clock time**, the **`Go packages being tested:` order** (a Python `set` iteration order — see Q1.b), and the **ctypes pointer/hex addresses** in the Tier 3 variable dumps. Any other short excerpt highlighted for the reader is labelled *(excerpt)* with the complete raw log shown adjacent. Every structural claim carries a `file:line` citation and names the function/method that performs the work. Anything not directly observed at runtime is prefixed exactly **`Inferred:`**. The canonical entry point (`./kitty/launcher/kitty +launch test.py`) was used throughout for all behavioral results; the two targeted probes that read internal state directly (the Q1.b hash-seed probe and the Q3 load-set probe) invoke the real, unmodified functions but bypass the full `test.py` → `main()` flow and are therefore explicitly labelled **non-canonical** — no mocks, no debug hooks, no synthetic stand-ins are used anywhere.

---

## Summary (one-paragraph answer)

kitty compiles **four C/Objective-C extensions** — `kitty/fast_data_types.so`, `kittens/transfer/rsync.so`, `kitty/glfw-x11.so`, `kitty/glfw-wayland.so` — plus a C launcher (`kitty/launcher/kitty`) and the Go `kitten` binary (`kitty/launcher/kitten`). In the normal from-source build the `kitten` binary is a **dynamically-linked** Go executable (linked against `libc`; `CGO_ENABLED=0` static linkage is used only for cross-platform release builds — see Q1). When the suite is run through its canonical entry point, exactly **two** of those extensions load as Python modules: **`kitty.fast_data_types`** (loaded *eagerly*, at harness-import time, because the `BaseTest` base class transitively then directly imports it) and **`kittens.transfer.rsync`** (loaded at *collection* time, because `kitty_tests/file_transmission.py` imports it at module top level). The two GLFW backends never load as Python modules in a headless run; they are touched *on demand* by two individual tests via `ctypes`/file-checks, and only the **x11** backend is exercised under canonical `CI=true`. This produces a **three-tier failure cascade** when an extension is made unavailable: removing `fast_data_types.so` is **suite-fatal at harness import** (the banner never prints, **0 tests run**); removing `rsync.so` is **suite-fatal at collection** (the banner prints, then **0 Python tests run**); removing `glfw-x11.so` is **optional/localized** (**all 145 tests still run**, adding exactly one FAIL and one ERROR on top of the baseline). `fast_data_types` is therefore the central single-point-of-failure for the entire suite, `rsync` is a second critical dependency (made fatal by the harness's non-defensive direct-import collection loop at `[kitty_tests/main.py:L64]`), and the GLFW backends are optional leaves.

---

## Environment / versions (observed)

All figures below are the **actual versions observed in this environment**, captured with the exact commands shown. Toolchain versions:

```text
$ python3 --version
Python 3.13.7
$ python3 -c "import sys; print(sys.executable)"
/usr/bin/python3
$ go version
go version go1.22.12 linux/amd64
$ command -v go
/usr/local/bin/go
$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
$ pkg-config --version
1.8.1
```

Native library versions (via `pkg-config --modversion`), including `wayland-protocols`, whose newer enum values drive the default-build Wayland abort documented in Q1:

```text
$ for lib in freetype2 fontconfig harfbuzz libpng lcms2 libxxhash libcrypto openssl xkbcommon wayland-client wayland-protocols gl dbus-1 x11 xcb; do printf "%-16s " "$lib"; pkg-config --modversion "$lib"; done
freetype2        26.2.20
fontconfig       2.15.0
harfbuzz         10.2.0
libpng           1.6.50
lcms2            2.16
libxxhash        0.8.3
libcrypto        3.5.3
openssl          3.5.3
xkbcommon        1.7.0
wayland-client   1.24.0
wayland-protocols 1.45
gl               1.2
dbus-1           1.16.2
x11              1.8.12
xcb              1.17.0
```

Fonts (installed families) — note the distinction between **font-face rows** and **unique family strings**:

```text
$ fc-list | wc -l          # total font FACE rows
511
$ fc-list : family | sort -u | wc -l   # unique family strings
267
$ fc-list | grep -ci "Source Code Pro"  # is that specific family present?
0
```

Summarised (each value backed by the raw output above):

| Component | Observed version | Note |
|---|---|---|
| python3 | **3.13.7** (`/usr/bin/python3`) | satisfies `pyproject.toml` `requires-python = ">=3.8"` |
| go | **go1.22.12 linux/amd64** (`/usr/local/bin/go`) | satisfies `[go.mod:L3]` `go 1.22` |
| gcc | **15.2.0** (Ubuntu 15.2.0-4ubuntu4) | C/Objective-C compiler |
| pkg-config | **1.8.1** | native-library discovery |
| freetype2 | 26.2.20 | `[docs/build.rst:L90]` |
| fontconfig | 2.15.0 | `[docs/build.rst:L91]` |
| harfbuzz | 10.2.0 | `[docs/build.rst:L84]` (`>= 2.2.0`) |
| libpng | 1.6.50 | `[docs/build.rst:L86]` |
| lcms2 | 2.16 | `[docs/build.rst:L87]` |
| libxxhash | 0.8.3 | `[docs/build.rst:L88]`; linked into `rsync.so` |
| openssl / libcrypto | 3.5.3 | `[docs/build.rst:L89]` |
| xkbcommon | 1.7.0 | X11 GLFW backend |
| wayland-client | 1.24.0 | Wayland GLFW backend |
| **wayland-protocols** | **1.45** | newer than this kitty snapshot targets → drives the default-build `-Werror=switch` abort (Q1) |
| gl | 1.2 | OpenGL |
| dbus-1 | 1.16.2 | desktop integration |
| x11 / xcb | 1.8.12 / 1.17.0 | X11 GLFW backend |
| Fonts | **511 font-face rows**, **267 unique families**; DejaVu / Fira Code present, **"Source Code Pro" absent** | drives one baseline artifact (see "Baseline failures") |
| PIL / pygments | Pillow 12.3.0 / pygments 2.20.0 | optional Python test deps |

> **Note on provenance / version drift.** The Agent Action Plan's dependency table listed slightly older versions (e.g., Python 3.12.3, Go 1.22.2, gcc 13.3.0). Per Rule 1, the values above are the versions **actually observed in this environment** at investigation time. Where an observed figure differs from the AAP's pre-captured "ground truth" — notably `_json` being a **builtin** rather than a `.so` (Q3) and the **Tier-3 failure totals** (Q4) — the difference is reported and explained rather than hidden.

---

## Prerequisites gating the canonical `test.py` path

Although the questions concern *C* extensions, the canonical entry point has three prerequisite classes; each is enumerated below with its citation and the observed (or, where noted, `Inferred:`) consequence when absent.

1. **Go toolchain (required by the canonical entry).** The runner enumerates Go packages before running Python tests: `reduce_go_pkgs()` at `[kitty_tests/main.py:L196]` calls `go_exe()` and, when `go` is absent, raises `SystemExit('go executable not found, current path: ' + repr(os.environ.get('PATH', '')))` at `[kitty_tests/main.py:L198]`. It is invoked from `run_tests()` at `[kitty_tests/main.py:L267]`. `[go.mod:L3]` mandates `go 1.22`. **Observed:** Go **1.22.12** on `PATH`; the baseline banner reports `Go executable: /usr/local/bin/go` and the run ends with `All Go tests succeeded` (see Q1.c). **Observed** (reproduced by invoking the canonical entry point with `go` removed from `PATH`): the run raises that `SystemExit` in `reduce_go_pkgs()` (`[kitty_tests/main.py:L196-L198]`, reached from `run_tests()` at `[kitty_tests/main.py:L267]`) **before any Python test executes** — the banner never prints and **0 tests run**. Command and complete, unedited output:

```text
$ env -i CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 HOME="$HOME" PATH=/usr/bin:/bin ./kitty/launcher/kitty +launch test.py
go executable not found, current path: '/usr/bin:/bin'
# exit code: 1
```

2. **Fontconfig + ≥1 font family.** `env_for_python_tests()` at `[kitty_tests/main.py:L297]` eagerly loads the system font DB — `from kitty.fonts.common import all_fonts_map` `[kitty_tests/main.py:L313]` then `all_fonts_map(True)` `[kitty_tests/main.py:L314]`. **Observed:** fontconfig 2.15.0 with 267 unique families (511 face rows) present, so env setup proceeds. The absence of one *specific* family ("Source Code Pro", grep count 0 above) produces the `test_font_selection` baseline artifact (below); it does **not** block env setup.
3. **Native libraries** per `[docs/build.rst:L76]` (§Dependencies): run-time `harfbuzz >= 2.2.0` `[docs/build.rst:L84]`, `libpng` `[docs/build.rst:L86]`, `liblcms2` `[docs/build.rst:L87]`, `libxxhash` `[docs/build.rst:L88]`, `openssl` `[docs/build.rst:L89]`, `freetype` `[docs/build.rst:L90]`, `fontconfig` `[docs/build.rst:L91]`; plus the build-time `-dev` packages `[docs/build.rst:L107-L120]`, a C compiler, and `pkg-config`; plus OpenGL, xkbcommon, wayland, x11, dbus for the GLFW backends. **Observed:** all present at the versions in the table above; the fallback build (Q1.a) links every artifact.

---
## Q1 — Build kitty from source and run the suite via the canonical entry point

### Q1.a — Canonical build commands (default, then documented fallback)

The default build action is `build` (`[setup.py:L175]` `action: str = 'build'`; the argparse positional `action` at `[setup.py:L1827]` has `nargs='?'` `[setup.py:L1828]` and `default=Options.action` `[setup.py:L1829]`), and `make all` maps to it (`[Makefile:L12-L13]` `all:` → `python3 setup.py $(VVAL)`). So the canonical build command is `python3 setup.py`.

**Step 1 — (optional) clear the git-ignored object cache for a from-scratch compile.** kitty caches compiled `.o` files under `build/` (git-ignored at `[.gitignore:L14]` `/build/`). Clearing it forces a full, first-principles recompile so the output below is the complete 122-unit build rather than an incremental subset. Clearing is **not** what makes the failure reproduce, however: the *populated-cache variant* observed after Step 2 shows that `wl_window.c` is recompiled and the abort re-triggered **even against a fully populated cache**. The clean-cache action, captured with its command and exit code:

```text
$ ls -d build 2>/dev/null && echo "build/ object cache present"
build
build/ object cache present
$ rm -rf build   # git-ignored C-object cache (.gitignore:L14 "/build/"); optional — yields a from-scratch compile (the -Werror abort also reproduces on a populated cache; see the populated-cache variant below)
# exit code: 0
$ ls -d build 2>&1
ls: cannot access 'build': No such file or directory
```

**Step 2 — run the default build.** **Observed: the default `python3 setup.py` ABORTS on the Wayland backend (exit 1).** The system's `wayland-protocols` is **1.45** (see versions table) — newer than this kitty snapshot targets — and introduces `XDG_TOPLEVEL_STATE_CONSTRAINED_{LEFT,RIGHT,TOP,BOTTOM}` enum values not handled by the `switch (*state)` at `[glfw/wl_window.c:L668]` (function `xdgToplevelHandleConfigure`). The default build compiles with `-pedantic-errors -Werror` (`werror = '' if ignore_compiler_warnings else '-pedantic-errors -Werror'` at `[setup.py:L491]`), so `-Werror=switch` promotes the unhandled-enum warning to a fatal error. The **complete, unedited** output (all 122 compile lines, the error, and the verbose failing `gcc` command that setup.py re-prints), with command and exit code:

```text
$ python3 setup.py
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
# exit code: 1
```

> **This is an ENVIRONMENT-VERSION ARTIFACT, not a defect in the C extensions under investigation, and it is deliberately NOT patched** (the source tree is read-only per MainRule). The last output line above is the exact failing compile command; note `-pedantic-errors -Werror` applied by the default build. The switch at `[glfw/wl_window.c:L668]` handles `XDG_TOPLEVEL_STATE_{RESIZING,MAXIMIZED,FULLSCREEN,ACTIVATED,TILED_*}` (and `SUSPENDED` under an `#ifdef`) but not the newer `CONSTRAINED_*` values shipped by `wayland-protocols 1.45`.

**Populated-cache variant (observed) — the abort does not depend on clearing `build/`.** To confirm the default-build failure is a property of the build *configuration* (not an artifact of starting from an empty cache), the default build was re-run inside a throwaway copy of the tree whose `build/` had first been **fully populated** by a successful `--ignore-compiler-warnings` build (122 cached `.o` files, including `build/glfw-wayland-glfw-wl_window.c.o` at 342048 bytes). With that cache present and **not cleared**, `python3 setup.py` **still aborts (exit 1)**: it recompiles `wl_window.c` (unit `[3/122]`) and re-emits the identical four `-Werror=switch` errors. **Cause → effect:** setup.py rebuilds any translation unit whose compile command changed — the incremental guard at `[setup.py:L117]` consults `cmd_changed()` (`[setup.py:L134-L136]`: `return bool(self.db.get(key) != cmd)`), which compares the command recorded in the compilation database against the current one. Dropping `--ignore-compiler-warnings` re-adds `-pedantic-errors -Werror` to the `wl_window.c` command (the `werror` gate at `[setup.py:L491]`), so `cmd_changed()` returns `True` and the unit is rebuilt with `-Werror` regardless of the cached object — which is why deleting `build/` is a convenience for a clean full log, not a precondition for the failure. Command and key output (excerpt — the full 122-line compile sequence and the failing `gcc` command are byte-identical to Step 2 above):

```text
$ python3 setup.py                # build/ fully populated by a prior --ignore-compiler-warnings build; NOT cleared
[3/122] Compiling [wayland] glfw/wl_window.c ...
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_LEFT’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_RIGHT’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_TOP’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_BOTTOM’ not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
# exit code: 1
```

**Step 3 — run the documented fallback.** The project ships a flag to drop `-Werror` for exactly this situation: `--ignore-compiler-warnings` (`[setup.py:L2003]`, `action='store_true'` at `[setup.py:L2004]`), which flips the same `werror` gate at `[setup.py:L491]` (and the launcher's gate at `[setup.py:L1231]`) to the empty string. **Observed: `python3 setup.py --ignore-compiler-warnings` completes with exit 0**, links all five artifact groups, then builds the Go `kitten`. Complete, unedited output with command and exit code:

```text
$ python3 setup.py --ignore-compiler-warnings
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
# exit code: 0
```

The final `kitty/tools/cmd` line above is the Go build of the `kitten` binary (`go build ... tools/cmd`, see linkage note below).

**Produced artifacts (all six), with linkage evidence.** The four `.so` files plus the two launcher binaries, then `file`/`ldd` on each executable — complete, unedited:

```text
$ ls -l kitty/fast_data_types.so kittens/transfer/rsync.so kitty/glfw-x11.so kitty/glfw-wayland.so kitty/launcher/kitty kitty/launcher/kitten
-rwxr-xr-x 1 root root    42824 Jul 14 20:38 kittens/transfer/rsync.so
-rwxr-xr-x 1 root root  1253792 Jul 14 20:38 kitty/fast_data_types.so
-rwxr-xr-x 1 root root   451016 Jul 14 20:38 kitty/glfw-wayland.so
-rwxr-xr-x 1 root root   373896 Jul 14 20:38 kitty/glfw-x11.so
-rwxr-xr-x 1 root root 15765764 Jul 14 20:38 kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 Jul 14 20:38 kitty/launcher/kitty
$ file kitty/launcher/kitten
kitty/launcher/kitten: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, Go BuildID=Z9nOhn8sTV1u7M99E6DY/gsKmmSe6ViPbUx6XUYp-/EFn9-HIvWxR5B0oYFn4A/IpzISy9u670xL2Wwu9cA, stripped
$ ldd kitty/launcher/kitten
	linux-vdso.so.1 (0x00007ffca736a000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007db799438000)
	/lib64/ld-linux-x86-64.so.2 (0x00007db799684000)
$ file kitty/launcher/kitty
kitty/launcher/kitty: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=4a693e4304285476522c8ac6a4eef4babf9072b7, for GNU/Linux 3.2.0, not stripped
$ ldd kitty/launcher/kitty
	linux-vdso.so.1 (0x00007ffed194c000)
	libpython3.13.so.1.0 => /lib/x86_64-linux-gnu/libpython3.13.so.1.0 (0x00007b1043b97000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007b1043954000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007b1043847000)
	libz.so.1 => /lib/x86_64-linux-gnu/libz.so.1 (0x00007b1043827000)
	libexpat.so.1 => /lib/x86_64-linux-gnu/libexpat.so.1 (0x00007b10437fa000)
	/lib64/ld-linux-x86-64.so.2 (0x00007b1044346000)
$ file kitty/fast_data_types.so kittens/transfer/rsync.so kitty/glfw-x11.so kitty/glfw-wayland.so
kitty/fast_data_types.so:  ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=3c8886f43ac8b1e7092372a27c2e11dcf86cebe5, not stripped
kittens/transfer/rsync.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=e77ae85d0520d5bd6e7983718fcf8b50c4c06bf3, not stripped
kitty/glfw-x11.so:         ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=ea79d3ae59f4902a43f9feadb77734f190e5388b, not stripped
kitty/glfw-wayland.so:     ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=1babec9fd6038260164f8b154cd4c1ac05fc07e6, not stripped
```

Runtime smoke test of the built binaries:

```text
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
$ ./kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
```

**The `kitten` binary is a *dynamically-linked* Go executable in the normal build (not static).** `file` reports `dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, Go BuildID=..., stripped`, and `ldd` resolves `libc.so.6` and `ld-linux` — i.e. it links the system C library. The build function is named `build_static_kittens()` (`[setup.py:L1130]`), but that name is a role label, **not** a linkage guarantee: `CGO_ENABLED='0'` (which forces a static, CGO-free build) is set **only** inside `if for_platform:` at `[setup.py:L1172-L1175]`, i.e. only for cross-platform release builds. The **normal** non-macOS build calls it *without* `for_platform` at `[setup.py:L2121]` (`build_static_kittens(args, launcher_dir=launcher_dir)`, inside the `args.action == 'build'` branch), so CGO stays enabled and the resulting `kitten` is dynamically linked — exactly as observed. (For contrast, the cross-platform path that sets `CGO_ENABLED='0'` is `[setup.py:L1203]`, used for standalone/other-arch release binaries.) The launcher `kitty` is itself a dynamically-linked PIE that loads `libpython3.13.so.1.0` (embedded interpreter), as `ldd` shows; the four `.so` extensions are ELF shared objects (`not stripped`).

**Which function builds what** (structural, cited):
- `kitty/fast_data_types.so` — `compile_c_extension()` `[setup.py:L856]`, invoked for `'kitty/fast_data_types'` at `[setup.py:L1091]`; sources gathered by `find_c_files()` `[setup.py:L906]`.
- `kittens/transfer/rsync.so` — `compile_kittens()` `[setup.py:L967]`, specifically the `files('transfer', 'rsync', libraries=pkg_config('libxxhash', ...))` entry at `[setup.py:L986]`.
- `kitty/glfw-x11.so`, `kitty/glfw-wayland.so` — `compile_glfw()` `[setup.py:L932]`, output name `f'kitty/glfw-{module}'` at `[setup.py:L953]`.
- `kitty/launcher/kitty` — `build_launcher()` `[setup.py:L1230]` (its own `-Werror` gate at `[setup.py:L1231]`). The launcher hard-codes the relative path to the kitty Python library: for the default `source` bundle, `klp = os.path.relpath('.', launcher_dir)` at `[setup.py:L1264]`, which is compiled in as `-DKITTY_LIB_PATH="{klp}"` at `[setup.py:L1270]`. At runtime `main()` resolves its own directory from `/proc/self/exe` (`read_exe_path()` at `[kitty/launcher/main.c:L281]`, called at `[kitty/launcher/main.c:L449]`) and concatenates it with `KITTY_LIB_PATH` to locate the library at `[kitty/launcher/main.c:L449-L460]` (`snprintf(lib, PATH_MAX, "%s/%s", exe_dir, KITTY_LIB_PATH)`). This is why the launcher always loads the extensions sitting next to it — the property the cascade experiment (Q4) relies on.
- `kitty/launcher/kitten` — `build_static_kittens()` `[setup.py:L1130]`, normal call at `[setup.py:L2121]` (Go `go build ... tools/cmd`).

### Q1.b — Canonical test invocation

`test.py` is the canonical entry point: its shebang is `#!./kitty/launcher/kitty +launch` `[test.py:L1]`, and it does `m = importlib.import_module('kitty_tests.main')` `[test.py:L8]` then `getattr(m, 'main')()` `[test.py:L9]`. (`setup.py`'s own `test` action executes the same launcher: `os.execl(texe, texe, '+launch', 'test.py')` at `[setup.py:L2103]`.) The exact canonical run command — with `CI=true` for headless mode and Go on `PATH` — is:

```bash
CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 PATH="$PATH:/usr/local/go/bin" ./kitty/launcher/kitty +launch test.py
```

The run begins by printing an environment banner from `env_for_python_tests()` `[kitty_tests/main.py:L297]` (`print('Running under CI:', BaseTest.is_ci)` at `[kitty_tests/main.py:L305]`). *(excerpt — banner only; the complete run output is embedded in full in Q1.c below):*

```text
Running under CI: True
Using PATH in test environment: /tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty_tests/kitty/launcher:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/go/bin
Python: /usr/bin/python
Intrinsics: has_avx2=True has_sse4_2=True
Go executable: /usr/local/bin/go
Go packages being tested: tools/tui/shell_integration tools/utils/shlex tools/utils/humanize kittens/hints kittens/transfer tools/utils/shm tools/cli tools/tui/sgr kittens/ssh kittens/hyperlinked_grep tools/cmd/at tools/config tools/rsync tools/tui tools/tui/readline tools/unicode_names tools/utils/style tools/utils tools/themes tools/tui/loop tools/simdstring tools/wcswidth tools/tui/graphics tools/tui/subseq tools/utils/base85 kittens/diff
```

**Observed — the `Go packages being tested:` order is produced by Python, not Go.** The list is enumerated in a **non-deterministic order** that differs run-to-run (compare the banner above with Tier 2 in Q4), while the *set* of packages is identical (26 packages) across runs. The order originates in a **Python `set`**: `find_testable_go_packages()` accumulates packages into `ans = set()` (`[kitty_tests/main.py:L129]`), adds each discovered directory with `ans.add(q)` (`[kitty_tests/main.py:L136]`), and returns it as `Set[str]` (`[kitty_tests/main.py:L141]`); `reduce_go_pkgs()` obtains that set at `[kitty_tests/main.py:L199]` and returns it (possibly filtered, still a set) at `[kitty_tests/main.py:L207]`; and `run_tests()` prints it **unsorted** via `print('Go packages being tested:', ' '.join(go_pkgs))` at `[kitty_tests/main.py:L277]`. Because CPython randomizes `str` hashing per process, the iteration order of that set of strings — and therefore the joined banner order — is fixed by `PYTHONHASHSEED`. Driving the *real* `find_testable_go_packages()` through the launcher under fixed seeds isolates the cause. *(This probe invokes the real, unmodified function but is **non-canonical**: it bypasses the full `test.py` → `main()` flow to print the set directly.)* A given seed reproduces the same order, a different seed reorders the list, and the *sorted* set is identical every time — complete, unedited output:

```text
$ cat /tmp/hashseed_probe.py
import os
from kitty_tests.main import find_testable_go_packages
pkgs, _ = find_testable_go_packages()
seed = os.environ.get("PYTHONHASHSEED")
print("SEED=" + str(seed) + " n=" + str(len(pkgs)))
print("ITER: " + " ".join(pkgs))
print("SORTED: " + " ".join(sorted(pkgs)))
$ for s in 1 2 1; do CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 PYTHONHASHSEED=$s PATH="$PATH:/usr/local/go/bin" ./kitty/launcher/kitty +launch /tmp/hashseed_probe.py; done
SEED=1 n=26
ITER: tools/simdstring tools/tui/sgr tools/tui/shell_integration tools/unicode_names tools/themes tools/utils/base85 tools/wcswidth tools/rsync tools/cmd/at tools/tui/loop tools/utils/shlex kittens/hyperlinked_grep tools/tui/graphics tools/config tools/utils/shm tools/utils kittens/transfer tools/cli tools/utils/humanize tools/tui/subseq kittens/diff kittens/ssh tools/tui/readline tools/utils/style tools/tui kittens/hints
SORTED: kittens/diff kittens/hints kittens/hyperlinked_grep kittens/ssh kittens/transfer tools/cli tools/cmd/at tools/config tools/rsync tools/simdstring tools/themes tools/tui tools/tui/graphics tools/tui/loop tools/tui/readline tools/tui/sgr tools/tui/shell_integration tools/tui/subseq tools/unicode_names tools/utils tools/utils/base85 tools/utils/humanize tools/utils/shlex tools/utils/shm tools/utils/style tools/wcswidth
SEED=2 n=26
ITER: kittens/ssh kittens/transfer kittens/hints tools/wcswidth tools/cmd/at tools/simdstring tools/utils tools/rsync kittens/hyperlinked_grep tools/utils/shlex tools/config tools/tui/graphics tools/tui/shell_integration tools/tui/readline tools/unicode_names tools/utils/humanize tools/themes tools/utils/style tools/tui/sgr tools/utils/base85 tools/tui/loop tools/tui tools/tui/subseq tools/cli kittens/diff tools/utils/shm
SORTED: kittens/diff kittens/hints kittens/hyperlinked_grep kittens/ssh kittens/transfer tools/cli tools/cmd/at tools/config tools/rsync tools/simdstring tools/themes tools/tui tools/tui/graphics tools/tui/loop tools/tui/readline tools/tui/sgr tools/tui/shell_integration tools/tui/subseq tools/unicode_names tools/utils tools/utils/base85 tools/utils/humanize tools/utils/shlex tools/utils/shm tools/utils/style tools/wcswidth
SEED=1 n=26
ITER: tools/simdstring tools/tui/sgr tools/tui/shell_integration tools/unicode_names tools/themes tools/utils/base85 tools/wcswidth tools/rsync tools/cmd/at tools/tui/loop tools/utils/shlex kittens/hyperlinked_grep tools/tui/graphics tools/config tools/utils/shm tools/utils kittens/transfer tools/cli tools/utils/humanize tools/tui/subseq kittens/diff kittens/ssh tools/tui/readline tools/utils/style tools/tui kittens/hints
SORTED: kittens/diff kittens/hints kittens/hyperlinked_grep kittens/ssh kittens/transfer tools/cli tools/cmd/at tools/config tools/rsync tools/simdstring tools/themes tools/tui tools/tui/graphics tools/tui/loop tools/tui/readline tools/tui/sgr tools/tui/shell_integration tools/tui/subseq tools/unicode_names tools/utils tools/utils/base85 tools/utils/humanize tools/utils/shlex tools/utils/shm tools/utils/style tools/wcswidth
```

The two `SEED=1` runs are **byte-identical** while `SEED=2` differs, and `SORTED` matches across all three — so the ordering is **Python set-iteration order (hash-seed dependent)**, not Go's doing; `go test` receives the same 26-package set regardless of the printed order. **Correction:** an earlier draft attributed this variation to "Go's map/goroutine iteration"; that is incorrect — the order is produced entirely on the Python side at `[kitty_tests/main.py:L277]` (`' '.join()` over a Python `set`) *before* any package name is handed to `go test` by `run_go()` at `[kitty_tests/main.py:L270]`.

### Q1.c — Two independent runs, complete output, and stability

Each baseline log was created by redirecting the canonical command's merged stdout/stderr to a file, e.g. `... ./kitty/launcher/kitty +launch test.py > baseline_run1_full.txt 2>&1; echo "# exit code: $?"`. Both runs were executed with the identical canonical command. **Complete, unedited output of RUN #1** (command and exit code included; all 145 test lines, three failure tracebacks, and the summary):

```text
$ CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 PATH="$PATH:/usr/local/go/bin" ./kitty/launcher/kitty +launch test.py   # (run #1; stdout+stderr)
Running under CI: True
Using PATH in test environment: /tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty_tests/kitty/launcher:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/go/bin
Python: /usr/bin/python
Intrinsics: has_avx2=True has_sse4_2=True
Go executable: /usr/local/bin/go
Go packages being tested: tools/tui/shell_integration tools/utils/shlex tools/utils/humanize kittens/hints kittens/transfer tools/utils/shm tools/cli tools/tui/sgr kittens/ssh kittens/hyperlinked_grep tools/cmd/at tools/config tools/rsync tools/tui tools/tui/readline tools/unicode_names tools/utils/style tools/utils tools/themes tools/tui/loop tools/simdstring tools/wcswidth tools/tui/graphics tools/tui/subseq tools/utils/base85 kittens/diff
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
test_transfer_receive (kitty_tests.file_transmission.TestFileTransmission.test_transfer_receive) ... FAIL
test_transfer_send (kitty_tests.file_transmission.TestFileTransmission.test_transfer_send) ... FAIL
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
test_font_selection (kitty_tests.fonts.Selection.test_font_selection) ... FAIL
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

======================================================================
FAIL: test_transfer_receive (kitty_tests.file_transmission.TestFileTransmission.test_transfer_receive)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/file_transmission.py", line 476, in test_transfer_receive
    self.basic_transfer_tests()
    ~~~~~~~~~~~~~~~~~~~~~~~~~^^
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/file_transmission.py", line 443, in basic_transfer_tests
    multiple_files()
    ~~~~~~~~~~~~~~^^
    d = <_io.BufferedWriter name='/tmp/tmpl0tttl8z/dest'>
    dest = '/tmp/tmpl0tttl8z/dest'
    multiple_files = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files at 0x7fc822454220>
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7fc8230c9160>
    s = <_io.BufferedWriter name='/tmp/tmpl0tttl8z/src'>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
    single_file = <function TestFileTransmission.basic_transfer_tests.<locals>.single_file at 0x7fc82270bd80>
    src = '/tmp/tmpl0tttl8z/src'
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/file_transmission.py", line 432, in multiple_files
    self.assertEqual(expected, actual)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^
    Entry = <class 'kitty_tests.file_transmission.Entry'>
    actual = {'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784061533649403356, mode='0o120777', nlink=1), 'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'sym': Entry(relpath='sym', mtime=1784061533649403356, mode='0o120777', nlink=1), 'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1)}
    b = PosixPath('/tmp/tmpl0tttl8z/msrc')
    cmd = ()
    de = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.de at 0x7fc822456c00>
    dest = '/tmp/tmpl0tttl8z/mdest'
    dirnames = []
    dirpath = '/tmp/tmpl0tttl8z/mdest/msrc/empty'
    entry = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.entry at 0x7fc822455f80>
    expected = {'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1), 'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784061533649403356, mode='0o120777', nlink=1), 'sym': Entry(relpath='sym', mtime=1784061533649403356, mode='0o120777', nlink=1)}
    f = <_io.BufferedWriter name='/tmp/tmpl0tttl8z/msrc/sub/reg'>
    filenames = []
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7fc823bb1c50>
    s = PosixPath('/tmp/tmpl0tttl8z/msrc/sub')
    se = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.se at 0x7fc822456020>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
    src = '/tmp/tmpl0tttl8z/msrc'
    x = 'reg'
AssertionError: {'simple': Entry(relpath='simple', mtime=130[497 chars]k=1)} != {'sub': Entry(relpath='sub', mtime=0, mode='[497 chars]k=1)}
  {'abssym': Entry(relpath='abssym', mtime=1784061533649403356, mode='0o120777', nlink=1),
-  'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2),
?                                                    ^

+  'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2),
?                                                    ^

   'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2),
   'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2),
-  'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2),
?                                                ^

+  'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2),
?                                                ^

   'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1),
   'sym': Entry(relpath='sym', mtime=1784061533649403356, mode='0o120777', nlink=1)}

======================================================================
FAIL: test_transfer_send (kitty_tests.file_transmission.TestFileTransmission.test_transfer_send)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/file_transmission.py", line 507, in test_transfer_send
    self.basic_transfer_tests()
    ~~~~~~~~~~~~~~~~~~~~~~~~~^^
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/file_transmission.py", line 443, in basic_transfer_tests
    multiple_files()
    ~~~~~~~~~~~~~~^^
    d = <_io.BufferedWriter name='/tmp/tmpvucds7qd/dest'>
    dest = '/tmp/tmpvucds7qd/dest'
    multiple_files = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files at 0x7fc82246d120>
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7fc82260a7b0>
    s = <_io.BufferedWriter name='/tmp/tmpvucds7qd/src'>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
    single_file = <function TestFileTransmission.basic_transfer_tests.<locals>.single_file at 0x7fc822456fc0>
    src = '/tmp/tmpvucds7qd/src'
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/file_transmission.py", line 432, in multiple_files
    self.assertEqual(expected, actual)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^
    Entry = <class 'kitty_tests.file_transmission.Entry'>
    actual = {'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784061534232409637, mode='0o120777', nlink=1), 'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'sym': Entry(relpath='sym', mtime=1784061534232409637, mode='0o120777', nlink=1), 'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1)}
    b = PosixPath('/tmp/tmpvucds7qd/msrc')
    cmd = ()
    de = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.de at 0x7fc82246e340>
    dest = '/tmp/tmpvucds7qd/mdest'
    dirnames = []
    dirpath = '/tmp/tmpvucds7qd/mdest/msrc/empty'
    entry = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.entry at 0x7fc82246d760>
    expected = {'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1), 'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784061534232409637, mode='0o120777', nlink=1), 'sym': Entry(relpath='sym', mtime=1784061534232409637, mode='0o120777', nlink=1)}
    f = <_io.BufferedWriter name='/tmp/tmpvucds7qd/msrc/sub/reg'>
    filenames = []
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7fc8226ccc00>
    s = PosixPath('/tmp/tmpvucds7qd/msrc/sub')
    se = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.se at 0x7fc82246d800>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
    src = '/tmp/tmpvucds7qd/msrc'
    x = 'reg'
AssertionError: {'simple': Entry(relpath='simple', mtime=130[497 chars]k=1)} != {'sub': Entry(relpath='sub', mtime=0, mode='[497 chars]k=1)}
  {'abssym': Entry(relpath='abssym', mtime=1784061534232409637, mode='0o120777', nlink=1),
-  'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2),
?                                                    ^

+  'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2),
?                                                    ^

   'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2),
   'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2),
-  'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2),
?                                                ^

+  'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2),
?                                                ^

   'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1),
   'sym': Entry(relpath='sym', mtime=1784061534232409637, mode='0o120777', nlink=1)}

======================================================================
FAIL: test_font_selection (kitty_tests.fonts.Selection.test_font_selection)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/fonts.py", line 64, in test_font_selection
    t('Source Code Pro', 'SourceCodePro', 'Semibold', 'It')
    ~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    both = <function Selection.test_font_selection.<locals>.both at 0x7fc8210e2ca0>
    has = <function Selection.test_font_selection.<locals>.has at 0x7fc8210e0180>
    names = {'noto sans signwriting', 'jetbrains mono', 'noto mono', 'jetbrains mono nl', 'dejavu sans mono', 'ubuntu mono', 'inconsolata', 'liberation mono', 'fira code', 'ubuntu sans mono'}
    opts = <kitty.options.types.Options object at 0x7fc8211120d0>
    s = <function Selection.test_font_selection.<locals>.s at 0x7fc8210e1f80>
    self = <kitty_tests.fonts.Selection testMethod=test_font_selection>
    t = <function Selection.test_font_selection.<locals>.t at 0x7fc8210e2fc0>
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/fonts.py", line 58, in t
    if has(family, allow_missing_in_ci=allow_missing_in_ci):
       ~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    allow_missing_in_ci = False
    alternate = None
    bi = ''
    bold = 'Semibold'
    both = <function Selection.test_font_selection.<locals>.both at 0x7fc8210e2ca0>
    family = 'Source Code Pro'
    has = <function Selection.test_font_selection.<locals>.has at 0x7fc8210e0180>
    italic = 'It'
    psprefix = 'SourceCodePro'
    reg = 'Regular'
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/fonts.py", line 54, in has
    raise AssertionError(f'The family: {family} is not available')
    allow_missing_in_ci = False
    ans = False
    family = 'Source Code Pro'
    names = {'noto sans signwriting', 'jetbrains mono', 'noto mono', 'jetbrains mono nl', 'dejavu sans mono', 'ubuntu mono', 'inconsolata', 'liberation mono', 'fira code', 'ubuntu sans mono'}
    self = <kitty_tests.fonts.Selection testMethod=test_font_selection>
AssertionError: The family: Source Code Pro is not available

----------------------------------------------------------------------
Ran 145 tests in 15.787s

FAILED (failures=3, skipped=4)
All Go tests succeeded, ran in 15.9 seconds
[31mError[39m: Some tests failed!
# exit code: 1
```

**Complete, unedited output of RUN #2** (identical command; counts identical, only wall-clock time and Go-package ordering differ):

```text
$ CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 PATH="$PATH:/usr/local/go/bin" ./kitty/launcher/kitty +launch test.py   # (run #2; stdout+stderr)
Running under CI: True
Using PATH in test environment: /tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty_tests/kitty/launcher:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/go/bin
Python: /usr/bin/python
Intrinsics: has_avx2=True has_sse4_2=True
Go executable: /usr/local/bin/go
Go packages being tested: tools/simdstring tools/tui/subseq tools/utils/shlex tools/cmd/at tools/tui kittens/hyperlinked_grep tools/cli tools/tui/sgr tools/utils/humanize tools/utils/style tools/themes kittens/hints tools/utils tools/config tools/rsync kittens/ssh tools/unicode_names kittens/diff kittens/transfer tools/tui/graphics tools/tui/readline tools/tui/loop tools/utils/shm tools/utils/base85 tools/tui/shell_integration tools/wcswidth
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
test_transfer_receive (kitty_tests.file_transmission.TestFileTransmission.test_transfer_receive) ... FAIL
test_transfer_send (kitty_tests.file_transmission.TestFileTransmission.test_transfer_send) ... FAIL
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
test_font_selection (kitty_tests.fonts.Selection.test_font_selection) ... FAIL
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

======================================================================
FAIL: test_transfer_receive (kitty_tests.file_transmission.TestFileTransmission.test_transfer_receive)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/file_transmission.py", line 476, in test_transfer_receive
    self.basic_transfer_tests()
    ~~~~~~~~~~~~~~~~~~~~~~~~~^^
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/file_transmission.py", line 443, in basic_transfer_tests
    multiple_files()
    ~~~~~~~~~~~~~~^^
    d = <_io.BufferedWriter name='/tmp/tmpvipyp6tg/dest'>
    dest = '/tmp/tmpvipyp6tg/dest'
    multiple_files = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files at 0x7b492f92c220>
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7b493519d160>
    s = <_io.BufferedWriter name='/tmp/tmpvipyp6tg/src'>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
    single_file = <function TestFileTransmission.basic_transfer_tests.<locals>.single_file at 0x7b492fbebd80>
    src = '/tmp/tmpvipyp6tg/src'
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/file_transmission.py", line 432, in multiple_files
    self.assertEqual(expected, actual)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^
    Entry = <class 'kitty_tests.file_transmission.Entry'>
    actual = {'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784061549592575112, mode='0o120777', nlink=1), 'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'sym': Entry(relpath='sym', mtime=1784061549592575112, mode='0o120777', nlink=1), 'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1)}
    b = PosixPath('/tmp/tmpvipyp6tg/msrc')
    cmd = ()
    de = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.de at 0x7b492f92ec00>
    dest = '/tmp/tmpvipyp6tg/mdest'
    dirnames = []
    dirpath = '/tmp/tmpvipyp6tg/mdest/msrc/empty'
    entry = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.entry at 0x7b492f92df80>
    expected = {'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1), 'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784061549592575112, mode='0o120777', nlink=1), 'sym': Entry(relpath='sym', mtime=1784061549592575112, mode='0o120777', nlink=1)}
    f = <_io.BufferedWriter name='/tmp/tmpvipyp6tg/msrc/sub/reg'>
    filenames = []
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7b4935141c50>
    s = PosixPath('/tmp/tmpvipyp6tg/msrc/sub')
    se = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.se at 0x7b492f92e020>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
    src = '/tmp/tmpvipyp6tg/msrc'
    x = 'reg'
AssertionError: {'simple': Entry(relpath='simple', mtime=130[497 chars]k=1)} != {'sub': Entry(relpath='sub', mtime=0, mode='[497 chars]k=1)}
  {'abssym': Entry(relpath='abssym', mtime=1784061549592575112, mode='0o120777', nlink=1),
-  'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2),
?                                                    ^

+  'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2),
?                                                    ^

   'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2),
   'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2),
-  'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2),
?                                                ^

+  'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2),
?                                                ^

   'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1),
   'sym': Entry(relpath='sym', mtime=1784061549592575112, mode='0o120777', nlink=1)}

======================================================================
FAIL: test_transfer_send (kitty_tests.file_transmission.TestFileTransmission.test_transfer_send)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/file_transmission.py", line 507, in test_transfer_send
    self.basic_transfer_tests()
    ~~~~~~~~~~~~~~~~~~~~~~~~~^^
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/file_transmission.py", line 443, in basic_transfer_tests
    multiple_files()
    ~~~~~~~~~~~~~~^^
    d = <_io.BufferedWriter name='/tmp/tmpyawe3ys3/dest'>
    dest = '/tmp/tmpyawe3ys3/dest'
    multiple_files = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files at 0x7b492f945120>
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7b492fae67b0>
    s = <_io.BufferedWriter name='/tmp/tmpyawe3ys3/src'>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
    single_file = <function TestFileTransmission.basic_transfer_tests.<locals>.single_file at 0x7b492f92efc0>
    src = '/tmp/tmpyawe3ys3/src'
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/file_transmission.py", line 432, in multiple_files
    self.assertEqual(expected, actual)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^
    Entry = <class 'kitty_tests.file_transmission.Entry'>
    actual = {'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784061550158581210, mode='0o120777', nlink=1), 'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'sym': Entry(relpath='sym', mtime=1784061550158581210, mode='0o120777', nlink=1), 'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1)}
    b = PosixPath('/tmp/tmpyawe3ys3/msrc')
    cmd = ()
    de = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.de at 0x7b492f946340>
    dest = '/tmp/tmpyawe3ys3/mdest'
    dirnames = []
    dirpath = '/tmp/tmpyawe3ys3/mdest/msrc/empty'
    entry = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.entry at 0x7b492f945760>
    expected = {'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1), 'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784061550158581210, mode='0o120777', nlink=1), 'sym': Entry(relpath='sym', mtime=1784061550158581210, mode='0o120777', nlink=1)}
    f = <_io.BufferedWriter name='/tmp/tmpyawe3ys3/msrc/sub/reg'>
    filenames = []
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7b492fbacc00>
    s = PosixPath('/tmp/tmpyawe3ys3/msrc/sub')
    se = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.se at 0x7b492f945800>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
    src = '/tmp/tmpyawe3ys3/msrc'
    x = 'reg'
AssertionError: {'simple': Entry(relpath='simple', mtime=130[497 chars]k=1)} != {'sub': Entry(relpath='sub', mtime=0, mode='[497 chars]k=1)}
  {'abssym': Entry(relpath='abssym', mtime=1784061550158581210, mode='0o120777', nlink=1),
-  'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2),
?                                                    ^

+  'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2),
?                                                    ^

   'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2),
   'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2),
-  'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2),
?                                                ^

+  'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2),
?                                                ^

   'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1),
   'sym': Entry(relpath='sym', mtime=1784061550158581210, mode='0o120777', nlink=1)}

======================================================================
FAIL: test_font_selection (kitty_tests.fonts.Selection.test_font_selection)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/fonts.py", line 64, in test_font_selection
    t('Source Code Pro', 'SourceCodePro', 'Semibold', 'It')
    ~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    both = <function Selection.test_font_selection.<locals>.both at 0x7b492e612ca0>
    has = <function Selection.test_font_selection.<locals>.has at 0x7b492e610180>
    names = {'jetbrains mono nl', 'noto sans signwriting', 'ubuntu sans mono', 'liberation mono', 'inconsolata', 'noto mono', 'dejavu sans mono', 'jetbrains mono', 'fira code', 'ubuntu mono'}
    opts = <kitty.options.types.Options object at 0x7b492e6420d0>
    s = <function Selection.test_font_selection.<locals>.s at 0x7b492e611f80>
    self = <kitty_tests.fonts.Selection testMethod=test_font_selection>
    t = <function Selection.test_font_selection.<locals>.t at 0x7b492e612fc0>
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/fonts.py", line 58, in t
    if has(family, allow_missing_in_ci=allow_missing_in_ci):
       ~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    allow_missing_in_ci = False
    alternate = None
    bi = ''
    bold = 'Semibold'
    both = <function Selection.test_font_selection.<locals>.both at 0x7b492e612ca0>
    family = 'Source Code Pro'
    has = <function Selection.test_font_selection.<locals>.has at 0x7b492e610180>
    italic = 'It'
    psprefix = 'SourceCodePro'
    reg = 'Regular'
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/fonts.py", line 54, in has
    raise AssertionError(f'The family: {family} is not available')
    allow_missing_in_ci = False
    ans = False
    family = 'Source Code Pro'
    names = {'jetbrains mono nl', 'noto sans signwriting', 'ubuntu sans mono', 'liberation mono', 'inconsolata', 'noto mono', 'dejavu sans mono', 'jetbrains mono', 'fira code', 'ubuntu mono'}
    self = <kitty_tests.fonts.Selection testMethod=test_font_selection>
AssertionError: The family: Source Code Pro is not available

----------------------------------------------------------------------
Ran 145 tests in 21.884s

FAILED (failures=3, skipped=4)
All Go tests succeeded, ran in 21.9 seconds
[31mError[39m: Some tests failed!
# exit code: 1
```

**Stability (Rule 1):** both runs report **`Ran 145 tests`**, **`FAILED (failures=3, skipped=4)`**, `All Go tests succeeded`, and both exit **1**. The three failing test ids are identical across both runs (verified with `diff` of the extracted `FAIL:`/`ERROR:` lines):

```text
FAIL: test_transfer_receive (kitty_tests.file_transmission.TestFileTransmission.test_transfer_receive)
FAIL: test_transfer_send (kitty_tests.file_transmission.TestFileTransmission.test_transfer_send)
FAIL: test_font_selection (kitty_tests.fonts.Selection.test_font_selection)
```

All three are **environment artifacts** (tests that executed and asserted false), not extension-load failures — dissected in the dedicated section near the end. Note the causal proof that **the extensions loaded**: all three failing tests *ran their bodies* (they reached assertions), which is only possible if `fast_data_types` and `rsync` imported successfully. As an independent corroboration, scanning each run's complete output for import-failure strings returns **zero** — this proves only the *absence of those strings*, not loading per se; the positive proof of loading is the module probe in Q3:

```text
$ grep -cE "ModuleNotFoundError|ImportError|Failed to import" baseline_run1_full.txt
0
$ grep -cE "ModuleNotFoundError|ImportError|Failed to import" baseline_run2_full.txt
0
```

---
## Q2 — How the compiled C extensions connect to test-execution mechanics · Q8 — The actual import chains

### Q2 — The relationship, one sentence per extension (observed)

- **`kitty/fast_data_types.so` → every test.** It backs the `BaseTest` base class (`kitty_tests/__init__.py`), which imports it both *transitively* (`from kitty.config import …` at `[kitty_tests/__init__.py:L21]`) and *directly* (`from kitty.fast_data_types import Cursor, HistoryBuf, LineBuf, Screen, …` at `[kitty_tests/__init__.py:L22]`); since all 22 test categories subclass `BaseTest`, the whole suite depends on it. **Observed loading:** the probe (Q3) shows `kitty.fast_data_types` loaded immediately after `import kitty_tests.main`.
- **`kittens/transfer/rsync.so` → the file-transmission category.** It is imported at module top level by `kitty_tests/file_transmission.py:L13` (`from kittens.transfer.rsync import Differ, Hasher, Patcher, parse_ftc`) and, more narrowly, *inside a test method* by `kitty_tests/check_build.py:L30`. **Observed loading:** the probe shows `kittens.transfer.rsync` appearing after `find_all_tests()` collects `file_transmission`.
- **`kitty/glfw-x11.so` / `kitty/glfw-wayland.so` → two individual build-verification tests, on demand.** They are never imported as Python modules; they are accessed by *path* — `ctypes.CDLL(glfw_path('x11'))` at `[kitty_tests/glfw.py:L50]` and `os.path.isfile(glfw_path('x11'))` at `[kitty_tests/check_build.py:L46]` (`glfw_path` is defined at `[kitty/constants.py:L191-L193]`). Under `CI=true` only the **x11** backend is referenced (see Q7). **Observed loading:** the probe shows **no** GLFW Python module loaded.

The harness mechanism that makes two of these *fatal*: test collection calls `importlib.import_module()` **directly** at `[kitty_tests/main.py:L64]` (inside `find_all_tests()`), with no per-module `try/except`. Therefore any exception raised at a test module's *top level* — including a failed extension import — propagates immediately and aborts collection. (The `ModuleImportFailure` safety net that unittest uses lives in `itertests()` at `[kitty_tests/main.py:L52-L53]`, but it is only reached for modules that were successfully imported, so it does not catch a hard top-level import error.)

### Q8 — The three import chains (each shown as a synthesized diagram *and* its verbatim runtime traceback)

The arrow diagrams below are **synthesized diagrams — not verbatim program output**. Each is immediately followed by a *verbatim excerpt* of the actual runtime traceback that establishes it (captured from the Q4 cascade experiments; the **complete** tracebacks, with before/during/after state, are in Q4). The tracebacks were captured from a temporary copy of the built tree at `/tmp/kitty_qa_sandbox`; the `file:line` structure is identical in the canonical run — only the leading directory differs, because the launcher resolves the library directory relative to its own location (`[kitty/launcher/main.c:L449-L460]`).

**Chain 1 — `fast_data_types` (eager, at harness-import time).** Synthesized diagram:

```text
test.py:L8  importlib.import_module('kitty_tests.main')
  └─> kitty_tests/__init__.py:L21  from kitty.config import finalize_keys, finalize_mouse_mappings   (transitive path, triggers first)
        └─> kitty/config.py:L10   from .conf.utils import BadLine, parse_config_base
              └─> kitty/conf/utils.py:L27   from ..fast_data_types import Color   ──> kitty/fast_data_types.so
  (and, had the transitive path not triggered first, the DIRECT import at kitty_tests/__init__.py:L22
   `from kitty.fast_data_types import Cursor, HistoryBuf, LineBuf, Screen, …` would load the same module)
```

Verbatim runtime traceback that establishes Chain 1 *(excerpt — kitty-relevant frames of the Tier-1 experiment; complete traceback in Q4 Tier 1)*:

```text
  File "test.py", line 13, in <module>
    main()
    ~~~~^^
  File "test.py", line 8, in main
    m = importlib.import_module('kitty_tests.main')
  File "/usr/lib/python3.13/importlib/__init__.py", line 88, in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
           ~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "<frozen importlib._bootstrap>", line 1387, in _gcd_import
  File "<frozen importlib._bootstrap>", line 1360, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1310, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 488, in _call_with_frames_removed
  File "<frozen importlib._bootstrap>", line 1387, in _gcd_import
  File "<frozen importlib._bootstrap>", line 1360, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1331, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 935, in _load_unlocked
  File "<frozen importlib._bootstrap_external>", line 1026, in exec_module
  File "<frozen importlib._bootstrap>", line 488, in _call_with_frames_removed
  File "/tmp/kitty_qa_sandbox/kitty/launcher/../../kitty_tests/__init__.py", line 21, in <module>
    from kitty.config import finalize_keys, finalize_mouse_mappings
  File "/tmp/kitty_qa_sandbox/kitty/launcher/../../kitty/config.py", line 10, in <module>
    from .conf.utils import BadLine, parse_config_base
  File "/tmp/kitty_qa_sandbox/kitty/launcher/../../kitty/conf/utils.py", line 27, in <module>
    from ..fast_data_types import Color
ModuleNotFoundError: No module named 'kitty.fast_data_types'
```

**Chain 2 — `rsync` (at collection time).** Synthesized diagram (note each `kitty_tests/main.py` hop is a distinct `file:line`):

```text
kitty_tests/main.py:L338  main() → run_tests()
  └─> kitty_tests/main.py:L279  run_tests() → run_python_tests(args, go_proc)
        └─> kitty_tests/main.py:L211  run_python_tests() → find_all_tests()
              └─> kitty_tests/main.py:L64  find_all_tests(): importlib.import_module(package + '.' + x…)
                    └─> kitty_tests/file_transmission.py:L13  from kittens.transfer.rsync import Differ, Hasher, Patcher, parse_ftc  ──> kittens/transfer/rsync.so
```

Verbatim runtime traceback that establishes Chain 2 *(excerpt — kitty-relevant frames of the Tier-2 experiment; complete traceback in Q4 Tier 2)*:

```text
  File "test.py", line 13, in <module>
    main()
    ~~~~^^
  File "test.py", line 9, in main
    getattr(m, 'main')()
    ~~~~~~~~~~~~~~~~~~^^
  File "/tmp/kitty_qa_sandbox/kitty/launcher/../../kitty_tests/main.py", line 338, in main
    run_tests()
    ~~~~~~~~~^^
  File "/tmp/kitty_qa_sandbox/kitty/launcher/../../kitty_tests/main.py", line 279, in run_tests
    run_python_tests(args, go_proc)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^
  File "/tmp/kitty_qa_sandbox/kitty/launcher/../../kitty_tests/main.py", line 211, in run_python_tests
    tests = find_all_tests()
  File "/tmp/kitty_qa_sandbox/kitty/launcher/../../kitty_tests/main.py", line 64, in find_all_tests
    m = importlib.import_module(package + '.' + x.partition('.')[0])
  File "/usr/lib/python3.13/importlib/__init__.py", line 88, in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
           ~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "<frozen importlib._bootstrap>", line 1387, in _gcd_import
  File "<frozen importlib._bootstrap>", line 1360, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1331, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 935, in _load_unlocked
  File "<frozen importlib._bootstrap_external>", line 1026, in exec_module
  File "<frozen importlib._bootstrap>", line 488, in _call_with_frames_removed
  File "/tmp/kitty_qa_sandbox/kitty/launcher/../../kitty_tests/file_transmission.py", line 13, in <module>
    from kittens.transfer.rsync import Differ, Hasher, Patcher, parse_ftc
ModuleNotFoundError: No module named 'kittens.transfer.rsync'
```

**Chain 3 — GLFW backends (on demand, by path, not `import`).** Synthesized diagram:

```text
kitty_tests/glfw.py:L50        ctypes.CDLL(glfw_path('x11'))        ──> dlopen kitty/glfw-x11.so   (test_utf_8_strndup)
kitty_tests/check_build.py:L46 os.path.isfile(glfw_path('x11'))      ──> stat  kitty/glfw-x11.so   (test_glfw_modules)
      (glfw_path defined at kitty/constants.py:L191-L193; under CI=true only 'x11' is referenced — see Q7)
```

Because these are `dlopen`/`stat` calls *inside individual test methods* (not module-level `import`s), a missing GLFW backend is caught by unittest per-test and does **not** abort collection — the verbatim `OSError`/`AssertionError` for these two tests is shown complete in **Q4 Tier 3**.

---
## Q3 — Which compiled extension modules are imported/loaded during a test run

**Method (positive proof of loading).** To observe exactly which compiled `.so` modules load, a temporary probe drove the **real** runner path — `importlib.import_module('kitty_tests.main')` (identical to `[test.py:L8]`) then the real `find_all_tests()` (`[kitty_tests/main.py:L57]`) — and inspected `sys.modules` for `.so`-backed modules at three points (before / after harness import / after collection). It was run through the canonical launcher and removed afterward (MainRule). Complete probe source:

```python
#!/usr/bin/env python
# Temporary observation probe (run via: ./kitty/launcher/kitty +launch load_probe.py).
# Drives the REAL runner path: importlib.import_module('kitty_tests.main') exactly as
# test.py:L8, then the real find_all_tests() (kitty_tests/main.py:L57), inspecting
# sys.modules for .so-backed extension modules at three points: BEFORE / AFTER harness
# import / AFTER collection. Not committed; removed after use (MainRule).
import importlib
import os
import sys


def so_backed():
    out = {}
    for name, mod in list(sys.modules.items()):
        f = getattr(mod, '__file__', None) or ''
        if f.endswith('.so'):
            out[name] = f
    return out


def dump(title):
    mods = so_backed()
    print('----- %s: %d .so-backed modules -----' % (title, len(mods)))
    for name in sorted(mods):
        print('  %s -> %s' % (name, mods[name]))
    return mods


print('===== BEFORE any kitty_tests import =====')
before = dump('BEFORE')

print('')
print('===== AFTER import kitty_tests.main (harness import, == test.py:L8) =====')
m = importlib.import_module('kitty_tests.main')
after_import = dump('AFTER import kitty_tests.main')

print('')
print('===== AFTER find_all_tests() (real collection, kitty_tests/main.py:L57) =====')
suite = m.find_all_tests()
after_collect = dump('AFTER find_all_tests')

import kitty
repo_root = os.path.dirname(os.path.dirname(os.path.abspath(kitty.__file__)))
print('')
print('(repo_root = %s)' % repo_root)
kitty_authored = {n: f for n, f in after_collect.items()
                  if os.path.abspath(f).startswith(repo_root + os.sep)}
other = {n: f for n, f in after_collect.items() if n not in kitty_authored}

print('')
print('===== KITTY-AUTHORED extension modules (.so under repo root) =====')
for n in sorted(kitty_authored):
    print('  %s -> %s' % (n, kitty_authored[n]))
print('  TOTAL kitty-authored: %d' % len(kitty_authored))

print('')
print('===== stdlib / third-party .so (NOT kitty-authored) =====')
for n in sorted(other):
    print('  %s -> %s' % (n, other[n]))
print('  TOTAL stdlib/third-party: %d' % len(other))

glfw_mods = [n for n in after_collect if 'glfw' in n.lower()]
print('')
print('===== GLFW backend python modules loaded: %s ====='
      % (glfw_mods if glfw_mods else 'NONE'))

import _json
print('')
print('===== _json builtin-vs-.so check =====')
print('  _json in sys.builtin_module_names: %s' % ('_json' in sys.builtin_module_names))
print('  _json.__file__ attr present: %s' % hasattr(_json, '__file__'))
print('  _json in sys.modules: %s' % ('_json' in sys.modules))

print('')
print('===== find_all_tests() collected test count: %d =====' % suite.countTestCases())
```

Exact invocation (canonical launcher, `CI=true`, Go on `PATH`):

```bash
CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 PATH="$PATH:/usr/local/go/bin" ./kitty/launcher/kitty +launch /tmp/kitty_qa_logs/load_probe.py
```

**Complete, unedited output — RUN #1:**

```text
===== BEFORE any kitty_tests import =====
----- BEFORE: 0 .so-backed modules -----

===== AFTER import kitty_tests.main (harness import, == test.py:L8) =====
----- AFTER import kitty_tests.main: 4 .so-backed modules -----
  _bz2 -> /usr/lib/python3.13/lib-dynload/_bz2.cpython-313-x86_64-linux-gnu.so
  _lzma -> /usr/lib/python3.13/lib-dynload/_lzma.cpython-313-x86_64-linux-gnu.so
  kitty.fast_data_types -> /tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty/fast_data_types.so
  termios -> /usr/lib/python3.13/lib-dynload/termios.cpython-313-x86_64-linux-gnu.so

===== AFTER find_all_tests() (real collection, kitty_tests/main.py:L57) =====
----- AFTER find_all_tests: 9 .so-backed modules -----
  PIL._imaging -> /usr/local/lib/python3.13/dist-packages/PIL/_imaging.cpython-313-x86_64-linux-gnu.so
  _bz2 -> /usr/lib/python3.13/lib-dynload/_bz2.cpython-313-x86_64-linux-gnu.so
  _ctypes -> /usr/lib/python3.13/lib-dynload/_ctypes.cpython-313-x86_64-linux-gnu.so
  _hashlib -> /usr/lib/python3.13/lib-dynload/_hashlib.cpython-313-x86_64-linux-gnu.so
  _lzma -> /usr/lib/python3.13/lib-dynload/_lzma.cpython-313-x86_64-linux-gnu.so
  kittens.transfer.rsync -> /tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kittens/transfer/rsync.so
  kitty.fast_data_types -> /tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty/fast_data_types.so
  mmap -> /usr/lib/python3.13/lib-dynload/mmap.cpython-313-x86_64-linux-gnu.so
  termios -> /usr/lib/python3.13/lib-dynload/termios.cpython-313-x86_64-linux-gnu.so

(repo_root = /tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d)

===== KITTY-AUTHORED extension modules (.so under repo root) =====
  kittens.transfer.rsync -> /tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kittens/transfer/rsync.so
  kitty.fast_data_types -> /tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty/fast_data_types.so
  TOTAL kitty-authored: 2

===== stdlib / third-party .so (NOT kitty-authored) =====
  PIL._imaging -> /usr/local/lib/python3.13/dist-packages/PIL/_imaging.cpython-313-x86_64-linux-gnu.so
  _bz2 -> /usr/lib/python3.13/lib-dynload/_bz2.cpython-313-x86_64-linux-gnu.so
  _ctypes -> /usr/lib/python3.13/lib-dynload/_ctypes.cpython-313-x86_64-linux-gnu.so
  _hashlib -> /usr/lib/python3.13/lib-dynload/_hashlib.cpython-313-x86_64-linux-gnu.so
  _lzma -> /usr/lib/python3.13/lib-dynload/_lzma.cpython-313-x86_64-linux-gnu.so
  mmap -> /usr/lib/python3.13/lib-dynload/mmap.cpython-313-x86_64-linux-gnu.so
  termios -> /usr/lib/python3.13/lib-dynload/termios.cpython-313-x86_64-linux-gnu.so
  TOTAL stdlib/third-party: 7

===== GLFW backend python modules loaded: NONE =====

===== _json builtin-vs-.so check =====
  _json in sys.builtin_module_names: True
  _json.__file__ attr present: False
  _json in sys.modules: True

===== find_all_tests() collected test count: 145 =====
```

**Complete, unedited output — RUN #2** (byte-identical to RUN #1; `diff probe_run1.log probe_run2.log` produces no output):

```text
===== BEFORE any kitty_tests import =====
----- BEFORE: 0 .so-backed modules -----

===== AFTER import kitty_tests.main (harness import, == test.py:L8) =====
----- AFTER import kitty_tests.main: 4 .so-backed modules -----
  _bz2 -> /usr/lib/python3.13/lib-dynload/_bz2.cpython-313-x86_64-linux-gnu.so
  _lzma -> /usr/lib/python3.13/lib-dynload/_lzma.cpython-313-x86_64-linux-gnu.so
  kitty.fast_data_types -> /tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty/fast_data_types.so
  termios -> /usr/lib/python3.13/lib-dynload/termios.cpython-313-x86_64-linux-gnu.so

===== AFTER find_all_tests() (real collection, kitty_tests/main.py:L57) =====
----- AFTER find_all_tests: 9 .so-backed modules -----
  PIL._imaging -> /usr/local/lib/python3.13/dist-packages/PIL/_imaging.cpython-313-x86_64-linux-gnu.so
  _bz2 -> /usr/lib/python3.13/lib-dynload/_bz2.cpython-313-x86_64-linux-gnu.so
  _ctypes -> /usr/lib/python3.13/lib-dynload/_ctypes.cpython-313-x86_64-linux-gnu.so
  _hashlib -> /usr/lib/python3.13/lib-dynload/_hashlib.cpython-313-x86_64-linux-gnu.so
  _lzma -> /usr/lib/python3.13/lib-dynload/_lzma.cpython-313-x86_64-linux-gnu.so
  kittens.transfer.rsync -> /tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kittens/transfer/rsync.so
  kitty.fast_data_types -> /tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty/fast_data_types.so
  mmap -> /usr/lib/python3.13/lib-dynload/mmap.cpython-313-x86_64-linux-gnu.so
  termios -> /usr/lib/python3.13/lib-dynload/termios.cpython-313-x86_64-linux-gnu.so

(repo_root = /tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d)

===== KITTY-AUTHORED extension modules (.so under repo root) =====
  kittens.transfer.rsync -> /tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kittens/transfer/rsync.so
  kitty.fast_data_types -> /tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty/fast_data_types.so
  TOTAL kitty-authored: 2

===== stdlib / third-party .so (NOT kitty-authored) =====
  PIL._imaging -> /usr/local/lib/python3.13/dist-packages/PIL/_imaging.cpython-313-x86_64-linux-gnu.so
  _bz2 -> /usr/lib/python3.13/lib-dynload/_bz2.cpython-313-x86_64-linux-gnu.so
  _ctypes -> /usr/lib/python3.13/lib-dynload/_ctypes.cpython-313-x86_64-linux-gnu.so
  _hashlib -> /usr/lib/python3.13/lib-dynload/_hashlib.cpython-313-x86_64-linux-gnu.so
  _lzma -> /usr/lib/python3.13/lib-dynload/_lzma.cpython-313-x86_64-linux-gnu.so
  mmap -> /usr/lib/python3.13/lib-dynload/mmap.cpython-313-x86_64-linux-gnu.so
  termios -> /usr/lib/python3.13/lib-dynload/termios.cpython-313-x86_64-linux-gnu.so
  TOTAL stdlib/third-party: 7

===== GLFW backend python modules loaded: NONE =====

===== _json builtin-vs-.so check =====
  _json in sys.builtin_module_names: True
  _json.__file__ attr present: False
  _json in sys.modules: True

===== find_all_tests() collected test count: 145 =====
```

**What loads (answer to Q3), read directly off the probe output:**

- **Exactly two kitty-authored compiled extensions load as Python modules:**
  - `kitty.fast_data_types` → `…/kitty/fast_data_types.so` — present **immediately after `import kitty_tests.main`** (eager; via the `BaseTest` chain of Q8).
  - `kittens.transfer.rsync` → `…/kittens/transfer/rsync.so` — appears **only after `find_all_tests()`** (collection-time; when `file_transmission` is imported).
- **The GLFW backends do NOT load as Python modules:** the probe prints `GLFW backend python modules loaded: NONE`. They are `dlopen`/`stat`-ed on demand by two tests (Q8 Chain 3), not imported.
- **`_json` is a *builtin*, not a `.so`** — this is **observed**, not inferred: the probe's direct check prints `_json in sys.builtin_module_names: True`, `_json.__file__ attr present: False`, `_json in sys.modules: True`. Because it is compiled **into** the CPython interpreter (statically), it never appears in the `.so`-backed list. *(This corrects the AAP's pre-captured note that had grouped `_json` among the standard-library `.so` files; per the version-drift note above, the observed reality is reported instead.)*
- The remaining `.so`-backed modules are all **standard-library or third-party**, not kitty's: `PIL._imaging` (Pillow), `_bz2`, `_ctypes`, `_hashlib`, `_lzma`, `mmap`, `termios`. The probe classifies these automatically by checking whether each module's file path is under the repo root (`TOTAL stdlib/third-party: 7`).
- `find_all_tests()` collected **145** test cases — matching the baseline run count of Q1.c (cross-check on the stability figure).

This probe — not the grep of Q1.c — is the **positive** proof that the extensions loaded. The Q1.c grep establishes only the *absence* of import-error strings; the probe establishes, by module object and resolved file path, that `kitty.fast_data_types` and `kittens.transfer.rsync` are actually present in `sys.modules`.

---
## Q4 — How test failures cascade when a compiled extension is unavailable · Q7 — Critical vs optional

**Safe reproduction method (MainRule / auditability).** The source tree is never mutated. Each extension was made unavailable inside a **temporary copy** of the built tree, created with a unique directory name via `mktemp -d`. Because the launcher resolves its library directory *relative to its own location* (`[kitty/launcher/main.c:L449-L460]`), running the copy's launcher loads the copy's own `.so` files — so moving a `.so` aside in the copy affects only the copy. Every command is shown with its exit code; each `.so` is moved aside (`mv`), the canonical suite is run, the `.so` is moved back (restore), and the whole copy is deleted at the end. Sandbox creation and the load-locality verification:

```text
$ SB="$(mktemp -d /tmp/kitty_qa_sandbox.XXXXXX)"; echo "$SB"
/tmp/kitty_qa_sandbox.To8dsh
# exit code: 0

$ tar cf - --exclude='./.git' -C "$REPO" . | ( cd "$SB" && tar xf - ); echo "copy_exit=${PIPESTATUS[0]}/${PIPESTATUS[1]}"
copy_exit=0/0

$ ls -d "$SB/.git" 2>&1   # confirm .git NOT copied
ls: cannot access '/tmp/kitty_qa_sandbox.To8dsh/.git': No such file or directory

$ ls -l "$SB"/kitty/fast_data_types.so "$SB"/kittens/transfer/rsync.so "$SB"/kitty/glfw-x11.so "$SB"/kitty/glfw-wayland.so "$SB"/kitty/launcher/kitty
-rwxr-xr-x 1 root root   42824 Jul 14 20:38 /tmp/kitty_qa_sandbox.To8dsh/kittens/transfer/rsync.so
-rwxr-xr-x 1 root root 1253792 Jul 14 20:38 /tmp/kitty_qa_sandbox.To8dsh/kitty/fast_data_types.so
-rwxr-xr-x 1 root root  451016 Jul 14 20:38 /tmp/kitty_qa_sandbox.To8dsh/kitty/glfw-wayland.so
-rwxr-xr-x 1 root root  373896 Jul 14 20:38 /tmp/kitty_qa_sandbox.To8dsh/kitty/glfw-x11.so
-rwxr-xr-x 1 root root   40384 Jul 14 20:38 /tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/kitty

# Verify the sandbox launcher loads the SANDBOX's own .so (so mutating the copy never touches the source repo):
$ ( cd "$SB" && CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./kitty/launcher/kitty +launch /tmp/kitty_qa_logs/load_probe.py ) | grep -E "fast_data_types ->|rsync ->"
  kitty.fast_data_types -> /tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty/fast_data_types.so
  kittens.transfer.rsync -> /tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kittens/transfer/rsync.so
  kitty.fast_data_types -> /tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty/fast_data_types.so
  kittens.transfer.rsync -> /tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kittens/transfer/rsync.so
  kitty.fast_data_types -> /tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty/fast_data_types.so
```

Each tier below was run **twice**. The `mv`/run/`mv`-back sequence and the before/during/after `ls` are part of each captured log.

### Tier 1 — `fast_data_types.so` missing → CRITICAL, fatal at harness import

**Cause → effect.** `fast_data_types` is imported at the *top level* of the `BaseTest` module, reached while `test.py:L8` is still importing `kitty_tests.main`. The import fails **before** `main()` ever runs, so the environment banner (printed by `env_for_python_tests()` at `[kitty_tests/main.py:L305]`) **never appears** and **zero tests run**. The traceback shows the transitive chain `[kitty_tests/__init__.py:L21]` → `[kitty/config.py:L10]` → `[kitty/conf/utils.py:L27]` → `ModuleNotFoundError`. Complete, unedited RUN #1 (before/during/run/exit/restore/after):

```text
########## TIER 1 (fast_data_types.so unavailable) — RUN 1 ##########
$ ls -l "$SB/kitty/fast_data_types.so"                      # BEFORE
-rwxr-xr-x 1 root root 1253792 Jul 14 20:38 /tmp/kitty_qa_sandbox.To8dsh/kitty/fast_data_types.so
$ mv "$SB/kitty/fast_data_types.so" "$SB/kitty/fast_data_types.so.qabak"     # render unavailable (temp copy only)
mv_exit=0
$ ls -l "$SB/kitty/fast_data_types.so" 2>&1                  # DURING (absent)
ls: cannot access '/tmp/kitty_qa_sandbox.To8dsh/kitty/fast_data_types.so': No such file or directory
$ ( cd "$SB" && CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./kitty/launcher/kitty +launch test.py )
Traceback (most recent call last):
  File "<frozen runpy>", line 198, in _run_module_as_main
  File "<frozen runpy>", line 88, in _run_code
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../__main__.py", line 7, in <module>
    main()
    ~~~~^^
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty/entry_points.py", line 192, in main
    namespaced(['+', first_arg[1:]] + sys.argv[2:])
    ~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty/entry_points.py", line 146, in namespaced
    func(args[1:])
    ~~~~^^^^^^^^^^
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty/entry_points.py", line 73, in launch
    runpy.run_path(exe, run_name='__main__')
    ~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "<frozen runpy>", line 287, in run_path
  File "<frozen runpy>", line 98, in _run_module_code
  File "<frozen runpy>", line 88, in _run_code
  File "test.py", line 13, in <module>
    main()
    ~~~~^^
  File "test.py", line 8, in main
    m = importlib.import_module('kitty_tests.main')
  File "/usr/lib/python3.13/importlib/__init__.py", line 88, in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
           ~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "<frozen importlib._bootstrap>", line 1387, in _gcd_import
  File "<frozen importlib._bootstrap>", line 1360, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1310, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 488, in _call_with_frames_removed
  File "<frozen importlib._bootstrap>", line 1387, in _gcd_import
  File "<frozen importlib._bootstrap>", line 1360, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1331, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 935, in _load_unlocked
  File "<frozen importlib._bootstrap_external>", line 1026, in exec_module
  File "<frozen importlib._bootstrap>", line 488, in _call_with_frames_removed
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/__init__.py", line 21, in <module>
    from kitty.config import finalize_keys, finalize_mouse_mappings
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty/config.py", line 10, in <module>
    from .conf.utils import BadLine, parse_config_base
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty/conf/utils.py", line 27, in <module>
    from ..fast_data_types import Color
ModuleNotFoundError: No module named 'kitty.fast_data_types'
# exit code: 1
$ mv "$SB/kitty/fast_data_types.so.qabak" "$SB/kitty/fast_data_types.so"     # RESTORE
restore_exit=0
$ ls -l "$SB/kitty/fast_data_types.so"                       # AFTER (restored)
-rwxr-xr-x 1 root root 1253792 Jul 14 20:38 /tmp/kitty_qa_sandbox.To8dsh/kitty/fast_data_types.so
```

**Repeat (RUN #2).** Both runs are byte-identical. Each complete log is a 43-line pure `ModuleNotFoundError` traceback — no banner, no timing, no pointer/hex addresses — so a direct `diff` of the two full logs (captured with the `RUN 1`/`RUN 2` header line excluded) produces **no output**, and both processes exit **1**. The banner is absent in both; **0 tests execute**:

```text
$ diff tier1_run1.log tier1_run2.log            # no output => the two complete logs are identical
$ echo "diff exit=$?  run1 exit=1  run2 exit=1"
diff exit=0  run1 exit=1  run2 exit=1
```

### Tier 2 — `rsync.so` missing → CRITICAL, fatal at collection

**Cause → effect.** `rsync` is imported at the top level of `kitty_tests/file_transmission.py:L13`. This time `main()` *does* run: the banner prints and the Go packages enumerate. But test **collection** — `main()` `[kitty_tests/main.py:L338]` → `run_tests()` `[kitty_tests/main.py:L279]` → `run_python_tests()` `[kitty_tests/main.py:L211]` → `find_all_tests()`'s direct `importlib.import_module()` at `[kitty_tests/main.py:L64]` — raises `ModuleNotFoundError` the moment it imports `file_transmission`, aborting the whole run with **0 Python tests executed**. Complete, unedited RUN #1:

```text
########## TIER 2 (rsync.so unavailable) — RUN 1 ##########
$ ls -l "$SB/kittens/transfer/rsync.so"                      # BEFORE
-rwxr-xr-x 1 root root 42824 Jul 14 20:38 /tmp/kitty_qa_sandbox.To8dsh/kittens/transfer/rsync.so
$ mv "$SB/kittens/transfer/rsync.so" "$SB/kittens/transfer/rsync.so.qabak"     # render unavailable (temp copy only)
mv_exit=0
$ ls -l "$SB/kittens/transfer/rsync.so" 2>&1                  # DURING (absent)
ls: cannot access '/tmp/kitty_qa_sandbox.To8dsh/kittens/transfer/rsync.so': No such file or directory
$ ( cd "$SB" && CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./kitty/launcher/kitty +launch test.py )
Running under CI: True
Using PATH in test environment: /tmp/kitty_qa_sandbox.To8dsh/kitty_tests/kitty/launcher:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/go/bin
Python: /usr/bin/python
Intrinsics: has_avx2=True has_sse4_2=True
Go executable: /usr/local/bin/go
Go packages being tested: tools/tui/subseq tools/unicode_names tools/utils/style kittens/ssh tools/tui/shell_integration tools/simdstring tools/tui/graphics tools/tui/sgr tools/cmd/at tools/tui/readline tools/tui tools/utils/shm tools/config tools/wcswidth tools/utils tools/utils/humanize kittens/hints tools/rsync tools/tui/loop tools/utils/base85 tools/cli kittens/transfer kittens/hyperlinked_grep tools/utils/shlex tools/themes kittens/diff
Traceback (most recent call last):
  File "<frozen runpy>", line 198, in _run_module_as_main
  File "<frozen runpy>", line 88, in _run_code
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../__main__.py", line 7, in <module>
    main()
    ~~~~^^
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty/entry_points.py", line 192, in main
    namespaced(['+', first_arg[1:]] + sys.argv[2:])
    ~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty/entry_points.py", line 146, in namespaced
    func(args[1:])
    ~~~~^^^^^^^^^^
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty/entry_points.py", line 73, in launch
    runpy.run_path(exe, run_name='__main__')
    ~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "<frozen runpy>", line 287, in run_path
  File "<frozen runpy>", line 98, in _run_module_code
  File "<frozen runpy>", line 88, in _run_code
  File "test.py", line 13, in <module>
    main()
    ~~~~^^
  File "test.py", line 9, in main
    getattr(m, 'main')()
    ~~~~~~~~~~~~~~~~~~^^
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/main.py", line 338, in main
    run_tests()
    ~~~~~~~~~^^
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/main.py", line 279, in run_tests
    run_python_tests(args, go_proc)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/main.py", line 211, in run_python_tests
    tests = find_all_tests()
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/main.py", line 64, in find_all_tests
    m = importlib.import_module(package + '.' + x.partition('.')[0])
  File "/usr/lib/python3.13/importlib/__init__.py", line 88, in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
           ~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "<frozen importlib._bootstrap>", line 1387, in _gcd_import
  File "<frozen importlib._bootstrap>", line 1360, in _find_and_load
  File "<frozen importlib._bootstrap>", line 1331, in _find_and_load_unlocked
  File "<frozen importlib._bootstrap>", line 935, in _load_unlocked
  File "<frozen importlib._bootstrap_external>", line 1026, in exec_module
  File "<frozen importlib._bootstrap>", line 488, in _call_with_frames_removed
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/file_transmission.py", line 13, in <module>
    from kittens.transfer.rsync import Differ, Hasher, Patcher, parse_ftc
ModuleNotFoundError: No module named 'kittens.transfer.rsync'
# exit code: 1
$ mv "$SB/kittens/transfer/rsync.so.qabak" "$SB/kittens/transfer/rsync.so"     # RESTORE
restore_exit=0
$ ls -l "$SB/kittens/transfer/rsync.so"                       # AFTER (restored)
-rwxr-xr-x 1 root root 42824 Jul 14 20:38 /tmp/kitty_qa_sandbox.To8dsh/kittens/transfer/rsync.so
```

**Repeat (RUN #2).** `main()` runs in both, so each complete 52-line log contains the environment banner. A raw `diff` of the two full logs shows **exactly one** differing line — the non-deterministic `Go packages being tested:` order (per Q1.b) — and nothing else. Excluding that single line the logs are byte-identical; the enumerated Go-package *set* is identical (26 packages, only the order varies); the `ModuleNotFoundError: No module named 'kittens.transfer.rsync'` line is byte-identical; and both exit **1**. **0 Python tests execute** in either run:

```text
$ diff tier2_run1.log tier2_run2.log
6c6
< Go packages being tested: tools/utils/base85 tools/config tools/cli tools/tui/loop tools/tui/readline tools/rsync kittens/transfer kittens/hyperlinked_grep tools/utils/style tools/wcswidth tools/tui/graphics tools/themes tools/unicode_names tools/tui/shell_integration kittens/hints tools/utils/humanize tools/utils tools/cmd/at tools/utils/shlex kittens/diff tools/utils/shm tools/tui/sgr tools/tui/subseq tools/tui kittens/ssh tools/simdstring
---
> Go packages being tested: kittens/hyperlinked_grep tools/utils tools/tui/shell_integration tools/tui/loop tools/config tools/wcswidth tools/tui/subseq kittens/transfer tools/utils/humanize tools/tui/sgr tools/simdstring tools/utils/base85 tools/tui tools/utils/style tools/cmd/at tools/utils/shlex kittens/hints tools/tui/graphics tools/rsync tools/tui/readline kittens/ssh tools/cli tools/unicode_names kittens/diff tools/utils/shm tools/themes

$ diff <(grep -v 'Go packages being tested:' tier2_run1.log) <(grep -v 'Go packages being tested:' tier2_run2.log)   # every line except the Go-order line
$ echo "non-Go diff exit=$?"
non-Go diff exit=0

$ diff <(grep 'Go packages being tested:' tier2_run1.log | sed 's/^.*being tested: //' | tr ' ' '\n' | sort) \
       <(grep 'Go packages being tested:' tier2_run2.log | sed 's/^.*being tested: //' | tr ' ' '\n' | sort)   # the Go-package SET (order-normalized)
$ echo "Go-set diff exit=$?  set size=26"
Go-set diff exit=0  set size=26

$ grep ModuleNotFoundError tier2_run1.log ; grep ModuleNotFoundError tier2_run2.log
ModuleNotFoundError: No module named 'kittens.transfer.rsync'
ModuleNotFoundError: No module named 'kittens.transfer.rsync'
```

### Tier 3 — `glfw-x11.so` missing → OPTIONAL, localized

**Cause → effect.** The GLFW backend is **not** a Python module (Q3 showed no GLFW `.so` in `sys.modules`); it is consumed on demand by exactly two tests. When `glfw-x11.so` is absent (with `glfw-wayland.so` left in place), collection completes and **all 145 tests still run**; only the two GLFW consumers fail:

- `test_utf_8_strndup` ERRORs — `kitty_tests/glfw.py:L50` calls `ctypes.CDLL(backend_utils)` where `backend_utils` is the `glfw-x11.so` path; the missing file yields `OSError: … cannot open shared object file`.
- `test_glfw_modules` FAILs — `kitty_tests/check_build.py:L46` asserts `os.path.isfile(path)`. Under CI, `kitty_tests/check_build.py:L40-L42` sets `linux_backends = ['x11']` (wayland is appended only `if not self.is_ci`), so only the x11 path is checked, and the assertion fails with `AssertionError: … is not a file`.

The total is therefore **`failures=4, errors=1`** = the **3 pre-existing baseline failures** (`test_font_selection`, `test_transfer_send`, `test_transfer_receive` — see the Environment-artifact section) **+ 1 new GLFW FAIL** (`test_glfw_modules`) **+ 1 new GLFW ERROR** (`test_utf_8_strndup`). The GLFW-specific delta caused by removing the extension is exactly **1 FAIL + 1 ERROR**. Complete, unedited RUN #1 (all 365 lines: before/during/the full 145-test run/summary/exit code/restore/after):

```text
########## TIER 3 (glfw-x11.so unavailable) — RUN 1 ##########
$ ls -l "$SB/kitty/glfw-x11.so"                      # BEFORE
-rwxr-xr-x 1 root root 373896 Jul 14 20:38 /tmp/kitty_qa_sandbox.To8dsh/kitty/glfw-x11.so
$ mv "$SB/kitty/glfw-x11.so" "$SB/kitty/glfw-x11.so.qabak"     # render unavailable (temp copy only)
mv_exit=0
$ ls -l "$SB/kitty/glfw-x11.so" 2>&1                  # DURING (absent)
ls: cannot access '/tmp/kitty_qa_sandbox.To8dsh/kitty/glfw-x11.so': No such file or directory
$ ( cd "$SB" && CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./kitty/launcher/kitty +launch test.py )
Running under CI: True
Using PATH in test environment: /tmp/kitty_qa_sandbox.To8dsh/kitty_tests/kitty/launcher:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/go/bin
Python: /usr/bin/python
Intrinsics: has_avx2=True has_sse4_2=True
Go executable: /usr/local/bin/go
Go packages being tested: tools/utils kittens/hints tools/utils/shlex tools/tui/graphics tools/utils/shm tools/wcswidth tools/themes tools/tui/subseq tools/simdstring tools/tui tools/utils/base85 tools/tui/shell_integration tools/cli kittens/hyperlinked_grep tools/tui/loop tools/unicode_names tools/tui/readline tools/config tools/utils/humanize kittens/transfer tools/tui/sgr tools/utils/style tools/rsync kittens/diff tools/cmd/at kittens/ssh
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
test_transfer_receive (kitty_tests.file_transmission.TestFileTransmission.test_transfer_receive) ... FAIL
test_transfer_send (kitty_tests.file_transmission.TestFileTransmission.test_transfer_send) ... FAIL
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
test_glfw_modules (kitty_tests.check_build.TestBuild.test_glfw_modules) ... FAIL
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
test_font_selection (kitty_tests.fonts.Selection.test_font_selection) ... FAIL
test_os_window_size_calculation (kitty_tests.glfw.TestGLFW.test_os_window_size_calculation) ... ok
test_utf_8_strndup (kitty_tests.glfw.TestGLFW.test_utf_8_strndup) ... ERROR
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

======================================================================
ERROR: test_utf_8_strndup (kitty_tests.glfw.TestGLFW.test_utf_8_strndup)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/glfw.py", line 50, in test_utf_8_strndup
    lib = ctypes.CDLL(backend_utils)
    backend_utils = '/tmp/kitty_qa_sandbox.To8dsh/kitty/glfw-x11.so'
    ctypes = <module 'ctypes' from '/usr/lib/python3.13/ctypes/__init__.py'>
    glfw_path = <function glfw_path at 0x7e124b1ef1a0>
    self = <kitty_tests.glfw.TestGLFW testMethod=test_utf_8_strndup>
  File "/usr/lib/python3.13/ctypes/__init__.py", line 390, in __init__
    self._handle = _dlopen(self._name, mode)
                   ~~~~~~~^^^^^^^^^^^^^^^^^^
    _FuncPtr = <class 'ctypes.CDLL.__init__.<locals>._FuncPtr'>
    flags = 1
    handle = None
    mode = 0
    name = '/tmp/kitty_qa_sandbox.To8dsh/kitty/glfw-x11.so'
    self = <CDLL '/tmp/kitty_qa_sandbox.To8dsh/kitty/glfw-x11.so', handle 0 at 0x7e12494bfcb0>
    use_errno = False
    use_last_error = False
    winmode = None
OSError: /tmp/kitty_qa_sandbox.To8dsh/kitty/glfw-x11.so: cannot open shared object file: No such file or directory

======================================================================
FAIL: test_transfer_receive (kitty_tests.file_transmission.TestFileTransmission.test_transfer_receive)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/file_transmission.py", line 476, in test_transfer_receive
    self.basic_transfer_tests()
    ~~~~~~~~~~~~~~~~~~~~~~~~~^^
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/file_transmission.py", line 443, in basic_transfer_tests
    multiple_files()
    ~~~~~~~~~~~~~~^^
    d = <_io.BufferedWriter name='/tmp/tmpo2a86s36/dest'>
    dest = '/tmp/tmpo2a86s36/dest'
    multiple_files = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files at 0x7e12488ed940>
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7e1249f45160>
    s = <_io.BufferedWriter name='/tmp/tmpo2a86s36/src'>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
    single_file = <function TestFileTransmission.basic_transfer_tests.<locals>.single_file at 0x7e12488ee020>
    src = '/tmp/tmpo2a86s36/src'
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/file_transmission.py", line 432, in multiple_files
    self.assertEqual(expected, actual)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^
    Entry = <class 'kitty_tests.file_transmission.Entry'>
    actual = {'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784062174618308537, mode='0o120777', nlink=1), 'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'sym': Entry(relpath='sym', mtime=1784062174618308537, mode='0o120777', nlink=1), 'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1)}
    b = PosixPath('/tmp/tmpo2a86s36/msrc')
    cmd = ()
    de = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.de at 0x7e1248734ea0>
    dest = '/tmp/tmpo2a86s36/mdest'
    dirnames = []
    dirpath = '/tmp/tmpo2a86s36/mdest/msrc/empty'
    entry = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.entry at 0x7e1248734220>
    expected = {'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1), 'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784062174618308537, mode='0o120777', nlink=1), 'sym': Entry(relpath='sym', mtime=1784062174618308537, mode='0o120777', nlink=1)}
    f = <_io.BufferedWriter name='/tmp/tmpo2a86s36/msrc/sub/reg'>
    filenames = []
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7e1249ee5c50>
    s = PosixPath('/tmp/tmpo2a86s36/msrc/sub')
    se = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.se at 0x7e12487342c0>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
    src = '/tmp/tmpo2a86s36/msrc'
    x = 'reg'
AssertionError: {'simple': Entry(relpath='simple', mtime=130[497 chars]k=1)} != {'sub': Entry(relpath='sub', mtime=0, mode='[497 chars]k=1)}
  {'abssym': Entry(relpath='abssym', mtime=1784062174618308537, mode='0o120777', nlink=1),
-  'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2),
?                                                    ^

+  'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2),
?                                                    ^

   'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2),
   'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2),
-  'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2),
?                                                ^

+  'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2),
?                                                ^

   'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1),
   'sym': Entry(relpath='sym', mtime=1784062174618308537, mode='0o120777', nlink=1)}

======================================================================
FAIL: test_transfer_send (kitty_tests.file_transmission.TestFileTransmission.test_transfer_send)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/file_transmission.py", line 507, in test_transfer_send
    self.basic_transfer_tests()
    ~~~~~~~~~~~~~~~~~~~~~~~~~^^
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/file_transmission.py", line 443, in basic_transfer_tests
    multiple_files()
    ~~~~~~~~~~~~~~^^
    d = <_io.BufferedWriter name='/tmp/tmpq3m8yhe0/dest'>
    dest = '/tmp/tmpq3m8yhe0/dest'
    multiple_files = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files at 0x7e1248737380>
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7e12488e67b0>
    s = <_io.BufferedWriter name='/tmp/tmpq3m8yhe0/src'>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
    single_file = <function TestFileTransmission.basic_transfer_tests.<locals>.single_file at 0x7e1248735260>
    src = '/tmp/tmpq3m8yhe0/src'
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/file_transmission.py", line 432, in multiple_files
    self.assertEqual(expected, actual)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^
    Entry = <class 'kitty_tests.file_transmission.Entry'>
    actual = {'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784062175114313881, mode='0o120777', nlink=1), 'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'sym': Entry(relpath='sym', mtime=1784062175114313881, mode='0o120777', nlink=1), 'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1)}
    b = PosixPath('/tmp/tmpq3m8yhe0/msrc')
    cmd = ()
    de = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.de at 0x7e12487585e0>
    dest = '/tmp/tmpq3m8yhe0/mdest'
    dirnames = []
    dirpath = '/tmp/tmpq3m8yhe0/mdest/msrc/empty'
    entry = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.entry at 0x7e12487379c0>
    expected = {'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1), 'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784062175114313881, mode='0o120777', nlink=1), 'sym': Entry(relpath='sym', mtime=1784062175114313881, mode='0o120777', nlink=1)}
    f = <_io.BufferedWriter name='/tmp/tmpq3m8yhe0/msrc/sub/reg'>
    filenames = []
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7e1248740520>
    s = PosixPath('/tmp/tmpq3m8yhe0/msrc/sub')
    se = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.se at 0x7e1248737a60>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
    src = '/tmp/tmpq3m8yhe0/msrc'
    x = 'reg'
AssertionError: {'simple': Entry(relpath='simple', mtime=130[497 chars]k=1)} != {'sub': Entry(relpath='sub', mtime=0, mode='[497 chars]k=1)}
  {'abssym': Entry(relpath='abssym', mtime=1784062175114313881, mode='0o120777', nlink=1),
-  'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2),
?                                                    ^

+  'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2),
?                                                    ^

   'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2),
   'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2),
-  'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2),
?                                                ^

+  'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2),
?                                                ^

   'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1),
   'sym': Entry(relpath='sym', mtime=1784062175114313881, mode='0o120777', nlink=1)}

======================================================================
FAIL: test_glfw_modules (kitty_tests.check_build.TestBuild.test_glfw_modules)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/check_build.py", line 46, in test_glfw_modules
    self.assertTrue(os.path.isfile(path), f'{path} is not a file')
    ~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    glfw_path = <function glfw_path at 0x7e124b1ef1a0>
    is_macos = False
    linux_backends = ['x11']
    modules = ['x11']
    name = 'x11'
    path = '/tmp/kitty_qa_sandbox.To8dsh/kitty/glfw-x11.so'
    self = <kitty_tests.check_build.TestBuild testMethod=test_glfw_modules>
AssertionError: False is not true : /tmp/kitty_qa_sandbox.To8dsh/kitty/glfw-x11.so is not a file

======================================================================
FAIL: test_font_selection (kitty_tests.fonts.Selection.test_font_selection)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/fonts.py", line 64, in test_font_selection
    t('Source Code Pro', 'SourceCodePro', 'Semibold', 'It')
    ~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    both = <function Selection.test_font_selection.<locals>.both at 0x7e124842cf40>
    has = <function Selection.test_font_selection.<locals>.has at 0x7e124842c0e0>
    names = {'liberation mono', 'jetbrains mono nl', 'inconsolata', 'jetbrains mono', 'ubuntu sans mono', 'noto mono', 'ubuntu mono', 'fira code', 'noto sans signwriting', 'dejavu sans mono'}
    opts = <kitty.options.types.Options object at 0x7e124844e0d0>
    s = <function Selection.test_font_selection.<locals>.s at 0x7e124842cc20>
    self = <kitty_tests.fonts.Selection testMethod=test_font_selection>
    t = <function Selection.test_font_selection.<locals>.t at 0x7e124842d260>
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/fonts.py", line 58, in t
    if has(family, allow_missing_in_ci=allow_missing_in_ci):
       ~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    allow_missing_in_ci = False
    alternate = None
    bi = ''
    bold = 'Semibold'
    both = <function Selection.test_font_selection.<locals>.both at 0x7e124842cf40>
    family = 'Source Code Pro'
    has = <function Selection.test_font_selection.<locals>.has at 0x7e124842c0e0>
    italic = 'It'
    psprefix = 'SourceCodePro'
    reg = 'Regular'
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/fonts.py", line 54, in has
    raise AssertionError(f'The family: {family} is not available')
    allow_missing_in_ci = False
    ans = False
    family = 'Source Code Pro'
    names = {'liberation mono', 'jetbrains mono nl', 'inconsolata', 'jetbrains mono', 'ubuntu sans mono', 'noto mono', 'ubuntu mono', 'fira code', 'noto sans signwriting', 'dejavu sans mono'}
    self = <kitty_tests.fonts.Selection testMethod=test_font_selection>
AssertionError: The family: Source Code Pro is not available

----------------------------------------------------------------------
Ran 145 tests in 21.661s

FAILED (failures=4, errors=1, skipped=4)
All Go tests succeeded, ran in 21.7 seconds
[31mError[39m: Some tests failed!
# exit code: 1
$ mv "$SB/kitty/glfw-x11.so.qabak" "$SB/kitty/glfw-x11.so"     # RESTORE
restore_exit=0
$ ls -l "$SB/kitty/glfw-x11.so"                       # AFTER (restored)
-rwxr-xr-x 1 root root 373896 Jul 14 20:38 /tmp/kitty_qa_sandbox.To8dsh/kitty/glfw-x11.so
```

**Repeat (RUN #2).** RUN #2 produced the same terminal counts — `Ran 145 tests`, `FAILED (failures=4, errors=1, skipped=4)`, exit **1** — and the same five failing test IDs. The 145 per-test progress lines are shown complete in RUN #1 above; the block below is a **labeled excerpt of RUN #2** (the elided middle is explicitly marked) showing the DURING-absent state, the two GLFW tracebacks, and the summary tail, to demonstrate stability without duplicating the 145 identical progress lines:

```text
$ ls -l "$SB/kitty/glfw-x11.so" 2>&1                  # DURING (absent; glfw-wayland.so left in place)
ls: cannot access '/tmp/kitty_qa_sandbox.To8dsh/kitty/glfw-x11.so': No such file or directory
[[ labelled elision — 145 per-test progress lines omitted here; byte-identical in kind to RUN #1, which is shown complete above ]]

ERROR: test_utf_8_strndup (kitty_tests.glfw.TestGLFW.test_utf_8_strndup)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/glfw.py", line 50, in test_utf_8_strndup
    lib = ctypes.CDLL(backend_utils)
    backend_utils = '/tmp/kitty_qa_sandbox.To8dsh/kitty/glfw-x11.so'
    ctypes = <module 'ctypes' from '/usr/lib/python3.13/ctypes/__init__.py'>
    glfw_path = <function glfw_path at 0x784d7f8971a0>
    self = <kitty_tests.glfw.TestGLFW testMethod=test_utf_8_strndup>
  File "/usr/lib/python3.13/ctypes/__init__.py", line 390, in __init__
    self._handle = _dlopen(self._name, mode)
                   ~~~~~~~^^^^^^^^^^^^^^^^^^
    _FuncPtr = <class 'ctypes.CDLL.__init__.<locals>._FuncPtr'>
    flags = 1
    handle = None
    mode = 0
    name = '/tmp/kitty_qa_sandbox.To8dsh/kitty/glfw-x11.so'
    self = <CDLL '/tmp/kitty_qa_sandbox.To8dsh/kitty/glfw-x11.so', handle 0 at 0x784d7db5fcb0>
    use_errno = False
    use_last_error = False
    winmode = None
OSError: /tmp/kitty_qa_sandbox.To8dsh/kitty/glfw-x11.so: cannot open shared object file: No such file or directory

======================================================================

FAIL: test_glfw_modules (kitty_tests.check_build.TestBuild.test_glfw_modules)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/kitty_qa_sandbox.To8dsh/kitty/launcher/../../kitty_tests/check_build.py", line 46, in test_glfw_modules
    self.assertTrue(os.path.isfile(path), f'{path} is not a file')
    ~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    glfw_path = <function glfw_path at 0x784d7f8971a0>
    is_macos = False
    linux_backends = ['x11']
    modules = ['x11']
    name = 'x11'
    path = '/tmp/kitty_qa_sandbox.To8dsh/kitty/glfw-x11.so'
    self = <kitty_tests.check_build.TestBuild testMethod=test_glfw_modules>
AssertionError: False is not true : /tmp/kitty_qa_sandbox.To8dsh/kitty/glfw-x11.so is not a file

======================================================================

Ran 145 tests in 21.683s
FAILED (failures=4, errors=1, skipped=4)
# exit code: 1
```

### Sandbox teardown

After the three tiers, the temporary copy was deleted and its removal verified; the source repository was untouched throughout. The residue check uses `find` (which recurses to any depth), **not** a `"$SB"/**/*.qabak` glob: under bash's default `globstar=off`, `**` behaves like a single `*`, so `"$SB"/**/*.qabak` expands like `"$SB"/*/*.qabak` and would silently **miss** a depth-2 residue such as `kittens/transfer/rsync.so.qabak` — precisely the file Tier 2 moves aside. The teardown check therefore reports absence soundly:

```text
$ find "$SB" -type f -name '*.qabak' -print | wc -l    # recursively confirm no leftover moved-aside files at ANY depth
0
$ rm -rf "$SB"; echo "rm_exit=$?"
rm_exit=0
$ ls -d "$SB" 2>&1                              # confirm sandbox gone
ls: cannot access '/tmp/kitty_qa_sandbox.To8dsh': No such file or directory
```

*Why `find` and not the glob (observed).* Planting one depth-1 residue (`kitty/glfw-x11.so.qabak`) and one depth-2 residue (`kittens/transfer/rsync.so.qabak`) in a scratch tree and running both checks with `globstar` off (the default) shows the glob under-counts while `find` catches every depth:

```text
$ shopt globstar
globstar       	off
$ ls -1 "$SB"/**/*.qabak 2>/dev/null            # BUGGY: non-recursive under globstar=off — sees only depth-1
$SB/kitty/glfw-x11.so.qabak
$ ls -1 "$SB"/**/*.qabak 2>/dev/null | wc -l
1
$ find "$SB" -type f -name '*.qabak' -print     # CORRECT: recurses to any depth
$SB/kittens/transfer/rsync.so.qabak
$SB/kitty/glfw-x11.so.qabak
$ find "$SB" -type f -name '*.qabak' -print | wc -l
2
```

### Tier 4 — `glfw-wayland.so` missing → OPTIONAL, and unexercised under canonical CI

The fourth kitty-authored artifact, `glfw-wayland.so`, is exercised **separately** from `glfw-x11.so`. The only reference to a wayland backend anywhere under `kitty_tests/` is `kitty_tests/check_build.py:L42` (`linux_backends.append('wayland')`), which runs **only** `if not self.is_ci` (`kitty_tests/check_build.py:L40-L42`); the ctypes probe in `kitty_tests/glfw.py:L49` hardcodes `glfw_path('x11')` (with `ctypes.CDLL` at `kitty_tests/glfw.py:L50`). Prediction from those code paths: under canonical CI (`CI=true`), removing `glfw-wayland.so` has **no effect**. This was confirmed by direct observation — a fresh temporary copy, with `glfw-wayland.so` moved aside and `glfw-x11.so` left in place, run twice under CI. Fresh sandbox creation:

```text
$ SB="$(mktemp -d /tmp/kitty_qa_sandbox.XXXXXX)"   # -> /tmp/kitty_qa_sandbox.qLOYON
$ tar cf - --exclude='./.git' -C "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d" . | ( cd "$SB" && tar xf - )
copy_exit=0
$ test -e "$SB/.git" && echo GIT_PRESENT || echo NO_GIT_COPIED
NO_GIT_COPIED
$ ls -l "$SB/kitty/glfw-x11.so" "$SB/kitty/glfw-wayland.so"
-rwxr-xr-x 1 root root 451016 Jul 14 20:38 /tmp/kitty_qa_sandbox.qLOYON/kitty/glfw-wayland.so
-rwxr-xr-x 1 root root 373896 Jul 14 20:38 /tmp/kitty_qa_sandbox.qLOYON/kitty/glfw-x11.so
```

Complete, unedited RUN #1 (327 lines: before/during/the full 145-test run/summary/exit code/restore/after):

```text
########## TIER 4 (glfw-wayland.so unavailable; glfw-x11.so left in place) — RUN 1 ##########
$ ls -l "$SB/kitty/glfw-wayland.so"                   # BEFORE
-rwxr-xr-x 1 root root 451016 Jul 14 20:38 /tmp/kitty_qa_sandbox.qLOYON/kitty/glfw-wayland.so
$ mv "$SB/kitty/glfw-wayland.so" "$SB/kitty/glfw-wayland.so.qabak"   # render unavailable (temp copy only)
mv_exit=0
$ ls -l "$SB/kitty/glfw-wayland.so" 2>&1              # DURING (absent; glfw-x11.so still present)
ls: cannot access '/tmp/kitty_qa_sandbox.qLOYON/kitty/glfw-wayland.so': No such file or directory
$ ls -l "$SB/kitty/glfw-x11.so"                       # DURING (glfw-x11.so present)
-rwxr-xr-x 1 root root 373896 Jul 14 20:38 /tmp/kitty_qa_sandbox.qLOYON/kitty/glfw-x11.so
$ ( cd "$SB" && CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./kitty/launcher/kitty +launch test.py )
Running under CI: True
Using PATH in test environment: /tmp/kitty_qa_sandbox.qLOYON/kitty_tests/kitty/launcher:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Python: /usr/bin/python
Intrinsics: has_avx2=True has_sse4_2=True
Go executable: /usr/local/bin/go
Go packages being tested: tools/utils tools/tui/shell_integration tools/utils/style tools/wcswidth tools/unicode_names tools/utils/humanize tools/tui/subseq kittens/ssh tools/cmd/at tools/cli tools/tui/graphics kittens/transfer kittens/hyperlinked_grep tools/themes tools/utils/base85 tools/simdstring tools/tui tools/utils/shm tools/tui/sgr tools/tui/loop kittens/diff tools/rsync tools/tui/readline tools/utils/shlex kittens/hints tools/config
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
test_transfer_receive (kitty_tests.file_transmission.TestFileTransmission.test_transfer_receive) ... FAIL
test_transfer_send (kitty_tests.file_transmission.TestFileTransmission.test_transfer_send) ... FAIL
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
test_font_selection (kitty_tests.fonts.Selection.test_font_selection) ... FAIL
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

======================================================================
FAIL: test_transfer_receive (kitty_tests.file_transmission.TestFileTransmission.test_transfer_receive)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/kitty_qa_sandbox.qLOYON/kitty/launcher/../../kitty_tests/file_transmission.py", line 476, in test_transfer_receive
    self.basic_transfer_tests()
    ~~~~~~~~~~~~~~~~~~~~~~~~~^^
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
  File "/tmp/kitty_qa_sandbox.qLOYON/kitty/launcher/../../kitty_tests/file_transmission.py", line 443, in basic_transfer_tests
    multiple_files()
    ~~~~~~~~~~~~~~^^
    d = <_io.BufferedWriter name='/tmp/tmpc7yxm_9f/dest'>
    dest = '/tmp/tmpc7yxm_9f/dest'
    multiple_files = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files at 0x783b305e7b00>
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x783b31b9d160>
    s = <_io.BufferedWriter name='/tmp/tmpc7yxm_9f/src'>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
    single_file = <function TestFileTransmission.basic_transfer_tests.<locals>.single_file at 0x783b304ee160>
    src = '/tmp/tmpc7yxm_9f/src'
  File "/tmp/kitty_qa_sandbox.qLOYON/kitty/launcher/../../kitty_tests/file_transmission.py", line 432, in multiple_files
    self.assertEqual(expected, actual)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^
    Entry = <class 'kitty_tests.file_transmission.Entry'>
    actual = {'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784062477624572839, mode='0o120777', nlink=1), 'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'sym': Entry(relpath='sym', mtime=1784062477624572839, mode='0o120777', nlink=1), 'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1)}
    b = PosixPath('/tmp/tmpc7yxm_9f/msrc')
    cmd = ()
    de = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.de at 0x783b30330fe0>
    dest = '/tmp/tmpc7yxm_9f/mdest'
    dirnames = []
    dirpath = '/tmp/tmpc7yxm_9f/mdest/msrc/empty'
    entry = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.entry at 0x783b30330360>
    expected = {'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1), 'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784062477624572839, mode='0o120777', nlink=1), 'sym': Entry(relpath='sym', mtime=1784062477624572839, mode='0o120777', nlink=1)}
    f = <_io.BufferedWriter name='/tmp/tmpc7yxm_9f/msrc/sub/reg'>
    filenames = []
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x783b31b3dc50>
    s = PosixPath('/tmp/tmpc7yxm_9f/msrc/sub')
    se = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.se at 0x783b30330400>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
    src = '/tmp/tmpc7yxm_9f/msrc'
    x = 'reg'
AssertionError: {'simple': Entry(relpath='simple', mtime=130[497 chars]k=1)} != {'sub': Entry(relpath='sub', mtime=0, mode='[497 chars]k=1)}
  {'abssym': Entry(relpath='abssym', mtime=1784062477624572839, mode='0o120777', nlink=1),
-  'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2),
?                                                    ^

+  'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2),
?                                                    ^

   'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2),
   'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2),
-  'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2),
?                                                ^

+  'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2),
?                                                ^

   'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1),
   'sym': Entry(relpath='sym', mtime=1784062477624572839, mode='0o120777', nlink=1)}

======================================================================
FAIL: test_transfer_send (kitty_tests.file_transmission.TestFileTransmission.test_transfer_send)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/kitty_qa_sandbox.qLOYON/kitty/launcher/../../kitty_tests/file_transmission.py", line 507, in test_transfer_send
    self.basic_transfer_tests()
    ~~~~~~~~~~~~~~~~~~~~~~~~~^^
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
  File "/tmp/kitty_qa_sandbox.qLOYON/kitty/launcher/../../kitty_tests/file_transmission.py", line 443, in basic_transfer_tests
    multiple_files()
    ~~~~~~~~~~~~~~^^
    d = <_io.BufferedWriter name='/tmp/tmpxnf4xh1h/dest'>
    dest = '/tmp/tmpxnf4xh1h/dest'
    multiple_files = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files at 0x783b303334c0>
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x783b304e27b0>
    s = <_io.BufferedWriter name='/tmp/tmpxnf4xh1h/src'>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
    single_file = <function TestFileTransmission.basic_transfer_tests.<locals>.single_file at 0x783b303313a0>
    src = '/tmp/tmpxnf4xh1h/src'
  File "/tmp/kitty_qa_sandbox.qLOYON/kitty/launcher/../../kitty_tests/file_transmission.py", line 432, in multiple_files
    self.assertEqual(expected, actual)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^
    Entry = <class 'kitty_tests.file_transmission.Entry'>
    actual = {'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784062478387581059, mode='0o120777', nlink=1), 'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'sym': Entry(relpath='sym', mtime=1784062478387581059, mode='0o120777', nlink=1), 'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1)}
    b = PosixPath('/tmp/tmpxnf4xh1h/msrc')
    cmd = ()
    de = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.de at 0x783b30354720>
    dest = '/tmp/tmpxnf4xh1h/mdest'
    dirnames = []
    dirpath = '/tmp/tmpxnf4xh1h/mdest/msrc/empty'
    entry = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.entry at 0x783b30333b00>
    expected = {'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1), 'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784062478387581059, mode='0o120777', nlink=1), 'sym': Entry(relpath='sym', mtime=1784062478387581059, mode='0o120777', nlink=1)}
    f = <_io.BufferedWriter name='/tmp/tmpxnf4xh1h/msrc/sub/reg'>
    filenames = []
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x783b303443c0>
    s = PosixPath('/tmp/tmpxnf4xh1h/msrc/sub')
    se = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.se at 0x783b30333ba0>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
    src = '/tmp/tmpxnf4xh1h/msrc'
    x = 'reg'
AssertionError: {'simple': Entry(relpath='simple', mtime=130[497 chars]k=1)} != {'sub': Entry(relpath='sub', mtime=0, mode='[497 chars]k=1)}
  {'abssym': Entry(relpath='abssym', mtime=1784062478387581059, mode='0o120777', nlink=1),
-  'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2),
?                                                    ^

+  'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2),
?                                                    ^

   'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2),
   'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2),
-  'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2),
?                                                ^

+  'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2),
?                                                ^

   'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1),
   'sym': Entry(relpath='sym', mtime=1784062478387581059, mode='0o120777', nlink=1)}

======================================================================
FAIL: test_font_selection (kitty_tests.fonts.Selection.test_font_selection)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/kitty_qa_sandbox.qLOYON/kitty/launcher/../../kitty_tests/fonts.py", line 64, in test_font_selection
    t('Source Code Pro', 'SourceCodePro', 'Semibold', 'It')
    ~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    both = <function Selection.test_font_selection.<locals>.both at 0x783b2af35120>
    has = <function Selection.test_font_selection.<locals>.has at 0x783b2af34220>
    names = {'dejavu sans mono', 'noto mono', 'ubuntu mono', 'noto sans signwriting', 'jetbrains mono nl', 'jetbrains mono', 'liberation mono', 'ubuntu sans mono', 'inconsolata', 'fira code'}
    opts = <kitty.options.types.Options object at 0x783b2af560d0>
    s = <function Selection.test_font_selection.<locals>.s at 0x783b30f70720>
    self = <kitty_tests.fonts.Selection testMethod=test_font_selection>
    t = <function Selection.test_font_selection.<locals>.t at 0x783b2af35260>
  File "/tmp/kitty_qa_sandbox.qLOYON/kitty/launcher/../../kitty_tests/fonts.py", line 58, in t
    if has(family, allow_missing_in_ci=allow_missing_in_ci):
       ~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    allow_missing_in_ci = False
    alternate = None
    bi = ''
    bold = 'Semibold'
    both = <function Selection.test_font_selection.<locals>.both at 0x783b2af35120>
    family = 'Source Code Pro'
    has = <function Selection.test_font_selection.<locals>.has at 0x783b2af34220>
    italic = 'It'
    psprefix = 'SourceCodePro'
    reg = 'Regular'
  File "/tmp/kitty_qa_sandbox.qLOYON/kitty/launcher/../../kitty_tests/fonts.py", line 54, in has
    raise AssertionError(f'The family: {family} is not available')
    allow_missing_in_ci = False
    ans = False
    family = 'Source Code Pro'
    names = {'dejavu sans mono', 'noto mono', 'ubuntu mono', 'noto sans signwriting', 'jetbrains mono nl', 'jetbrains mono', 'liberation mono', 'ubuntu sans mono', 'inconsolata', 'fira code'}
    self = <kitty_tests.fonts.Selection testMethod=test_font_selection>
AssertionError: The family: Source Code Pro is not available

----------------------------------------------------------------------
Ran 145 tests in 34.537s

FAILED (failures=3, skipped=4)
All Go tests succeeded, ran in 34.7 seconds
[31mError[39m: Some tests failed!
# exit code: 1
$ mv "$SB/kitty/glfw-wayland.so.qabak" "$SB/kitty/glfw-wayland.so"   # RESTORE
restore_exit=0
$ ls -l "$SB/kitty/glfw-wayland.so"                    # AFTER (restored)
-rwxr-xr-x 1 root root 451016 Jul 14 20:38 /tmp/kitty_qa_sandbox.qLOYON/kitty/glfw-wayland.so
```

**Repeat (RUN #2).** The behavioral result is identical — `Ran 145 tests`, `FAILED (failures=3, skipped=4)`, exit **1**, and the failing IDs are exactly the **three baseline** failures (`test_font_selection`, `test_transfer_send`, `test_transfer_receive`) with **no** GLFW FAIL or ERROR. Only the wall-clock time differs (RUN #1 `34.537s` vs RUN #2 `22.022s`), which does not affect any count. Labeled excerpt of RUN #2:

```text
$ ls -l "$SB/kitty/glfw-wayland.so" 2>&1              # DURING (absent; glfw-x11.so still present)
ls: cannot access '/tmp/kitty_qa_sandbox.qLOYON/kitty/glfw-wayland.so': No such file or directory
$ ls -l "$SB/kitty/glfw-x11.so"                       # DURING (glfw-x11.so present)
-rwxr-xr-x 1 root root 373896 Jul 14 20:38 /tmp/kitty_qa_sandbox.qLOYON/kitty/glfw-x11.so
[[ labelled elision — 145 per-test progress lines omitted here; byte-identical in kind to RUN #1 (shown complete above), with no GLFW FAIL or ERROR ]]

Ran 145 tests in 22.022s

FAILED (failures=3, skipped=4)
All Go tests succeeded, ran in 22.1 seconds
[31mError[39m: Some tests failed!
# exit code: 1
```

This is **identical to the happy-path baseline** (Q1.b: `Ran 145 tests`, `FAILED (failures=3, skipped=4)`), which is the observed proof that `glfw-wayland.so` is not consumed at all by the suite under canonical CI. Sandbox teardown for Tier 4:

```text
$ find "$SB" -type f -name '*.qabak' -print | wc -l    # recursive residue check (see the Tier 3 teardown note on why not a **/*.qabak glob)
0
$ rm -rf "$SB"; echo rm_exit=$?
rm_exit=0
$ ls -d "$SB" 2>&1
ls: cannot access '/tmp/kitty_qa_sandbox.qLOYON': No such file or directory
```

### Q7 — Critical vs optional (all four kitty-authored extensions, each observed)

The classification below is grounded in the four experiments above; every row was **observed** (not inferred), including `glfw-wayland.so`:

| Extension `.so` | Classification | Observed effect when missing (canonical CI) | Tests that run | Fatal point (cause) |
|---|---|---|---|---|
| `kitty/fast_data_types.so` | **CRITICAL** | `ModuleNotFoundError: No module named 'kitty.fast_data_types'` while importing `kitty_tests.main`; banner never prints | **0** | Harness import — top-level import in `BaseTest` module chain (`kitty_tests/__init__.py:L21` → `kitty/config.py:L10` → `kitty/conf/utils.py:L27`, plus direct `kitty_tests/__init__.py:L22`) |
| `kittens/transfer/rsync.so` | **CRITICAL** | Banner prints and Go pkgs enumerate, then `ModuleNotFoundError: No module named 'kittens.transfer.rsync'` at collection | **0 Python** | Collection — `find_all_tests()` direct `importlib.import_module()` at `kitty_tests/main.py:L64` importing `kitty_tests/file_transmission.py:L13` |
| `kitty/glfw-x11.so` | **OPTIONAL** (localized) | `Ran 145 tests … FAILED (failures=4, errors=1)`; delta vs baseline = **1 FAIL + 1 ERROR** | **145** | Not fatal — on-demand `ctypes.CDLL` at `kitty_tests/glfw.py:L50` (ERROR) and `os.path.isfile` at `kitty_tests/check_build.py:L46` (FAIL) |
| `kitty/glfw-wayland.so` | **OPTIONAL** (unexercised under CI) | `Ran 145 tests … FAILED (failures=3, skipped=4)` — **identical to baseline**, delta = **0** | **145** | Never referenced under CI — the only reference (`kitty_tests/check_build.py:L42`) is gated by `if not self.is_ci` |

**Why the two critical extensions differ in *where* they are fatal (cause → effect).** `fast_data_types` is fatal *earlier* — at **harness import time** — because it is imported at the top level of the `BaseTest` module, which `test.py:L8` pulls in while importing `kitty_tests.main`; the failure precedes `main()`, so the banner from `env_for_python_tests()` (`kitty_tests/main.py:L305`) never prints. `rsync` is fatal *later* — at **collection time** — because it is imported at the top level of a single test module (`kitty_tests/file_transmission.py:L13`) that `find_all_tests()` imports directly at `kitty_tests/main.py:L64`; by then `main()` has already printed the banner and enumerated Go packages. Both are suite-fatal (0 tests / 0 Python tests), but at different stages, which is exactly what the tier-1 vs tier-2 logs show.

**Why the two GLFW backends are optional.** Neither backend is imported as a Python module (Q3: no GLFW `.so` in `sys.modules`). `glfw-x11.so` is consumed only on demand by two tests, so its absence is localized to those two. `glfw-wayland.so` is not consumed at all under CI, so its absence is invisible. In both cases collection completes and all 145 tests run.

## Q5 — What the test output reveals about the extension dependency structure

The observed load set (Q3), the three-tier cascade (Q4), and the enumerated import sites (Q6 below) together reveal a **three-level dependency hierarchy** among the four kitty-authored extensions:

**Level 0 — `fast_data_types.so` is the universal root (single point of failure).** It is loaded **eagerly**, at module-import time, before any test runs. `kitty_tests/__init__.py` — the module that defines the `BaseTest` superclass every test class extends — imports it *twice* at the top level: first **transitively** at `kitty_tests/__init__.py:L21` (`from kitty.config import …` → `kitty/config.py:L10` → `kitty/conf/utils.py:L27` `from ..fast_data_types import Color`), then **directly** at `kitty_tests/__init__.py:L22` (`from kitty.fast_data_types import Cursor, HistoryBuf, LineBuf, Screen, get_options, monotonic, set_options`). Because *every* test class subclasses `BaseTest`, **all 22 test categories depend on `fast_data_types`** whether or not they name it themselves. This is what the Q3 probe observed (it appears in `sys.modules` immediately after importing `kitty_tests.main`) and what Q4 Tier 1 confirmed by removal (its absence is fatal at *harness import*, before the banner, 0 tests run). *This is the reason `fast_data_types` is the central single-point-of-failure: it sits at the root of the import graph via `BaseTest`.*

**Level 1 — `rsync.so` is a secondary hard dependency of exactly one collected module.** It is loaded at **collection time** (not before), because it is imported at the top level of a single test module, `kitty_tests/file_transmission.py:L13` (`from kittens.transfer.rsync import Differ, Hasher, Patcher, parse_ftc`). The Q3 probe observed it entering `sys.modules` only during `find_all_tests()` (the collection step), and Q4 Tier 2 confirmed its absence is fatal at *collection* (after the banner, 0 Python tests). A **second, lazy** consumer exists — `kitty_tests/check_build.py:L30` (`from kittens.transfer import rsync`, inside the `test_loading_extensions()` method at `kitty_tests/check_build.py:L28`) — but because it is inside a method body it is not evaluated at collection time, so it does not participate in the collection-time cascade; it would only be exercised when that one test runs. (This corrects any notion that "only one module needs rsync": **two** modules reference it — one hard/top-level, one lazy/in-test.)

**Level 2 — the GLFW backends are leaves, loaded on demand, x11-only under CI.** `glfw-x11.so` and `glfw-wayland.so` are **never imported as Python modules** (the Q3 probe found no GLFW `.so` in `sys.modules`). They are consumed only on demand, by path, in exactly two modules: `kitty_tests/glfw.py:L50` (`ctypes.CDLL(glfw_path('x11'))`) and `kitty_tests/check_build.py:L46` (`os.path.isfile(glfw_path(name))`). Under `CI=true`, `kitty_tests/check_build.py:L40-L42` restricts the checked backends to `['x11']` and `kitty_tests/glfw.py:L49` hardcodes `glfw_path('x11')`, so **only `glfw-x11.so` is touched**; `glfw-wayland.so` is not referenced at all (Q4 Tier 4 observed zero effect on its removal). Their absence is therefore *localized* (Q4 Tier 3: 145 tests still run; only 1 FAIL + 1 ERROR).

**What the structure implies (cause → effect).** The test output reveals a dependency *tree* rooted at `fast_data_types` (every test transitively needs it), with `rsync` as a single hard branch (one collected module) and the GLFW backends as optional leaves (two tests, on demand). This is precisely why the failure severity is tiered: removing the root aborts everything before it starts; removing the single-branch dependency aborts collection; removing a leaf degrades only the two tests that reach for it. The observed load order in Q3 (`fast_data_types` at import, `rsync` at collection, GLFW never) is the runtime signature of exactly this hierarchy.

## Q6 — How the compiled extensions connect to the different test categories

`find_all_tests()` (`kitty_tests/main.py:L57`) imports **every** `.py` module under `kitty_tests/` except `main` and `gr` (its `excludes=('main', 'gr')`), i.e. **23 modules** (22 test categories + the `__init__` base module). The enumeration below is the reproducible basis for the mapping table; it lists every extension import site by `file:line`:

```text
$ # Enumeration of extension import sites across kitty_tests/ (source references; file:line facts)
$ ls kitty_tests/*.py | sed "s#kitty_tests/##;s#\.py##" | grep -vE "^(main|gr)$" | wc -l   # modules find_all_tests imports (excludes main,gr)
23

$ # (A) Modules that reference kitty.fast_data_types in their OWN source (direct importers, incl lazy in-method):
$ grep -lE "kitty\.fast_data_types" kitty_tests/*.py | sed "s#kitty_tests/##;s#\.py##"
__init__
check_build
crypto
datatypes
fonts
graphics
keys
main
mouse
options
parser
screen
shell_integration
shm
ssh
utmp

$ # (B) Every fast_data_types import site with line number:
$ grep -nE "kitty\.fast_data_types" kitty_tests/*.py
kitty_tests/__init__.py:22:from kitty.fast_data_types import Cursor, HistoryBuf, LineBuf, Screen, get_options, monotonic, set_options
kitty_tests/check_build.py:29:        import kitty.fast_data_types as fdt
kitty_tests/crypto.py:28:        from kitty.fast_data_types import AES256GCMDecrypt, AES256GCMEncrypt, CryptoError, EllipticCurveKey
kitty_tests/datatypes.py:9:from kitty.fast_data_types import (
kitty_tests/datatypes.py:22:from kitty.fast_data_types import Cursor as C
kitty_tests/datatypes.py:578:        from kitty.fast_data_types import GLFW_MOD_KITTY, GLFW_MOD_SHIFT, SingleKey
kitty_tests/fonts.py:11:from kitty.fast_data_types import DECAWM, get_fallback_font, sprite_map_set_layout, sprite_map_set_limits, test_render_line, test_sprite_position_for, wcwidth
kitty_tests/graphics.py:14:from kitty.fast_data_types import base64_decode, base64_encode, has_avx2, has_sse4_2, load_png_data, shm_unlink, shm_write, test_xor64
kitty_tests/keys.py:6:import kitty.fast_data_types as defines
kitty_tests/main.py:309:        from kitty.fast_data_types import has_avx2, has_sse4_2
kitty_tests/mouse.py:6:from kitty.fast_data_types import (
kitty_tests/options.py:5:from kitty.fast_data_types import Color
kitty_tests/parser.py:8:from kitty.fast_data_types import (
kitty_tests/screen.py:4:from kitty.fast_data_types import DECAWM, DECCOLM, DECOM, IRM, VT_PARSER_BUFFER_SIZE, Cursor
kitty_tests/shell_integration.py:16:from kitty.fast_data_types import CURSOR_BEAM, CURSOR_BLOCK, CURSOR_UNDERLINE
kitty_tests/shm.py:9:from kitty.fast_data_types import shm_unlink
kitty_tests/ssh.py:16:from kitty.fast_data_types import CURSOR_BEAM, shm_unlink
kitty_tests/utmp.py:3:from kitty.fast_data_types import num_users

$ # (C) Every rsync import site with line number:
$ grep -nE "transfer\.rsync|transfer import rsync" kitty_tests/*.py
kitty_tests/check_build.py:30:        from kittens.transfer import rsync
kitty_tests/file_transmission.py:13:from kittens.transfer.rsync import Differ, Hasher, Patcher, parse_ftc

$ # (D) Every GLFW backend reference (glfw_path / ctypes.CDLL / os.path.isfile on backend) with line number:
$ grep -nE "glfw_path|ctypes\.CDLL\(backend" kitty_tests/glfw.py kitty_tests/check_build.py
kitty_tests/glfw.py:47:        from kitty.constants import glfw_path
kitty_tests/glfw.py:49:        backend_utils = glfw_path('x11')
kitty_tests/glfw.py:50:        lib = ctypes.CDLL(backend_utils)
kitty_tests/check_build.py:39:        from kitty.constants import glfw_path, is_macos
kitty_tests/check_build.py:45:            path = glfw_path(name)
```

**Classification of the 23 modules by how they reach `fast_data_types`:**

- **15 modules import it directly** (a `kitty.fast_data_types` statement appears in their own source): the `__init__` base module plus 14 categories — `check_build`, `crypto`, `datatypes`, `fonts`, `graphics`, `keys`, `mouse`, `options`, `parser`, `screen`, `shell_integration`, `shm`, `ssh`, `utmp`. (The grep above also lists `main`, but `main` is one of the two excluded modules, so it is **not** among the 23 discovered modules.) Of these 15, **13 import at module top level** and **2 import lazily inside a method** — `kitty_tests/check_build.py:L29` (inside `test_loading_extensions()`) and `kitty_tests/crypto.py:L28`.
- **8 categories depend on it only transitively**, via the `BaseTest` superclass in `kitty_tests/__init__.py` — `clipboard`, `completion`, **`file_transmission`**, `glfw`, `layout`, `open_actions`, `search_query_parser`, `tui`. (`file_transmission` belongs here: it has **no** direct `fast_data_types` import; its only compiled-extension import is `rsync`.)

**15 direct + 8 transitive-only = 23.** Every one of the 22 test categories ultimately depends on `fast_data_types` (directly or via `BaseTest`); `datatypes` is notable for having **three** distinct import sites — two top level (`kitty_tests/datatypes.py:L9` for `Color, ColorProfile, HistoryBuf, LineBuf, …` and `kitty_tests/datatypes.py:L22` for `Cursor as C`) and one lazy (`kitty_tests/datatypes.py:L578` for `GLFW_MOD_KITTY, GLFW_MOD_SHIFT, SingleKey`, inside `test_single_key()`).

**`rsync` connects to exactly two categories, and the GLFW backends to exactly two categories:**

- `rsync.so`: `kitty_tests/file_transmission.py:L13` (top-level — the sole *hard*/collection-time consumer) and `kitty_tests/check_build.py:L30` (lazy, inside `test_loading_extensions()` — a second in-test consumer).
- `glfw-x11.so`: `kitty_tests/glfw.py:L50` (`ctypes.CDLL`, in `test_utf_8_strndup`) and `kitty_tests/check_build.py:L46` (`os.path.isfile`, in `test_glfw_modules`). `glfw-wayland.so` is referenced by **no** category under CI (its only would-be reference, `kitty_tests/check_build.py:L42`, is gated by `if not self.is_ci`).

### Category → extension dependency table (all 23 discovered modules: 22 test categories + the `__init__` base module)

The first row is the `__init__` base module (which defines `BaseTest`); the remaining 22 rows are the test categories. All line numbers in a cell are scoped to that row's module (`kitty_tests/<module>.py`).

| Discovered module | `fast_data_types` | `rsync` | GLFW backend |
|---|---|---|---|
| `__init__` (base module) | direct `L22` + transitive `L21` | — | — |
| `check_build` | direct, lazy `L29` | **yes** — lazy `L30` | x11 — `L45`/`L46` (`os.path.isfile`, CI: x11 only) |
| `clipboard` | transitive (via `BaseTest`) | — | — |
| `completion` | transitive (via `BaseTest`) | — | — |
| `crypto` | direct, lazy `L28` | — | — |
| `datatypes` | direct `L9`, `L22` (top level) + `L578` (lazy) | — | — |
| `file_transmission` | transitive (via `BaseTest`) | **yes** — top level `L13` (hard) | — |
| `fonts` | direct `L11` | — | — |
| `glfw` | transitive (via `BaseTest`) | — | x11 — `L50` (`ctypes.CDLL`) |
| `graphics` | direct `L14` | — | — |
| `keys` | direct `L6` | — | — |
| `layout` | transitive (via `BaseTest`) | — | — |
| `mouse` | direct `L6` | — | — |
| `open_actions` | transitive (via `BaseTest`) | — | — |
| `options` | direct `L5` | — | — |
| `parser` | direct `L8` | — | — |
| `screen` | direct `L4` | — | — |
| `search_query_parser` | transitive (via `BaseTest`) | — | — |
| `shell_integration` | direct `L16` | — | — |
| `shm` | direct `L9` | — | — |
| `ssh` | direct `L16` | — | — |
| `tui` | transitive (via `BaseTest`) | — | — |
| `utmp` | direct `L3` | — | — |

The first row, `__init__`, is the `BaseTest`-defining base module — not itself a test category, but imported by `find_all_tests()` alongside the 22 categories, for **23** discovered modules in total. Counting `__init__`, **15** modules contain a direct `fast_data_types` import (`__init__` plus 14 categories) and the remaining **8** categories reach it transitive-only via `BaseTest` (15 + 8 = 23).

## Environment-artifact baseline failures (distinct from extension-load failures)

On the happy path (Q1.b), both canonical runs reported the **same three** failures — `Ran 145 tests … FAILED (failures=3, skipped=4)` — and the same three IDs in both runs: `test_transfer_receive`, `test_transfer_send`, `test_font_selection`. These are **environment artifacts, not extension-load failures**. The decisive distinction: **all three tests executed** (each reached an assertion deep inside its own test body), which is only possible if `fast_data_types` and `rsync` had already loaded successfully — the exact opposite of the Q4 cascade failures, where the missing extension aborts *before* any test runs. Complete, unedited failure blocks:

**(1) `test_font_selection` — missing font family "Source Code Pro" (not a code defect).** The test asserts a specific font family is installed; it is not present in this container. The available families are listed in the `names` set in the traceback:

```text
FAIL: test_font_selection (kitty_tests.fonts.Selection.test_font_selection)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/fonts.py", line 64, in test_font_selection
    t('Source Code Pro', 'SourceCodePro', 'Semibold', 'It')
    ~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    both = <function Selection.test_font_selection.<locals>.both at 0x78f47453aca0>
    has = <function Selection.test_font_selection.<locals>.has at 0x78f474538180>
    names = {'ubuntu mono', 'jetbrains mono', 'fira code', 'noto mono', 'inconsolata', 'liberation mono', 'dejavu sans mono', 'jetbrains mono nl', 'ubuntu sans mono', 'noto sans signwriting'}
    opts = <kitty.options.types.Options object at 0x78f47456a0d0>
    s = <function Selection.test_font_selection.<locals>.s at 0x78f474539f80>
    self = <kitty_tests.fonts.Selection testMethod=test_font_selection>
    t = <function Selection.test_font_selection.<locals>.t at 0x78f47453afc0>
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/fonts.py", line 58, in t
    if has(family, allow_missing_in_ci=allow_missing_in_ci):
       ~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    allow_missing_in_ci = False
    alternate = None
    bi = ''
    bold = 'Semibold'
    both = <function Selection.test_font_selection.<locals>.both at 0x78f47453aca0>
    family = 'Source Code Pro'
    has = <function Selection.test_font_selection.<locals>.has at 0x78f474538180>
    italic = 'It'
    psprefix = 'SourceCodePro'
    reg = 'Regular'
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/fonts.py", line 54, in has
    raise AssertionError(f'The family: {family} is not available')
    allow_missing_in_ci = False
    ans = False
    family = 'Source Code Pro'
    names = {'ubuntu mono', 'jetbrains mono', 'fira code', 'noto mono', 'inconsolata', 'liberation mono', 'dejavu sans mono', 'jetbrains mono nl', 'ubuntu sans mono', 'noto sans signwriting'}
    self = <kitty_tests.fonts.Selection testMethod=test_font_selection>
AssertionError: The family: Source Code Pro is not available
```

The assertion is raised at `kitty_tests/fonts.py:L54` (`raise AssertionError(f'The family: {family} is not available')`), reached from `kitty_tests/fonts.py:L58` and the call at `kitty_tests/fonts.py:L64` (`t('Source Code Pro', …)`). "Source Code Pro" is absent from the observed `names` set (which contains `dejavu sans mono`, `liberation mono`, `noto mono`, etc.). *Cause: the font is not installed in the environment; not an extension problem.*

**(2) `test_transfer_send` — a directory setgid-bit mismatch (filesystem artifact).** The assertion at `kitty_tests/file_transmission.py:L432` (`self.assertEqual(expected, actual)`, via `L443` and `L507`) compares directory-entry metadata. The only difference is the **setgid bit** on the two directory entries `empty` and `sub`: one side reports `mode='0o42755'` (the setgid bit `0o2000` **set**) and the other `mode='0o40755'` (setgid **clear**):

```text
FAIL: test_transfer_send (kitty_tests.file_transmission.TestFileTransmission.test_transfer_send)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/file_transmission.py", line 507, in test_transfer_send
    self.basic_transfer_tests()
    ~~~~~~~~~~~~~~~~~~~~~~~~~^^
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/file_transmission.py", line 443, in basic_transfer_tests
    multiple_files()
    ~~~~~~~~~~~~~~^^
    d = <_io.BufferedWriter name='/tmp/tmpnwx6k0ld/dest'>
    dest = '/tmp/tmpnwx6k0ld/dest'
    multiple_files = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files at 0x78f47506d120>
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x78f47520a7b0>
    s = <_io.BufferedWriter name='/tmp/tmpnwx6k0ld/src'>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
    single_file = <function TestFileTransmission.basic_transfer_tests.<locals>.single_file at 0x78f475052fc0>
    src = '/tmp/tmpnwx6k0ld/src'
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/file_transmission.py", line 432, in multiple_files
    self.assertEqual(expected, actual)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^
    Entry = <class 'kitty_tests.file_transmission.Entry'>
    actual = {'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784060444966675159, mode='0o120777', nlink=1), 'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'sym': Entry(relpath='sym', mtime=1784060444966675159, mode='0o120777', nlink=1), 'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1)}
    b = PosixPath('/tmp/tmpnwx6k0ld/msrc')
    cmd = ()
    de = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.de at 0x78f47506e340>
    dest = '/tmp/tmpnwx6k0ld/mdest'
    dirnames = []
    dirpath = '/tmp/tmpnwx6k0ld/mdest/msrc/empty'
    entry = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.entry at 0x78f47506d760>
    expected = {'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1), 'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784060444966675159, mode='0o120777', nlink=1), 'sym': Entry(relpath='sym', mtime=1784060444966675159, mode='0o120777', nlink=1)}
    f = <_io.BufferedWriter name='/tmp/tmpnwx6k0ld/msrc/sub/reg'>
    filenames = []
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x78f4752ccc00>
    s = PosixPath('/tmp/tmpnwx6k0ld/msrc/sub')
    se = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.se at 0x78f47506d800>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
    src = '/tmp/tmpnwx6k0ld/msrc'
    x = 'reg'
AssertionError: {'simple': Entry(relpath='simple', mtime=130[497 chars]k=1)} != {'sub': Entry(relpath='sub', mtime=0, mode='[497 chars]k=1)}
```

**(3) `test_transfer_receive` — the same setgid-bit mismatch** (`0o42755` vs `0o40755` on the directory entries):

```text
FAIL: test_transfer_receive (kitty_tests.file_transmission.TestFileTransmission.test_transfer_receive)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/file_transmission.py", line 476, in test_transfer_receive
    self.basic_transfer_tests()
    ~~~~~~~~~~~~~~~~~~~~~~~~~^^
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/file_transmission.py", line 443, in basic_transfer_tests
    multiple_files()
    ~~~~~~~~~~~~~~^^
    d = <_io.BufferedWriter name='/tmp/tmpd4jet1ub/dest'>
    dest = '/tmp/tmpd4jet1ub/dest'
    multiple_files = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files at 0x78f475050220>
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x78f475d51160>
    s = <_io.BufferedWriter name='/tmp/tmpd4jet1ub/src'>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
    single_file = <function TestFileTransmission.basic_transfer_tests.<locals>.single_file at 0x78f47530bd80>
    src = '/tmp/tmpd4jet1ub/src'
  File "/tmp/blitzy/kitty/blitzy-93df6d1d-62be-44c5-8cbd-2c5ec0295825_acd18d/kitty/launcher/../../kitty_tests/file_transmission.py", line 432, in multiple_files
    self.assertEqual(expected, actual)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^
    Entry = <class 'kitty_tests.file_transmission.Entry'>
    actual = {'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784060444386668911, mode='0o120777', nlink=1), 'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'sym': Entry(relpath='sym', mtime=1784060444386668911, mode='0o120777', nlink=1), 'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1)}
    b = PosixPath('/tmp/tmpd4jet1ub/msrc')
    cmd = ()
    de = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.de at 0x78f475052c00>
    dest = '/tmp/tmpd4jet1ub/mdest'
    dirnames = []
    dirpath = '/tmp/tmpd4jet1ub/mdest/msrc/empty'
    entry = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.entry at 0x78f475051f80>
    expected = {'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1), 'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1784060444386668911, mode='0o120777', nlink=1), 'sym': Entry(relpath='sym', mtime=1784060444386668911, mode='0o120777', nlink=1)}
    f = <_io.BufferedWriter name='/tmp/tmpd4jet1ub/msrc/sub/reg'>
    filenames = []
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x78f476859c50>
    s = PosixPath('/tmp/tmpd4jet1ub/msrc/sub')
    se = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.se at 0x78f475052020>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
    src = '/tmp/tmpd4jet1ub/msrc'
    x = 'reg'
AssertionError: {'simple': Entry(relpath='simple', mtime=130[497 chars]k=1)} != {'sub': Entry(relpath='sub', mtime=0, mode='[497 chars]k=1)}
```

**The `/tmp` filesystem is `ext4`, not `tmpfs`** — this is the observed value (any attribution of these failures to `tmpfs` would be incorrect):

```text
### command: df -T /tmp
Filesystem     Type   1K-blocks      Used   Available Use% Mounted on
/dev/nvme0n1p1 ext4 25800365232 363995284 25436353564   2% /tmp
### command: stat -f -c '%T' /tmp
ext2/ext3
### command: mount | grep ' /tmp '
/dev/nvme0n1p1 on /tmp type ext4 (rw,relatime,commit=30)
```

*Observed:* the mismatch is exactly `0o42755` vs `0o40755` (the setgid bit on newly-created directories under the test's `/tmp` scratch dir). The setgid bit is visible in full in the `expected=` and `actual=` local-variable dumps above; the complete element-by-element unittest diff (the native `-`/`+`/`?` marker lines that point at the differing `4`↔`0` character) appears verbatim in the Q1.b baseline output — the `[497 chars]` notation in the one-line `AssertionError` summary is Python's own `safe_repr` truncation of the dict repr, not an edit of this document. *Inferred:* the root cause is directory setgid-bit inheritance behavior of the mount (a new directory inherits the parent's setgid bit), which differs from what the test hard-codes as expected; this is a property of the container's filesystem/mount, not of the compiled extensions. Both transfer tests ran to completion (reaching `kitty_tests/file_transmission.py:L432`), which again confirms `rsync` and `fast_data_types` loaded.

**Bottom line:** these three failures are orthogonal to the extension-load question. They are stable across both runs (same three IDs), they occur *after* the extensions load, and none of them is an import/collection error. They are explicitly **out of scope** for the extension-cascade analysis and are reported here only to account for every line of the observed test output.

## Read-only compliance — the source repository is unchanged

Per the MainRule, no existing file in the source repository was modified; the **only** repository change is the authoring of this one deliverable document. This document was committed on top of the pre-deliverable **branch-point** across more than one reconciliation commit; because a committed document cannot embed its own final commit hash, all evidence below is anchored on the **stable, permanent branch-point commit** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (an ancestor of `HEAD`), never on a volatile relative ref such as `HEAD~1`. The `git diff --name-status` from that branch-point to the current `HEAD` lists exactly one path — the deliverable — proving **no tracked source file changed** (evidence shown **unfiltered**: all untracked files, no directory filter):

```text
$ # Stable pre-deliverable baseline = the branch-point commit (permanent hash, an ancestor of HEAD):
$ git rev-parse 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1

$ # Confirm the branch-point is the pre-deliverable source state (an ancestor of the current HEAD):
$ git merge-base --is-ancestor 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD && echo "ancestor: yes"
ancestor: yes

$ # Every tracked path that differs between the branch-point and the current HEAD:
$ git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD
A	blitzy/documentation/kitty_815df1e210e0.md

$ # The six build artifacts are git-ignored, so building leaves tracked state clean:
$ git check-ignore -v kitty/fast_data_types.so kittens/transfer/rsync.so kitty/glfw-x11.so kitty/glfw-wayland.so kitty/launcher/kitty kitty/launcher/kitten
.gitignore:1:*.so	kitty/fast_data_types.so
.gitignore:1:*.so	kittens/transfer/rsync.so
.gitignore:1:*.so	kitty/glfw-x11.so
.gitignore:1:*.so	kitty/glfw-wayland.so
.gitignore:18:/kitty/launcher/kitt*	kitty/launcher/kitty
.gitignore:18:/kitty/launcher/kitt*	kitty/launcher/kitten
```

Notes on why this is a complete accounting (no concealment):

- **Only the deliverable differs from baseline.** `git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD` returns a single line, `A blitzy/documentation/kitty_815df1e210e0.md`. No `.py`, `.c`, `.m`, `.go`, or configuration file appears. (This is a *tracked-state* claim, not a "byte-for-byte" claim: building the project necessarily writes ignored artifacts to disk, so a literal byte-for-byte assertion about the whole tree would be false — instead, no *tracked* file other than the deliverable is changed.)
- **The six build artifacts are git-ignored.** `git check-ignore -v` shows `kitty/fast_data_types.so`, `kittens/transfer/rsync.so`, `kitty/glfw-x11.so`, `kitty/glfw-wayland.so` matched by `.gitignore:1` (`*.so`) and both launchers (`kitty/launcher/kitty`, `kitty/launcher/kitten`) matched by `.gitignore:18` (`/kitty/launcher/kitt*`). That is why building the project — a prerequisite for the whole investigation — leaves tracked repository state clean, and why the working-tree status shown in the final block below has no artifact entries.
- **All observation scripts and sandboxes lived outside the repository.** The load probe (`/tmp/kitty_qa_logs/load_probe.py`), the enumeration/evidence logs (`/tmp/kitty_qa_logs/…`), and the cascade sandboxes (`/tmp/kitty_qa_sandbox.*`, created via `mktemp -d`) were all created under `/tmp`, never inside the working tree. The cascade experiments (Q4) mutated only a *temporary copy* of the built tree, restored each moved `.so`, and deleted the copy (see the Tier teardown logs). No temporary file was ever written into the repository, so none can appear in that clean working-tree status.

**Final post-commit state (observed).** After this deliverable was committed, the working tree is clean and the only change from the pre-deliverable baseline is the addition of this one document:

```text
$ # After the deliverable is committed, the working tree is clean (unfiltered, all untracked files shown):
$ git status --porcelain --untracked-files=all
$ echo "clean == the command above printed no lines"
clean == the command above printed no lines

$ # From the pre-deliverable baseline to HEAD, the ONE and only change is the deliverable being added:
$ git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD
A	blitzy/documentation/kitty_815df1e210e0.md
```

## Final coverage pass — every sub-question answered, by name

| # | Sub-question | Where answered | One-line observed answer |
|---|---|---|---|
| **Q1** | Build from source and run the suite via the canonical entry point | §Q1.a/b/c | Default `python3 setup.py` aborts at `glfw/wl_window.c:668` (`-Werror=switch`, exit 1); documented fallback `python3 setup.py --ignore-compiler-warnings` exits **0** and produces all six artifacts; `CI=true … ./kitty/launcher/kitty +launch test.py` runs `Ran 145 tests … FAILED (failures=3, skipped=4)`, exit 1, stable across two runs. |
| **Q2** | How the compiled C extensions connect to test-execution mechanics | §Q2 | `fast_data_types` loads eagerly at harness import (via `BaseTest`); `rsync` loads at collection (one top-level importer); GLFW backends are loaded on demand by path, never as modules. |
| **Q3** | Which compiled extension modules load during a run | §Q3 | The real-runner probe observed exactly **two kitty-authored** `.so` modules — `kitty.fast_data_types` (after harness import) and `kittens.transfer.rsync` (during collection); **no** GLFW module; `_json` is a **builtin**, not a `.so` (observed via `sys.builtin_module_names`). |
| **Q4** | How failures cascade when an extension is unavailable | §Q4 | Four observed tiers: `fast_data_types` fatal at harness import (0 tests); `rsync` fatal at collection (0 Python tests); `glfw-x11` localized (145 run, +1 FAIL +1 ERROR); `glfw-wayland` zero effect under CI. |
| **Q5** | What the output reveals about the dependency structure | §Q5 | A three-level tree: `fast_data_types` universal root (every `BaseTest` subclass), `rsync` a single hard branch (one collected module), the GLFW backends optional leaves (two tests, x11-only under CI). |
| **Q6** | How the extensions connect to the test categories | §Q6 | Of 23 discovered modules, **15 import `fast_data_types` directly** + **8 transitive-only** (incl. `file_transmission`) = 23; `rsync` → 2 categories (`file_transmission` hard `L13`, `check_build` lazy `L30`); GLFW → 2 categories (`glfw` `L50`, `check_build` `L46`), x11-only under CI. |
| **Q7** | Which extensions are critical vs optional | §Q4/Q7 | **Critical:** `fast_data_types.so`, `rsync.so` (both suite-fatal, at different stages). **Optional:** `glfw-x11.so` (localized), `glfw-wayland.so` (unexercised under CI). All four **observed**. |
| **Q8** | The actual import chains established during the run | §Q8 | Tier-1 chain `test.py:L8` → `kitty_tests/__init__.py:L21` → `kitty/config.py:L10` → `kitty/conf/utils.py:L27` → `fast_data_types`; Tier-2 chain `kitty_tests/main.py:L64` → `kitty_tests/file_transmission.py:L13` → `rsync`; Tier-3 on-demand `kitty_tests/glfw.py:L50` / `kitty_tests/check_build.py:L46` → `glfw-x11.so`. Each shown as a diagram **and** its verbatim traceback. |

**Corrected-fact ledger (relative to the AAP's pre-run expectations):** (1) the `kitten` binary is a **dynamically-linked** Go executable in the normal build (static linking `CGO_ENABLED=0` applies only to cross-platform release builds); (2) `rsync` has **two** consumers (hard + lazy), not one; (3) the category split is **15 + 8 = 23**, not 15 + 7 = 22, and `file_transmission` is transitive-only; (4) the Tier-3 delta is **+1 FAIL +1 ERROR** on top of the 3 baseline failures (totalling `failures=4, errors=1`); (5) `glfw-wayland.so` is **unexercised under CI** (observed, not inferred); (6) `_json` is a **builtin**, not a `.so`; (7) `/tmp` is **ext4**, not `tmpfs`; (8) the font count is **511 faces / 267 families**. Every one of these is grounded in an embedded command-and-output block above.
