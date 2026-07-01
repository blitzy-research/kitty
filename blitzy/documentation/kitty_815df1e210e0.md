# Compiled C Extensions and the Test-Execution Flow in `kitty`

**Repository:** kovidgoyal/kitty
**Commit:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
**Branch:** `kitty_815df1e210e0`

---

## 0. Title, Scope & Methodology

This document answers a single technical question about the `kitty` terminal emulator:

> Build the project from source; execute the test suite; trace the relationship between the compiled C
> extensions and test execution; observe which extension modules actually load during test execution;
> observe how test failures cascade when those modules are unavailable; determine what the test output
> reveals about the dependency structure; map how compiled extensions connect to different test categories
> (e.g., core data types, transfer, windowing); identify which modules are critical vs. optional; and trace
> the actual import chains established during the test run.

**Methodology — investigate by running first, then write.** Every behavioral claim below is grounded in
output I captured by *actually building and running* the code in this environment — the build, the full test
suite, instrumented import traces, and controlled destructive experiments performed inside an isolated copy
of the tree. Each claim is paired with the command that produced it, the verbatim output line, and an exact
`file:line` citation. Where my observed numbers differ from any prior reference (timings, failure/skip
counts), I report **my own** observed values and note the difference explicitly.

**Environment (observed).** Python `3.13.7`, Go `go1.22.12`, gcc `15.2.0` (`gcc --version`), running on a
headless Linux container. The suite was launched with `LANG=C.UTF-8`/`LC_ALL=C.UTF-8`.

**Read-only guarantee.** The source tree was treated as strictly read-only. The only file created is this
document. All destructive experiments were performed in an isolated copy at `/tmp/kitty_iso`; the build
produces only git-ignored artifacts. After cleanup, `git status --porcelain` on the real tree is empty
(proof in §9).

**Naming convention used below.** "C extension" / "compiled extension" means a `.so` shared object produced
by the build. The kitty-specific ones are `kitty/fast_data_types.so`, `kittens/transfer/rsync.so`,
`kitty/glfw-x11.so`, and `kitty/glfw-wayland.so`.

---

## 1. Building from Source

### 1.1 The default build fails under `-Werror`

kitty's build treats compiler warnings as errors by default. The switch is a single expression in
`setup.py`:

```
werror = '' if ignore_compiler_warnings else '-pedantic-errors -Werror'
```
— `setup.py:491`

Running the default build from a clean tree (`python3 setup.py clean` first) fails while compiling the
Wayland windowing backend. The full build output was redirected to `build_default.log`; the lines below are
the **complete, verbatim** output of the `sed` extraction command shown (an explicit line-range slice of that
log — no elision inside it):

```console
$ python3 setup.py build > build_default.log 2>&1 ; echo "exit=$?"
exit=1
$ sed -n '/wl_window.c: In function/,/being treated as errors/p' build_default.log
glfw/wl_window.c: In function ‘xdgToplevelHandleConfigure’:
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_LEFT’ not handled in switch [-Werror=switch]
  668 |         switch (*state) {
      |         ^~~~~~
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_RIGHT’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_TOP’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_BOTTOM’ not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
```

**Claim:** the failure is caused by the host's newer `wayland-protocols` introducing
`XDG_TOPLEVEL_STATE_CONSTRAINED_*` enum values that this commit's `switch` does not handle, promoted to a
hard error by `-pedantic-errors -Werror` (`setup.py:491`). **Evidence:** the four
`error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_{LEFT,RIGHT,TOP,BOTTOM}’ not handled in switch [-Werror=switch]`
lines above — each carrying the literal `[-Werror=switch]` suffix that proves `-Werror` is active — followed
by `cc1: all warnings being treated as errors`, with the build exiting `exit=1`.

### 1.2 The `--ignore-compiler-warnings` flag produces a clean build

kitty ships an in-tree flag for exactly this situation. It is declared in the argument parser:

```
'--ignore-compiler-warnings',
default=Options.ignore_compiler_warnings, action='store_true',
```
— `setup.py:2003-2004`

Re-running with that flag succeeds. This flag is a **legitimate, in-tree build option — not a source
edit**; no product file was modified to obtain a clean build.

The full build output was redirected to `build_ignore.log`; the block below is the **complete, verbatim**
output of the `sed` extraction command shown (the linking phase of that log):

```console
$ python3 setup.py build --ignore-compiler-warnings > build_ignore.log 2>&1 ; echo "exit=$?"
exit=0
$ sed -n '/^\[1\/5\] Linking/,/^ done/p' build_ignore.log
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
```

**Claim:** the flag disables `-Werror` (making `werror` the empty string at `setup.py:491`) and the build
links cleanly with exit `0`. **Evidence:** the `[1/5] … [5/5] Linking … done` block above and `exit=0`.

### 1.3 Artifacts produced

```console
$ ls -la kitty/fast_data_types.so kittens/transfer/rsync.so kitty/glfw-x11.so kitty/glfw-wayland.so kitty/launcher/kitten kitty/launcher/kitty
-rwxr-xr-x 1 root root    42824 Jul  1 22:06 kittens/transfer/rsync.so
-rwxr-xr-x 1 root root  1253792 Jul  1 22:06 kitty/fast_data_types.so
-rwxr-xr-x 1 root root   451016 Jul  1 22:06 kitty/glfw-wayland.so
-rwxr-xr-x 1 root root   373896 Jul  1 22:06 kitty/glfw-x11.so
-rwxr-xr-x 1 root root 15765764 Jul  1 22:07 kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 Jul  1 22:06 kitty/launcher/kitty
```

The build produces four kitty `.so` extensions plus the Go `kitten` and `kitty` launcher binaries. All are
**git-ignored build outputs**, not tracked source. The ignore rules include `*.so` at `.gitignore:1` and
`/kitty/launcher/kitt*`:

```console
$ git check-ignore kitty/fast_data_types.so kittens/transfer/rsync.so kitty/glfw-x11.so kitty/glfw-wayland.so kitty/launcher/kitten kitty/launcher/kitty
kitty/fast_data_types.so
kittens/transfer/rsync.so
kitty/glfw-x11.so
kitty/glfw-wayland.so
kitty/launcher/kitten
kitty/launcher/kitty
```

