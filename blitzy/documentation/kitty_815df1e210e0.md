# kitty — Compiled C Extensions vs. Test-Suite Execution

**Question answered:** How do kitty's compiled C extensions relate to its test-suite execution? Specifically: build kitty from source; execute the test suite through its real entry point; trace the relationship between the compiled C extensions and the test-execution flow; identify which extension modules actually get loaded during test execution; determine how test failures cascade when those extension modules are unavailable; explain what the test output reveals about the dependency structure; map how the compiled extensions connect to the different test categories; classify each extension as CRITICAL or OPTIONAL with respect to test execution; and document the actual import chains established during the test run.

## Methodology and provenance

Every factual claim below was produced by **building kitty from source and running its real test suite first**, then writing this document from the captured output. Beside every behavioural claim is the **exact command that produced it** together with that command's output. Two honest scope notes on the phrase "complete output":

1. The `setup.py build --verbose` log is **319 lines** of largely-repetitive per-file compiler invocations (the longest single line — the `fast_data_types` link command — reaches **3,626 characters**). Rather than paste all 319 lines, §1 embeds the build's **exit code, wall-clock duration, verbatim preamble, and self-contained `grep`/`nm`/`ls` commands whose own output is shown complete and unedited**. Every command that is *shown* has its full, verbatim output beside it; nothing inside a shown output block is paraphrased or elided.
2. Where a value is stated "stable across two runs", **both runs were executed**. The second run's output is shown in full when it differs materially (e.g. the two full test runs, §2.1/§2.2), and stated as byte-identical (with the exact re-run command given, so it is reproducible) when it is (e.g. the loaded-module and import-chain observers, §4/§9).

Statements that are **not** direct observations — conclusions derived by reading source rather than watching runtime behaviour — are explicitly marked **(inferred)**. Source locations are given as `file:line` against the checkout described below.

- **Source commit under investigation:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (kitty). This is the code that was read for all `file:line` citations.
- **Delivery branch (where this document is added):** `blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9`. This document is the *only* file added to the repository; no existing file is modified. The compiled `*.so` extensions are git-ignored generated build artifacts (`.gitignore:1` = `*.so`); any that were temporarily moved during the failure-cascade reproduction (§5) were restored byte-for-byte (verified by SHA-256 in the Appendix).
- **Canonical build command — exactly what kitty's CI runs (`.github/workflows/ci.py:104` = `{python} setup.py build --verbose`, with *no* `CC` override):** `python3 setup.py build --verbose`. On this container the host default compiler is `gcc` 15.2.0; the environment setup additionally recommends the `gcc-13` variant `CC=gcc-13 python3 setup.py build --verbose`. **Both build cleanly (exit 0)** — see §1.1–§1.3, which show each build's duration and the (differing) artifact sizes. The compiled artifacts used for every runtime observation in this document (§1.1 symbols, §1.3 sizes, §4 loading, §5 cascades, Appendix hashes) are from the `CC=gcc-13` build.
- **Canonical test command:** `CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./test.py` (single module: `./test.py --module <name>`).

**Shell setup assumed for every command block in this document.** The inherited container PATH does **not** include the Go toolchain (`/usr/local/go/bin`), and the C UTF-8 locale is required; unless a block shows a different environment inline, assume this setup is in effect:

```console
$ export PATH="/usr/local/go/bin:$PATH"    # kitten's `go build` and the Go test suite need `go` on PATH; otherwise it is "not found"
$ export LANG=C.UTF-8 LC_ALL=C.UTF-8        # required, or two zsh shell-integration tests error on UTF-8 handling
$ command -v go
/usr/local/go/bin/go
```

All temporary observation scripts lived under `/tmp` (never inside the repository) and were removed after use; their **full, runnable source is shown inline** where used (no elided bodies).

### Direct answer (summary)

- The canonical build produces **three** compiled shared objects, but they are **not** three of a kind: **two are Python C-extension modules** — `kitty/fast_data_types.so` (exports `PyInit_fast_data_types`) and `kittens/transfer/rsync.so` (exports `PyInit_rsync`) — and **one is a native (non-Python) shared library** — `kitty/glfw-x11.so` (has **no** `PyInit_*`; it is loaded via `dlopen`/`ctypes`, not `import`).
- During test execution, the two Python extensions are loaded as Python modules: `kitty.fast_data_types` at **bootstrap** (before any test runs) and `kittens.transfer.rsync` during **discovery**. `kitty/glfw-x11.so` is **never imported as a Python module**; instead it is **loaded natively via `ctypes.CDLL`** when the test body `kitty_tests/glfw.py:test_utf_8_strndup` actually runs.
- Classification: `fast_data_types.so` = **CRITICAL (root)**; `rsync.so` = **CRITICAL for the Python suite**; `glfw-x11.so` = **OPTIONAL for suite completion**, but its removal breaks **two** tests (one native-load `ERROR`, one file-check `FAIL`).

Each of these is proved with captured output in the sections that follow.

## Environment observed

The following versions were captured with the commands shown (each command precedes its output). This is the exact toolchain used for the canonical build and test runs.

```text
### command: uname -a
Linux reverse-code-generator-0dd512d2-st54t 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64 GNU/Linux

### command: cat /etc/os-release | head -1
PRETTY_NAME="Ubuntu 25.10"

### command: python3 --version
Python 3.13.7

### command: gcc-13 --version | head -1
gcc-13 (Ubuntu 13.4.0-4ubuntu1) 13.4.0

### command: gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0

### command: /usr/local/go/bin/go version
go version go1.22.12 linux/amd64

### command: pkg-config --version
1.8.1

### command: for p in harfbuzz libxxhash libcrypto libpng lcms2 fontconfig xkbcommon; do printf '%s: %s\n' "$p" "$(pkg-config --modversion "$p")"; done
harfbuzz: 10.2.0
libxxhash: 0.8.3
libcrypto: 3.5.3
libpng: 1.6.50
lcms2: 2.16
fontconfig: 2.15.0
xkbcommon: 1.7.0

### command: python3 -c 'import PIL, pygments; print("Pillow", PIL.__version__); print("pygments", pygments.__version__)'
Pillow 12.3.0
pygments 2.20.0
```

The build/test libraries were provisioned in this ephemeral container only (no repository manifest was changed). The one dependency that kitty's own CI apt list omits but `setup.py` requires — `libssl-dev`, which provides `libcrypto.pc` — was present, as confirmed below:

```text
### command: dpkg-query -W -f='${Package} ${Version} ${Status}\n' libssl-dev libharfbuzz-dev libxxhash-dev build-essential golang-go 2>&1
dpkg-query: no packages found matching golang-go
build-essential 12.12ubuntu1 install ok installed
libharfbuzz-dev 10.2.0-1 install ok installed
libssl-dev 3.5.3-1ubuntu3.4 install ok installed
libxxhash-dev 0.8.3-2 install ok installed

### command: pkg-config --exists libcrypto && echo 'libcrypto.pc FOUND'; echo "libcrypto.pc dir: $(pkg-config --variable=pcfiledir libcrypto)"
libcrypto.pc FOUND
libcrypto.pc dir: /usr/lib/x86_64-linux-gnu/pkgconfig

### command: pip show Pillow pygments | grep -E '^(Name|Version):'
Name: pillow
Version: 12.3.0
Name: Pygments
Version: 2.20.0
```

In the `dpkg-query` block above, the leading `dpkg-query: no packages found matching golang-go` line reflects that the Go toolchain is installed under `/usr/local/go` (per the `go version` line above), not via the apt `golang-go` package; the four library packages queried alongside it are each reported `install ok installed`. (`Pillow` and `pygments` were installed with `pip install --break-system-packages`, as this container is a PEP 668 externally-managed environment.)

## 1. Building kitty from source (canonical configuration)

kitty is built by its multi-language orchestrator `setup.py` (it uses `sysconfig`, not distutils, so it builds cleanly on Python 3.13). The **canonical** build command is exactly the one kitty's own CI runs — `build_kitty()` at `.github/workflows/ci.py:102-104` sets `cmd = f'{python} setup.py build --verbose'`, i.e. **`python3 setup.py build --verbose` with no `CC` override**. On this container the host default compiler is `gcc` 15.2.0; the environment setup additionally recommends building with `CC=gcc-13` (gcc 13.4.0). **Both configurations build cleanly (exit code 0).** Each was run twice from a clean tree, timing the wall clock with the shell `time` builtin:

```console
$ export TIMEFORMAT='BUILD_REAL_SECONDS=%R'

# (a) canonical CI command — default compiler (gcc 15.2.0)
$ python3 setup.py clean ;            { time python3 setup.py build --verbose ; } ; echo "exit=$?"   # run 1
$ python3 setup.py clean ;            { time python3 setup.py build --verbose ; } ; echo "exit=$?"   # run 2

# (b) container-recommended variant — CC=gcc-13 (gcc 13.4.0); this is the build left on disk
$ CC=gcc-13 python3 setup.py clean ;  { time CC=gcc-13 python3 setup.py build --verbose ; } ; echo "exit=$?"   # run 1
$ CC=gcc-13 python3 setup.py clean ;  { time CC=gcc-13 python3 setup.py build --verbose ; } ; echo "exit=$?"   # run 2
```

All four builds reported `exit=0`. The captured `BUILD_REAL_SECONDS` values are stable to the same order across the two runs of each configuration:

```text
# (a) default gcc 15.2.0
BUILD_REAL_SECONDS=65.081      # run 1
BUILD_REAL_SECONDS=64.060      # run 2
# (b) CC=gcc-13
BUILD_REAL_SECONDS=61.630      # run 1
BUILD_REAL_SECONDS=61.590      # run 2
```

Both builds emit an identical **319-line** verbose log. Its verbatim preamble records two facts directly relevant to this investigation — the Wayland backend is **disabled** (no `wayland-protocols` on this host), and the detected compiler:

```text
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
CC: ['gcc-13'] (13, 0)
gcc-13 (Ubuntu 13.4.0-4ubuntu1) 13.4.0
Copyright (C) 2023 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Detected: CompilerType.gcc
```

