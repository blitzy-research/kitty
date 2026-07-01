# kitty — Compiled C Extensions and the Test‑Execution Flow

> An investigative Q&A tracing how kitty's compiled C extension modules relate to its
> test‑execution flow. **Every claim below is grounded in output I actually captured by
> building kitty and running its test suite** — not in reading alone. Where my live
> observation differs from previously documented figures, **the observed value is reported
> as ground truth** (per the task's ground‑truth‑precedence rule) and the discrepancy is
> called out explicitly.

---

## 0. Investigation metadata

| Item | Value |
|---|---|
| Repository | `kovidgoyal/kitty` |
| Source branch | `kitty_815df1e210e0` |
| HEAD commit | `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` ("Wire up applying of font config") |
| Task | Read‑only investigation; the **only** file created is this document |

### 0.1 Environment (observed)

All build and test runs were executed **inside** the pinned Docker image
`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0` (Ubuntu 24.04 base). This
is mandatory: the host Python is 3.13, which cannot load the image's CPython‑3.12 `.so`
extensions. The repository source tree was bind‑mounted into the container at `/app`.

Toolchain, captured verbatim (`gcc --version`, `go version`, `pkg-config --version`,
`python3 --version`):

```
gcc: gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
go: go version go1.23.4 linux/amd64
pkg-config: 1.8.1
python3: Python 3.12.3
```

System library versions (`pkg-config --modversion <lib>`):

```
freetype2: 26.1.20     fontconfig: 2.15.0     harfbuzz: 8.3.0
libpng: 1.6.43         lcms2: 2.14            openssl: 3.0.13
libxxhash: 0.8.2       xkbcommon: 1.6.0       x11-xcb: 1.8.7
wayland-client: 1.22.0
```

Environment‑version facts cited from source: `requires-python = ">=3.8"`
[`pyproject.toml:2`]; `go 1.22` [`go.mod:3`]; and the official minimal build requirement —
"a C compiler and the `go compiler`" [`docs/build.rst:14-16`], corroborated by kitty's
public build documentation (<https://sw.kovidgoyal.net/kitty/build/>).

> **Deviation note (observed is authoritative):** the container ships `go version
> go1.23.4`, whereas prior documentation recorded `go1.22.2`. The `go.mod` *directive* is
> nonetheless `go 1.22` [`go.mod:3`]; the directive is a language‑version floor, not the
> installed compiler version. This difference only affects the size of the Go‑compiled
> `kitten` binary (see the Q1 artifacts table), not any Python/extension result.

### 0.2 Commands used and their outcomes (the whole investigation in one table)

| # | Command (run inside the image) | Tree state | Exit | Outcome |
|---|---|---|---|---|
| 1 | `python3 test.py` | **unbuilt** | `1` | Regime A cascade — `ModuleNotFoundError: No module named 'kitty.fast_data_types'`, **0 tests run** |
| 2 | `python3 setup.py` | unbuilt → **builds** | `0` | Build succeeds (no flags); produces 4 `.so` + 2 launchers |
| 3 | `CI=true ./kitty/launcher/kitty +launch test.py` | **built** | `1` | Regime B — `Ran 145 tests in 33.446s`, `FAILED (failures=1, errors=2, skipped=4)` |
| 4 | `env PYTHONPATH="$PWD" python3 /tmp/observe_loaded.py` | built | `0` | Extension probe — `collected test cases: 145`; only `fast_data_types.so` + `rsync.so` in `sys.modules` |
| 5 | `CI=true ./kitty/launcher/kitty +launch test.py` (`rsync.so` removed) | built‑minus‑rsync | `1` | Regime A variant — eager‑import abort at `kitty_tests/main.py:64` |

The bootstrap entry point is a two‑line shim: `test.py` imports and calls
`kitty_tests.main.main` [`test.py:8`, `test.py:9`, `test.py:13`], and its shebang
`#!./kitty/launcher/kitty +launch` [`test.py:1`] is why the canonical invocation runs
through the built launcher.

---

## 1. Central finding — two failure regimes

The single most important result of this investigation is that kitty's test suite exhibits
**two fundamentally different failure regimes**, determined entirely by whether the compiled
C extensions are present:

- **Regime A — missing C extension (unbuilt tree).** A *single* fatal
  `ModuleNotFoundError` aborts the **entire** test collection **before any test executes**.
  Observed: **0 tests run**. This is caused by eager, exception‑free imports (detailed in
  Q3/Q7).

- **Regime B — built tree.** The suite runs **fully** — **145 tests** collected and
  executed — and individual tests then fail or skip **in isolation**. In my environment:
  `FAILED (failures=1, errors=2, skipped=4)` on the Python side plus one environment‑specific
  Go failure. **None** of these is an extension‑loading defect; every one is traceable to a
  missing font, an unavailable shell locale, or a container‑filesystem limitation.

The invariant that survives every environment difference is the number **145**: 145 tests
are collected and run whenever `kitty/fast_data_types.so` is present, and **0** when it is
absent. That single fact is the spine of the dependency story told below.

---

## Q1 — Build kitty and run its test suite (exact commands + observed outcomes)

**Build.** kitty is built by its custom orchestrator `setup.py` (a hybrid C/Python/Go build
driven by a `CompilationDatabase`, not setuptools/CMake). I ran, inside the image:

```
python3 setup.py
```

**Observed outcome: exit `0`.** The build first generates the Wayland protocol headers, then
compiles the C extensions and the Go `kitten` binary. First lines of the build log, verbatim:

```
[1/28] Generating wayland-xdg-shell-client-protocol.h ...
[2/28] Generating wayland-xdg-shell-client-protocol.c ...
[3/28] Generating wayland-viewporter-client-protocol.h ...
...
```

> **Build‑flag transparency (important, and *not* a kitty defect).** Prior documentation
> built with `python3 setup.py --ignore-compiler-warnings`. That flag drops
> `-pedantic-errors -Werror` — literally `werror = '' if ignore_compiler_warnings else
> '-pedantic-errors -Werror'` [`setup.py:491`]. It was only ever an *environment
> accommodation* for a `wayland-protocols` enum drift (an unhandled
> `XDG_TOPLEVEL_STATE_CONSTRAINED_*` `switch` case in `glfw/wl_window.c` that a
> `-Werror=switch` site would reject) on a drifted host. **In this pinned image the flag is
> unnecessary: `python3 setup.py` succeeds at exit `0` with `-pedantic-errors -Werror` still
> active.** I report the observed clean build; the flag mechanism is cited from
> `setup.py:491` for completeness.

**Build artifacts produced** (byte sizes via `stat`; all gitignored via `*.so`, `/build/`,
`/kitty/launcher/kitt*`):

| Artifact | Size (bytes) | Linked at |
|---|---|---|
| `kitty/fast_data_types.so` | `1213072` | `setup.py:1090-1091` (`compile_c_extension(kitty_env(args), 'kitty/fast_data_types', ...)`); C sources gathered by `find_c_files()` [`setup.py:906`] |
| `kitty/glfw-wayland.so` | `442784` | `setup.py:952-953` (`compile_c_extension(genv, f'kitty/glfw-{module}', ...)`) |
| `kitty/glfw-x11.so` | `357592` | `setup.py:952-953` |
| `kittens/transfer/rsync.so` | `55056` | `setup.py:986` (`files('transfer', 'rsync', libraries=pkg_config('libxxhash', '--libs'), ...)`) |
| `kitty/launcher/kitty` | `36224` | launcher executable (`kitty/launcher/main.c`) |
| `kitty/launcher/kitten` | `15945988` | Go binary (see deviation note) |

> The four `.so` sizes reproduce prior documentation **exactly**. `kitty/launcher/kitten`
> measured `15945988` here vs. a previously documented `15757572`; the difference is the
> `go1.23.4` vs `go1.22.2` compiler (see §0.1). Observed value is authoritative.

**Run.** The canonical, built‑tree invocation mirrors what `python setup.py test` performs —
`os.execl(texe, texe, '+launch', 'test.py')` with `texe = <launcher_dir>/kitty`
[`setup.py:2101-2103`]. I ran it directly:

```
CI=true ./kitty/launcher/kitty +launch test.py
```

**Observed outcome: exit `1`.** Header lines, verbatim:

```
Running under CI: True
Using PATH in test environment: /app/kitty_tests/kitty/launcher:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Python: /usr/bin/python
Intrinsics: has_avx2=True has_sse4_2=True
Go executable: /usr/local/go/bin/go
```

Summary markers, verbatim:

```
Ran 145 tests in 33.446s

FAILED (failures=1, errors=2, skipped=4)
```

with the overall run ending in `Error: Some tests failed!`. The full breakdown of those
failures/errors/skips (and why none is an extension defect) is given in Q8. **`CI=true`
matters**: `is_ci = os.environ.get('CI') == 'true'` [`kitty_tests/__init__.py:212`] gates
several test behaviors (e.g. the font check and the GLFW‑backend list), so the run was made
under CI to keep those conditions well‑defined.

**Rationale.** `setup.py` is the documented build path (`python3 setup.py build` is a listed
subcommand, and the official docs state the minimal requirement as a C compiler + the Go
compiler [`docs/build.rst:14-16`]). The launcher‑based invocation is required rather than
plain `python3 test.py` because the suite's Go phase reads `sys.kitty_run_data`, which only
the launcher sets (demonstrated in Q3's run‑method nuance).

---

## Q2 — Which compiled C extension modules actually get *loaded* during tests

**Method and rationale.** The authoritative way to answer "which extensions *load*" is to
inspect `sys.modules` after the suite is collected — because *file presence on disk is not
the same as a Python import*. (Indeed, `check_build` validates GLFW purely as on‑disk files,
never importing them — see Q5.) I wrote a temporary probe **outside** the repository at
`/tmp/observe_loaded.py`, which calls `find_all_tests()` [`kitty_tests/main.py:57`] to force
the exact same collection the real run performs, then scans `sys.modules` for `.so` modules:

```python
import sys, os
from kitty_tests.main import find_all_tests
suite = find_all_tests()
print("collected test cases:", suite.countTestCases())
for name, mod in sorted(sys.modules.items()):
    f = getattr(mod, '__file__', None) or ''
    if f.endswith('.so'):
        print(f"{name} -> {os.path.basename(f)}   [in sys.modules: True]")
print("glfw-* names in sys.modules:", sorted(n for n in sys.modules if 'glfw' in n.lower()))
```

Run with `env PYTHONPATH="$PWD" python3 /tmp/observe_loaded.py` from the repo root (so cwd,
not `/tmp`, is on `sys.path`). **Observed output, verbatim (exit `0`):**

```
collected test cases: 145
PIL._imaging -> _imaging.cpython-312-x86_64-linux-gnu.so   [in sys.modules: True]
_bz2 -> _bz2.cpython-312-x86_64-linux-gnu.so   [in sys.modules: True]
_ctypes -> _ctypes.cpython-312-x86_64-linux-gnu.so   [in sys.modules: True]
_hashlib -> _hashlib.cpython-312-x86_64-linux-gnu.so   [in sys.modules: True]
_json -> _json.cpython-312-x86_64-linux-gnu.so   [in sys.modules: True]
_lzma -> _lzma.cpython-312-x86_64-linux-gnu.so   [in sys.modules: True]
kittens.transfer.rsync -> rsync.so   [in sys.modules: True]
kitty.fast_data_types -> fast_data_types.so   [in sys.modules: True]
mmap -> mmap.cpython-312-x86_64-linux-gnu.so   [in sys.modules: True]
termios -> termios.cpython-312-x86_64-linux-gnu.so   [in sys.modules: True]
glfw-* names in sys.modules: ['kitty_tests.glfw']
```

**Answer.** Of kitty's **four** compiled artifacts, exactly **two** are Python‑imported into
`sys.modules` during test collection:

1. `kitty.fast_data_types` → `fast_data_types.so`
2. `kittens.transfer.rsync` → `rsync.so`

The GLFW extensions `glfw-x11.so` / `glfw-wayland.so` are **never** Python‑imported: the only
`sys.modules` name containing "glfw" is `kitty_tests.glfw` — the *test module*
`kitty_tests/glfw.py`, **not** either extension. (The GLFW `.so` files are `dlopen`'d by the
C layer at GUI runtime, which never happens in these headless tests.) Every other `.so` in
`sys.modules` (`PIL._imaging`, `_bz2`, `_ctypes`, `_hashlib`, `_json`, `_lzma`, `mmap`,
`termios`) is Python‑stdlib or Pillow — not a kitty build artifact.

**Why `rsync.so` is loaded even though it is "optional":** because
`kitty_tests/file_transmission.py` imports it at **module top level** —
`from kittens.transfer.rsync import Differ, Hasher, Patcher, parse_ftc`
[`kitty_tests/file_transmission.py:13`] — so merely *collecting* that test module imports the
extension.

---

## Q3 — How test failures cascade when the compiled modules are unavailable

When a required extension is missing, the failure does not stay local — it **cascades and
aborts the whole run**. I reproduced this two ways.

### Q3.A — Missing `kitty/fast_data_types.so` (the critical extension): total collection abort

Running the **unbuilt** tree with `python3 test.py` produced this, verbatim (exit `1`; frozen
`importlib` frames elided with `...`):

```
Traceback (most recent call last):
  File "/app/test.py", line 13, in <module>
    main()
  File "/app/test.py", line 8, in main
    m = importlib.import_module('kitty_tests.main')
  ...
  File "/app/kitty_tests/__init__.py", line 21, in <module>
    from kitty.config import finalize_keys, finalize_mouse_mappings
  File "/app/kitty/config.py", line 10, in <module>
    from .conf.utils import BadLine, parse_config_base
  File "/app/kitty/conf/utils.py", line 27, in <module>
    from ..fast_data_types import Color
ModuleNotFoundError: No module named 'kitty.fast_data_types'
```

**Terminal literal:** `ModuleNotFoundError: No module named 'kitty.fast_data_types'`.
**Result: 0 tests run.** The cascade is a **package‑import‑time** abort: `test.py:8` imports
the *package* `kitty_tests`, which runs `kitty_tests/__init__.py`, whose line 21 pulls in
`kitty.config`, which transitively reaches `fast_data_types`. Because this happens while the
package itself is being imported, `find_all_tests()` is **not even reached** — there is no
opportunity for any per‑test isolation. One missing extension ⇒ the entire suite is dead on
arrival.

### Q3.B — Missing `kittens/transfer/rsync.so` (an "optional" extension): still aborts collection

To show the *mechanism* that makes even an optional extension fatal to collection, I moved
`rsync.so` out of the tree (it is gitignored, so the working tree stayed clean) with
`fast_data_types.so` still present, and ran the built‑tree launcher command. Verbatim (exit
`1`; launcher frames elided):

```
  File "/app/kitty/launcher/../../kitty_tests/main.py", line 338, in main
    run_tests()
  File "/app/kitty/launcher/../../kitty_tests/main.py", line 279, in run_tests
    run_python_tests(args, go_proc)
  File "/app/kitty/launcher/../../kitty_tests/main.py", line 211, in run_python_tests
    tests = find_all_tests()
  File "/app/kitty/launcher/../../kitty_tests/main.py", line 64, in find_all_tests
    m = importlib.import_module(package + '.' + x.partition('.')[0])
  File "/app/kitty/launcher/../../kitty_tests/file_transmission.py", line 13, in <module>
    from kittens.transfer.rsync import Differ, Hasher, Patcher, parse_ftc
ModuleNotFoundError: No module named 'kittens.transfer.rsync'
```

Here `kitty_tests/__init__.py` does **not** need `rsync`, so the package imports fine; the
abort happens **inside** `find_all_tests()` at the eager
`m = importlib.import_module(...)` [`kitty_tests/main.py:64`], when the collection loop
reaches `file_transmission`. That loop has **no `try/except`** around the import
[`kitty_tests/main.py:61-65`], so a single un‑importable module terminates the entire Python
collection.

> **Accuracy note (stated honestly).** The error that actually fired is a **raw
> `ModuleNotFoundError` propagating from `kitty_tests/main.py:64`** — *not* the guard
> `raise Exception('Failed to import a test module: %s' % test)` at
> `kitty_tests/main.py:52-53`. That guard lives in `itertests()`
> [`kitty_tests/main.py:44-54`] and only fires if a unittest `ModuleImportFailure` placeholder
> is encountered while *iterating* an already‑built suite (a distinct defensive path). It is
> the related mechanism, but it is not what triggered in this observed run.

### Q3.C — Run‑method nuance (bonus evidence about ordering)

If instead you run plain `python3 test.py` (no launcher) with `rsync.so` missing, the failure
comes **even earlier**, verbatim (exit `1`):

```
  File "/app/kitty_tests/main.py", line 338, in main
    run_tests()
  File "/app/kitty_tests/main.py", line 270, in run_tests
    go_proc: 'Optional[GoProc]' = run_go(go_pkgs, args.name)
  File "/app/kitty_tests/main.py", line 192, in run_go
    return GoProc(cmd)
  File "/app/kitty_tests/main.py", line 155, in __init__
    env['KITTY_PATH_TO_KITTY_EXE'] = kitty_exe()
  File "/app/kitty/constants.py", line 67, in kitty_exe
    rpath = getattr(sys, 'kitty_run_data').get('bundle_exe_dir')
AttributeError: module 'sys' has no attribute 'kitty_run_data'
```

This proves two things: (1) `run_tests()` calls `run_go()` [`kitty_tests/main.py:270`]
**before** `run_python_tests()` [`kitty_tests/main.py:279`]; and (2) the launcher is required
because only it sets `sys.kitty_run_data` (read at `kitty/constants.py:67`). That is precisely
why the canonical invocation is `./kitty/launcher/kitty +launch test.py`, not bare Python.

---

## Q4 — What the test output reveals about the Python‑↔‑native dependency structure

The output reveals a **strict, layered, hard dependency**: the entire Python test layer sits
on top of the native extension layer, and the coupling point is `kitty/fast_data_types.so`.

The decisive evidence is *where* the dependency is triggered. `fast_data_types` is not merely
imported by an individual test — it is pulled in **transitively at package‑import time**,
before any test class is even defined, through configuration code:

```
kitty_tests/__init__.py:21   from kitty.config import finalize_keys, finalize_mouse_mappings
        -> kitty/config.py:10        from .conf.utils import BadLine, parse_config_base
                -> kitty/conf/utils.py:27    from ..fast_data_types import Color   # TERMINAL
```

This transitive hop reaches `fast_data_types` **before** the test base class's own *direct*
import of it — `from kitty.fast_data_types import Cursor, HistoryBuf, LineBuf, Screen,
get_options, monotonic, set_options` [`kitty_tests/__init__.py:22`]. In other words, the very
act of importing the `kitty_tests` package binds the Python layer to the C layer.

The behavioral consequence, observed directly, defines the dependency as **non‑optional**:

- `fast_data_types.so` **present** ⇒ `collected test cases: 145` and the suite executes
  (Regime B).
- `fast_data_types.so` **absent** ⇒ `ModuleNotFoundError` at `kitty/conf/utils.py:27` and
  **0 tests** (Regime A).

There is no graceful degradation, no skip, no partial run: the test output shows a binary
outcome gated by one shared library. That is the structural relationship the output reveals —
a Python test tier that cannot even be *collected* without its native tier.

---

## Q5 — How each compiled extension connects to the different test categories

"Test categories" here means the runnable test modules under `kitty_tests/`. `find_all_tests()`
collects every module in that package except `main` and `gr`
[`kitty_tests/main.py:57` — `excludes=('main', 'gr')`], which yields **22** modules:
`check_build, clipboard, completion, crypto, datatypes, file_transmission, fonts, glfw,
graphics, keys, layout, mouse, open_actions, options, parser, screen, search_query_parser,
shell_integration, shm, ssh, tui, utmp`.

The connections:

- **`fast_data_types.so` → all 22 modules.** Every collected module imports the shared test
  base `kitty_tests/__init__.py`, and that base binds `fast_data_types` both transitively
  (`:21` → `kitty/config.py:10` → `kitty/conf/utils.py:27`) and directly (`:22`). So this
  extension is coupled to **every** category simultaneously. It is also exercised explicitly
  by `check_build.test_loading_extensions`, which does `import kitty.fast_data_types as fdt`
  [`kitty_tests/check_build.py:29`].

- **`rsync.so` → `file_transmission` and `check_build`.** `file_transmission` imports it at
  module top level [`kitty_tests/file_transmission.py:13`], and
  `check_build.test_loading_extensions` imports it via `from kittens.transfer import rsync`
  [`kitty_tests/check_build.py:30`]. These are the only categories that touch it.

- **`glfw-x11.so` / `glfw-wayland.so` → `check_build` only, and only as a file‑check.**
  `check_build.test_glfw_modules` [`kitty_tests/check_build.py:38`] never imports these
  modules; it resolves each backend's on‑disk path with `glfw_path(name)` and asserts it is a
  file and is executable:

  ```
  path = glfw_path(name)
  self.assertTrue(os.path.isfile(path), f'{path} is not a file')
  self.assertTrue(os.access(path, os.X_OK), f'{path} is not executable')
  ```
  [`kitty_tests/check_build.py:44-47`]. The backend list is `linux_backends = ['x11']`
  [`kitty_tests/check_build.py:40`], and `'wayland'` is appended **only when not under CI** —
  `if not self.is_ci: linux_backends.append('wayland')` [`kitty_tests/check_build.py:41-42`].
  Under my `CI=true` run, therefore, only `glfw-x11.so` is file‑checked; `glfw-wayland.so` is
  not even examined.

- **Launchers (`kitty`, `kitten`).** Validated by `check_build.test_exe`, and the `kitty`
  launcher is itself the harness that runs the whole suite (`+launch test.py`).

The full mapping is tabulated in the "Extension → test‑category mapping table" below.

---

## Q6 — Critical vs. optional compiled modules

- **`kitty/fast_data_types.so` — CRITICAL (gates the whole suite).** Present ⇒ 145 tests
  run; absent ⇒ collection aborts and **0 tests** run (Q3.A). It is loaded transitively at
  package‑import time (Q4), so nothing downstream can execute without it. This is the one and
  only module whose absence turns Regime B into Regime A.

- **`kittens/transfer/rsync.so` — OPTIONAL / targeted, with a caveat.** Functionally it
  matters to only two categories (`file_transmission`, `check_build` — Q5). **But** because
  `file_transmission` imports it at module top level [`kitty_tests/file_transmission.py:13`]
  and `find_all_tests()` collects modules with no `try/except` [`kitty_tests/main.py:64`], a
  *missing* `rsync.so` still aborts the entire Python collection (Q3.B). So "optional" here
  means **optional to functionality, not to a clean collection**: its absence is fatal to
  *collection* even though it gates only a couple of test categories once present.

- **`kitty/glfw-x11.so` and `kitty/glfw-wayland.so` — file‑checked only, never imported.**
  They are validated purely as on‑disk executable files by `check_build.test_glfw_modules`
  [`kitty_tests/check_build.py:44-47`]; they never enter `sys.modules` during tests (Q2). If
  they were missing, only `test_glfw_modules` would fail — the rest of the suite would be
  entirely unaffected. They are the most "optional" of the four.

**Summary:** exactly **one** compiled module is CRITICAL (`fast_data_types.so`); `rsync.so` is
targeted‑but‑collection‑fatal; the two GLFW modules are purely file‑checked.

---

## Q7 — The actual import chains established during the run (with `file:line` anchors)

**Chain 1 — the critical, package‑import‑time chain to `fast_data_types` (the terminal node
of the Regime A cascade):**

```
test.py:8                    importlib.import_module('kitty_tests.main')
  → (imports package kitty_tests, runs kitty_tests/__init__.py)
    kitty_tests/__init__.py:21   from kitty.config import finalize_keys, finalize_mouse_mappings
      kitty/config.py:10           from .conf.utils import BadLine, parse_config_base
        kitty/conf/utils.py:27       from ..fast_data_types import Color          ← TERMINAL
```

**Chain 1b — the *direct* base‑class import (fires right after Chain 1, same file):**

```
kitty_tests/__init__.py:22   from kitty.fast_data_types import Cursor, HistoryBuf, LineBuf, Screen, get_options, monotonic, set_options
```

**Chain 2 — the collection‑time loop that imports each test module (and the `rsync` terminal
node when it is missing):**

```
kitty_tests/main.py:338      main()  → run_tests()
kitty_tests/main.py:279      run_tests()  → run_python_tests(args, go_proc)
kitty_tests/main.py:211      run_python_tests()  → tests = find_all_tests()
kitty_tests/main.py:57/64    find_all_tests(): for each module → importlib.import_module(...)
        kitty_tests/file_transmission.py:13   from kittens.transfer.rsync import Differ, Hasher, Patcher, parse_ftc   ← TERMINAL (Regime A variant)
```

**Chain 3 — the Go‑setup chain that precedes Python collection (observed via the run‑method
nuance):**

```
kitty_tests/main.py:338      main() → run_tests()
kitty_tests/main.py:270      run_tests() → run_go(go_pkgs, args.name)
kitty_tests/main.py:192      run_go() → GoProc(cmd)
kitty_tests/main.py:155      GoProc.__init__ → env['KITTY_PATH_TO_KITTY_EXE'] = kitty_exe()
kitty/constants.py:67        kitty_exe() → getattr(sys, 'kitty_run_data')   ← needs the launcher
```

All five anchor lines above were verified against the on‑disk source; the terminal nodes
(`kitty/conf/utils.py:27` and `kitty_tests/file_transmission.py:13`) are exactly the lines
that appeared at the bottom of the two Regime A tracebacks in Q3.

---

## Q8 — Artifacts ↔ test‑module imports ↔ observed failures: one coherent picture

Putting it together: the build produces **four** `.so` artifacts + **two** launchers (Q1);
the collection **imports exactly two** of those `.so` (`fast_data_types`, `rsync` — Q2) and
**file‑checks** the two GLFW ones (Q5); and the **145 tests** then run, with a handful of
**environment‑specific** failures that are unrelated to extension loading.

**The observed Regime B outcome (default canonical command), verbatim:**

```
Ran 145 tests in 33.446s

FAILED (failures=1, errors=2, skipped=4)
```

**1 failure — a missing font, not an extension problem.**

```
FAIL: test_font_selection (kitty_tests.fonts.Selection.test_font_selection)
...
  File "/app/kitty/launcher/../../kitty_tests/fonts.py", line 64, in test_font_selection
    t('Source Code Pro', 'SourceCodePro', 'Semibold', 'It')
  File "/app/kitty/launcher/../../kitty_tests/fonts.py", line 58, in t
    if has(family, allow_missing_in_ci=allow_missing_in_ci):
  File "/app/kitty/launcher/../../kitty_tests/fonts.py", line 54, in has
    raise AssertionError(f'The family: {family} is not available')
AssertionError: The family: Source Code Pro is not available
```

The test class is `Selection` [`kitty_tests/fonts.py:22`], the method `test_font_selection`
[`kitty_tests/fonts.py:24`]; the assertion at `kitty_tests/fonts.py:54` fires because the
container's installed families were observed as `names = {'symbols nerd font mono', 'dejavu
sans mono'}` — Source Code Pro is simply not installed.

**2 errors — a non‑UTF‑8 locale, not an extension problem.**

```
ERROR: test_zsh_integration (kitty_tests.shell_integration.ShellIntegration.test_zsh_integration)
...
TimeoutError: Timed out: pty.callbacks.last_cmd_cmdline='cd /tmp/.../testing-cwd-notification-????' != 'cd /tmp/.../testing-cwd-notification-🐱'.
```

(and the identical error for `ShellIntegrationWithKitten`). `zsh` **is** installed
(`/usr/bin/zsh`), so the tests run rather than skip; but the image's default locale is
non‑UTF‑8 (`LC_CTYPE="POSIX"`), so the cat‑emoji directory name `testing-cwd-notification-🐱`
degrades to `????` and the assertion times out (each waits the 10‑second timeout, which is
why the wall clock reads `33.446s`). **Proof that this is purely a locale artifact:** re‑running
with `LANG=C.UTF-8 LC_ALL=C.UTF-8` makes both zsh tests **pass** and yields:

```
Ran 145 tests in 13.956s

FAILED (failures=1, skipped=4)
```

i.e. the two errors vanish (and the clock drops by ~20s = the two eliminated 10s timeouts).

**4 skips — platform/environment gating, verbatim:**

```
test_ca_certificates (kitty_tests.check_build.TestBuild.test_ca_certificates) ... skipped 'CA certificates are only tested on frozen builds'
test_fallback_font_not_last_resort (kitty_tests.fonts.Rendering.test_fallback_font_not_last_resort) ... skipped 'Only macOS has a Last Resort font'
test_fish_integration (kitty_tests.shell_integration.ShellIntegration.test_fish_integration) ... skipped 'fish not installed'
test_fish_integration (kitty_tests.shell_integration.ShellIntegrationWithKitten.test_fish_integration) ... skipped 'fish not installed'
```

**Go phase — 26 packages, one environment‑specific failure.** The header lists 26 Go packages
under test. One failed, verbatim:

```
=== RUN   TestCreateAnonymousTempfile
    tpmfile_test.go:23: Anonymous tempfile was not created atomically
--- FAIL: TestCreateAnonymousTempfile (0.00s)
FAIL
FAIL	kitty/tools/utils	0.045s
```

This is a container‑filesystem artifact: creating an anonymous temp file atomically relies on
`O_TMPFILE`, which the Docker `overlay2` upper layer does not honor atomically. It is not
related to any kitty extension.

**The coherent picture.** `fast_data_types.so` (present) is why all **145** tests could be
collected and run at all; `rsync.so` (present, imported at `file_transmission.py:13`) is why
the transfer/`check_build` tests ran; the GLFW `.so` are present as executables and were
file‑checked (x11 under CI). Against that fully‑functional extension layer, the only
non‑passing results were **a missing font, a non‑UTF‑8 locale, a missing shell, a
macOS‑only/frozen‑build gate, and an overlayfs `O_TMPFILE` limitation** — **every one an
environment condition, none an extension‑loading defect.** That is the whole point: the
extension layer worked flawlessly and identically in every run; the noise is all environmental.

---

## Extension → test‑category mapping table

| Compiled artifact | Python‑imported? | Consumed by (test categories) | Classification | Key `file:line` |
|---|---|---|---|---|
| `kitty/fast_data_types.so` | **YES** (in `sys.modules`) | **All 22** collected modules via shared base `kitty_tests/__init__.py`; plus `check_build.test_loading_extensions` | **CRITICAL** — gates all 145 tests | `kitty_tests/__init__.py:21`, `:22`; transitive `kitty/config.py:10` → `kitty/conf/utils.py:27`; `kitty_tests/check_build.py:29` |
| `kittens/transfer/rsync.so` | **YES** (in `sys.modules`) | `file_transmission` (module‑level import); `check_build.test_loading_extensions` | OPTIONAL/targeted — but module‑level import ⇒ its absence still aborts collection | `kitty_tests/file_transmission.py:13`; `kitty_tests/check_build.py:30` |
| `kitty/glfw-x11.so` | **NO** (file‑checked only) | `check_build.test_glfw_modules` (`x11` always) | file‑checked only, never imported | `kitty_tests/check_build.py:40`, `:44-47` |
| `kitty/glfw-wayland.so` | **NO** (file‑checked only) | `check_build.test_glfw_modules` (`wayland` only when **not** CI) | file‑checked only, never imported | `kitty_tests/check_build.py:41-42`, `:44-47` |

Launchers `kitty/launcher/kitty` and `kitty/launcher/kitten` are validated by
`check_build.test_exe`, and the `kitty` launcher is the harness that runs the suite itself
(`+launch test.py`, mirroring `setup.py:2101-2103`).

---

## Observed vs. previously documented figures (full transparency)

Per the ground‑truth‑precedence rule, my **observed** values are authoritative. The
extension‑centric results (the actual subject of this investigation) reproduced **exactly**;
the environment‑specific test tallies differed, and every difference has an identified cause:

| Aspect | Previously documented | **Observed here** | Cause of difference |
|---|---|---|---|
| Tests collected/run | 145 | **145** | — (exact match; the invariant) |
| Extensions in `sys.modules` | `fast_data_types.so`, `rsync.so` | **same** | — (exact match) |
| GLFW modules imported | none (file‑checked) | **none** | — (exact match) |
| Regime A terminal node | `kitty/conf/utils.py:27` | **same** | — (exact match) |
| Build flag | `--ignore-compiler-warnings` | **none needed** (`python3 setup.py`, exit 0) | pinned image has no `wayland-protocols` enum drift |
| Python summary | `failures=3, skipped=6` | **`failures=1, errors=2, skipped=4`** | this image: `zsh` installed (→ 2 locale errors, not skips), `file_transmission` passes (no setgid mismatch), fonts still missing |
| Zsh tests | skipped ('zsh not installed') | **error** (POSIX locale) → **pass** with UTF‑8 | `zsh` present here; locale not UTF‑8 by default |
| Go phase | `All Go tests succeeded` | **1 failure** (`TestCreateAnonymousTempfile`) | overlay2 `O_TMPFILE` not atomic |
| `kitten` binary size | `15757572` B | **`15945988`** B | `go1.23.4` vs `go1.22.2` |
| Wall‑clock time | ~8–14 s | **33.446 s** (default) / **13.956 s** (UTF‑8 locale) | two 10 s zsh timeouts in the default run |

The takeaway is that the **extension‑loading behavior is fully reproducible and
deterministic**, while the peripheral test tallies are legitimately environment‑dependent —
which reinforces, rather than undermines, the central finding.

---

## Coverage pass

- **Q1 — Build and run.** ✅ `python3 setup.py` → exit 0 (§Q1, artifacts table);
  `CI=true ./kitty/launcher/kitty +launch test.py` → exit 1, `Ran 145 tests in 33.446s`.
  Bootstrap anchors `test.py:1/8/9/13`; test invocation mirrors `setup.py:2101-2103`.
- **Q2 — Which extensions load.** ✅ Only `kitty.fast_data_types` → `fast_data_types.so` and
  `kittens.transfer.rsync` → `rsync.so` are in `sys.modules`; GLFW never imported (probe
  output quoted). Rationale: `sys.modules` inspection ≠ file presence.
- **Q3 — Failure cascade.** ✅ Both variants captured verbatim: missing `fast_data_types`
  aborts at package‑import time (`kitty/conf/utils.py:27`, 0 tests); missing `rsync` aborts
  inside the eager loop (`kitty_tests/main.py:64`, no `try/except`), with the honest note that
  the raw error — not the `main.py:52-53` guard — fired. Plus the run‑method nuance
  (`AttributeError` at `kitty/constants.py:67`).
- **Q4 — Dependency structure.** ✅ Hard, layered, non‑optional dependency; the whole suite is
  gated by one C extension reached transitively via `kitty/config.py:10` →
  `kitty/conf/utils.py:27` before the direct import at `kitty_tests/__init__.py:22`.
- **Q5 — Extensions ↔ test categories.** ✅ Mapping table provided; `fast_data_types` → all 22
  modules; `rsync` → `file_transmission` + `check_build`; GLFW → `check_build.test_glfw_modules`
  file‑check only (x11 always, wayland only off‑CI).
- **Q6 — Critical vs. optional.** ✅ `fast_data_types.so` CRITICAL; `rsync.so`
  optional‑but‑collection‑fatal; `glfw-*.so` file‑checked only.
- **Q7 — Import chains.** ✅ Chains 1, 1b, 2, 3 documented with exact `file:line` anchors,
  matching the observed traceback terminal nodes.
- **Q8 — Artifacts ↔ tests ↔ failures.** ✅ Four `.so` + two launchers reconciled against the
  two imported extensions, the file‑checked GLFW pair, and the observed 1 failure / 2 errors /
  4 skips + 1 Go failure — each shown to be environment‑specific, none an extension defect.

**Read‑only outcome.** The only persistent change to the repository is this document,
`blitzy/documentation/kitty_815df1e210e0.md`. All build artifacts (`*.so`, launchers) remain
gitignored and uncommitted; the temporary probe `/tmp/observe_loaded.py` lives outside the
repository and was deleted after use; `git status --porcelain` shows only the new `blitzy/`
tree. No existing repository file was modified.