**Claim:** none of the six artifacts is tracked source. **Evidence:** `git check-ignore` echoes back all six
paths (a path is printed only when it is ignored).

### 1.4 Supporting build facts

- The `Makefile` is a thin wrapper over `setup.py`: `all` runs `python3 setup.py $(VVAL)` (`Makefile:13`),
  `test` runs `python3 setup.py $(VVAL) test` (`Makefile:16`), and `clean` runs `python3 setup.py $(VVAL)
  clean` (`Makefile:19`).
- The Python floor is `requires-python = ">=3.8"` (`pyproject.toml:2`); the Go baseline is `go 1.22`
  (`go.mod:3`). The extension build environment is assembled in `def kitty_env(args: Options) -> Env:`
  (`setup.py:598`).
- The canonical dependency list lives in `docs/build.rst`: run-time deps begin with `* ``python`` >= 3.8`
  (`docs/build.rst:83`) and `* ``harfbuzz`` >= 2.2.0` (`docs/build.rst:84`); build-time deps are headed
  `Build-time dependencies:` (`docs/build.rst:97`) and include `* ``gcc`` or ``clang``` (`docs/build.rst:99`),
  `* ``simde``` (`docs/build.rst:100`), `* ``go`` >= _build_go_version` (`docs/build.rst:101`), and
  `* ``pkg-config``` (`docs/build.rst:102`).

---

## 2. Baseline Test Run

### 2.1 How the suite is launched

The test entry point is executed *through the built launcher*. `test.py`'s shebang is
`#!./kitty/launcher/kitty +launch` (`test.py:1`); its `main()` imports the runner and calls it:

```
    m = importlib.import_module('kitty_tests.main')
    getattr(m, 'main')()
```
— `test.py:8-9`

`python setup.py test` routes to the same place — it re-executes the launcher via
`os.execl(texe, texe, '+launch', 'test.py')` (`setup.py:2101-2103`).

### 2.2 Verbatim runner output (my observed run)

The full runner output was redirected to `test_baseline.log`; the three lines below are the **complete,
verbatim** output of the `grep` command shown (the two `unittest` summary lines and the Go summary line):

```console
$ ./kitty/launcher/kitty +launch test.py > test_baseline.log 2>&1 ; echo "exit=$?"
exit=1
$ grep -E '^Ran [0-9]+ tests|^FAILED|All Go tests' test_baseline.log
Ran 145 tests in 27.964s
FAILED (failures=6, skipped=2)
All Go tests succeeded, ran in 28.4 seconds
```

- **Python `unittest` summary — Claim:** the suite ran at representative scale — 145 tests. **Evidence:**
  `Ran 145 tests in 27.964s`.
- **Python result line — Claim:** the run reported 6 failures and 2 skips. **Evidence:**
  `FAILED (failures=6, skipped=2)`.
- **Go summary — Claim:** all Go tests passed. **Evidence:** `All Go tests succeeded, ran in 28.4 seconds`,
  printed by `kitty_tests/main.py:216`.

> **Difference from prior reference values.** A prior reference recorded `Ran 145 tests in 13.642s`,
> `FAILED (failures=2, skipped=6)`, and `14.1 seconds`. My run has the **same test count (145)** but a
> different failure/skip split (6 failures / 2 skips) and longer timings (`27.964s` / `28.4 seconds`). Both
> differences are explained by environment, below — and neither involves compiled-extension loading.

### 2.3 The 6 failures (verbatim, and why they are unrelated to extensions)

Two failures are file-mode assertions in the transfer tests:

```console
$ grep -E '^FAIL: test_transfer' test_baseline.log
FAIL: test_transfer_receive (kitty_tests.file_transmission.TestFileTransmission.test_transfer_receive)
FAIL: test_transfer_send (kitty_tests.file_transmission.TestFileTransmission.test_transfer_send)
```

The assertion that fails is `self.assertEqual(expected, actual)` at `kitty_tests/file_transmission.py:432`.
The `AssertionError` carries `unittest`'s unified diff of the two directory listings (`-` lines are the
`expected` values, `+` lines are the `actual` values); it isolates the difference to exactly two directory
entries. The block below is the **complete, verbatim** output of the `grep` command shown (the four differing
diff lines from the `test_transfer_receive` failure):

```console
$ sed -n '/^FAIL: test_transfer_receive/,/^====/p' test_baseline.log | grep -E "^[-+]  '(empty|sub)':"
-  'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2),
+  'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2),
-  'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2),
+  'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2),
```

**Claim:** these are environment-specific file-mode assertions, **not** extension-loading problems.
**Evidence:** the diff is purely `mode='0o42755'` vs `mode='0o40755'` — the two values differ only in the
`0o2000` set-gid permission bit (`S_ISGID`), which is set in `0o42755` but not in `0o40755`
(`0o42755 - 0o40755 = 0o2000`); the `0o40000` bit is the directory file-type bit (`S_IFDIR`) common to **both**
values, not the differing bit. The assertion is raised at `file_transmission.py:432`.

The other four failures are font-selection mismatches:

```console
$ grep -E '^FAIL: test_font_selection' test_baseline.log
FAIL: test_font_selection (kitty_tests.fonts.Selection.test_font_selection) (spec='fira code')
FAIL: test_font_selection (kitty_tests.fonts.Selection.test_font_selection) (spec='family="fira code"')
FAIL: test_font_selection (kitty_tests.fonts.Selection.test_font_selection) (spec='ubuntu mono')
FAIL: test_font_selection (kitty_tests.fonts.Selection.test_font_selection) (spec='family="ubuntu mono"')
```

The assertion is `self.ae(expected, actual)` at `kitty_tests/fonts.py:40`. Because `runner.tb_locals = True`
(`kitty_tests/main.py:118`), the traceback prints the full `expected` and `actual` tuples, which show the
installed font reports different internal names than the test expects. The two lines below are the
**complete, verbatim** output of the `sed`/`grep` command shown (the `actual` and `expected` locals captured
from the `spec='fira code'` failure):