(The default-compiler build's preamble is identical except for the three compiler-identity lines, which read `CC: ['gcc'] (15, 0)` / `gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0` / `Copyright (C) 2025 Free Software Foundation, Inc.`.) The line **`Disabling building of wayland backend`** is the *observed* reason `glfw-wayland.so` is never produced here — the root cause of the non-CI `test_glfw_modules` result in §2.4.

Rather than reproduce all 319 largely-repetitive per-file compile lines, the three extension link steps and the two launcher/Go build steps are isolated with a self-contained `grep` over the build output. The command below is exactly reproducible and its output is complete:

```console
$ CC=gcc-13 python3 setup.py build --verbose 2>&1 \
    | grep -oE '\-o (build/kitty/fast_data_types\.so|build/kitty/glfw-x11\.so|build/kittens/transfer/rsync\.so|kitty/launcher/kitty|kitty/launcher/kitten)'
```

```text
-o build/kitty/fast_data_types.so
-o build/kitty/glfw-x11.so
-o build/kittens/transfer/rsync.so
-o kitty/launcher/kitty
-o kitty/launcher/kitten
```

The final step is the `go build` that produces `kitty/launcher/kitten`. Its `-ldflags` embed `-X kitty.VCSRevision=<delivery-commit>` — the *destination* delivery commit, not the source commit (see Appendix). Shown verbatim below, with only that 40-hex revision replaced by the placeholder `<delivery-commit>` because it is the hash of the commit that adds this document (which cannot embed its own hash):

```text
/usr/local/go/bin/go build -v -ldflags '-X kitty.VCSRevision=<delivery-commit> -s -w' -o kitty/launcher/kitten /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/tools/cmd
```

### 1.1 Extension taxonomy — two Python extensions + one native library (observed)

A key correction to any "three compiled Python C-extensions" framing: only **two** of the three shared objects are Python extension modules. This was verified with `nm -D` (dynamic symbol table): a CPython extension must export a `PyInit_<name>` initializer, which the interpreter calls on `import`. `kitty/glfw-x11.so` exports **no** `PyInit_*` — it instead exports the native symbol `utf_8_strndup` that the GLFW test resolves through `ctypes` (§4, §5.3).

```console
$ for so in kitty/fast_data_types.so kittens/transfer/rsync.so kitty/glfw-x11.so; do
    echo "--- $so ---"; nm -D "$so" | grep -E 'PyInit|utf_8_strndup'; done
```

```text
--- kitty/fast_data_types.so ---
0000000000028c70 T PyInit_fast_data_types
--- kittens/transfer/rsync.so ---
0000000000008600 T PyInit_rsync
--- kitty/glfw-x11.so ---
0000000000021f60 T utf_8_strndup
```

Interpreting the observed output: `fast_data_types.so` and `rsync.so` each print a `PyInit_*` line (`PyInit_fast_data_types` at `0x28c70`, `PyInit_rsync` at `0x8600`), confirming they are importable CPython extension modules. Under the `--- kitty/glfw-x11.so ---` header the command prints **only** the `utf_8_strndup` line (`0x21f60`) and **no** `PyInit_*` line — so `glfw-x11.so` is a native shared library, not a Python extension module; it is consumed through `ctypes` rather than `import` (see §4 and §5.3). The exact symbol addresses are compiler/link dependent (captured here from the on-disk `CC=gcc-13` build); the **presence/absence** of each symbol is not. **(inferred that the missing `PyInit_glfw_x11` means "not importable as a Python module" — the runtime confirmation of that inference is the actual `ModuleNotFoundError`-free `ctypes` load path exercised in §4.1 and the cascade in §5.3.)**

### 1.2 What actually gets compiled into each extension (source selection, observed)

The repository globs are **not** the exact set of build inputs. Two self-contained, reproducible steps show the difference: first count the on-disk source globs, then count the object files (`.o` tokens) on each link line of a verbose build.

**Step 1 — count the on-disk source globs:**

```console
$ for g in 'kitty/*.c' 'glfw/*.c' 'kittens/transfer/*.c'; do
    printf '%-22s : %s\n' "$g" "$(ls $g | wc -l)"; done
```

```text
kitty/*.c              : 49
glfw/*.c               : 31
kittens/transfer/*.c   : 1
```

**Step 2 — count the objects actually linked** (capture the verbose build once to a scratch log *outside* the repository, then count the `.o` tokens on each `-o <build-dir>/<name>.so` link line):

```console
$ CC=gcc-13 python3 setup.py build --verbose > /tmp/build.log 2>&1
$ for so in kitty/fast_data_types.so kitty/glfw-x11.so kittens/transfer/rsync.so; do
    n=$(grep -E "\-o build/$so( |\$)" /tmp/build.log | grep -oE '[^ ]+\.o( |$)' | wc -l)
    echo "$so : $n objects linked"; done
```

```text
kitty/fast_data_types.so : 62 objects linked
kitty/glfw-x11.so : 20 objects linked
kittens/transfer/rsync.so : 1 objects linked
```

**Step 3 — break down the 62 `fast_data_types` objects** by categorizing the link-line tokens:

```console
$ grep -E '\-o build/kitty/fast_data_types\.so( |$)' /tmp/build.log \
    | tr ' ' '\n' | grep -E '\.o$' > /tmp/fdt_objs.txt
$ echo "kitty-prefixed  : $(grep -cE 'fast_data_types-kitty-'          /tmp/fdt_objs.txt)"
$ echo "3rdparty base64 : $(grep -cE 'fast_data_types-3rdparty-base64-'  /tmp/fdt_objs.txt)"
$ echo "3rdparty ringbuf: $(grep -cE 'fast_data_types-3rdparty-ringbuf-' /tmp/fdt_objs.txt)"
```

```text
kitty-prefixed  : 49
3rdparty base64 : 12
3rdparty ringbuf: 1
```

The glob counts (49 / 31 / 1) versus the linked-object counts (62 / 20 / 1), and the 49 = 48 real + 1 generated / 13 = 12 + 1 breakdown, are summarized below:

| Extension | Repo glob | Glob count | Objects actually linked | Composition of the linked objects |
|-----------|-----------|-----------:|------------------------:|-----------------------------------|
| `kitty/fast_data_types.so` | `kitty/*.c` | 49 | **62** | 49 `kitty-`prefixed objects (**48** real `kitty/*.c` — `kitty/macos_process_info.c` is **excluded** on non-macOS by `find_c_files()` — **plus 1 generated** `kitty/vt-parser-dump.c` that is not in the glob) **+ 13 third-party objects** (12 `3rdparty/base64/*` + 1 `3rdparty/ringbuf/*`) |
| `kitty/glfw-x11.so` | `glfw/*.c` | 31 | **20** | only the 20 sources selected for the `x11` backend by `compile_glfw()` (the other 11 belong to `cocoa`/`wayland`/unused backends) |
| `kittens/transfer/rsync.so` | `kittens/transfer/*.c` | 1 | **1** | the single transfer C source |

The source-selection logic lives in `setup.py`:

- `find_c_files()` is defined at `setup.py:906` (body `setup.py:906-928`): it gathers `kitty/*.c`, **excludes** the macOS-only source on non-macOS, and **appends** the generated `kitty/vt-parser-dump.c` and the `3rdparty` sources — which is why the linked object count (62) exceeds the glob (49). **(inferred, from reading `setup.py:906-928`)**
- `compile_glfw()` at `setup.py:932-953` selects the per-platform backend: `modules = 'cocoa' if is_macos else 'x11 wayland'` (`setup.py:933`). On this Linux host only the `x11` backend's 20 sources are compiled into `glfw-x11.so`; `glfw-wayland.so` was **not** built because no Wayland dev libraries were installed (this is the root cause of the non-CI `test_glfw_modules` result in §2.4). **(inferred, from reading `setup.py:932-953`)**
- the rsync extension's sources are listed at `setup.py:986`. **(inferred, from reading `setup.py:986`)**
- `build()` (`setup.py:1084`) drives the `fast_data_types` compile via `compile_c_extension(...'kitty/fast_data_types'...)` at `setup.py:1090-1093`. **(inferred, from reading `setup.py:1084-1093`)**

### 1.3 Observed build artifacts (both compilers)

The three `*.so` extensions plus the two launcher binaries are the observable products of the build. Their **C-artifact sizes are compiler-dependent**, so both builds are shown. All runtime evidence in §§2, 4, 5 and the Appendix was captured against the **`CC=gcc-13` build left on disk**; the default-compiler (`gcc 15.2.0`) build is the one kitty's CI actually produces (`.github/workflows/ci.py:104` runs `{python} setup.py build --verbose` with no `CC` override).

**Canonical CI build — default compiler `gcc (Ubuntu 15.2.0)` (`ls -l`):**

```text
-rwxr-xr-x 1 root root    42824 Jul 13 22:21 kittens/transfer/rsync.so
-rwxr-xr-x 1 root root  1253792 Jul 13 22:21 kitty/fast_data_types.so
-rwxr-xr-x 1 root root   373896 Jul 13 22:21 kitty/glfw-x11.so
-rwxr-xr-x 1 root root 15765764 Jul 13 22:22 kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 Jul 13 22:21 kitty/launcher/kitty
```

**On-disk build used for all runtime observations — `CC=gcc-13` (`gcc-13 13.4.0`) (`ls -l`):**

```text
-rwxr-xr-x 1 root root    55032 Jul 13 22:23 kittens/transfer/rsync.so
-rwxr-xr-x 1 root root  1213072 Jul 13 22:23 kitty/fast_data_types.so
-rwxr-xr-x 1 root root   357584 Jul 13 22:23 kitty/glfw-x11.so
-rwxr-xr-x 1 root root 15765764 Jul 13 22:24 kitty/launcher/kitten
-rwxr-xr-x 1 root root    36288 Jul 13 22:23 kitty/launcher/kitty
```

The `kitten` launcher is **15,765,764 bytes in both builds** — it is produced by `go build` (§1), so its size does not depend on the C compiler. The three C artifacts and the C `kitty` launcher differ between the two toolchains: `fast_data_types.so` is 1,253,792 under gcc-15 vs 1,213,072 under gcc-13; `glfw-x11.so` 373,896 vs 357,584; and `rsync.so` 42,824 vs 55,032 (gcc-13's `rsync.so` is *larger*). Within each compiler, the sizes were byte-identical across the two paired runs captured in §1 (stable). The `nm -D` symbol addresses shown in §1.1 (e.g. `PyInit_fast_data_types` at `0x28c70`) correspond to this on-disk `CC=gcc-13` build.

### 1.4 The `libssl-dev` / CI dependency gap (observed, isolated reproduction)

`setup.py` requires `libcrypto` via `libcrypto_flags()` (defined `setup.py:253`, called `setup.py:616`), which calls the `pkg_config()` helper (defined `setup.py:220`). When a required package is missing, `pkg_config()` raises at `setup.py:239`:

```console
$ sed -n '239p' setup.py
            raise SystemExit(f'The package {error(pkg)} was not found on your system')
```

kitty's CI apt list (`.github/workflows/ci.py:85-88`) does **not** include `libssl-dev`, yet `setup.py` needs the `libcrypto.pc` it provides. To observe the resulting failure **without touching any system file**, the build was re-run with `PKG_CONFIG_LIBDIR` pointed at a private symlink farm (created with `mktemp -d`) that mirrored every `.pc` file from **all six** pkg-config search directories (112 files) **except** `libcrypto.pc`/`openssl.pc`/`libssl.pc`. The system `pkg-config` directory was never modified — its `libcrypto` modversion was `3.5.3` both before and after the reproduction. The gap surfaces inside `libcrypto_flags()`' `pkg_config()` call **before any C source is compiled**, so it is independent of the C compiler; it is shown here with the **default compiler** (no `CC` override), exactly as kitty's CI would encounter it. The command and its complete output:

```console
$ ISO_DIR="$(mktemp -d)"     # private farm: every *.pc from all 6 search dirs EXCEPT libcrypto/openssl/libssl
$ PKG_CONFIG_LIBDIR="$ISO_DIR" python3 setup.py build --verbose   # default compiler (no CC), exit 1
```

```text
Package libcrypto was not found in the pkg-config search path.
Perhaps you should add the directory containing `libcrypto.pc'
to the PKG_CONFIG_PATH environment variable
Package 'libcrypto', required by 'virtual:world', not found
CC: ['gcc'] (15, 0)
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
Copyright (C) 2025 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Detected: CompilerType.gcc
The package libcrypto was not found on your system
```

## 2. Executing the test suite through its real entry point

The canonical entry point is `./test.py`. Its shebang routes through the compiled C launcher (`test.py:1` = `#!./kitty/launcher/kitty +launch`), and `main()` (defined `test.py:7`) imports the runner and calls it:

```console
$ sed -n '1,13p' test.py
#!./kitty/launcher/kitty +launch
# License: GPL v3 Copyright: 2016, Kovid Goyal <kovid at kovidgoyal.net>

import importlib


def main() -> None:
    m = importlib.import_module('kitty_tests.main')
    getattr(m, 'main')()


if __name__ == '__main__':
    main()
```

`test.py:8` imports `kitty_tests.main`; `test.py:9` calls its `main()`. The runner's `run_tests()` (`kitty_tests/main.py:246`) launches the Go tests, runs the Python suite, and reports. Two complete runs of the **full** suite follow, for stability (rule: confirm magnitude/count stability across ≥2 runs).

### 2.1 Full suite — complete unedited output (run 1)

```console
$ CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./test.py    # exit 1
```

```text
Running under CI: True
Using PATH in test environment: /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty_tests/kitty/launcher:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/go/bin
Python: /usr/bin/python
Intrinsics: has_avx2=True has_sse4_2=True
Go executable: /usr/local/go/bin/go
Go packages being tested: tools/tui/sgr tools/cli tools/utils/base85 tools/utils/shm tools/tui/shell_integration tools/unicode_names tools/wcswidth tools/simdstring tools/cmd/at tools/tui/readline kittens/transfer kittens/ssh kittens/diff tools/tui/subseq tools/utils/style tools/utils kittens/hyperlinked_grep tools/rsync kittens/hints tools/utils/humanize tools/tui tools/themes tools/config tools/utils/shlex tools/tui/graphics tools/tui/loop
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
test_font_selection (kitty_tests.fonts.Selection.test_font_selection) ...
  test_font_selection (kitty_tests.fonts.Selection.test_font_selection) (spec='ubuntu mono') ... FAIL
  test_font_selection (kitty_tests.fonts.Selection.test_font_selection) (spec='family="ubuntu mono"') ... FAIL
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
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 476, in test_transfer_receive
    self.basic_transfer_tests()
    ~~~~~~~~~~~~~~~~~~~~~~~~~^^
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 443, in basic_transfer_tests
    multiple_files()
    ~~~~~~~~~~~~~~^^
    d = <_io.BufferedWriter name='/tmp/tmp_aqmismc/dest'>
    dest = '/tmp/tmp_aqmismc/dest'
    multiple_files = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files at 0x7f6228fb9940>
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7f6229ad23c0>
    s = <_io.BufferedWriter name='/tmp/tmp_aqmismc/src'>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
    single_file = <function TestFileTransmission.basic_transfer_tests.<locals>.single_file at 0x7f6228fb8d60>
    src = '/tmp/tmp_aqmismc/src'
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 432, in multiple_files
    self.assertEqual(expected, actual)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^
    Entry = <class 'kitty_tests.file_transmission.Entry'>
    actual = {'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1783964162657439239, mode='0o120777', nlink=1), 'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'sym': Entry(relpath='sym', mtime=1783964162657439239, mode='0o120777', nlink=1), 'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1)}
    b = PosixPath('/tmp/tmp_aqmismc/msrc')
    cmd = ()
    de = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.de at 0x7f6228fbb600>
    dest = '/tmp/tmp_aqmismc/mdest'
    dirnames = []
    dirpath = '/tmp/tmp_aqmismc/mdest/msrc/empty'
    entry = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.entry at 0x7f6228fb85e0>
    expected = {'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1), 'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1783964162657439239, mode='0o120777', nlink=1), 'sym': Entry(relpath='sym', mtime=1783964162657439239, mode='0o120777', nlink=1)}
    f = <_io.BufferedWriter name='/tmp/tmp_aqmismc/msrc/sub/reg'>
    filenames = []
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7f622a54e050>
    s = PosixPath('/tmp/tmp_aqmismc/msrc/sub')
    se = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.se at 0x7f6228fb8400>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
    src = '/tmp/tmp_aqmismc/msrc'
    x = 'reg'
AssertionError: {'simple': Entry(relpath='simple', mtime=130[497 chars]k=1)} != {'sub': Entry(relpath='sub', mtime=0, mode='[497 chars]k=1)}
  {'abssym': Entry(relpath='abssym', mtime=1783964162657439239, mode='0o120777', nlink=1),
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
   'sym': Entry(relpath='sym', mtime=1783964162657439239, mode='0o120777', nlink=1)}

======================================================================
FAIL: test_transfer_send (kitty_tests.file_transmission.TestFileTransmission.test_transfer_send)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 507, in test_transfer_send
    self.basic_transfer_tests()
    ~~~~~~~~~~~~~~~~~~~~~~~~~^^
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 443, in basic_transfer_tests
    multiple_files()
    ~~~~~~~~~~~~~~^^
    d = <_io.BufferedWriter name='/tmp/tmpwhmndm2d/dest'>
    dest = '/tmp/tmpwhmndm2d/dest'
    multiple_files = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files at 0x7f6228fd1b20>
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7f6228f56d50>
    s = <_io.BufferedWriter name='/tmp/tmpwhmndm2d/src'>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
    single_file = <function TestFileTransmission.basic_transfer_tests.<locals>.single_file at 0x7f6228fbb9c0>
    src = '/tmp/tmpwhmndm2d/src'
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 432, in multiple_files
    self.assertEqual(expected, actual)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^
    Entry = <class 'kitty_tests.file_transmission.Entry'>
    actual = {'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1783964164627460462, mode='0o120777', nlink=1), 'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'sym': Entry(relpath='sym', mtime=1783964164627460462, mode='0o120777', nlink=1), 'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1)}
    b = PosixPath('/tmp/tmpwhmndm2d/msrc')
    cmd = ()
    de = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.de at 0x7f6228fd2d40>
    dest = '/tmp/tmpwhmndm2d/mdest'
    dirnames = []
    dirpath = '/tmp/tmpwhmndm2d/mdest/msrc/empty'
    entry = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.entry at 0x7f6228fd2160>
    expected = {'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1), 'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1783964164627460462, mode='0o120777', nlink=1), 'sym': Entry(relpath='sym', mtime=1783964164627460462, mode='0o120777', nlink=1)}
    f = <_io.BufferedWriter name='/tmp/tmpwhmndm2d/msrc/sub/reg'>
    filenames = []
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7f62299cb070>
    s = PosixPath('/tmp/tmpwhmndm2d/msrc/sub')
    se = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.se at 0x7f6228fd2200>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
    src = '/tmp/tmpwhmndm2d/msrc'
    x = 'reg'
AssertionError: {'simple': Entry(relpath='simple', mtime=130[497 chars]k=1)} != {'sub': Entry(relpath='sub', mtime=0, mode='[497 chars]k=1)}
  {'abssym': Entry(relpath='abssym', mtime=1783964164627460462, mode='0o120777', nlink=1),
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
   'sym': Entry(relpath='sym', mtime=1783964164627460462, mode='0o120777', nlink=1)}

======================================================================
FAIL: test_font_selection (kitty_tests.fonts.Selection.test_font_selection) (spec='ubuntu mono')
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/fonts.py", line 40, in s
    self.ae(expected, actual)
    ~~~~~~~^^^^^^^^^^^^^^^^^^
    actual = ('UbuntuMonoRoman-Regular', 'UbuntuMonoRoman-Bold', 'UbuntuMonoRoman-MediumItalic', 'UbuntuMonoRoman-BoldItalic')
    alternate = None
    expected = ('UbuntuMono-Regular', 'UbuntuMono-Bold', 'UbuntuMono-Italic', 'UbuntuMono-BoldItalic')
    family = 'ubuntu mono'
    opts = <kitty.options.types.Options object at 0x7f6228466ad0>
    self = <kitty_tests.fonts.Selection testMethod=test_font_selection>
    x = 'UbuntuMonoRoman-BoldItalic'
AssertionError: Tuples differ: ('UbuntuMono-Regular', 'UbuntuMono-Bold', 'UbuntuMono[29 chars]lic') != ('UbuntuMonoRoman-Regular', 'UbuntuMonoRoman-Bold', '[55 chars]lic')

First differing element 0:
'UbuntuMono-Regular'
'UbuntuMonoRoman-Regular'

- ('UbuntuMono-Regular',
+ ('UbuntuMonoRoman-Regular',
?             +++++

-  'UbuntuMono-Bold',
+  'UbuntuMonoRoman-Bold',
?             +++++

-  'UbuntuMono-Italic',
+  'UbuntuMonoRoman-MediumItalic',
?             +++++ ++++++

-  'UbuntuMono-BoldItalic')
+  'UbuntuMonoRoman-BoldItalic')
?             +++++


======================================================================
FAIL: test_font_selection (kitty_tests.fonts.Selection.test_font_selection) (spec='family="ubuntu mono"')
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/fonts.py", line 40, in s
    self.ae(expected, actual)
    ~~~~~~~^^^^^^^^^^^^^^^^^^
    actual = ('UbuntuMonoRoman-Regular', 'UbuntuMonoRoman-Bold', 'UbuntuMonoRoman-MediumItalic', 'UbuntuMonoRoman-BoldItalic')
    alternate = None
    expected = ('UbuntuMono-Regular', 'UbuntuMono-Bold', 'UbuntuMono-Italic', 'UbuntuMono-BoldItalic')
    family = 'family="ubuntu mono"'
    opts = <kitty.options.types.Options object at 0x7f6228466ad0>
    self = <kitty_tests.fonts.Selection testMethod=test_font_selection>
    x = 'UbuntuMonoRoman-BoldItalic'
AssertionError: Tuples differ: ('UbuntuMono-Regular', 'UbuntuMono-Bold', 'UbuntuMono[29 chars]lic') != ('UbuntuMonoRoman-Regular', 'UbuntuMonoRoman-Bold', '[55 chars]lic')

First differing element 0:
'UbuntuMono-Regular'
'UbuntuMonoRoman-Regular'

- ('UbuntuMono-Regular',
+ ('UbuntuMonoRoman-Regular',
?             +++++

-  'UbuntuMono-Bold',
+  'UbuntuMonoRoman-Bold',
?             +++++

-  'UbuntuMono-Italic',
+  'UbuntuMonoRoman-MediumItalic',
?             +++++ ++++++

-  'UbuntuMono-BoldItalic')
+  'UbuntuMonoRoman-BoldItalic')
?             +++++


----------------------------------------------------------------------
Ran 145 tests in 38.932s

FAILED (failures=4, skipped=4)
All Go tests succeeded, ran in 47.4 seconds
[31mError[39m: Some tests failed!
```

### 2.2 Full suite — complete unedited output (run 2, stability)

The counts are stable across both runs: **`Ran 145 tests`**, **`FAILED (failures=4, skipped=4)`** (zero errors), and **`All Go tests succeeded`**, with the identical set of four failing tests (`test_transfer_receive`, `test_transfer_send`, and the two `test_font_selection` parametrizations). Only the wall-clock timings differ (run 1: `38.932s`, Go `47.4s`; run 2 below), which is expected. The extension-loading tests (`test_loading_extensions`, `test_loading_shaders`, `test_glfw_modules`, and `test_utf_8_strndup`) all **pass** in both runs when the artifacts are present.

```console
$ CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./test.py    # exit 1  (second run)
```

```text
Running under CI: True
Using PATH in test environment: /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty_tests/kitty/launcher:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/go/bin
Python: /usr/bin/python
Intrinsics: has_avx2=True has_sse4_2=True
Go executable: /usr/local/go/bin/go
Go packages being tested: tools/cmd/at tools/utils/humanize tools/utils/shm tools/utils/style kittens/ssh tools/tui/readline tools/utils/base85 tools/tui/loop tools/utils tools/wcswidth tools/simdstring tools/config tools/tui/subseq tools/tui/shell_integration kittens/hyperlinked_grep tools/unicode_names tools/themes kittens/transfer kittens/diff tools/tui tools/tui/graphics tools/tui/sgr tools/rsync tools/cli tools/utils/shlex kittens/hints
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
test_font_selection (kitty_tests.fonts.Selection.test_font_selection) ...
  test_font_selection (kitty_tests.fonts.Selection.test_font_selection) (spec='ubuntu mono') ... FAIL
  test_font_selection (kitty_tests.fonts.Selection.test_font_selection) (spec='family="ubuntu mono"') ... FAIL
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
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 476, in test_transfer_receive
    self.basic_transfer_tests()
    ~~~~~~~~~~~~~~~~~~~~~~~~~^^
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 443, in basic_transfer_tests
    multiple_files()
    ~~~~~~~~~~~~~~^^
    d = <_io.BufferedWriter name='/tmp/tmpc519ddwm/dest'>
    dest = '/tmp/tmpc519ddwm/dest'
    multiple_files = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files at 0x7f4f4818ede0>
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7f4f48e25160>
    s = <_io.BufferedWriter name='/tmp/tmpc519ddwm/src'>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
    single_file = <function TestFileTransmission.basic_transfer_tests.<locals>.single_file at 0x7f4f48e2d080>
    src = '/tmp/tmpc519ddwm/src'
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 432, in multiple_files
    self.assertEqual(expected, actual)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^
    Entry = <class 'kitty_tests.file_transmission.Entry'>
    actual = {'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1783964218708043080, mode='0o120777', nlink=1), 'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'sym': Entry(relpath='sym', mtime=1783964218708043080, mode='0o120777', nlink=1), 'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1)}
    b = PosixPath('/tmp/tmpc519ddwm/msrc')
    cmd = ()
    de = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.de at 0x7f4f4818ff60>
    dest = '/tmp/tmpc519ddwm/mdest'
    dirnames = []
    dirpath = '/tmp/tmpc519ddwm/mdest/msrc/empty'
    entry = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.entry at 0x7f4f4818f2e0>
    expected = {'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1), 'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1783964218708043080, mode='0o120777', nlink=1), 'sym': Entry(relpath='sym', mtime=1783964218708043080, mode='0o120777', nlink=1)}
    f = <_io.BufferedWriter name='/tmp/tmpc519ddwm/msrc/sub/reg'>
    filenames = []
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7f4f49975e50>
    s = PosixPath('/tmp/tmpc519ddwm/msrc/sub')
    se = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.se at 0x7f4f4818f380>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
    src = '/tmp/tmpc519ddwm/msrc'
    x = 'reg'
AssertionError: {'simple': Entry(relpath='simple', mtime=130[497 chars]k=1)} != {'sub': Entry(relpath='sub', mtime=0, mode='[497 chars]k=1)}
  {'abssym': Entry(relpath='abssym', mtime=1783964218708043080, mode='0o120777', nlink=1),
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
   'sym': Entry(relpath='sym', mtime=1783964218708043080, mode='0o120777', nlink=1)}

======================================================================
FAIL: test_transfer_send (kitty_tests.file_transmission.TestFileTransmission.test_transfer_send)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 507, in test_transfer_send
    self.basic_transfer_tests()
    ~~~~~~~~~~~~~~~~~~~~~~~~~^^
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 443, in basic_transfer_tests
    multiple_files()
    ~~~~~~~~~~~~~~^^
    d = <_io.BufferedWriter name='/tmp/tmp9k8qkfg2/dest'>
    dest = '/tmp/tmp9k8qkfg2/dest'
    multiple_files = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files at 0x7f4f481a2480>
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7f4f4814e7b0>
    s = <_io.BufferedWriter name='/tmp/tmp9k8qkfg2/src'>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
    single_file = <function TestFileTransmission.basic_transfer_tests.<locals>.single_file at 0x7f4f481a0360>
    src = '/tmp/tmp9k8qkfg2/src'
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 432, in multiple_files
    self.assertEqual(expected, actual)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^
    Entry = <class 'kitty_tests.file_transmission.Entry'>
    actual = {'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1783964219152047863, mode='0o120777', nlink=1), 'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'sym': Entry(relpath='sym', mtime=1783964219152047863, mode='0o120777', nlink=1), 'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1)}
    b = PosixPath('/tmp/tmp9k8qkfg2/msrc')
    cmd = ()
    de = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.de at 0x7f4f481a36a0>
    dest = '/tmp/tmp9k8qkfg2/mdest'
    dirnames = []
    dirpath = '/tmp/tmp9k8qkfg2/mdest/msrc/empty'
    entry = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.entry at 0x7f4f481a2ac0>
    expected = {'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1), 'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1783964219152047863, mode='0o120777', nlink=1), 'sym': Entry(relpath='sym', mtime=1783964219152047863, mode='0o120777', nlink=1)}
    f = <_io.BufferedWriter name='/tmp/tmp9k8qkfg2/msrc/sub/reg'>
    filenames = []
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7f4f48770c00>
    s = PosixPath('/tmp/tmp9k8qkfg2/msrc/sub')
    se = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.se at 0x7f4f481a2b60>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
    src = '/tmp/tmp9k8qkfg2/msrc'
    x = 'reg'
AssertionError: {'simple': Entry(relpath='simple', mtime=130[497 chars]k=1)} != {'sub': Entry(relpath='sub', mtime=0, mode='[497 chars]k=1)}
  {'abssym': Entry(relpath='abssym', mtime=1783964219152047863, mode='0o120777', nlink=1),
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
   'sym': Entry(relpath='sym', mtime=1783964219152047863, mode='0o120777', nlink=1)}

======================================================================
FAIL: test_font_selection (kitty_tests.fonts.Selection.test_font_selection) (spec='ubuntu mono')
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/fonts.py", line 40, in s
    self.ae(expected, actual)
    ~~~~~~~^^^^^^^^^^^^^^^^^^
    actual = ('UbuntuMonoRoman-Regular', 'UbuntuMonoRoman-Bold', 'UbuntuMonoRoman-MediumItalic', 'UbuntuMonoRoman-BoldItalic')
    alternate = None
    expected = ('UbuntuMono-Regular', 'UbuntuMono-Bold', 'UbuntuMono-Italic', 'UbuntuMono-BoldItalic')
    family = 'ubuntu mono'
    opts = <kitty.options.types.Options object at 0x7f4f42e920d0>
    self = <kitty_tests.fonts.Selection testMethod=test_font_selection>
    x = 'UbuntuMonoRoman-BoldItalic'
AssertionError: Tuples differ: ('UbuntuMono-Regular', 'UbuntuMono-Bold', 'UbuntuMono[29 chars]lic') != ('UbuntuMonoRoman-Regular', 'UbuntuMonoRoman-Bold', '[55 chars]lic')

First differing element 0:
'UbuntuMono-Regular'
'UbuntuMonoRoman-Regular'

- ('UbuntuMono-Regular',
+ ('UbuntuMonoRoman-Regular',
?             +++++

-  'UbuntuMono-Bold',
+  'UbuntuMonoRoman-Bold',
?             +++++

-  'UbuntuMono-Italic',
+  'UbuntuMonoRoman-MediumItalic',
?             +++++ ++++++

-  'UbuntuMono-BoldItalic')
+  'UbuntuMonoRoman-BoldItalic')
?             +++++


======================================================================
FAIL: test_font_selection (kitty_tests.fonts.Selection.test_font_selection) (spec='family="ubuntu mono"')
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/fonts.py", line 40, in s
    self.ae(expected, actual)
    ~~~~~~~^^^^^^^^^^^^^^^^^^
    actual = ('UbuntuMonoRoman-Regular', 'UbuntuMonoRoman-Bold', 'UbuntuMonoRoman-MediumItalic', 'UbuntuMonoRoman-BoldItalic')
    alternate = None
    expected = ('UbuntuMono-Regular', 'UbuntuMono-Bold', 'UbuntuMono-Italic', 'UbuntuMono-BoldItalic')
    family = 'family="ubuntu mono"'
    opts = <kitty.options.types.Options object at 0x7f4f42e920d0>
    self = <kitty_tests.fonts.Selection testMethod=test_font_selection>
    x = 'UbuntuMonoRoman-BoldItalic'
AssertionError: Tuples differ: ('UbuntuMono-Regular', 'UbuntuMono-Bold', 'UbuntuMono[29 chars]lic') != ('UbuntuMonoRoman-Regular', 'UbuntuMonoRoman-Bold', '[55 chars]lic')

First differing element 0:
'UbuntuMono-Regular'
'UbuntuMonoRoman-Regular'

- ('UbuntuMono-Regular',
+ ('UbuntuMonoRoman-Regular',
?             +++++

-  'UbuntuMono-Bold',
+  'UbuntuMonoRoman-Bold',
?             +++++

-  'UbuntuMono-Italic',
+  'UbuntuMonoRoman-MediumItalic',
?             +++++ ++++++

-  'UbuntuMono-BoldItalic')
+  'UbuntuMonoRoman-BoldItalic')
?             +++++


----------------------------------------------------------------------
Ran 145 tests in 22.271s

FAILED (failures=4, skipped=4)
All Go tests succeeded, ran in 22.4 seconds
[31mError[39m: Some tests failed!
```

### 2.3 Filtered run — `--module check_build` (the extension-validating module)

`kitty_tests/check_build.py` is the module most directly tied to the extensions: `test_loading_extensions` (defined `check_build.py:28`) imports `kitty.fast_data_types` (`check_build.py:29`) and `kittens.transfer.rsync` (`check_build.py:30`); `test_loading_shaders` is at `check_build.py:33`; `test_glfw_modules` (defined `check_build.py:38`) file-checks the GLFW backend `.so` at `check_build.py:46`. Under `CI=true` it passes cleanly:

```console
$ CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./test.py --module check_build    # exit 0
```

```text
Running under CI: True
Using PATH in test environment: /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty_tests/kitty/launcher:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/go/bin
Python: /usr/bin/python
Intrinsics: has_avx2=True has_sse4_2=True
test_all_kitten_names (kitty_tests.check_build.TestBuild.test_all_kitten_names) ... ok
test_ca_certificates (kitty_tests.check_build.TestBuild.test_ca_certificates) ... skipped 'CA certificates are only tested on frozen builds'
test_docs_url (kitty_tests.check_build.TestBuild.test_docs_url) ... ok
test_exe (kitty_tests.check_build.TestBuild.test_exe) ... ok
test_filesystem_locations (kitty_tests.check_build.TestBuild.test_filesystem_locations) ... ok
test_glfw_modules (kitty_tests.check_build.TestBuild.test_glfw_modules) ... ok
test_launcher_ensures_stdio (kitty_tests.check_build.TestBuild.test_launcher_ensures_stdio) ... ok
test_loading_extensions (kitty_tests.check_build.TestBuild.test_loading_extensions) ... ok
test_loading_shaders (kitty_tests.check_build.TestBuild.test_loading_shaders) ... ok

----------------------------------------------------------------------
Ran 9 tests in 0.073s

OK (skipped=1)
```

### 2.4 `test_glfw_modules` depends on CI mode (exact identifier, complete output)

`test_glfw_modules` builds its expected backend list as `linux_backends = ['x11']` (`check_build.py:40`) and **appends `'wayland'` only when not under CI** (`check_build.py:41-42`, guarded by `is_ci`). It then asserts each backend `.so` exists (`check_build.py:46`). Because this environment built only `glfw-x11.so` (no Wayland libs; §1.2), the outcome flips with `CI`:

- **`CI=true`** → expected backends `['x11']` → `glfw-x11.so` exists → **passes** (§2.3 above).
- **not CI** → expected backends `['x11','wayland']` → `glfw-wayland.so` is missing → **fails**.

The exact failing test identifier is **`kitty_tests.check_build.TestBuild.test_glfw_modules`** (the class is `TestBuild`). Complete output of the non-CI run:

```console
$ env -u CI LANG=C.UTF-8 LC_ALL=C.UTF-8 ./test.py --module check_build    # exit 1
```

```text
Running under CI: False
test_all_kitten_names (kitty_tests.check_build.TestBuild.test_all_kitten_names) ... ok
test_ca_certificates (kitty_tests.check_build.TestBuild.test_ca_certificates) ... skipped 'CA certificates are only tested on frozen builds'
test_docs_url (kitty_tests.check_build.TestBuild.test_docs_url) ... ok
test_exe (kitty_tests.check_build.TestBuild.test_exe) ... ok
test_filesystem_locations (kitty_tests.check_build.TestBuild.test_filesystem_locations) ... ok
test_glfw_modules (kitty_tests.check_build.TestBuild.test_glfw_modules) ... FAIL
test_launcher_ensures_stdio (kitty_tests.check_build.TestBuild.test_launcher_ensures_stdio) ... ok
test_loading_extensions (kitty_tests.check_build.TestBuild.test_loading_extensions) ... ok
test_loading_shaders (kitty_tests.check_build.TestBuild.test_loading_shaders) ... ok

======================================================================
FAIL: test_glfw_modules (kitty_tests.check_build.TestBuild.test_glfw_modules)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/check_build.py", line 46, in test_glfw_modules
    self.assertTrue(os.path.isfile(path), f'{path} is not a file')
    ~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    glfw_path = <function glfw_path at 0x7f93b67d7600>
    is_macos = False
    linux_backends = ['x11', 'wayland']
    modules = ['x11', 'wayland']
    name = 'wayland'
    path = '/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/glfw-wayland.so'
    self = <kitty_tests.check_build.TestBuild testMethod=test_glfw_modules>
AssertionError: False is not true : /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/glfw-wayland.so is not a file

----------------------------------------------------------------------
Ran 9 tests in 0.075s

FAILED (failures=1, skipped=1)
[31mError[39m: Some tests failed!
```

### 2.5 What the four full-suite failures actually are (not extension failures)

None of the four full-suite failures is an extension-loading failure; the extension-loading tests themselves pass (§2.3). Characterized from the captured tracebacks in §2.1:

- **`test_transfer_receive` / `test_transfer_send`** (`kitty_tests/file_transmission.py`) fail on a **setgid-bit mismatch on directories**, not content corruption. The assertion `self.assertEqual(expected, actual)` compares directory-entry modes: `expected` has the two directories `empty` and `sub` at mode **`0o42755`** (i.e. `S_ISGID` set) while `actual` has them at **`0o40755`** (no setgid). The delta is exactly octal `2000` — the setgid bit. Excerpt from the run-1 diff (full context in §2.1):

  ```text
  -  'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2),
  +  'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2),
  -  'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2),
  +  'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2),
  ```

  Crucially, the rsync-backed tests `test_rsync_hashers` and `test_rsync_roundtrip` **pass** in the same run — so `rsync.so` loaded and functioned correctly; the transfer failures are directory-permission assertion mismatches attributable to the container filesystem's default directory setgid behaviour, **not** to the rsync extension. **(the setgid vs non-setgid cause is inferred from the mode octals in the captured diff)**
- **`test_font_selection` (spec=`'ubuntu mono'`) and (spec=`'family="ubuntu mono"'`)** (`kitty_tests/fonts.py`) fail because the installed Ubuntu Mono font reports the PostScript family names `UbuntuMonoRoman-*` where the test expects `UbuntuMono-*`. This is a font-packaging difference in the environment, unrelated to any compiled kitty extension. **(font-naming cause inferred from the captured `AssertionError` tuples in §2.1)**

## 3. The relationship between the compiled C extensions and the test-execution flow

The relationship is a **hard import-time gating dependency at the root, a discovery-time dependency in the middle, and a run-time native dependency at the leaf**:

1. **Bootstrap (before any test):** Importing the runner package pulls in `kitty.fast_data_types` immediately. The command `./test.py` → `test.py:8` `importlib.import_module('kitty_tests.main')` first initializes the **package** `kitty_tests`, executing `kitty_tests/__init__.py`. That initializer imports `kitty.config` at `kitty_tests/__init__.py:21`, and `kitty/config.py:10` imports `kitty.conf.utils`, whose `kitty/conf/utils.py:27` does `from ..fast_data_types import Color`. It also imports `fast_data_types` **directly** at `kitty_tests/__init__.py:22`. So `fast_data_types.so` must load before a single test is collected. *(Note: it is the **package** `kitty_tests/__init__.py` that triggers this, reached directly from `test.py:8`; the `kitty_tests/main.py` module body is not on this particular chain — see the captured trace in §9.)*
2. **Discovery:** The runner's own `find_all_tests()` (`kitty_tests/main.py:57`) does a direct `importlib.import_module(...)` (`kitty_tests/main.py:64`) for each discovered test module. One of them, `kitty_tests/file_transmission.py:13`, does `from kittens.transfer.rsync import Differ, Hasher, Patcher, parse_ftc` — so `rsync.so` must load during discovery.
3. **Run time (leaf):** `kitty/glfw-x11.so` is not needed to import anything. It is loaded **natively** via `ctypes.CDLL` only when the test body `kitty_tests/glfw.py:test_utf_8_strndup` runs (`kitty_tests/glfw.py:50`), and it is file-checked by `kitty_tests/check_build.py:test_glfw_modules` (`check_build.py:46`).

Sections 4 and 9 show this captured live; section 5 shows what breaks at each level when the corresponding artifact is removed.

## 4. Which extension modules actually get loaded during test execution

This was observed by running the runner's **own** discovery routine, `find_all_tests()`, through the **real** launcher (`./kitty/launcher/kitty +launch <script>`), and inspecting which entries in `sys.modules` are backed by a `.so` file (matched by module `__spec__.origin`). The temporary script (created under `/tmp`, outside the repository, and removed after the investigation so the tracked tree is left untouched) was, in full:

```python
#!/usr/bin/env python3
# observe_loading.py  (temporary observation script; lives under /tmp, never in the repo)
# Purpose: observe which .so-backed modules are loaded (a) at bootstrap, just after
# importing the test runner package, and (b) after the runner's own find_all_tests()
# discovery has imported every test module. Run through the REAL launcher:
#   ./kitty/launcher/kitty +launch <path>/observe_loading.py
import sys, os


def so_modules():
    """Return {module_name: origin} for every entry in sys.modules backed by a .so file."""
    out = {}
    for name, mod in list(sys.modules.items()):
        origin = getattr(getattr(mod, '__spec__', None), 'origin', None) or getattr(mod, '__file__', None)
        if origin and origin.endswith('.so'):
            out[name] = origin
    return out


def dump(title, mapping):
    print(f"=== {title} ===")
    print(f"count={len(mapping)}")
    for name in sorted(mapping):
        print(f"  {name} -> {mapping[name]}")
    print()


import kitty_tests.main as ktmain            # BOOTSTRAP: importing the package runs kitty_tests/__init__.py
boot = so_modules()
suite = ktmain.find_all_tests()             # DISCOVERY: main.py:57,64 import every test module
post = so_modules()

dump("BOOTSTRAP .so-backed modules (after importing kitty_tests.main)", boot)
dump("POST-DISCOVERY .so-backed modules (after find_all_tests())", post)

kitty_built = {n: o for n, o in post.items() if n.split('.')[0] in ('kitty', 'kittens')}
stdlib = {n: o for n, o in post.items() if n.split('.')[0] not in ('kitty', 'kittens')}
dump(f"KITTY-BUILT extensions loaded (count={len(kitty_built)})", kitty_built)
dump(f"stdlib / third-party .so loaded (count={len(stdlib)})", stdlib)

print(f"=== discovered test cases: {suite.countTestCases()} ===")

glfw_as_pymod = any(('glfw' in n.lower()) and o.endswith('.so') for n, o in post.items())
print(f"=== 'kitty.glfw-x11' or glfw backend imported as a python module? {glfw_as_pymod} ===")

with open('/proc/self/maps') as fh:
    maps = fh.read()
print(f"=== glfw-x11.so present in /proc/self/maps at discovery time? {'glfw-x11.so' in maps} ===")
```

Complete unedited output (identical across two runs):

```console
$ CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./kitty/launcher/kitty +launch /tmp/kitty_fix_evidence/observe_loading.py
```

```text
=== BOOTSTRAP .so-backed modules (after importing kitty_tests.main) ===
count=4
  _bz2 -> /usr/lib/python3.13/lib-dynload/_bz2.cpython-313-x86_64-linux-gnu.so
  _lzma -> /usr/lib/python3.13/lib-dynload/_lzma.cpython-313-x86_64-linux-gnu.so
  kitty.fast_data_types -> /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty/fast_data_types.so
  termios -> /usr/lib/python3.13/lib-dynload/termios.cpython-313-x86_64-linux-gnu.so

=== POST-DISCOVERY .so-backed modules (after find_all_tests()) ===
count=9
  PIL._imaging -> /usr/local/lib/python3.13/dist-packages/PIL/_imaging.cpython-313-x86_64-linux-gnu.so
  _bz2 -> /usr/lib/python3.13/lib-dynload/_bz2.cpython-313-x86_64-linux-gnu.so
  _ctypes -> /usr/lib/python3.13/lib-dynload/_ctypes.cpython-313-x86_64-linux-gnu.so
  _hashlib -> /usr/lib/python3.13/lib-dynload/_hashlib.cpython-313-x86_64-linux-gnu.so
  _lzma -> /usr/lib/python3.13/lib-dynload/_lzma.cpython-313-x86_64-linux-gnu.so
  kittens.transfer.rsync -> /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kittens/transfer/rsync.so
  kitty.fast_data_types -> /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty/fast_data_types.so
  mmap -> /usr/lib/python3.13/lib-dynload/mmap.cpython-313-x86_64-linux-gnu.so
  termios -> /usr/lib/python3.13/lib-dynload/termios.cpython-313-x86_64-linux-gnu.so

=== KITTY-BUILT extensions loaded (count=2) ===
count=2
  kittens.transfer.rsync -> /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kittens/transfer/rsync.so
  kitty.fast_data_types -> /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty/fast_data_types.so

=== stdlib / third-party .so loaded (count=7) ===
count=7
  PIL._imaging -> /usr/local/lib/python3.13/dist-packages/PIL/_imaging.cpython-313-x86_64-linux-gnu.so
  _bz2 -> /usr/lib/python3.13/lib-dynload/_bz2.cpython-313-x86_64-linux-gnu.so
  _ctypes -> /usr/lib/python3.13/lib-dynload/_ctypes.cpython-313-x86_64-linux-gnu.so
  _hashlib -> /usr/lib/python3.13/lib-dynload/_hashlib.cpython-313-x86_64-linux-gnu.so
  _lzma -> /usr/lib/python3.13/lib-dynload/_lzma.cpython-313-x86_64-linux-gnu.so
  mmap -> /usr/lib/python3.13/lib-dynload/mmap.cpython-313-x86_64-linux-gnu.so
  termios -> /usr/lib/python3.13/lib-dynload/termios.cpython-313-x86_64-linux-gnu.so

=== discovered test cases: 145 ===
=== 'kitty.glfw-x11' or glfw backend imported as a python module? False ===
=== glfw-x11.so present in /proc/self/maps at discovery time? False ===
```

#### What the numbers mean

- **At bootstrap** (just after importing `kitty_tests.main`, before discovery) exactly **one kitty-built extension** is loaded: `kitty.fast_data_types`. The other three `.so` at that point are CPython stdlib modules (`_bz2`, `_lzma`, `termios`). This is the observable proof that `fast_data_types` is the **root** dependency loaded before any test.
- **After discovery** (`find_all_tests()` imported all 22 test modules) there are **9** `.so`-backed modules in `sys.modules`: **2 are kitty-built** (`kitty.fast_data_types`, `kittens.transfer.rsync`) and **7 are stdlib / third-party** (`PIL._imaging`, `_bz2`, `_ctypes`, `_hashlib`, `_lzma`, `mmap`, `termios`). *(Reported exactly as observed on this Python 3.13.7 build; `_json` is **not** a separate `.so` here.)*
- **`145` test cases** are discovered — the same count as the full run's `Ran 145 tests` (§2.1/§2.2), i.e. stable.
- **`kitty/glfw-x11.so` is NOT loaded as a Python module during discovery**, and is not even mapped into the process yet: the script reports `imported as a python module? False` and `present in /proc/self/maps at discovery time? False`. It is loaded later, natively — proved next.

### 4.1 `glfw-x11.so` is loaded **natively** (not imported) when the GLFW test runs

The GLFW backend is not a Python extension (it has no `PyInit_*`; §1.1). The test `kitty_tests/glfw.py:test_utf_8_strndup` obtains its path from `kitty.constants.glfw_path` (defined `kitty/constants.py:191`, returns `os.path.join(extensions_dir, f'{prefix}glfw-{module}.so')` at `constants.py:193`) and loads it with `ctypes.CDLL` at `kitty_tests/glfw.py:50` (`lib = ctypes.CDLL(backend_utils)`), then calls the native `utf_8_strndup` symbol. This was observed with an `sys.addaudithook` on `ctypes.dlopen` plus `/proc/self/maps` snapshots, running the **real** `test_utf_8_strndup` through `unittest`. The temporary script (created under `/tmp`, outside the repository, removed afterward) and its complete, unedited output follow. The output is byte-identical across two runs **except** the sub-second wall-clock figure on the `unittest` summary line — `Ran 1 test in 0.005s` (run 1) vs `Ran 1 test in 0.006s` (run 2) — which is expected to vary; every other line, including all state snapshots and the captured `dlopen` event, is stable. Run 1 is shown:

```python
#!/usr/bin/env python3
# observe_native_glfw.py  (temporary observation script; lives under /tmp, never in the repo)
# Purpose: prove that kitty/glfw-x11.so is loaded NATIVELY via ctypes.dlopen (not imported as a
# Python module) when the real kitty_tests/glfw.py:test_utf_8_strndup runs. We install an audit
# hook on 'ctypes.dlopen', snapshot /proc/self/maps and sys.modules before and after, and run the
# real test through unittest. Run through the REAL launcher:
#   ./kitty/launcher/kitty +launch <path>/observe_native_glfw.py
import sys, os, unittest, traceback

REPO_MARKER = 'kitty_tests'                 # repo test frames contain this path component


def glfw_mapped():
    """OS truth: is glfw-x11.so mapped into this process address space right now?"""
    with open('/proc/self/maps') as f:
        return 'glfw-x11.so' in f.read()


def glfw_so_backed_module():
    """Correct test: is any loaded module backed by a glfw .so ORIGIN (not just a name match)?"""
    for name, mod in list(sys.modules.items()):
        origin = getattr(getattr(mod, '__spec__', None), 'origin', None) or getattr(mod, '__file__', None)
        if origin and origin.endswith('.so') and 'glfw' in os.path.basename(origin):
            return (name, origin)
    return None


def glfw_name_only():
    """sys.modules keys that merely CONTAIN 'glfw' as a substring (name-only, can mislead)."""
    return [k for k in sys.modules if 'glfw' in k.lower()]


def print_state():
    print(f'glfw-x11.so as a .so-backed python module in sys.modules? {glfw_so_backed_module()}')
    print(f'glfw-x11.so mapped in /proc/self/maps (OS truth)?         {glfw_mapped()}')
    print(f'sys.modules keys merely containing "glfw" (name-only):    {glfw_name_only()}')


dlopen_hits = []


def _audit(ev, args):
    if ev == 'ctypes.dlopen' and args and 'glfw' in str(args[0]):
        dlopen_hits.append((str(args[0]), traceback.extract_stack()))


sys.addaudithook(_audit)

import kitty_tests.main                      # bootstrap (runs kitty_tests/__init__.py)

print("=== BEFORE running glfw.test_utf_8_strndup ===")
print_state()
print()

# Run the REAL test method through unittest, output interleaved on stdout in order.
from kitty_tests.glfw import TestGLFW
suite = unittest.TestLoader().loadTestsFromName('test_utf_8_strndup', TestGLFW)
result = unittest.TextTestRunner(stream=sys.stdout, verbosity=2).run(suite)
print()

print("=== AFTER running glfw.test_utf_8_strndup ===")
print(f'test outcome: wasSuccessful={result.wasSuccessful()} (failures={len(result.failures)}, errors={len(result.errors)})')
print_state()
print()

print(f"=== native ctypes.dlopen(glfw) events captured: {len(dlopen_hits)} ===")
for target, stack in dlopen_hits:
    print(f"dlopen target: {target}")
    print("call stack (repo frames):")
    for fr in stack:
        if REPO_MARKER in fr.filename:
            print(f"  {fr.filename}:{fr.lineno} ({fr.name})  |  {fr.line}")
```

```console
$ CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./kitty/launcher/kitty +launch /tmp/kitty_fix_evidence/observe_native_glfw.py
```

```text
=== BEFORE running glfw.test_utf_8_strndup ===
glfw-x11.so as a .so-backed python module in sys.modules? None
glfw-x11.so mapped in /proc/self/maps (OS truth)?         False
sys.modules keys merely containing "glfw" (name-only):    []

test_utf_8_strndup (kitty_tests.glfw.TestGLFW.test_utf_8_strndup) ... ok

----------------------------------------------------------------------
Ran 1 test in 0.005s

OK

=== AFTER running glfw.test_utf_8_strndup ===
test outcome: wasSuccessful=True (failures=0, errors=0)
glfw-x11.so as a .so-backed python module in sys.modules? None
glfw-x11.so mapped in /proc/self/maps (OS truth)?         True
sys.modules keys merely containing "glfw" (name-only):    ['kitty_tests.glfw']

=== native ctypes.dlopen(glfw) events captured: 1 ===
dlopen target: /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/glfw-x11.so
call stack (repo frames):
  /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/glfw.py:50 (test_utf_8_strndup)  |  lib = ctypes.CDLL(backend_utils)
```

**Reading the output:** *before* the test runs, `glfw-x11.so` is neither a `.so`-backed Python module (`None`) nor mapped in `/proc/self/maps` (`False`). *After* `test_utf_8_strndup` runs (and passes), it is **still not** a `.so`-backed Python module (`None`) — confirming it is **never imported** — yet it is now **mapped in `/proc/self/maps` (`True`)**, because the audit hook captured exactly **one** `ctypes.dlopen` of `.../kitty/glfw-x11.so` whose call stack is `kitty_tests/glfw.py:50 (test_utf_8_strndup) | lib = ctypes.CDLL(backend_utils)`. The `['kitty_tests.glfw']` entry in the name-only list is the **test module** `kitty_tests.glfw`, not the extension (a name-substring match), which is why the correct origin-based check still reports `None`. This is the definitive evidence that `glfw-x11.so` participates in the test run via **native `dlopen`**, contradicting any claim that it is "never loaded" or "only file-checked".

## 5. How test failures cascade when the extension modules are unavailable

Each compiled extension was moved out of its real import path **one at a time**, the **full** `./test.py` was re-run, the complete output and exit code captured, and the artifact restored. The reproduction used a safe harness: a private `mktemp -d` stash (mode 0700), an **unconditional** `trap restore EXIT INT TERM` that always moves the `.so` back, and SHA-256 verification that each artifact was restored byte-for-byte (before/after hashes shown inline in each subsection below and summarized in the Appendix). These `.so` are git-ignored generated artifacts (`.gitignore:1`), so moving and restoring them leaves the tracked tree unchanged.

The complete harness, `cascade_one.sh`, invoked once per extension as `bash cascade_one.sh <relative-.so-path>`, is reproduced here in full:

```bash
#!/usr/bin/env bash
# cascade_one.sh <relative-.so-path>
# Move ONE compiled extension out of its real import path, run the FULL ./test.py through the real
# launcher, capture exit code + complete output, then restore the artifact byte-for-byte. An
# unconditional trap restores the .so even if the run is interrupted. .so are git-ignored
# (.gitignore:1 = *.so), so moving/restoring leaves the tracked tree unchanged.
set -u
export PATH="/usr/local/go/bin:$PATH"              # canonical prerequisite: Go on PATH (see Methodology)
SO="$1"
BASENAME="$(basename "$SO")"
STASH="$(mktemp -d)"; chmod 700 "$STASH"           # private stash, mode 0700
restore() {                                        # always move the .so back
    if [ -f "$STASH/$BASENAME" ] && [ ! -e "$SO" ]; then mv "$STASH/$BASENAME" "$SO"; fi
    rmdir "$STASH" 2>/dev/null || true
}
trap restore EXIT INT TERM

sha_before="$(sha256sum "$SO" | awk '{print $1}')"; mode_before="$(stat -c '%a' "$SO")"
echo "### CASCADE for $SO"
echo "BEFORE : sha256=$sha_before mode=$mode_before present=$([ -e "$SO" ] && echo yes || echo no)"
mv "$SO" "$STASH/$BASENAME"
echo "MOVED  : present=$([ -e "$SO" ] && echo yes || echo no) stashed=$([ -e "$STASH/$BASENAME" ] && echo yes || echo no)"
CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./test.py > "$STASH/out.txt" 2>&1
rc=$?
mv "$STASH/$BASENAME" "$SO"                         # explicit restore (trap is the safety net)
sha_after="$(sha256sum "$SO" | awk '{print $1}')"; mode_after="$(stat -c '%a' "$SO")"
echo "AFTER  : sha256=$sha_after mode=$mode_after present=$([ -e "$SO" ] && echo yes || echo no)"
echo "EXIT   : ./test.py exit_code=$rc"
[ "$sha_before" = "$sha_after" ] && echo "INTEGRITY: RESTORED BYTE-FOR-BYTE (sha256 match), mode preserved=$([ "$mode_before" = "$mode_after" ] && echo yes || echo no)" || echo "INTEGRITY: MISMATCH!"
echo "===== BEGIN ./test.py OUTPUT (exit=$rc) ====="
cat "$STASH/out.txt"
echo "===== END ./test.py OUTPUT ====="
```

Each subsection below shows the invocation, the harness's restoration proof (SHA-256 and mode before/after, plus the `./test.py` exit code), then the captured `./test.py` output.

### 5.1 `kitty/fast_data_types.so` removed → entire run aborts at bootstrap → **CRITICAL (root)**

With `fast_data_types.so` absent, the run dies **before any test is collected or run**, during the package-init import chain of §3/§9. The complete output is the traceback only — no test lines, no Go line:

```console
$ bash cascade_one.sh kitty/fast_data_types.so
```

Harness restoration proof — SHA-256-identical before and after, mode preserved, with the captured exit code:

```text
### CASCADE for kitty/fast_data_types.so
BEFORE : sha256=a3f7ec88487e1ab7b9ecba83415c5abcaf4011b68ca87db326e6e45e8ba62c11 mode=755 present=yes
MOVED  : present=no stashed=yes
AFTER  : sha256=a3f7ec88487e1ab7b9ecba83415c5abcaf4011b68ca87db326e6e45e8ba62c11 mode=755 present=yes
EXIT   : ./test.py exit_code=1
INTEGRITY: RESTORED BYTE-FOR-BYTE (sha256 match), mode preserved=yes
```

```text
Traceback (most recent call last):
  File "<frozen runpy>", line 198, in _run_module_as_main
  File "<frozen runpy>", line 88, in _run_code
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../__main__.py", line 7, in <module>
    main()
    ~~~~^^
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty/entry_points.py", line 192, in main
    namespaced(['+', first_arg[1:]] + sys.argv[2:])
    ~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty/entry_points.py", line 146, in namespaced
    func(args[1:])
    ~~~~^^^^^^^^^^
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty/entry_points.py", line 73, in launch
    runpy.run_path(exe, run_name='__main__')
    ~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "<frozen runpy>", line 287, in run_path
  File "<frozen runpy>", line 98, in _run_module_code
  File "<frozen runpy>", line 88, in _run_code
  File "./test.py", line 13, in <module>
    main()
    ~~~~^^
  File "./test.py", line 8, in main
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
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/__init__.py", line 21, in <module>
    from kitty.config import finalize_keys, finalize_mouse_mappings
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty/config.py", line 10, in <module>
    from .conf.utils import BadLine, parse_config_base
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty/conf/utils.py", line 27, in <module>
    from ..fast_data_types import Color
ModuleNotFoundError: No module named 'kitty.fast_data_types'
```

The terminal frame is `kitty/conf/utils.py:27` → `ModuleNotFoundError: No module named 'kitty.fast_data_types'`, reached from `test.py:8` → `kitty_tests/__init__.py:21` → `kitty/config.py:10` → `kitty/conf/utils.py:27`. Because the failure is in package initialization, `find_all_tests()` is never called and **zero** tests run. This is the root: everything depends on it.

### 5.2 `kittens/transfer/rsync.so` removed → Python suite aborts at discovery → **CRITICAL for the Python suite**

With `rsync.so` absent, the environment/preamble prints, the Go tests are **launched** (the line `Go packages being tested:` comes from `kitty_tests/main.py:277`, after `go_proc = run_go(...)` at `main.py:270`), and then discovery crashes:

```console
$ bash cascade_one.sh kittens/transfer/rsync.so
```

Harness restoration proof — SHA-256-identical before and after, mode preserved, with the captured exit code:

```text
### CASCADE for kittens/transfer/rsync.so
BEFORE : sha256=c0bf7b038558d5dfabd176602d7a05458459bc635ee8dd869e1125695dea35a8 mode=755 present=yes
MOVED  : present=no stashed=yes
AFTER  : sha256=c0bf7b038558d5dfabd176602d7a05458459bc635ee8dd869e1125695dea35a8 mode=755 present=yes
EXIT   : ./test.py exit_code=1
INTEGRITY: RESTORED BYTE-FOR-BYTE (sha256 match), mode preserved=yes
```

```text
Running under CI: True
Using PATH in test environment: /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty_tests/kitty/launcher:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Python: /usr/bin/python
Intrinsics: has_avx2=True has_sse4_2=True
Go executable: /usr/local/go/bin/go
Go packages being tested: tools/rsync tools/utils/base85 tools/wcswidth tools/tui/shell_integration kittens/hints tools/tui/graphics tools/utils/humanize tools/utils/style tools/tui/sgr tools/simdstring tools/unicode_names tools/themes tools/utils/shlex kittens/hyperlinked_grep tools/tui/subseq tools/tui tools/cmd/at tools/utils tools/config tools/cli tools/tui/readline tools/utils/shm kittens/diff kittens/transfer tools/tui/loop kittens/ssh
Traceback (most recent call last):
  File "<frozen runpy>", line 198, in _run_module_as_main
  File "<frozen runpy>", line 88, in _run_code
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../__main__.py", line 7, in <module>
    main()
    ~~~~^^
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty/entry_points.py", line 192, in main
    namespaced(['+', first_arg[1:]] + sys.argv[2:])
    ~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty/entry_points.py", line 146, in namespaced
    func(args[1:])
    ~~~~^^^^^^^^^^
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty/entry_points.py", line 73, in launch
    runpy.run_path(exe, run_name='__main__')
    ~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "<frozen runpy>", line 287, in run_path
  File "<frozen runpy>", line 98, in _run_module_code
  File "<frozen runpy>", line 88, in _run_code
  File "./test.py", line 13, in <module>
    main()
    ~~~~^^
  File "./test.py", line 9, in main
    getattr(m, 'main')()
    ~~~~~~~~~~~~~~~~~~^^
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/main.py", line 338, in main
    run_tests()
    ~~~~~~~~~^^
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/main.py", line 279, in run_tests
    run_python_tests(args, go_proc)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/main.py", line 211, in run_python_tests
    tests = find_all_tests()
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/main.py", line 64, in find_all_tests
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
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 13, in <module>
    from kittens.transfer.rsync import Differ, Hasher, Patcher, parse_ftc
ModuleNotFoundError: No module named 'kittens.transfer.rsync'
```

The terminal frame is `kitty_tests/file_transmission.py:13` → `ModuleNotFoundError: No module named 'kittens.transfer.rsync'`, reached from `test.py:9` → `kitty_tests/main.py:338` (`main` → `run_tests()`) → `main.py:279` (`run_tests` → `run_python_tests`) → `main.py:211` (`run_python_tests` → `find_all_tests()`) → `main.py:64` (the direct `importlib.import_module`) → `file_transmission.py:13`.

**Important nuance about the Go tests (observed):** the output proves the Go tests were **launched** (`Go packages being tested:`) but it does **not** contain `All Go tests succeeded`. The following mechanism is **(inferred from the source structure at `main.py:211`-`238`)**: that completion line is printed by `print_go()` (`main.py:213`, which calls `go_proc.wait()` at `main.py:214`), and `print_go()` is invoked only later (`main.py:238`), **after** `find_all_tests()` at `main.py:211`. Since the crash happens at `main.py:211`, the Go result is **never waited on or reported** in this cascade. So the correct statement is: the Go tests *launch*, but their completion is **not reported** — not that they "succeed".

**Observed run-to-run variance (not a result inconsistency):** the *order* of packages on the `Go packages being tested:` line is not stable across runs (it reflects Go's unordered package discovery) — compare this section's line with the one in §5.3; the *set* is invariably the same 26 packages, and the ordering affects neither which tests run nor the pass/fail outcome.

### 5.3 `kitty/glfw-x11.so` removed → **two** tests break (native-load ERROR + file-check FAIL) → **OPTIONAL for suite completion**

Unlike the two Python extensions, removing the GLFW backend does **not** abort the run: because `glfw-x11.so` is not imported at discovery, the suite still collects and runs all **145** tests, and the Go tests still complete (`All Go tests succeeded`). But **two** tests break — not one — and the delta versus the baseline (`failures=4, errors=0` in §2) is exactly `+1 failure` and `+1 error`:

```console
$ bash cascade_one.sh kitty/glfw-x11.so
```

Harness restoration proof — SHA-256-identical before and after, mode preserved, with the captured exit code:

```text
### CASCADE for kitty/glfw-x11.so
BEFORE : sha256=99db5778b34c5370637a2fce743fd6379ebb879f9fbee34bb38f67f0cbd83a56 mode=755 present=yes
MOVED  : present=no stashed=yes
AFTER  : sha256=99db5778b34c5370637a2fce743fd6379ebb879f9fbee34bb38f67f0cbd83a56 mode=755 present=yes
EXIT   : ./test.py exit_code=1
INTEGRITY: RESTORED BYTE-FOR-BYTE (sha256 match), mode preserved=yes
```

```text
Running under CI: True
Using PATH in test environment: /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty_tests/kitty/launcher:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Python: /usr/bin/python
Intrinsics: has_avx2=True has_sse4_2=True
Go executable: /usr/local/go/bin/go
Go packages being tested: tools/unicode_names tools/tui/loop tools/simdstring tools/wcswidth kittens/ssh tools/tui/graphics kittens/hyperlinked_grep tools/utils/humanize kittens/hints tools/config kittens/diff tools/tui tools/themes tools/tui/shell_integration tools/utils/style kittens/transfer tools/cmd/at tools/tui/readline tools/utils/base85 tools/utils tools/tui/subseq tools/cli tools/utils/shlex tools/tui/sgr tools/utils/shm tools/rsync
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
test_font_selection (kitty_tests.fonts.Selection.test_font_selection) ... 
  test_font_selection (kitty_tests.fonts.Selection.test_font_selection) (spec='ubuntu mono') ... FAIL
  test_font_selection (kitty_tests.fonts.Selection.test_font_selection) (spec='family="ubuntu mono"') ... FAIL
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
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/glfw.py", line 50, in test_utf_8_strndup
    lib = ctypes.CDLL(backend_utils)
    backend_utils = '/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/glfw-x11.so'
    ctypes = <module 'ctypes' from '/usr/lib/python3.13/ctypes/__init__.py'>
    glfw_path = <function glfw_path at 0x787366ceb600>
    self = <kitty_tests.glfw.TestGLFW testMethod=test_utf_8_strndup>
  File "/usr/lib/python3.13/ctypes/__init__.py", line 390, in __init__
    self._handle = _dlopen(self._name, mode)
                   ~~~~~~~^^^^^^^^^^^^^^^^^^
    _FuncPtr = <class 'ctypes.CDLL.__init__.<locals>._FuncPtr'>
    flags = 1
    handle = None
    mode = 0
    name = '/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/glfw-x11.so'
    self = <CDLL '/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/glfw-x11.so', handle 0 at 0x787364affcb0>
    use_errno = False
    use_last_error = False
    winmode = None
OSError: /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/glfw-x11.so: cannot open shared object file: No such file or directory

======================================================================
FAIL: test_transfer_receive (kitty_tests.file_transmission.TestFileTransmission.test_transfer_receive)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 476, in test_transfer_receive
    self.basic_transfer_tests()
    ~~~~~~~~~~~~~~~~~~~~~~~~~^^
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 443, in basic_transfer_tests
    multiple_files()
    ~~~~~~~~~~~~~~^^
    d = <_io.BufferedWriter name='/tmp/tmpe5njcxbs/dest'>
    dest = '/tmp/tmpe5njcxbs/dest'
    multiple_files = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files at 0x78735ff6ede0>
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x787364c65160>
    s = <_io.BufferedWriter name='/tmp/tmpe5njcxbs/src'>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
    single_file = <function TestFileTransmission.basic_transfer_tests.<locals>.single_file at 0x787364c71080>
    src = '/tmp/tmpe5njcxbs/src'
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 432, in multiple_files
    self.assertEqual(expected, actual)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^
    Entry = <class 'kitty_tests.file_transmission.Entry'>
    actual = {'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1783983422937923359, mode='0o120777', nlink=1), 'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'sym': Entry(relpath='sym', mtime=1783983422937923359, mode='0o120777', nlink=1), 'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1)}
    b = PosixPath('/tmp/tmpe5njcxbs/msrc')
    cmd = ()
    de = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.de at 0x78735ff6ff60>
    dest = '/tmp/tmpe5njcxbs/mdest'
    dirnames = []
    dirpath = '/tmp/tmpe5njcxbs/mdest/msrc/empty'
    entry = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.entry at 0x78735ff6f2e0>
    expected = {'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1), 'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1783983422937923359, mode='0o120777', nlink=1), 'sym': Entry(relpath='sym', mtime=1783983422937923359, mode='0o120777', nlink=1)}
    f = <_io.BufferedWriter name='/tmp/tmpe5njcxbs/msrc/sub/reg'>
    filenames = []
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7873657b5e50>
    s = PosixPath('/tmp/tmpe5njcxbs/msrc/sub')
    se = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.se at 0x78735ff6f380>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
    src = '/tmp/tmpe5njcxbs/msrc'
    x = 'reg'
AssertionError: {'simple': Entry(relpath='simple', mtime=130[497 chars]k=1)} != {'sub': Entry(relpath='sub', mtime=0, mode='[497 chars]k=1)}
  {'abssym': Entry(relpath='abssym', mtime=1783983422937923359, mode='0o120777', nlink=1),
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
   'sym': Entry(relpath='sym', mtime=1783983422937923359, mode='0o120777', nlink=1)}

======================================================================
FAIL: test_transfer_send (kitty_tests.file_transmission.TestFileTransmission.test_transfer_send)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 507, in test_transfer_send
    self.basic_transfer_tests()
    ~~~~~~~~~~~~~~~~~~~~~~~~~^^
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 443, in basic_transfer_tests
    multiple_files()
    ~~~~~~~~~~~~~~^^
    d = <_io.BufferedWriter name='/tmp/tmpwk6wo7p2/dest'>
    dest = '/tmp/tmpwk6wo7p2/dest'
    multiple_files = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files at 0x78735ff86480>
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x78735ff2e7b0>
    s = <_io.BufferedWriter name='/tmp/tmpwk6wo7p2/src'>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
    single_file = <function TestFileTransmission.basic_transfer_tests.<locals>.single_file at 0x78735ff84360>
    src = '/tmp/tmpwk6wo7p2/src'
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 432, in multiple_files
    self.assertEqual(expected, actual)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^
    Entry = <class 'kitty_tests.file_transmission.Entry'>
    actual = {'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1783983423357927873, mode='0o120777', nlink=1), 'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'sym': Entry(relpath='sym', mtime=1783983423357927873, mode='0o120777', nlink=1), 'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1)}
    b = PosixPath('/tmp/tmpwk6wo7p2/msrc')
    cmd = ()
    de = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.de at 0x78735ff876a0>
    dest = '/tmp/tmpwk6wo7p2/mdest'
    dirnames = []
    dirpath = '/tmp/tmpwk6wo7p2/mdest/msrc/empty'
    entry = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.entry at 0x78735ff86ac0>
    expected = {'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1), 'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1783983423357927873, mode='0o120777', nlink=1), 'sym': Entry(relpath='sym', mtime=1783983423357927873, mode='0o120777', nlink=1)}
    f = <_io.BufferedWriter name='/tmp/tmpwk6wo7p2/msrc/sub/reg'>
    filenames = []
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7873645b0c00>
    s = PosixPath('/tmp/tmpwk6wo7p2/msrc/sub')
    se = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.se at 0x78735ff86b60>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
    src = '/tmp/tmpwk6wo7p2/msrc'
    x = 'reg'
AssertionError: {'simple': Entry(relpath='simple', mtime=130[497 chars]k=1)} != {'sub': Entry(relpath='sub', mtime=0, mode='[497 chars]k=1)}
  {'abssym': Entry(relpath='abssym', mtime=1783983423357927873, mode='0o120777', nlink=1),
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
   'sym': Entry(relpath='sym', mtime=1783983423357927873, mode='0o120777', nlink=1)}

======================================================================
FAIL: test_glfw_modules (kitty_tests.check_build.TestBuild.test_glfw_modules)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/check_build.py", line 46, in test_glfw_modules
    self.assertTrue(os.path.isfile(path), f'{path} is not a file')
    ~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    glfw_path = <function glfw_path at 0x787366ceb600>
    is_macos = False
    linux_backends = ['x11']
    modules = ['x11']
    name = 'x11'
    path = '/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/glfw-x11.so'
    self = <kitty_tests.check_build.TestBuild testMethod=test_glfw_modules>
AssertionError: False is not true : /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/glfw-x11.so is not a file

======================================================================
FAIL: test_font_selection (kitty_tests.fonts.Selection.test_font_selection) (spec='ubuntu mono')
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/fonts.py", line 40, in s
    self.ae(expected, actual)
    ~~~~~~~^^^^^^^^^^^^^^^^^^
    actual = ('UbuntuMonoRoman-Regular', 'UbuntuMonoRoman-Bold', 'UbuntuMonoRoman-MediumItalic', 'UbuntuMonoRoman-BoldItalic')
    alternate = None
    expected = ('UbuntuMono-Regular', 'UbuntuMono-Bold', 'UbuntuMono-Italic', 'UbuntuMono-BoldItalic')
    family = 'ubuntu mono'
    opts = <kitty.options.types.Options object at 0x78735ec8e0d0>
    self = <kitty_tests.fonts.Selection testMethod=test_font_selection>
    x = 'UbuntuMonoRoman-BoldItalic'
AssertionError: Tuples differ: ('UbuntuMono-Regular', 'UbuntuMono-Bold', 'UbuntuMono[29 chars]lic') != ('UbuntuMonoRoman-Regular', 'UbuntuMonoRoman-Bold', '[55 chars]lic')

First differing element 0:
'UbuntuMono-Regular'
'UbuntuMonoRoman-Regular'

- ('UbuntuMono-Regular',
+ ('UbuntuMonoRoman-Regular',
?             +++++

-  'UbuntuMono-Bold',
+  'UbuntuMonoRoman-Bold',
?             +++++

-  'UbuntuMono-Italic',
+  'UbuntuMonoRoman-MediumItalic',
?             +++++ ++++++

-  'UbuntuMono-BoldItalic')
+  'UbuntuMonoRoman-BoldItalic')
?             +++++


======================================================================
FAIL: test_font_selection (kitty_tests.fonts.Selection.test_font_selection) (spec='family="ubuntu mono"')
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/fonts.py", line 40, in s
    self.ae(expected, actual)
    ~~~~~~~^^^^^^^^^^^^^^^^^^
    actual = ('UbuntuMonoRoman-Regular', 'UbuntuMonoRoman-Bold', 'UbuntuMonoRoman-MediumItalic', 'UbuntuMonoRoman-BoldItalic')
    alternate = None
    expected = ('UbuntuMono-Regular', 'UbuntuMono-Bold', 'UbuntuMono-Italic', 'UbuntuMono-BoldItalic')
    family = 'family="ubuntu mono"'
    opts = <kitty.options.types.Options object at 0x78735ec8e0d0>
    self = <kitty_tests.fonts.Selection testMethod=test_font_selection>
    x = 'UbuntuMonoRoman-BoldItalic'
AssertionError: Tuples differ: ('UbuntuMono-Regular', 'UbuntuMono-Bold', 'UbuntuMono[29 chars]lic') != ('UbuntuMonoRoman-Regular', 'UbuntuMonoRoman-Bold', '[55 chars]lic')

First differing element 0:
'UbuntuMono-Regular'
'UbuntuMonoRoman-Regular'

- ('UbuntuMono-Regular',
+ ('UbuntuMonoRoman-Regular',
?             +++++

-  'UbuntuMono-Bold',
+  'UbuntuMonoRoman-Bold',
?             +++++

-  'UbuntuMono-Italic',
+  'UbuntuMonoRoman-MediumItalic',
?             +++++ ++++++

-  'UbuntuMono-BoldItalic')
+  'UbuntuMonoRoman-BoldItalic')
?             +++++


----------------------------------------------------------------------
Ran 145 tests in 22.060s

FAILED (failures=5, errors=1, skipped=4)
All Go tests succeeded, ran in 22.2 seconds
[31mError[39m: Some tests failed!
```

The two newly-broken tests, both attributable to the missing `glfw-x11.so`, are:

1. **`ERROR: test_utf_8_strndup (kitty_tests.glfw.TestGLFW.test_utf_8_strndup)`** — the **native load** fails: `kitty_tests/glfw.py:50` `lib = ctypes.CDLL(backend_utils)` with `backend_utils = '.../kitty/glfw-x11.so'` raises `OSError: .../kitty/glfw-x11.so: cannot open shared object file: No such file or directory`. This is the runtime `dlopen`, proving the extension is genuinely loaded (not merely file-checked) during the suite.
2. **`FAIL: test_glfw_modules (kitty_tests.check_build.TestBuild.test_glfw_modules)`** — the **file check** fails: `kitty_tests/check_build.py:46` `self.assertTrue(os.path.isfile(path), ...)` → `AssertionError: False is not true : .../kitty/glfw-x11.so is not a file`.

The other four failures (`test_transfer_receive`, `test_transfer_send`, the two `test_font_selection`) are the same environment-specific failures as the baseline (§2.5) and are unrelated to GLFW. So the true blast radius of removing `glfw-x11.so` is **exactly two tests**; the remaining 143 are unaffected by its absence.

### 5.4 The `itertests()` guard is NOT reached (nuance, verified from the tracebacks)

The runner has a defensive guard: `itertests()` (`kitty_tests/main.py:52`) raises `Failed to import a test module` when it encounters a unittest `ModuleImportFailure` placeholder. In **neither** the `fast_data_types` nor the `rsync` cascade is this guard reached: `find_all_tests()` uses a **direct** `importlib.import_module(...)` at `main.py:64`, which raises the `ModuleNotFoundError` immediately (see the tracebacks in §5.1/§5.2), before any placeholder-based iteration. For `fast_data_types` the crash is even earlier — during package init, before `find_all_tests()` is called at all.

## 6. What the test output reveals about the dependency structure

Reading the observed output end-to-end, the dependency structure between build artifacts, test-module imports, and failure patterns is:

- **A single hard root.** The very first `.so`-backed kitty module in `sys.modules` at bootstrap (§4) is `kitty.fast_data_types`, and its removal (§5.1) aborts the process during package init with `ModuleNotFoundError` at `kitty/conf/utils.py:27`. The entire test framework — every one of the 145 cases — sits behind this import. This is the strongest possible coupling: build-artifact-missing ⇒ zero tests.
- **A discovery-time chokepoint for the Python suite.** `rsync.so` is not needed to bootstrap, but `find_all_tests()` (`main.py:64`) imports **every** test module including `file_transmission.py`, whose top-level import (`file_transmission.py:13`) needs `kittens.transfer.rsync`. So one missing leaf extension imported at module top level takes down the **whole Python suite** at discovery (§5.2) — while the independently-launched Go tests are left un-waited-on.
- **A run-time-only native leaf.** `glfw-x11.so` is decoupled from import entirely: it is `dlopen`'d via `ctypes` only inside one test body (§4.1). Its absence therefore surfaces **only** as failures of the two tests that touch it (§5.3), with no effect on discovery or on the other 143 tests.
- **The failure *stage* encodes the coupling.** Bootstrap-stage crash (fast_data_types) ⇒ CRITICAL root; discovery-stage crash (rsync) ⇒ CRITICAL for the Python suite; single-test-stage failure (glfw) ⇒ OPTIONAL. The exit code is `1` in all three cases, but *where* the run dies (or whether it completes) is the real signal.
- **Build selection drives which artifacts exist.** The platform-conditional source selection in `setup.py` (§1.2) means `glfw-wayland.so` is never built here, which is exactly why the non-CI `test_glfw_modules` fails (§2.4) — a direct line from build configuration to a specific test outcome.

## 7. Mapping the compiled extensions to the different test categories

### 7.1 The 22 discovered test modules

`find_all_tests()` (`kitty_tests/main.py:57`, `excludes=('main', 'gr')`) discovers 22 test modules under `kitty_tests/`: `check_build`, `clipboard`, `completion`, `crypto`, `datatypes`, `file_transmission`, `fonts`, `glfw`, `graphics`, `keys`, `layout`, `mouse`, `open_actions`, `options`, `parser`, `screen`, `search_query_parser`, `shell_integration`, `shm`, `ssh`, `tui`, `utmp`. Every one of them executes `from . import BaseTest` — **observed 22/22** by grepping each discovered module:

```console
$ for m in check_build clipboard completion crypto datatypes file_transmission fonts glfw \
           graphics keys layout mouse open_actions options parser screen search_query_parser \
           shell_integration shm ssh tui utmp; do
      grep -qE '^from \. import .*BaseTest' "kitty_tests/$m.py" && echo "$m: yes"; done | wc -l
```

```text
22
```

Executing `from . import BaseTest` runs `kitty_tests/__init__.py`, which is what loads `fast_data_types` transitively (import chain in §9). This was also confirmed dynamically through the **real** launcher with a temporary `observe_modules.py` that diffs `sys.modules` around `find_all_tests()` (byte-identical across two runs). It shows the discovery loop newly imports the 22 test modules — each exposing `BaseTest` — **plus** the package initializer `kitty_tests.__init__` itself (where `BaseTest` is *defined* and where `fast_data_types` is imported at `__init__.py:21-22`); the loop imports `__init__.py` because its name is not in `excludes`, but that module contributes no test cases. `main` and `gr` are **not** imported (both observed `False`), and **145** test cases are discovered:

```console
$ CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./kitty/launcher/kitty +launch /tmp/kitty_fix_evidence/observe_modules.py
```

```text
=== test modules newly imported by find_all_tests(): 23 ===
  kitty_tests.__init__  (exposes BaseTest? True)
  kitty_tests.check_build  (exposes BaseTest? True)
  kitty_tests.clipboard  (exposes BaseTest? True)
  kitty_tests.completion  (exposes BaseTest? True)
  kitty_tests.crypto  (exposes BaseTest? True)
  kitty_tests.datatypes  (exposes BaseTest? True)
  kitty_tests.file_transmission  (exposes BaseTest? True)
  kitty_tests.fonts  (exposes BaseTest? True)
  kitty_tests.glfw  (exposes BaseTest? True)
  kitty_tests.graphics  (exposes BaseTest? True)
  kitty_tests.keys  (exposes BaseTest? True)
  kitty_tests.layout  (exposes BaseTest? True)
  kitty_tests.mouse  (exposes BaseTest? True)
  kitty_tests.open_actions  (exposes BaseTest? True)
  kitty_tests.options  (exposes BaseTest? True)
  kitty_tests.parser  (exposes BaseTest? True)
  kitty_tests.screen  (exposes BaseTest? True)
  kitty_tests.search_query_parser  (exposes BaseTest? True)
  kitty_tests.shell_integration  (exposes BaseTest? True)
  kitty_tests.shm  (exposes BaseTest? True)
  kitty_tests.ssh  (exposes BaseTest? True)
  kitty_tests.tui  (exposes BaseTest? True)
  kitty_tests.utmp  (exposes BaseTest? True)
=== every discovered module exposes BaseTest? True ===
=== 'kitty_tests.main' discovered? False | 'kitty_tests.gr' discovered? False ===
=== total test cases discovered: 145 ===
```

(The header count of 23 = the 22 test modules + the `kitty_tests.__init__` initializer; the 22 figure in the module list above excludes the initializer, which defines `BaseTest` rather than being a test module.)

### 7.2 Extension → test-category mapping (observed)

| Extension | How it is consumed | Test categories that depend on it |
|-----------|--------------------|-----------------------------------|
| `kitty/fast_data_types.so` (Python ext) | Imported at **bootstrap** via `kitty_tests/__init__.py:21-22`; also directly by `check_build.test_loading_extensions` (`check_build.py:29`) | **All 22 modules** (every test transitively; the package cannot even be imported without it) |
| `kittens/transfer/rsync.so` (Python ext) | Imported at **discovery** via `file_transmission.py:13`; also directly by `check_build.test_loading_extensions` (`check_build.py:30`) | `file_transmission` (all its tests) and `check_build.test_loading_extensions`; the whole **Python** suite depends on it being importable at discovery |
| `kitty/glfw-x11.so` (native lib) | **Native `dlopen`** via `ctypes.CDLL` at `glfw.py:50`; **file-existence check** at `check_build.py:46` | `glfw.test_utf_8_strndup` (native load) and `check_build.test_glfw_modules` (file check) — **only these two** |

## 8. CRITICAL vs OPTIONAL classification (with observed rationale)

| Extension | Kind | Classification | Failure stage when removed | Observed effect (from §5) |
|-----------|------|----------------|----------------------------|---------------------------|
| `kitty/fast_data_types.so` | Python C-extension (`PyInit_fast_data_types`) | **CRITICAL (root)** | Runner **bootstrap** (`test.py:8` → package init → `kitty/conf/utils.py:27`) | `ModuleNotFoundError`; **0** of 145 tests run; no Go line |
| `kittens/transfer/rsync.so` | Python C-extension (`PyInit_rsync`) | **CRITICAL for the Python suite** | **Discovery** (`find_all_tests()` `main.py:64` → `file_transmission.py:13`) | `ModuleNotFoundError`; **0** Python tests run; Go tests only *launched*, completion **not reported** |
| `kitty/glfw-x11.so` | **Native** shared library (no `PyInit`; native symbol `utf_8_strndup`) | **OPTIONAL for suite completion** | Single-test **run time** | Suite still runs all **145**; Go completes; exactly **2** tests break — `test_utf_8_strndup` **ERROR** (native `dlopen` at `glfw.py:50`) + `test_glfw_modules` **FAIL** (file check at `check_build.py:46`) |

**Rationale:** "CRITICAL" here means *its absence prevents tests from running at all* (bootstrap or discovery), whereas "OPTIONAL" means *the suite still runs to completion and only the handful of tests that specifically exercise the artifact fail*. Note that `glfw-x11.so` being OPTIONAL does **not** mean it is unused during tests — §4.1/§5.3 show it is genuinely loaded natively; it is simply not on any import path that gates collection.

## 9. The actual import chains established during the test run

To capture the **real** chains (not a synthetic import), a non-invasive `sys.meta_path` finder recorded the repo-frame call stack at the **first** import of each target, then the **real** entry point `./test.py` was executed via `runpy.run_path(test_py, run_name='__main__')` so the frames include the genuine launcher preamble, `test.py:8`, and `find_all_tests()` at `main.py:64`. The finder returns `None` from `find_spec` (so the real import machinery still resolves everything). Because this observer is itself launched by `kitty +launch`, the launcher's own `__main__.py`/`entry_points.py` frames are already on the stack; the observer's own `/tmp` frame is filtered out (it is not under the repo root), which is why `entry_points.py:73`'s `runpy.run_path(...)` appears directly above `test.py:13`. The temporary script (created under `/tmp`, outside the repository, removed afterward) was, in full:

```python
#!/usr/bin/env python3
# observe_importchain.py  (temporary observation script; lives under /tmp, never in the repo)
# Capture the REAL import chains by recording the repo-frame call stack at the FIRST import of
# each target module, then execute the REAL entry point ./test.py via runpy so the frames include
# the genuine launcher preamble (this observer was itself launched by kitty +launch, so the
# launcher's __main__.py/entry_points.py frames are already on the stack). The finder returns None
# from find_spec, so the real import machinery still resolves everything (non-invasive).
# Run through the REAL launcher:
#   CHAIN_TRACE_FILE=<path>/chain.txt ./kitty/launcher/kitty +launch <path>/observe_importchain.py
import sys, os, runpy, traceback
from importlib.abc import MetaPathFinder

TARGETS = {'kitty.config', 'kitty.conf.utils', 'kitty.fast_data_types', 'kittens.transfer.rsync'}
REPO = os.getcwd()
_seen = set()
_out = open(os.environ['CHAIN_TRACE_FILE'], 'w')


def emit(line=''):
    _out.write(line + '\n')
    _out.flush()


def is_repo_frame(fr):
    # Keep genuine repo source frames; drop this /tmp observer, the stdlib, and <frozen ...> frames.
    if fr.filename.startswith('<'):
        return False
    return os.path.abspath(os.path.join(REPO, fr.filename)).startswith(REPO + os.sep)


class StackTracer(MetaPathFinder):
    def find_spec(self, fullname, path, target=None):
        if fullname in TARGETS and fullname not in _seen:
            _seen.add(fullname)
            emit(f'>>> FIRST IMPORT OF: {fullname}')
            for fr in traceback.extract_stack():
                if is_repo_frame(fr):
                    disp = fr.filename[len(REPO) + 1:] if fr.filename.startswith(REPO + os.sep) else fr.filename
                    emit(f'    {disp}:{fr.lineno} ({fr.name})  |  {fr.line}')
            if fullname == 'kitty.fast_data_types':
                emit(f"    after this: 'kitty.fast_data_types' in sys.modules -> {'kitty.fast_data_types' in sys.modules}")
                emit(f"    at this point: 'kittens.transfer.rsync' in sys.modules -> {'kittens.transfer.rsync' in sys.modules}")
            emit()
        return None                          # non-invasive: real finders resolve the import


sys.meta_path.insert(0, StackTracer())
test_py = os.path.join(REPO, 'test.py')
sys.argv = [test_py, '--module', 'check_build']
emit(f"========== EXECUTING REAL ENTRY POINT: runpy.run_path('{test_py}') argv={sys.argv} ==========")
emit()
try:
    runpy.run_path(test_py, run_name='__main__')     # the REAL entry point
    code = 0
except SystemExit as e:
    code = e.code if e.code is not None else 0
    emit(f'(real entry point exited with SystemExit code={code})')
emit()
emit(f"FINAL: 'kittens.transfer.rsync' in sys.modules -> {'kittens.transfer.rsync' in sys.modules}")
_out.close()
```

Complete, unedited captured trace written to `$CHAIN_TRACE_FILE` (byte-identical across two runs). Frame paths are shown repo-relative — the script strips the `$REPO/` prefix for readability — while the banner records the absolute `test.py` path exactly as `runpy.run_path` received it:

```console
$ CHAIN_TRACE_FILE=/tmp/kitty_fix_evidence/chain.txt CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 \
    ./kitty/launcher/kitty +launch /tmp/kitty_fix_evidence/observe_importchain.py
```

```text
========== EXECUTING REAL ENTRY POINT: runpy.run_path('/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/test.py') argv=['/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/test.py', '--module', 'check_build'] ==========

>>> FIRST IMPORT OF: kitty.config
    kitty/launcher/../../__main__.py:7 (<module>)  |  main()
    kitty/launcher/../../kitty/entry_points.py:192 (main)  |  namespaced(['+', first_arg[1:]] + sys.argv[2:])
    kitty/launcher/../../kitty/entry_points.py:146 (namespaced)  |  func(args[1:])
    kitty/launcher/../../kitty/entry_points.py:73 (launch)  |  runpy.run_path(exe, run_name='__main__')
    test.py:13 (<module>)  |  main()
    test.py:8 (main)  |  m = importlib.import_module('kitty_tests.main')
    kitty/launcher/../../kitty_tests/__init__.py:21 (<module>)  |  from kitty.config import finalize_keys, finalize_mouse_mappings

>>> FIRST IMPORT OF: kitty.conf.utils
    kitty/launcher/../../__main__.py:7 (<module>)  |  main()
    kitty/launcher/../../kitty/entry_points.py:192 (main)  |  namespaced(['+', first_arg[1:]] + sys.argv[2:])
    kitty/launcher/../../kitty/entry_points.py:146 (namespaced)  |  func(args[1:])
    kitty/launcher/../../kitty/entry_points.py:73 (launch)  |  runpy.run_path(exe, run_name='__main__')
    test.py:13 (<module>)  |  main()
    test.py:8 (main)  |  m = importlib.import_module('kitty_tests.main')
    kitty/launcher/../../kitty_tests/__init__.py:21 (<module>)  |  from kitty.config import finalize_keys, finalize_mouse_mappings
    kitty/launcher/../../kitty/config.py:10 (<module>)  |  from .conf.utils import BadLine, parse_config_base

>>> FIRST IMPORT OF: kitty.fast_data_types
    kitty/launcher/../../__main__.py:7 (<module>)  |  main()
    kitty/launcher/../../kitty/entry_points.py:192 (main)  |  namespaced(['+', first_arg[1:]] + sys.argv[2:])
    kitty/launcher/../../kitty/entry_points.py:146 (namespaced)  |  func(args[1:])
    kitty/launcher/../../kitty/entry_points.py:73 (launch)  |  runpy.run_path(exe, run_name='__main__')
    test.py:13 (<module>)  |  main()
    test.py:8 (main)  |  m = importlib.import_module('kitty_tests.main')
    kitty/launcher/../../kitty_tests/__init__.py:21 (<module>)  |  from kitty.config import finalize_keys, finalize_mouse_mappings
    kitty/launcher/../../kitty/config.py:10 (<module>)  |  from .conf.utils import BadLine, parse_config_base
    kitty/launcher/../../kitty/conf/utils.py:27 (<module>)  |  from ..fast_data_types import Color
    after this: 'kitty.fast_data_types' in sys.modules -> False
    at this point: 'kittens.transfer.rsync' in sys.modules -> False

>>> FIRST IMPORT OF: kittens.transfer.rsync
    kitty/launcher/../../__main__.py:7 (<module>)  |  main()
    kitty/launcher/../../kitty/entry_points.py:192 (main)  |  namespaced(['+', first_arg[1:]] + sys.argv[2:])
    kitty/launcher/../../kitty/entry_points.py:146 (namespaced)  |  func(args[1:])
    kitty/launcher/../../kitty/entry_points.py:73 (launch)  |  runpy.run_path(exe, run_name='__main__')
    test.py:13 (<module>)  |  main()
    test.py:9 (main)  |  getattr(m, 'main')()
    kitty/launcher/../../kitty_tests/main.py:338 (main)  |  run_tests()
    kitty/launcher/../../kitty_tests/main.py:279 (run_tests)  |  run_python_tests(args, go_proc)
    kitty/launcher/../../kitty_tests/main.py:211 (run_python_tests)  |  tests = find_all_tests()
    kitty/launcher/../../kitty_tests/main.py:64 (find_all_tests)  |  m = importlib.import_module(package + '.' + x.partition('.')[0])
    kitty/launcher/../../kitty_tests/file_transmission.py:13 (<module>)  |  from kittens.transfer.rsync import Differ, Hasher, Patcher, parse_ftc

(real entry point exited with SystemExit code=0)

FINAL: 'kittens.transfer.rsync' in sys.modules -> True
```

The same invocation's **combined stdout and stderr** (`2>&1`): the runner prints its environment banner — `Running under CI`, the test `PATH`, `Python`, and `Intrinsics` — to **stdout** via `env_for_python_tests()` (`kitty_tests/main.py:297-331`), while `unittest` writes the per-test results to **stderr**; the two are merged here so the whole run is visible in order. The filtered `check_build` suite still triggers discovery of **all** modules at `main.py:211`, which is why `rsync` loads even under `--module check_build`. This output is byte-identical across two runs **except** the sub-second `Ran 9 tests in 0.076s` timing (run 2: `0.078s`), which is expected to vary. Run 1:

```console
$ CHAIN_TRACE_FILE=/tmp/kitty_fix_evidence/chain.txt CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 \
    ./kitty/launcher/kitty +launch /tmp/kitty_fix_evidence/observe_importchain.py 2>&1
```

```text
Running under CI: True
Using PATH in test environment: /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty_tests/kitty/launcher:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Python: /usr/bin/python
Intrinsics: has_avx2=True has_sse4_2=True
test_all_kitten_names (kitty_tests.check_build.TestBuild.test_all_kitten_names) ... ok
test_ca_certificates (kitty_tests.check_build.TestBuild.test_ca_certificates) ... skipped 'CA certificates are only tested on frozen builds'
test_docs_url (kitty_tests.check_build.TestBuild.test_docs_url) ... ok
test_exe (kitty_tests.check_build.TestBuild.test_exe) ... ok
test_filesystem_locations (kitty_tests.check_build.TestBuild.test_filesystem_locations) ... ok
test_glfw_modules (kitty_tests.check_build.TestBuild.test_glfw_modules) ... ok
test_launcher_ensures_stdio (kitty_tests.check_build.TestBuild.test_launcher_ensures_stdio) ... ok
test_loading_extensions (kitty_tests.check_build.TestBuild.test_loading_extensions) ... ok
test_loading_shaders (kitty_tests.check_build.TestBuild.test_loading_shaders) ... ok

----------------------------------------------------------------------
Ran 9 tests in 0.076s

OK (skipped=1)
```

### 9.1 The two Python-extension chains, annotated

- **`kitty.fast_data_types` (bootstrap chain):**
  `kitty/launcher/../../__main__.py:7` `main()` → `kitty/entry_points.py:192` → `:146` → `:73` `runpy.run_path(...)` → `test.py:13` `main()` → **`test.py:8`** `importlib.import_module('kitty_tests.main')` → **`kitty_tests/__init__.py:21`** `from kitty.config import ...` → `kitty/config.py:10` `from .conf.utils import ...` → **`kitty/conf/utils.py:27`** `from ..fast_data_types import Color`. This confirms the root import is triggered by the **package** initializer reached directly from `test.py:8`; the `kitty_tests/main.py` module body is not on this chain.
- **`kittens.transfer.rsync` (discovery chain):**
  (launcher preamble as in the bootstrap chain above) → `test.py:9` `getattr(m,'main')()` → `kitty_tests/main.py:338` `run_tests()` → `main.py:279` `run_python_tests(args, go_proc)` → **`main.py:211`** `tests = find_all_tests()` → **`main.py:64`** `importlib.import_module(...)` → **`kitty_tests/file_transmission.py:13`** `from kittens.transfer.rsync import Differ, Hasher, Patcher, parse_ftc`.

### 9.2 The native GLFW chain (not an import)

`glfw-x11.so` establishes no Python import chain. Its runtime chain is the **native** one captured in §4.1: `kitty_tests/glfw.py:test_utf_8_strndup` → `kitty.constants.glfw_path('x11')` (`constants.py:191-193`) → `kitty_tests/glfw.py:50` `ctypes.CDLL(backend_utils)` → `dlopen(".../kitty/glfw-x11.so")`. The observed `ctypes.dlopen` audit event and the `/proc/self/maps` transition (`False` → `True`) in §4.1 are the evidence for this chain.

## Appendix — repository integrity and provenance

This QnA investigation is read-only with respect to tracked files: the **only** repository change is the addition/update of this document. The distinction between the two relevant commits is important:

- **Source commit under investigation:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` — the kitty code that all `file:line` citations refer to.
- **Destination delivery branch:** `blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9` — the branch this document is committed to. (The `go build` line shown verbatim in §1 — `/usr/local/go/bin/go build -v -ldflags '-X kitty.VCSRevision=<delivery-commit> -s -w' -o kitty/launcher/kitten` — embeds `-X kitty.VCSRevision=<this delivery commit>`, which is why the launcher's version string reflects the destination commit, not the source commit **(inferred from the `-ldflags` shown; not separately observed at runtime)**.)

The three compiled `*.so` are git-ignored generated artifacts (`.gitignore:1` = `*.so`). **All three** — `kitty/fast_data_types.so`, `kittens/transfer/rsync.so`, and `kitty/glfw-x11.so` — were temporarily moved during §5 (one at a time), each with its SHA-256 and mode verified before the move and again after restoration (see the per-cascade restoration-proof blocks in §5.1–§5.3), and all three were restored **byte-for-byte**; the SHA-256 values below match the post-build baseline. The captured working-tree state at delivery, the restored-artifact hashes, and the confirmation that all `/tmp` observation scratch was removed:

```console
# Repository state at delivery (destination branch), captured with the commands shown.

$ git rev-parse --abbrev-ref HEAD          # destination delivery branch
blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9

$ git log -1 --format="%H  %s"             # source commit under investigation is 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1;
                                            # the delivery commit that adds this document is created on the branch above
c732e9295e99e1d7091953f9317997b35b7bc5aa  docs: add kitty C-extension vs test-suite QnA investigation

$ git status --porcelain                    # only the answer document is added/changed; no source file is touched
 M blitzy/documentation/kitty_815df1e210e0.md

$ sha256sum kitty/fast_data_types.so kitty/glfw-x11.so kittens/transfer/rsync.so   # git-ignored artifacts, restored byte-for-byte (== post-build baseline)
a3f7ec88487e1ab7b9ecba83415c5abcaf4011b68ca87db326e6e45e8ba62c11  kitty/fast_data_types.so
99db5778b34c5370637a2fce743fd6379ebb879f9fbee34bb38f67f0cbd83a56  kitty/glfw-x11.so
c0bf7b038558d5dfabd176602d7a05458459bc635ee8dd869e1125695dea35a8  kittens/transfer/rsync.so

$ ls -d /tmp/blitzy_qa_obs.* /tmp/blitzy_qa_stash.* 2>/dev/null || echo "no temporary observation scratch remains under /tmp"
no temporary observation scratch remains under /tmp
```

> **Extension-hash provenance note.** Of the three restored-artifact hashes above, `kittens/transfer/rsync.so` (`c0bf7b03…`) and `kitty/glfw-x11.so` (`99db5778…`) are **commit-independent** — they rebuild byte-for-byte from any checkout with this toolchain (both were reproduced identically during validation). `kitty/fast_data_types.so` (`a3f7ec88…`, shown here and in the §5.1 restoration-proof block) is **not**: its `kitty/data-types.c` object is compiled with `-DKITTY_VCS_REV="<git HEAD>"`. `get_source_specific_defines()` (`setup.py:720-726`) injects that define for `kitty/data-types.c` (`setup.py:723`), and its value comes from `get_vcs_rev()` (`setup.py:674`, which runs `git rev-parse HEAD` at `setup.py:678`) — so the **current HEAD commit is embedded into `fast_data_types.so`** (observed: the `--verbose` build log emits `-DKITTY_VCS_REV="…"` on the `data-types.c` compile line, and `strings kitty/fast_data_types.so` contains the HEAD hash). Its SHA-256 is therefore **authoring-time-specific**: the `a3f7ec88…` shown was captured while this document was being written; a rebuild at the (necessarily different) delivery commit yields a **different** `fast_data_types.so` hash of the **same size** (`1,213,072` bytes) — precisely analogous to the launcher's embedded `VCSRevision` masked as `<delivery-commit>` (§1 and the delivery-branch bullet above). This does **not** weaken the §5.1 **integrity** claim: "restored byte-for-byte" means the before-hash equals the after-hash *within* each cascade run (the harness only moves the artifact out and back), which holds for whatever `fast_data_types.so` is on disk regardless of the embedded revision.

> **Self-reference note.** The `git log -1` and `git status --porcelain` lines above are the *authoring-time* snapshot captured while this document was being written — hence `git status` still shows the answer document as modified (` M`), and `git log -1` shows the commit that first added it. The commit that ultimately delivers this document (including this correction of it) necessarily carries a **different** hash from any shown here, because a document cannot embed the hash of the commit that adds it. After that commit the working tree is clean and `git diff 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1..HEAD --name-status` reports exactly one entry — `A blitzy/documentation/kitty_815df1e210e0.md` — confirming the read-only mandate: only the answer document is added; no existing source file is modified.

### Coverage of the nine sub-questions

| # | Sub-question | Where answered | Key observed evidence |
|---|--------------|----------------|-----------------------|
| 1 | Build kitty from source (canonical) | §1, §1.3 | canonical `python3 setup.py build --verbose` (gcc 15.2.0) + on-disk `CC=gcc-13` variant, both exit 0; gcc-13 `BUILD_REAL_SECONDS=61.630`/`61.590` (×2 runs); 3 artifacts on disk |
| 2 | Execute the test suite via its real entry point | §2.1, §2.2 | `./test.py` → `Ran 145 tests`, `FAILED (failures=4, skipped=4)`, `All Go tests succeeded` (×2 runs) |
| 3 | Trace the extension ↔ test-execution relationship | §3, §9 | root/discovery/native three-level gating; captured import trace |
| 4 | Which extension modules actually get loaded | §4, §4.1 | bootstrap=1 kitty ext; post-discovery 9 `.so` (2 kitty + 7 stdlib); glfw loaded natively (maps `False`→`True`) |
| 5 | How failures cascade when extensions are unavailable | §5.1–§5.3 | fast_data_types→0 tests; rsync→0 Python tests; glfw→2 tests break |
| 6 | What the output reveals about the dependency structure | §6 | one hard root, one discovery chokepoint, one native leaf; stage encodes coupling |
| 7 | Map extensions → test categories | §7.1, §7.2 | 22 modules; per-extension consumption table |
| 8 | Classify each extension CRITICAL vs OPTIONAL | §8 | fast_data_types & rsync CRITICAL; glfw-x11 OPTIONAL (native lib) |
| 9 | Document the actual import chains | §9.1, §9.2 | two real Python-extension chains + the native GLFW `ctypes` chain |

Named items explicitly covered: the three compiled extensions (`fast_data_types.so`, `glfw-x11.so`, `rsync.so`); the real entry point (`./test.py` → `kitty_tests.main`); `find_all_tests()`; the `itertests()` guard; `check_build` (`test_loading_extensions`, `test_loading_shaders`, `test_glfw_modules`); `file_transmission`; the `glfw.test_utf_8_strndup` native load; CI-vs-non-CI `test_glfw_modules`; and the environmental failures/skips.