```console
$ sed -n "/^FAIL: test_font_selection.*spec='fira code')$/,/^====/p" test_baseline.log | grep -E "^    (expected|actual) = "
    actual = ('FiraCode-Regular', 'FiraCode-SemiBold', 'FiraCode-Retina', 'FiraCode-SemiBold')
    expected = ('FiraCodeRoman-Regular', 'FiraCodeRoman-SemiBold', 'FiraCodeRoman-Regular', 'FiraCodeRoman-SemiBold')
```

**Claim:** these are environment-specific font-availability/naming differences, **not** extension-loading
problems. **Evidence:** `expected` is
`('FiraCodeRoman-Regular', 'FiraCodeRoman-SemiBold', 'FiraCodeRoman-Regular', 'FiraCodeRoman-SemiBold')` while
`actual` is `('FiraCode-Regular', 'FiraCode-SemiBold', 'FiraCode-Retina', 'FiraCode-SemiBold')`, raised at
`fonts.py:40`.

### 2.4 The 2 skips (verbatim reasons)

```console
$ grep "skipped '" test_baseline.log
test_ca_certificates (kitty_tests.check_build.TestBuild.test_ca_certificates) ... skipped 'CA certificates are only tested on frozen builds'
test_fallback_font_not_last_resort (kitty_tests.fonts.Rendering.test_fallback_font_not_last_resort) ... skipped 'Only macOS has a Last Resort font'
```

- **Claim:** the CA-certificates test is a frozen-build-only test. **Evidence:**
  `skipped 'CA certificates are only tested on frozen builds'`.
- **Claim:** the Last-Resort font test is macOS-only. **Evidence:**
  `skipped 'Only macOS has a Last Resort font'`.

**Why my skip count (2) is lower than the reference (6).** The prior reference environment lacked the `fish`
and `zsh` shells, so their shell-integration tests skipped. Here, both shells are installed, so those tests
*run* rather than skip:

```console
$ command -v fish zsh
/usr/bin/fish
/usr/bin/zsh
$ grep -E 'test_(bash|fish|zsh)_integration' test_baseline.log
test_bash_integration (kitty_tests.shell_integration.ShellIntegration.test_bash_integration) ... ok
test_fish_integration (kitty_tests.shell_integration.ShellIntegration.test_fish_integration) ... ok
test_zsh_integration (kitty_tests.shell_integration.ShellIntegration.test_zsh_integration) ... ok
test_bash_integration (kitty_tests.shell_integration.ShellIntegrationWithKitten.test_bash_integration) ... ok
test_fish_integration (kitty_tests.shell_integration.ShellIntegrationWithKitten.test_fish_integration) ... ok
test_zsh_integration (kitty_tests.shell_integration.ShellIntegrationWithKitten.test_zsh_integration) ... ok
```

**Claim:** the lower skip count is because `fish`/`zsh` are present and their integration tests ran `ok`.
**Evidence:** `/usr/bin/fish` and `/usr/bin/zsh` exist, and both `test_fish_integration` and
`test_zsh_integration` report the `... ok` status marker (shown verbatim in the block above).

**Bottom line for §2:** the suite fully collected and ran **145** tests, which by itself proves that both
required compiled modules (`fast_data_types`, `rsync`) loaded — no failure or skip relates to extension
loading.

---

## 3. The Extension ↔ Test Binding, and Which Modules Actually Load

### 3.1 The concrete binding code path

The test framework binds to the compiled C core at *module-import time*. The base test-infrastructure
package `kitty_tests/__init__.py` imports the C core directly at its top level:

```
from kitty.fast_data_types import Cursor, HistoryBuf, LineBuf, Screen, get_options, monotonic, set_options
```
— `kitty_tests/__init__.py:22`

Every test class ultimately derives from the `BaseTest` defined in that module, so importing *any* test
module pulls in `kitty_tests/__init__.py`, which requires `kitty.fast_data_types` to already be importable.
That single line is the concrete binding between the `unittest`-based test framework and the compiled
extension.

### 3.2 The authoritative in-repo assertion of what must load

kitty encodes its own definition of "the compiled modules must load" in a dedicated build-verification test:

```
    def test_loading_extensions(self) -> None:
        import kitty.fast_data_types as fdt
        from kittens.transfer import rsync
        del fdt, rsync
```
— `kitty_tests/check_build.py:28-31`

**Claim:** the two kitty-specific compiled Python modules the project expects to load are
`kitty.fast_data_types` and `kittens.transfer.rsync`. **Evidence:** `test_loading_extensions` imports exactly
those two (`import kitty.fast_data_types as fdt`; `from kittens.transfer import rsync`) at
`check_build.py:29-30`.

### 3.3 Enumerating the `.so` modules that actually load (not the theoretically-available ones)

I instrumented the import machinery with a temporary `sys.meta_path` finder (a `blitzy_adhoc_test_*` script,
since removed) and ran the runner's real discovery step, `find_all_tests()`, then listed every `.so`-backed
module in `sys.modules`:

```console
$ ./kitty/launcher/kitty +launch blitzy_adhoc_test_discovery_trace.py > discovery_trace.log 2>&1
$ sed -n '/loaded after discovery/,$p' discovery_trace.log
=== Kitty-specific compiled extension modules loaded after discovery ===
  kittens.transfer.rsync -> kittens/transfer/rsync.so
  kitty.fast_data_types -> kitty/fast_data_types.so

TOTAL kitty-specific .so modules after full discovery: 2
```

**Claim:** across the full test-discovery pass, exactly **two** kitty-specific compiled modules load —
`kitty.fast_data_types` (→ `kitty/fast_data_types.so`) and `kittens.transfer.rsync`
(→ `kittens/transfer/rsync.so`). **Evidence:** the two-line enumeration above and
`TOTAL kitty-specific .so modules after full discovery: 2`.

A subtle but important distinction I observed: during a *plain* `import kitty_tests` (before any test module
is discovered) only **one** kitty-specific extension is present — `kitty.fast_data_types`. The other three
`.so` in `sys.modules` at that point are Python standard-library extensions, not kitty's:

```console
$ ./kitty/launcher/kitty +launch blitzy_adhoc_test_import_trace.py > import_trace.log 2>&1
$ grep -A4 'present in sys.modules' import_trace.log
=== Compiled extension modules (.so) present in sys.modules ===
  _bz2 -> /usr/lib/python3.13/lib-dynload/_bz2.cpython-313-x86_64-linux-gnu.so
  _lzma -> /usr/lib/python3.13/lib-dynload/_lzma.cpython-313-x86_64-linux-gnu.so
  kitty.fast_data_types -> kitty/fast_data_types.so
  termios -> /usr/lib/python3.13/lib-dynload/termios.cpython-313-x86_64-linux-gnu.so
```

**Claim:** at base-package import, the only kitty-specific compiled module loaded is `kitty.fast_data_types`;
`kittens.transfer.rsync` has not loaded yet. **Evidence:** the list shows `kitty.fast_data_types` alongside
only stdlib extensions `_bz2`, `_lzma`, `termios` — no `rsync`. (This split is what separates Tier 1 from
Tier 2 in §5.)

### 3.4 The GLFW backends are *not* Python-imported

The instrumentation explicitly reported that no GLFW backend is imported as a Python module:

```console
$ grep 'GLFW backend modules' import_trace.log
GLFW backend modules python-imported into sys.modules: []
```

Instead, the GLFW `.so` paths are *resolved as filesystem paths* by `glfw_path()` and loaded on demand (by
the C layer, or via `ctypes.CDLL` in one specific test):

```
def glfw_path(module: str) -> str:
    prefix = 'kitty.' if getattr(sys, 'frozen', False) else ''
    return os.path.join(extensions_dir, f'{prefix}glfw-{module}.so')
```
— `kitty/constants.py:191-193`

**Claim:** `glfw-x11.so` and `glfw-wayland.so` never enter `sys.modules` as Python modules; they are located
by path via `glfw_path()` and opened on demand. **Evidence:** the empty
`GLFW backend modules python-imported into sys.modules: []` list, and `glfw_path()` returning
`os.path.join(extensions_dir, f'{prefix}glfw-{module}.so')` at `kitty/constants.py:193`.

---

## 4. The Actual Import Chains Established During the Test Run

### 4.1 Primary chain — the C core loads *before* the explicit import

The explicit `from kitty.fast_data_types import …` sits at `kitty_tests/__init__.py:22`, but the C core is
actually loaded *earlier*, by the line above it. I confirmed this by tracing the **first** `find_spec`
request for `kitty.fast_data_types` and printing the import stack at that instant:

```console
$ awk '/FIRST import request for: kitty.fast_data_types/{f=1} f&&/FIRST import request for: kitty.window/{exit} f' import_trace.log
=== FIRST import request for: kitty.fast_data_types ===
  __main__.py:7:  main()
  kitty/entry_points.py:192:  namespaced(['+', first_arg[1:]] + sys.argv[2:])
  kitty/entry_points.py:146:  func(args[1:])
  kitty/entry_points.py:73:  runpy.run_path(exe, run_name='__main__')
  kitty_tests/__init__.py:21:  from kitty.config import finalize_keys, finalize_mouse_mappings
  kitty/config.py:10:  from .conf.utils import BadLine, parse_config_base
  kitty/conf/utils.py:27:  from ..fast_data_types import Color
```

**Claim:** the true first load of `kitty.fast_data_types` is triggered by the *preceding* line
`kitty_tests/__init__.py:21` (`from kitty.config import …`) → `kitty/config.py:10`
(`from .conf.utils import …`) → `kitty/conf/utils.py:27` (`from ..fast_data_types import Color`) — i.e.
**before** the explicit import at `kitty_tests/__init__.py:22`. **Evidence:** the trace's terminal frame is
`kitty/conf/utils.py:27: from ..fast_data_types import Color`, reached via `config.py:10` from
`__init__.py:21`.

Note the *first absolute* import of the core is `from kitty.fast_data_types import Color, SingleKey`
(`kitty/options/types.py:8`), reached via `from .options.types import …` at `kitty/config.py:13` — but that
runs *after* the relative import at `config.py:10`, so it is not the first load.

### 4.2 Secondary chain — via the windowing subsystem

The base package pulls in the window subsystem a few lines later, forming a second real path to the C core.
Tracing the first load of `kitty.window` and `kitty.child`:

```console
$ awk '/FIRST import request for: kitty.window/{f=1} /^$/{if(f)exit} f' import_trace.log
=== FIRST import request for: kitty.window ===
  kitty_tests/__init__.py:27:  from kitty.window import decode_cmdline, process_remote_print, process_title_from_child
=== FIRST import request for: kitty.child ===
  kitty_tests/__init__.py:27:  from kitty.window import decode_cmdline, process_remote_print, process_title_from_child
  kitty/window.py:33:  from .child import ProcessDesc
```

**Claim:** a second import path runs `kitty_tests/__init__.py:27` → `kitty/window.py:33`
(`from .child import ProcessDesc`) → `kitty/child.py`, which does `import kitty.fast_data_types as
fast_data_types` (`kitty/child.py:11`). **Evidence:** the trace shows `kitty.child` first requested from
`kitty/window.py:33`, itself requested from `kitty_tests/__init__.py:27`. (By the time this path runs, the
core is already cached from §4.1, so `child.py:11` resolves from cache.)

### 4.3 The `rsync` extension loads during discovery, not base import

The transfer extension enters via a *different* trigger — the runner's eager discovery loop importing the
`file_transmission` test module:

```console
$ awk '/FIRST import request for: kittens.transfer.rsync/{f=1} /^$/{if(f)exit} f' discovery_trace.log
=== FIRST import request for: kittens.transfer.rsync ===
  kitty_tests/main.py:64:  m = importlib.import_module(package + '.' + x.partition('.')[0])
  kitty_tests/file_transmission.py:13:  from kittens.transfer.rsync import Differ, Hasher, Patcher, parse_ftc
```

**Claim:** `kittens.transfer.rsync` first loads when eager discovery (`kitty_tests/main.py:64`) imports
`kitty_tests/file_transmission.py`, whose module-scope line `from kittens.transfer.rsync import Differ,
Hasher, Patcher, parse_ftc` (`file_transmission.py:13`) requires it. **Evidence:** the trace's terminal
frame is `file_transmission.py:13`, reached from the eager `importlib.import_module(...)` at `main.py:64`.

---

## 5. Failure Cascade & the Three-Tier Criticality Gradient

All experiments in this section were run inside an isolated copy at `/tmp/kitty_iso` (see §9). Each removed
a compiled `.so` and re-ran `./kitty/launcher/kitty +launch test.py`.

### 5.1 Tier 1 — `kitty/fast_data_types.so`: critical (total failure at base-package import)

```console
$ mv kitty/fast_data_types.so /tmp/kitty_iso_fast_data_types.so.bak
$ ./kitty/launcher/kitty +launch test.py > tier1.log 2>&1 ; echo "exit=$?"
exit=1
$ tail -n 7 tier1.log
  File "/tmp/kitty_iso/kitty/launcher/../../kitty_tests/__init__.py", line 21, in <module>
    from kitty.config import finalize_keys, finalize_mouse_mappings
  File "/tmp/kitty_iso/kitty/launcher/../../kitty/config.py", line 10, in <module>
    from .conf.utils import BadLine, parse_config_base
  File "/tmp/kitty_iso/kitty/launcher/../../kitty/conf/utils.py", line 27, in <module>
    from ..fast_data_types import Color
ModuleNotFoundError: No module named 'kitty.fast_data_types'
$ grep -c '^Ran [0-9]* tests' tier1.log
0
```

**Claim:** removing `fast_data_types.so` aborts the run at base-package import, raising
`ModuleNotFoundError: No module named 'kitty.fast_data_types'` at `kitty/conf/utils.py:27`, and **zero** tests
run. **Evidence:** the `ModuleNotFoundError` terminates at `conf/utils.py:27: from ..fast_data_types import
Color`; no `Ran N tests` line is ever printed (a `grep -c '^Ran [0-9]* tests'` of the log returns `0`). The
failing chain matches the primary import chain from §4.1 exactly.

### 5.2 Tier 2 — `kittens/transfer/rsync.so`: collection-critical (total failure during discovery)

The full Go-package line is shown verbatim via `grep`; the traceback is then shown via a filter pipeline that
strips only the Python-internal `<frozen importlib._bootstrap>` / `importlib/__init__.py` frames and the
`~~~`/`^^^` caret lines for readability — the lines printed below are the **complete, verbatim** output of
each command shown (real `/tmp/kitty_iso/...` paths, no elision):

```console
$ mv kittens/transfer/rsync.so /tmp/kitty_iso_rsync.so.bak
$ ./kitty/launcher/kitty +launch test.py > tier2.log 2>&1 ; echo "exit=$?"
exit=1
$ grep '^Go packages being tested:' tier2.log
Go packages being tested: tools/cmd/at tools/utils/humanize tools/tui/graphics tools/utils/style tools/tui/readline tools/cli tools/simdstring tools/config tools/wcswidth tools/utils/shlex tools/tui/loop tools/tui/sgr tools/tui kittens/hints kittens/ssh tools/themes kittens/transfer tools/utils/shm tools/rsync tools/utils/base85 tools/unicode_names tools/tui/subseq tools/utils tools/tui/shell_integration kittens/diff kittens/hyperlinked_grep
$ grep -vE '<frozen|importlib/__init__\.py|_bootstrap|~~~|\^\^\^' tier2.log | sed -n '/^Traceback/,/^ModuleNotFoundError/p'
Traceback (most recent call last):
  File "/tmp/kitty_iso/kitty/launcher/../../__main__.py", line 7, in <module>
    main()
  File "/tmp/kitty_iso/kitty/launcher/../../kitty/entry_points.py", line 192, in main
    namespaced(['+', first_arg[1:]] + sys.argv[2:])
  File "/tmp/kitty_iso/kitty/launcher/../../kitty/entry_points.py", line 146, in namespaced
    func(args[1:])
  File "/tmp/kitty_iso/kitty/launcher/../../kitty/entry_points.py", line 73, in launch
    runpy.run_path(exe, run_name='__main__')
  File "test.py", line 13, in <module>
    main()
  File "test.py", line 9, in main
    getattr(m, 'main')()
  File "/tmp/kitty_iso/kitty/launcher/../../kitty_tests/main.py", line 338, in main
    run_tests()
  File "/tmp/kitty_iso/kitty/launcher/../../kitty_tests/main.py", line 279, in run_tests
    run_python_tests(args, go_proc)
  File "/tmp/kitty_iso/kitty/launcher/../../kitty_tests/main.py", line 211, in run_python_tests
    tests = find_all_tests()
  File "/tmp/kitty_iso/kitty/launcher/../../kitty_tests/main.py", line 64, in find_all_tests
    m = importlib.import_module(package + '.' + x.partition('.')[0])
  File "/tmp/kitty_iso/kitty/launcher/../../kitty_tests/file_transmission.py", line 13, in <module>
    from kittens.transfer.rsync import Differ, Hasher, Patcher, parse_ftc
ModuleNotFoundError: No module named 'kittens.transfer.rsync'
$ grep -c '^Ran [0-9]* tests' tier2.log
0
```

**Claim:** removing `rsync.so` lets the base package import, but the run still aborts with **zero** tests —
eager discovery `find_all_tests()` fails at the eager `importlib.import_module(...)` (`main.py:64`) while
importing `file_transmission.py`, whose `file_transmission.py:13` import cannot be satisfied. **Evidence:**
`ModuleNotFoundError: No module named 'kittens.transfer.rsync'` terminating at `file_transmission.py:13`,
reached through `main.py:64`; again no `Ran N tests` line appears. The traceback also confirms the runner's
call chain: `main()` (defined `main.py:334`) calls `run_tests()` at `main.py:338`; `run_tests()` (defined
`main.py:246`) calls `run_python_tests(...)` at `main.py:279`; `run_python_tests()` (defined `main.py:210`)
calls `find_all_tests()` at `main.py:211`.

### 5.3 Tier 3 — GLFW backends: optional/platform (localized failure only)

```console
$ mv kitty/glfw-x11.so /tmp/kitty_iso_glfw-x11.so.bak
$ mv kitty/glfw-wayland.so /tmp/kitty_iso_glfw-wayland.so.bak
$ ./kitty/launcher/kitty +launch test.py > tier3.log 2>&1 ; echo "exit=$?"
exit=1
$ grep -E '^Ran [0-9]+ tests|^FAILED|All Go tests' tier3.log
Ran 145 tests in 15.476s
FAILED (failures=7, errors=1, skipped=2)
All Go tests succeeded, ran in 15.5 seconds
```

Unlike Tiers 1–2, the suite **still runs to completion**. Compared to the §2 baseline
(`failures=6, skipped=2`), Tier 3 adds exactly one error and one failure — both GLFW-specific.

The added **error** is the on-demand `ctypes` load in the GLFW test. The block below is the **complete,
verbatim** output of the bounded `awk`/`grep` command shown (it selects the `test_utf_8_strndup` error section
and keeps the header, the failing frame, its captured locals, and the `OSError` line — the intervening
Python-internal frames are dropped by the `grep` filter; real `/tmp/kitty_iso/...` paths, no elision):

```console
$ awk '/^ERROR: test_utf_8_strndup/{f=1} f; /^OSError:/{if(f)exit}' tier3.log \
    | grep -E '^ERROR: test_utf_8_strndup|glfw\.py", line 50|^    lib = ctypes\.CDLL|^    backend_utils =|^OSError:'
ERROR: test_utf_8_strndup (kitty_tests.glfw.TestGLFW.test_utf_8_strndup)
  File "/tmp/kitty_iso/kitty/launcher/../../kitty_tests/glfw.py", line 50, in test_utf_8_strndup
    lib = ctypes.CDLL(backend_utils)
    backend_utils = '/tmp/kitty_iso/kitty/glfw-x11.so'
OSError: /tmp/kitty_iso/kitty/glfw-x11.so: cannot open shared object file: No such file or directory
```

**Claim:** with the GLFW backends gone, `test_utf_8_strndup` errors when it tries to `dlopen` the backend by
path. **Evidence:** `OSError: /tmp/kitty_iso/kitty/glfw-x11.so: cannot open shared object file: No such file
or directory`, raised at `kitty_tests/glfw.py:50` (`lib = ctypes.CDLL(backend_utils)`), where `backend_utils`
was assigned from `glfw_path('x11')` at `kitty_tests/glfw.py:49`.

The added **failure** is the build-verification check. The block below is the **complete, verbatim** output of
the bounded `awk`/`grep` command shown (same filtering approach — header, failing frame, captured locals, and
the `AssertionError` line; real `/tmp/kitty_iso/...` paths, no elision):

```console
$ awk '/^FAIL: test_glfw_modules/{f=1} f; /^AssertionError:/{if(f)exit}' tier3.log \
    | grep -E '^FAIL: test_glfw_modules|check_build\.py", line 46|^    self\.assertTrue|^    modules = |^AssertionError:'
FAIL: test_glfw_modules (kitty_tests.check_build.TestBuild.test_glfw_modules)
  File "/tmp/kitty_iso/kitty/launcher/../../kitty_tests/check_build.py", line 46, in test_glfw_modules
    self.assertTrue(os.path.isfile(path), f'{path} is not a file')
    modules = ['x11', 'wayland']
AssertionError: False is not true : /tmp/kitty_iso/kitty/glfw-x11.so is not a file
```

**Claim:** `test_glfw_modules` fails because the expected backend file is absent. **Evidence:**
`AssertionError: False is not true : /tmp/kitty_iso/kitty/glfw-x11.so is not a file`, raised at
`kitty_tests/check_build.py:46` (`self.assertTrue(os.path.isfile(path), f'{path} is not a file')`) with
`modules = ['x11', 'wayland']`.

**Claim:** the GLFW backends are optional to the run as a whole — their absence leaves all 145 tests
collected and running, breaking only the two GLFW-specific tests. **Evidence:** `Ran 145 tests in 15.476s`
with `FAILED (failures=7, errors=1, skipped=2)` — exactly one more failure and one more error than the
baseline's `failures=6, skipped=2`.

> **Difference from a prior reference value.** A prior reference stated "134 tests still run" for Tier 3. My
> observed run still *collects and runs* all **145** tests (`Ran 145 tests`), with only the two GLFW-specific
> tests failing/erroring. I report my own observed values.

---

## 6. What the Test Output Reveals About the Dependency Structure

The three tiers are not arbitrary — they are a direct consequence of **when** each extension is imported
relative to **eager test discovery**. The decisive mechanism is the discovery loop:

```
def find_all_tests(package: str = '', excludes: Sequence[str] = ('main', 'gr')) -> unittest.TestSuite:
```
— `kitty_tests/main.py:57`

```
            m = importlib.import_module(package + '.' + x.partition('.')[0])
```
— `kitty_tests/main.py:64`

**Claim:** discovery is *eager* — every test module (except `main` and `gr`) is imported up front, before a
single test executes. **Evidence:** `find_all_tests(... excludes=('main', 'gr') ...)` (`main.py:57`) loops
and calls `importlib.import_module(...)` for each module at `main.py:64`.

The consequence, read directly off the experiments:

- **A module-scope import of a compiled extension is a hard collection prerequisite.** If the extension is
  missing, the module fails to import during discovery, and because discovery is eager and unguarded, the
  *entire* run aborts with **zero** tests — even the hundreds of tests that have nothing to do with that
  extension. This is exactly Tier 1 (`fast_data_types` at `conf/utils.py:27`, reached from base import) and
  Tier 2 (`rsync` at `file_transmission.py:13`, reached from `main.py:64`).
- **An on-demand load inside a single test method is *not* a collection prerequisite.** The GLFW backend is
  opened by `ctypes.CDLL(...)` *inside* `test_utf_8_strndup` (`kitty_tests/glfw.py:50`), so its absence
  cannot break discovery; it can only fail that one test (plus the file-existence check in
  `test_glfw_modules`). This is Tier 3.

**What the output "reveals," stated plainly:** the pass/fail/skip signals show a dependency structure with a
sharp cliff. Either the two module-scope compiled dependencies are present — and then the suite runs in full
(145 tests, with only environment-specific failures/skips) — or one is missing and the suite produces **zero**
tests. There is no partial-collection middle ground for the module-scope extensions, precisely because
`find_all_tests()` imports everything eagerly (`main.py:64`).

---

## 7. Mapping Compiled Extensions to Test Categories

The question names three example categories — **core data types, transfer, windowing**. Each is covered
below, along with the other categories that bind the core, using the *actual* import sites.

| Test category | Test module(s) | Compiled extension | Binding site (import statement) |
|---|---|---|---|
| **Core data types** | `kitty_tests/screen.py`, `kitty_tests/datatypes.py`, `kitty_tests/keys.py` | `kitty.fast_data_types` | `from kitty.fast_data_types import DECAWM, DECCOLM, DECOM, IRM, VT_PARSER_BUFFER_SIZE, Cursor` (`screen.py:4`); `from kitty.fast_data_types import (Color, ColorProfile, HistoryBuf, LineBuf, expand_ansi_c_escapes, parse_input_from_terminal, replace_c0_codes_except_nl_space_tab, strip_csi, truncate_point_for_length, wcswidth, wcwidth)` (multi-line, `datatypes.py:9-21`); `import kitty.fast_data_types as defines` (`keys.py:6`) |
| **Parsing + SIMD** | `kitty_tests/parser.py` | `kitty.fast_data_types` | `from kitty.fast_data_types import (CURSOR_BLOCK, VT_PARSER_BUFFER_SIZE, base64_decode, base64_encode, has_avx2, has_sse4_2, test_find_either_of_two_bytes, test_utf8_decode_to_sentinel)` (multi-line, `parser.py:8-17`; SIMD symbols `has_avx2`, `has_sse4_2` at `parser.py:13-14`) |
| **Graphics protocol** | `kitty_tests/graphics.py` | `kitty.fast_data_types` | `from kitty.fast_data_types import base64_decode, base64_encode, has_avx2, has_sse4_2, load_png_data, shm_unlink, shm_write, test_xor64` (`graphics.py:14`) |
| **Crypto (ECDH / AES-256-GCM)** | `kitty_tests/crypto.py` | `kitty.fast_data_types` | `from kitty.fast_data_types import AES256GCMDecrypt, AES256GCMEncrypt, CryptoError, EllipticCurveKey` (`crypto.py:28`) — **method-scope**, see note |
| **Transfer** | `kitty_tests/file_transmission.py` | `kittens.transfer.rsync` | `from kittens.transfer.rsync import Differ, Hasher, Patcher, parse_ftc` (`file_transmission.py:13`) — **module-scope** |
| **Windowing (GLFW)** | `kitty_tests/glfw.py` | `glfw-x11.so` (on demand) | `backend_utils = glfw_path('x11')` (`glfw.py:49`); `lib = ctypes.CDLL(backend_utils)` (`glfw.py:50`) — **method-scope `dlopen`** |
| **Build verification** | `kitty_tests/check_build.py` | all of the above | `test_loading_extensions` (`check_build.py:28-31`); `test_loading_shaders` (`check_build.py:33-36`); `test_glfw_modules` (`check_build.py:38-47`) |

- **Core data types — Claim:** the core-datatype tests bind `kitty.fast_data_types` at module scope.
  **Evidence:** `from kitty.fast_data_types import DECAWM, DECCOLM, DECOM, IRM, VT_PARSER_BUFFER_SIZE, Cursor`
  at `screen.py:4`.
- **Transfer — Claim:** the transfer test binds `kittens.transfer.rsync` at module scope (this is the Tier 2
  driver). **Evidence:** `from kittens.transfer.rsync import Differ, Hasher, Patcher, parse_ftc` at
  `file_transmission.py:13`.
- **Windowing — Claim:** the windowing test loads a GLFW backend on demand by path (this is the Tier 3
  driver). **Evidence:** `lib = ctypes.CDLL(backend_utils)` at `glfw.py:50`, with `backend_utils` from
  `glfw_path('x11')` at `glfw.py:49`.

**Note on the crypto nuance.** The crypto import is *not* module-scope — it lives **inside a test method**,
so it is indented in the source (`crypto.py:28`). It therefore binds `kitty.fast_data_types` only when that
test runs, but since the core is already a hard base-import prerequisite (Tier 1), this distinction never
changes the crypto outcome in practice.

**Shaders (a related build-verification detail).** `test_loading_shaders` (`check_build.py:33-36`) does not
name `.glsl` files directly; it constructs `Program(name)` for
`for name in 'cell border bgimage tint graphics'.split():` (`check_build.py:35`). `Program.__init__` then
derives the filenames as `f'{name}_vertex.glsl'` (`kitty/shaders.py:54`) and `f'{name}_fragment.glsl'`
(`kitty/shaders.py:55`) — the `Program` class is defined at `kitty/shaders.py:43` and its `compile_program`
work is performed through the `kitty.fast_data_types` binding.

**Full runnable test-module inventory (22 modules; `main` and `gr` excluded per `main.py:57`):**
`check_build`, `clipboard`, `completion`, `crypto`, `datatypes`, `file_transmission`, `fonts`, `glfw`,
`graphics`, `keys`, `layout`, `mouse`, `open_actions`, `options`, `parser`, `screen`,
`search_query_parser`, `shell_integration`, `shm`, `ssh`, `tui`, `utmp` — plus the ever-present base package
`kitty_tests/__init__.py`. The overwhelming majority bind the single core extension `kitty.fast_data_types`
(directly or transitively through `BaseTest`), which is why its absence zeroes out the run.

---

## 8. Critical vs. Optional Classification (by Observed Impact of Absence)

| Tier | Extension | Import style | Observed impact when removed | Classification |
|---|---|---|---|---|
| **1** | `kitty/fast_data_types.so` | module-scope, base package | `ModuleNotFoundError` at `conf/utils.py:27`; **0** tests run | **Critical** |
| **2** | `kittens/transfer/rsync.so` | module-scope, `file_transmission.py:13` | `ModuleNotFoundError` at `file_transmission.py:13` via eager `main.py:64`; **0** tests run | **Collection-critical** |
| **3** | `kitty/glfw-x11.so`, `kitty/glfw-wayland.so` | on-demand `ctypes.CDLL`, single test | `Ran 145 tests`; only `test_glfw_modules` fails + `test_utf_8_strndup` errors | **Optional / platform** |

- **Critical — Claim:** `kitty/fast_data_types.so` is critical; without it, **nothing** runs. **Evidence:**
  Tier 1 produced `ModuleNotFoundError: No module named 'kitty.fast_data_types'` (`conf/utils.py:27`) and no
  `Ran N tests` line.
- **Collection-critical — Claim:** `kittens/transfer/rsync.so` is critical to *collection*; without it,
  discovery aborts and **nothing** runs, even though the base package imports fine. **Evidence:** Tier 2
  produced `ModuleNotFoundError: No module named 'kittens.transfer.rsync'` (`file_transmission.py:13`, via
  `main.py:64`) and no `Ran N tests` line.
- **Optional / platform — Claim:** the GLFW backends are optional to the run; without them the suite still
  runs and only the two GLFW-specific tests break. **Evidence:** Tier 3 produced `Ran 145 tests in 15.476s`
  with `FAILED (failures=7, errors=1, skipped=2)` — exactly the baseline plus the GLFW error/failure.

---

## 9. Read-Only Guarantee

Every destructive experiment (§5) was performed in an isolated copy of the tree at `/tmp/kitty_iso`, created
with `tar` (excluding `.git` and `build`). The real repository was never mutated: the build touches only
git-ignored artifacts (`*.so`, `build/`, `/kitty/launcher/kitt*`), and the temporary observation scripts
(`blitzy_adhoc_test_*.py`) were removed. After all experiments and cleanup:

```console
$ git status --porcelain
$
```

**Claim:** the source tree is byte-for-byte unchanged after the entire investigation. **Evidence:**
`git status --porcelain` returns **empty** output at the end of the investigation/cleanup step (no modified,
added, or deleted tracked files). The compiled `.so` and launcher artifacts are git-ignored, so they do not
appear; `.pyc`/`__pycache__` produced by running the tests are also ignored (`.gitignore` includes `*.pyc`
and `__pycache__/`).

The **only** subsequent addition to the repository is this document itself — the sole permitted new file.
Once it is written, `git status --porcelain -uall` shows exactly one untracked entry and nothing else:

```console
$ git status --porcelain -uall
?? blitzy/documentation/kitty_815df1e210e0.md
```

**Claim:** no existing/tracked source file was modified, added, or deleted; the lone repository change is
this deliverable. **Evidence:** the single `?? blitzy/documentation/kitty_815df1e210e0.md` line (an untracked
new file) is the complete porcelain output.

---

## 10. Coverage Pass — Every Named Item, Answered

Re-reading the question and confirming each named item is addressed in the body:

- [x] **Build the project from source** — §1: default build fails under `-Werror` (`setup.py:491`);
  `--ignore-compiler-warnings` (`setup.py:2003-2004`) yields a clean exit-0 build with all six artifacts.
- [x] **Execute the test suite at representative scale, capturing real runner output** — §2:
  `Ran 145 tests in 27.964s`, `FAILED (failures=6, skipped=2)`, `All Go tests succeeded, ran in 28.4 seconds`.
- [x] **Trace the relationship between the compiled C extensions and test execution (concrete binding code
  path)** — §3.1: `from kitty.fast_data_types import …` at `kitty_tests/__init__.py:22`, plus the
  authoritative `test_loading_extensions` (`check_build.py:28-31`).
- [x] **Observe which extension modules actually load (enumerate loaded `.so`)** — §3.3: exactly two
  kitty-specific modules — `kitty.fast_data_types` and `kittens.transfer.rsync` — enumerated from
  `sys.modules` after real discovery.
- [x] **Observe how failures cascade when modules are unavailable** — §5: Tier 1/2 → zero tests
  (`ModuleNotFoundError`); Tier 3 → 145 tests run, two GLFW tests break (`OSError` / `AssertionError`).
- [x] **Determine what the output reveals about the dependency structure** — §6: eager discovery
  (`main.py:64`) makes module-scope compiled imports hard collection prerequisites; a sharp cliff (full run
  vs. zero tests).
- [x] **Map extensions to different test categories — e.g., core data types, transfer, windowing** — §7:
  each named example covered — core data types (`screen.py:4`), transfer (`file_transmission.py:13`),
  windowing (`glfw.py:49-50`) — plus parsing/SIMD, graphics, crypto, build verification.
- [x] **Identify critical vs. optional modules (by observed impact)** — §8: `fast_data_types.so` critical,
  `rsync.so` collection-critical, GLFW backends optional/platform.
- [x] **Trace the actual import chains established during the test run** — §4: primary chain
  `__init__.py:21` → `config.py:10` → `conf/utils.py:27`; secondary chain `__init__.py:27` → `window.py:33`
  → `child.py:11`; transfer chain `main.py:64` → `file_transmission.py:13`.
- [x] **Confirm every "e.g./such as" example is addressed** — the three named categories (core data types,
  transfer, windowing) are each answered explicitly in §7.

### Honesty note on what was *not* executed here

The macOS GLFW backend `glfw-cocoa.so` was **not** built or executed in this Linux container. Its handling is
described only from code and skip signals: `glfw_path()` resolves it by the same rule
(`kitty/constants.py:193`), and `test_glfw_modules` selects `['cocoa']` only when `is_macos` is true
(`check_build.py:38-47`), while `test_utf_8_strndup` is decorated `@unittest.skipIf(is_macos, …)`
(`kitty_tests/glfw.py:43`). I did not observe macOS behavior directly and do not claim to.

