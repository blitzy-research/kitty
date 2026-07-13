# kitty — Compiled C Extensions vs. Test-Suite Execution

**Question answered:** How do kitty's compiled C extensions relate to its test-suite execution? Specifically: build kitty from source; execute the test suite through its real entry point; trace the relationship between the compiled C extensions and the test-execution flow; identify which extension modules actually get loaded during test execution; determine how test failures cascade when those extension modules are unavailable; explain what the test output reveals about the dependency structure; map how the compiled extensions connect to the different test categories; classify each extension as CRITICAL or OPTIONAL with respect to test execution; and document the actual import chains established during the test run.

## Methodology and provenance

Every factual claim below was produced by **building kitty from source and running its real test suite first**, then writing this document from the captured output. The complete, unedited output of each command is embedded beside the claim it supports, together with the exact command that produced it. Statements that are **not** direct observations — i.e. conclusions derived by reading source code rather than watching runtime behaviour — are explicitly marked **(inferred)**. Source locations are given as `file:line` against the checkout described below.

- **Source commit under investigation:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (kitty). This is the code that was read for all `file:line` citations.
- **Delivery branch (where this document is added):** `blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9`. This document is the *only* file added to the repository; no existing file is modified. The compiled `*.so` extensions are git-ignored generated build artifacts (`.gitignore:1` = `*.so`); any that were temporarily moved during the failure-cascade reproduction (§5) were restored byte-for-byte (verified by SHA-256 in the Appendix).
- **Canonical build command:** `CC=gcc-13 python3 setup.py build --verbose`
- **Canonical test command:** `CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./test.py` (single module: `./test.py --module <name>`)

All temporary observation scripts lived under `/tmp` (never inside the repository) and were removed after use; their full source is shown inline where used.

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

### command: pkg-config --modversion harfbuzz libxxhash libcrypto libpng lcms2 fontconfig xkbcommon
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
build-essential 12.12ubuntu1	[install ok installed]
libharfbuzz-dev 10.2.0-1	[install ok installed]
libssl-dev 3.5.3-1ubuntu3.4	[install ok installed]
libxxhash-dev 0.8.3-2	[install ok installed]

### command: pkg-config --exists libcrypto && echo 'libcrypto.pc FOUND' ; pkg-config --variable=pcfiledir libcrypto
libcrypto.pc FOUND
libcrypto.pc dir: /usr/lib/x86_64-linux-gnu/pkgconfig

### command: pip show Pillow pygments (installed via --break-system-packages, PEP668)
Name: pillow
Version: 12.3.0
Name: Pygments
Version: 2.20.0
```

## 1. Building kitty from source (canonical configuration)

kitty is built by its multi-language orchestrator `setup.py` (it uses `sysconfig`, not distutils, so it builds cleanly on Python 3.13). The canonical build command — the one kitty's CI uses (`.github/workflows/ci.py:104`, `{python} setup.py build --verbose`) — was run after a clean:

```console
$ CC=gcc-13 python3 setup.py clean
$ export TIMEFORMAT='BUILD_REAL_SECONDS=%R'
$ { time CC=gcc-13 python3 setup.py build --verbose ; }
```

The build completed successfully (exit code 0). Its wall-clock duration was captured with the shell `time` builtin and confirmed to be of the same order across the clean rebuild:

```text
BUILD_REAL_SECONDS=60.871
```

The three extension link steps and the two launcher/Go build steps appear in the verbose build log at these lines (`build/…` object paths abbreviated):

- `build.log:98` — link `kitty/fast_data_types.so`
- `build.log:99` — link `kitty/glfw-x11.so`
- `build.log:100` — link `kittens/transfer/rsync.so`
- `build.log:101` — link the C launcher `kitty/launcher/kitty`
- `build.log:319` — `go build` of `kitty/launcher/kitten` (its ldflags embed `-X kitty.VCSRevision=<delivery-commit>`, i.e. the *destination* commit, not the source commit — see Appendix)

### 1.1 Extension taxonomy — two Python extensions + one native library (observed)

A key correction to any "three compiled Python C-extensions" framing: only **two** of the three shared objects are Python extension modules. This was verified with `nm -D` (dynamic symbol table): a CPython extension must export a `PyInit_<name>` initializer, which the interpreter calls on `import`. `kitty/glfw-x11.so` exports **no** `PyInit_*` — it instead exports the native symbol `utf_8_strndup` that the GLFW test resolves through `ctypes` (§4, §5.3).

```console
$ for so in kitty/fast_data_types.so kittens/transfer/rsync.so kitty/glfw-x11.so; do
    echo "--- $so ---"; nm -D "$so" | grep -E 'PyInit|utf_8_strndup'; done
```

```text
### command: for f in fast_data_types glfw-x11 rsync: nm -D | grep -E 'PyInit|utf_8_strndup'
--- kitty/fast_data_types.so ---
0000000000028c70 T PyInit_fast_data_types
--- kittens/transfer/rsync.so ---
0000000000008600 T PyInit_rsync
--- kitty/glfw-x11.so ---
(no PyInit -> NOT a Python extension module)
    glfw-x11.so exported native symbol used by test_utf_8_strndup:
0000000000021f60 T utf_8_strndup
```

### 1.2 What actually gets compiled into each extension (source selection, observed)

The repository globs are **not** the exact set of build inputs. Counting the object files on each link line in `build.log` (`grep -oE 'build/…\.o'`) versus the on-disk globs shows the difference:

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

### 1.3 Observed build artifacts

The compiled artifacts on disk after the canonical build (`ls -l`):

```text
-rwxr-xr-x 1 root root    55032 Jul 13 17:31 kittens/transfer/rsync.so
-rwxr-xr-x 1 root root  1213072 Jul 13 17:31 kitty/fast_data_types.so
-rwxr-xr-x 1 root root   357584 Jul 13 17:31 kitty/glfw-x11.so
-rwxr-xr-x 1 root root 15765764 Jul 13 17:32 kitty/launcher/kitten
-rwxr-xr-x 1 root root    36288 Jul 13 17:31 kitty/launcher/kitty
```

### 1.4 The `libssl-dev` / CI dependency gap (observed, isolated reproduction)

`setup.py` requires `libcrypto` via `libcrypto_flags()` (defined `setup.py:253`, called `setup.py:616`), which calls the `pkg_config()` helper (defined `setup.py:220`). When a required package is missing, `pkg_config()` raises at `setup.py:239`:

```console
$ sed -n '239p' setup.py
        raise SystemExit(f'The package {name} was not found on your system')
```

kitty's CI apt list (`.github/workflows/ci.py:85-88`) does **not** include `libssl-dev`, yet `setup.py` needs the `libcrypto.pc` it provides. To observe the resulting failure **without touching any system file**, the build was re-run with `PKG_CONFIG_LIBDIR` pointed at a private symlink farm (created with `mktemp -d`) that contained every `.pc` file **except** `libcrypto.pc`/`openssl.pc`/`libssl.pc`. The system `pkg-config` directory was never modified (its `libcrypto` modversion remained `3.5.3` afterward). The command and its complete tail:

```console
$ ISO_DIR="$(mktemp -d)"     # private; populated with all *.pc except libcrypto/openssl/libssl
$ PKG_CONFIG_LIBDIR="$ISO_DIR" CC=gcc-13 python3 setup.py build --verbose   # exit 1
```

```text
Package libcrypto was not found in the pkg-config search path.
Perhaps you should add the directory containing `libcrypto.pc'
to the PKG_CONFIG_PATH environment variable
Package 'libcrypto', required by 'virtual:world', not found
CC: ['gcc-13'] (13, 0)
gcc-13 (Ubuntu 13.4.0-4ubuntu1) 13.4.0
Copyright (C) 2023 Free Software Foundation, Inc.
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

This was observed by running the runner's **own** discovery routine, `find_all_tests()`, through the **real** launcher (`./kitty/launcher/kitty +launch <script>`), and inspecting which entries in `sys.modules` are backed by a `.so` file (matched by module `__spec__.origin`). The temporary script (created under a private `mktemp -d`, mode 0700, removed by an `EXIT`/`INT`/`TERM` trap afterward) was:

```python
# /tmp/…/observe_loading.py  (temporary; removed after the run)
import sys, os
def so_modules():
    out = {}
    for name, mod in list(sys.modules.items()):
        origin = getattr(getattr(mod, '__spec__', None), 'origin', None) or getattr(mod, '__file__', None)
        if origin and origin.endswith('.so'):
            out[name] = origin
    return out
import kitty_tests.main as ktmain            # BOOTSTRAP: runs kitty_tests/__init__.py
boot = so_modules()
suite = ktmain.find_all_tests()             # DISCOVERY: main.py:57,64 import every test module
post = so_modules()
# … prints the two snapshots, the kitty-built vs stdlib split, the discovered case count,
#   and whether glfw is imported as a python module / mapped in /proc/self/maps …
```

Complete unedited output (identical across two runs):

```console
$ CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./kitty/launcher/kitty +launch /tmp/…/observe_loading.py
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

The GLFW backend is not a Python extension (it has no `PyInit_*`; §1.1). The test `kitty_tests/glfw.py:test_utf_8_strndup` obtains its path from `kitty.constants.glfw_path` (defined `kitty/constants.py:191`, returns `os.path.join(extensions_dir, f'{prefix}glfw-{module}.so')` at `constants.py:193`) and loads it with `ctypes.CDLL` at `kitty_tests/glfw.py:50` (`lib = ctypes.CDLL(backend_utils)`), then calls the native `utf_8_strndup` symbol. This was observed with an `sys.addaudithook` on `ctypes.dlopen` plus `/proc/self/maps` snapshots, running the **real** `test_utf_8_strndup` through `unittest`. The temporary script and its complete output (identical across two runs):

```python
# /tmp/…/observe_native_glfw.py  (temporary; removed after the run)
import sys, os, unittest, traceback
def glfw_mapped():
    with open('/proc/self/maps') as f: return 'glfw-x11.so' in f.read()
def glfw_so_backed_module():           # correct test: match on .so ORIGIN, not module NAME
    for name, mod in list(sys.modules.items()):
        origin = getattr(getattr(mod,'__spec__',None),'origin',None) or getattr(mod,'__file__',None)
        if origin and origin.endswith('.so') and 'glfw' in os.path.basename(origin): return (name, origin)
    return None
dlopen_hits = []
def _audit(ev, args):
    if ev == 'ctypes.dlopen' and args and 'glfw' in str(args[0]): dlopen_hits.append((str(args[0]), traceback.extract_stack()))
sys.addaudithook(_audit)
import kitty_tests.main                 # bootstrap
# … print BEFORE state, run the real test_utf_8_strndup via unittest, print AFTER state + dlopen stack …
```

```console
$ CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./kitty/launcher/kitty +launch /tmp/…/observe_native_glfw.py
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

Each compiled extension was moved out of its real import path **one at a time**, the **full** `./test.py` was re-run, the complete output and exit code captured, and the artifact restored. The reproduction used a safe harness: a private `mktemp -d` stash (mode 0700), an **unconditional** `trap … EXIT INT TERM` that always moves the `.so` back, and SHA-256 verification that each artifact was restored byte-for-byte (hashes in the Appendix). These `.so` are git-ignored generated artifacts (`.gitignore:1`), so moving and restoring them leaves the tracked tree unchanged.

### 5.1 `kitty/fast_data_types.so` removed → entire run aborts at bootstrap → **CRITICAL (root)**

With `fast_data_types.so` absent, the run dies **before any test is collected or run**, during the package-init import chain of §3/§9. The complete output is the traceback only — no test lines, no Go line:

```console
$ mv kitty/fast_data_types.so <private-stash>/     # git-ignored artifact
$ CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./test.py    # exit 1
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

With `rsync.so` absent, the environment/preamble prints, the Go tests are **launched** (the line `Go packages being tested: …` comes from `kitty_tests/main.py:277`, after `go_proc = run_go(...)` at `main.py:270`), and then discovery crashes:

```console
$ mv kittens/transfer/rsync.so <private-stash>/    # git-ignored artifact
$ CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./test.py    # exit 1
```

```text
Running under CI: True
Using PATH in test environment: /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty_tests/kitty/launcher:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/go/bin
Python: /usr/bin/python
Intrinsics: has_avx2=True has_sse4_2=True
Go executable: /usr/local/go/bin/go
Go packages being tested: kittens/hints tools/themes tools/tui tools/tui/loop tools/wcswidth tools/tui/shell_integration tools/utils/humanize tools/config tools/tui/readline tools/simdstring kittens/diff tools/cmd/at tools/utils/shlex kittens/ssh kittens/transfer tools/rsync tools/utils/style tools/utils tools/cli tools/utils/base85 tools/tui/subseq tools/tui/graphics tools/unicode_names tools/utils/shm tools/tui/sgr kittens/hyperlinked_grep
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

**Important nuance about the Go tests (observed):** the output proves the Go tests were **launched** (`Go packages being tested: …`) but it does **not** contain `All Go tests succeeded`. That completion line is printed by `print_go()` (`main.py:213`, which calls `go_proc.wait()` at `main.py:214`), and `print_go()` is invoked only later (`main.py:238`), **after** `find_all_tests()` at `main.py:211`. Since the crash happens at `main.py:211`, the Go result is **never waited on or reported** in this cascade. So the correct statement is: the Go tests *launch*, but their completion is **not reported** — not that they "succeed".

### 5.3 `kitty/glfw-x11.so` removed → **two** tests break (native-load ERROR + file-check FAIL) → **OPTIONAL for suite completion**

Unlike the two Python extensions, removing the GLFW backend does **not** abort the run: because `glfw-x11.so` is not imported at discovery, the suite still collects and runs all **145** tests, and the Go tests still complete (`All Go tests succeeded`). But **two** tests break — not one — and the delta versus the baseline (`failures=4, errors=0` in §2) is exactly `+1 failure` and `+1 error`:

```console
$ mv kitty/glfw-x11.so <private-stash>/            # git-ignored artifact
$ CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 ./test.py    # exit 1
```

```text
Running under CI: True
Using PATH in test environment: /tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty_tests/kitty/launcher:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/go/bin
Python: /usr/bin/python
Intrinsics: has_avx2=True has_sse4_2=True
Go executable: /usr/local/go/bin/go
Go packages being tested: tools/tui kittens/diff kittens/hints tools/cli tools/rsync tools/tui/loop tools/wcswidth kittens/transfer tools/themes tools/utils/humanize tools/tui/readline tools/cmd/at kittens/ssh tools/simdstring tools/utils/style tools/tui/graphics tools/utils tools/utils/shm tools/tui/subseq tools/tui/shell_integration tools/tui/sgr tools/unicode_names kittens/hyperlinked_grep tools/utils/shlex tools/config tools/utils/base85
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
    glfw_path = <function glfw_path at 0x7b9985347600>
    self = <kitty_tests.glfw.TestGLFW testMethod=test_utf_8_strndup>
  File "/usr/lib/python3.13/ctypes/__init__.py", line 390, in __init__
    self._handle = _dlopen(self._name, mode)
                   ~~~~~~~^^^^^^^^^^^^^^^^^^
    _FuncPtr = <class 'ctypes.CDLL.__init__.<locals>._FuncPtr'>
    flags = 1
    handle = None
    mode = 0
    name = '/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/glfw-x11.so'
    self = <CDLL '/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/glfw-x11.so', handle 0 at 0x7b998315fcb0>
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
    d = <_io.BufferedWriter name='/tmp/tmpip7ihigb/dest'>
    dest = '/tmp/tmpip7ihigb/dest'
    multiple_files = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files at 0x7b998268ede0>
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7b99832c5160>
    s = <_io.BufferedWriter name='/tmp/tmpip7ihigb/src'>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
    single_file = <function TestFileTransmission.basic_transfer_tests.<locals>.single_file at 0x7b99832cd080>
    src = '/tmp/tmpip7ihigb/src'
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 432, in multiple_files
    self.assertEqual(expected, actual)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^
    Entry = <class 'kitty_tests.file_transmission.Entry'>
    actual = {'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1783964859056941457, mode='0o120777', nlink=1), 'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'sym': Entry(relpath='sym', mtime=1783964859056941457, mode='0o120777', nlink=1), 'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1)}
    b = PosixPath('/tmp/tmpip7ihigb/msrc')
    cmd = ()
    de = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.de at 0x7b998268ff60>
    dest = '/tmp/tmpip7ihigb/mdest'
    dirnames = []
    dirpath = '/tmp/tmpip7ihigb/mdest/msrc/empty'
    entry = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.entry at 0x7b998268f2e0>
    expected = {'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1), 'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1783964859056941457, mode='0o120777', nlink=1), 'sym': Entry(relpath='sym', mtime=1783964859056941457, mode='0o120777', nlink=1)}
    f = <_io.BufferedWriter name='/tmp/tmpip7ihigb/msrc/sub/reg'>
    filenames = []
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7b9983ddde50>
    s = PosixPath('/tmp/tmpip7ihigb/msrc/sub')
    se = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.se at 0x7b998268f380>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_receive>
    src = '/tmp/tmpip7ihigb/msrc'
    x = 'reg'
AssertionError: {'simple': Entry(relpath='simple', mtime=130[497 chars]k=1)} != {'sub': Entry(relpath='sub', mtime=0, mode='[497 chars]k=1)}
  {'abssym': Entry(relpath='abssym', mtime=1783964859056941457, mode='0o120777', nlink=1),
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
   'sym': Entry(relpath='sym', mtime=1783964859056941457, mode='0o120777', nlink=1)}

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
    d = <_io.BufferedWriter name='/tmp/tmphfs5ac1l/dest'>
    dest = '/tmp/tmphfs5ac1l/dest'
    multiple_files = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files at 0x7b998269e480>
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7b998264e7b0>
    s = <_io.BufferedWriter name='/tmp/tmphfs5ac1l/src'>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
    single_file = <function TestFileTransmission.basic_transfer_tests.<locals>.single_file at 0x7b998269c360>
    src = '/tmp/tmphfs5ac1l/src'
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/file_transmission.py", line 432, in multiple_files
    self.assertEqual(expected, actual)
    ~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^
    Entry = <class 'kitty_tests.file_transmission.Entry'>
    actual = {'sub': Entry(relpath='sub', mtime=0, mode='0o40755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1783964859528946542, mode='0o120777', nlink=1), 'empty': Entry(relpath='empty', mtime=0, mode='0o40755', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'sym': Entry(relpath='sym', mtime=1783964859528946542, mode='0o120777', nlink=1), 'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1)}
    b = PosixPath('/tmp/tmphfs5ac1l/msrc')
    cmd = ()
    de = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.de at 0x7b998269f6a0>
    dest = '/tmp/tmphfs5ac1l/mdest'
    dirnames = []
    dirpath = '/tmp/tmphfs5ac1l/mdest/msrc/empty'
    entry = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.entry at 0x7b998269eac0>
    expected = {'simple': Entry(relpath='simple', mtime=1300000000, mode='0o100766', nlink=2), 'hardlink': Entry(relpath='hardlink', mtime=1300000000, mode='0o100766', nlink=2), 'empty': Entry(relpath='empty', mtime=0, mode='0o42755', nlink=2), 'sub/reg': Entry(relpath='sub/reg', mtime=1171299999999, mode='0o100644', nlink=1), 'sub': Entry(relpath='sub', mtime=0, mode='0o42755', nlink=2), 'abssym': Entry(relpath='abssym', mtime=1783964859528946542, mode='0o120777', nlink=1), 'sym': Entry(relpath='sym', mtime=1783964859528946542, mode='0o120777', nlink=1)}
    f = <_io.BufferedWriter name='/tmp/tmphfs5ac1l/msrc/sub/reg'>
    filenames = []
    pty = <kitty_tests.file_transmission.TransferPTY object at 0x7b99829f0c00>
    s = PosixPath('/tmp/tmphfs5ac1l/msrc/sub')
    se = <function TestFileTransmission.basic_transfer_tests.<locals>.multiple_files.<locals>.se at 0x7b998269eb60>
    self = <kitty_tests.file_transmission.TestFileTransmission testMethod=test_transfer_send>
    src = '/tmp/tmphfs5ac1l/msrc'
    x = 'reg'
AssertionError: {'simple': Entry(relpath='simple', mtime=130[497 chars]k=1)} != {'sub': Entry(relpath='sub', mtime=0, mode='[497 chars]k=1)}
  {'abssym': Entry(relpath='abssym', mtime=1783964859528946542, mode='0o120777', nlink=1),
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
   'sym': Entry(relpath='sym', mtime=1783964859528946542, mode='0o120777', nlink=1)}

======================================================================
FAIL: test_glfw_modules (kitty_tests.check_build.TestBuild.test_glfw_modules)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9_cfa475/kitty/launcher/../../kitty_tests/check_build.py", line 46, in test_glfw_modules
    self.assertTrue(os.path.isfile(path), f'{path} is not a file')
    ~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    glfw_path = <function glfw_path at 0x7b9985347600>
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
    opts = <kitty.options.types.Options object at 0x7b99813560d0>
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
    opts = <kitty.options.types.Options object at 0x7b99813560d0>
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
Ran 145 tests in 21.945s

FAILED (failures=5, errors=1, skipped=4)
All Go tests succeeded, ran in 22.0 seconds
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

`find_all_tests()` (`kitty_tests/main.py:57`, excluding `main` and `gr`) discovers 22 modules under `kitty_tests/`: `check_build`, `clipboard`, `completion`, `crypto`, `datatypes`, `file_transmission`, `fonts`, `glfw`, `graphics`, `keys`, `layout`, `mouse`, `open_actions`, `options`, `parser`, `screen`, `search_query_parser`, `shell_integration`, `shm`, `ssh`, `tui`, `utmp`. Every one of them executes `from . import BaseTest`, which runs `kitty_tests/__init__.py` and therefore loads `fast_data_types` transitively.

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

To capture the **real** chains (not a synthetic import), a non-invasive `sys.meta_path` finder recorded the repo-frame call stack at the **first** import of each target, then the **real** entry point `./test.py` was executed via `runpy.run_path(test_py, run_name='__main__')` so the frames include the genuine launcher preamble, `test.py:8`, and `find_all_tests()` at `main.py:64`. The finder returns `None` from `find_spec` (so the real import machinery still resolves everything). The temporary script:

```python
# /tmp/…/observe_importchain.py  (temporary; removed after the run)
import sys, os, runpy, traceback
from importlib.abc import MetaPathFinder
TARGETS = {'kitty.config','kitty.conf.utils','kitty.fast_data_types','kittens.transfer.rsync'}
_seen = set(); _out = open(os.environ['CHAIN_TRACE_FILE'], 'w')
class StackTracer(MetaPathFinder):
    def find_spec(self, fullname, path, target=None):
        if fullname in TARGETS and fullname not in _seen:
            _seen.add(fullname); _out.write(f'>>> FIRST IMPORT OF: {fullname}\n')
            for fr in traceback.extract_stack(): _out.write(f'    {fr.filename}:{fr.lineno} ({fr.name}) | {fr.line}\n')
            _out.write('\n'); _out.flush()
        return None                      # non-invasive: real finders resolve the import
sys.meta_path.insert(0, StackTracer())
sys.argv = [os.path.join(os.getcwd(),'test.py'), '--module', 'check_build']
runpy.run_path(sys.argv[0], run_name='__main__')   # the REAL entry point
```

Complete captured trace (identical across two runs; repo-path prefix shown as recorded):

```console
$ CHAIN_TRACE_FILE=/tmp/…/chain.txt CI=true LANG=C.UTF-8 LC_ALL=C.UTF-8 \
    ./kitty/launcher/kitty +launch /tmp/…/observe_importchain.py
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

The launcher's own stdout for that run (the filtered `check_build` suite still triggers discovery of **all** modules at `main.py:211`, which is why `rsync` loads even under `--module check_build`):

```console
(stdout of the same invocation)
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
Ran 9 tests in 0.106s

OK (skipped=1)
```

### 9.1 The two Python-extension chains, annotated

- **`kitty.fast_data_types` (bootstrap chain):**
  `kitty/launcher/../../__main__.py:7` `main()` → `kitty/entry_points.py:192` → `:146` → `:73` `runpy.run_path(...)` → `test.py:13` `main()` → **`test.py:8`** `importlib.import_module('kitty_tests.main')` → **`kitty_tests/__init__.py:21`** `from kitty.config import ...` → `kitty/config.py:10` `from .conf.utils import ...` → **`kitty/conf/utils.py:27`** `from ..fast_data_types import Color`. This confirms the root import is triggered by the **package** initializer reached directly from `test.py:8`; the `kitty_tests/main.py` module body is not on this chain.
- **`kittens.transfer.rsync` (discovery chain):**
  … → `test.py:9` `getattr(m,'main')()` → `kitty_tests/main.py:338` `run_tests()` → `main.py:279` `run_python_tests(args, go_proc)` → **`main.py:211`** `tests = find_all_tests()` → **`main.py:64`** `importlib.import_module(...)` → **`kitty_tests/file_transmission.py:13`** `from kittens.transfer.rsync import Differ, Hasher, Patcher, parse_ftc`.

### 9.2 The native GLFW chain (not an import)

`glfw-x11.so` establishes no Python import chain. Its runtime chain is the **native** one captured in §4.1: `kitty_tests/glfw.py:test_utf_8_strndup` → `kitty.constants.glfw_path('x11')` (`constants.py:191-193`) → `kitty_tests/glfw.py:50` `ctypes.CDLL(backend_utils)` → `dlopen(".../kitty/glfw-x11.so")`. The observed `ctypes.dlopen` audit event and the `/proc/self/maps` transition (`False` → `True`) in §4.1 are the evidence for this chain.

## Appendix — repository integrity and provenance

This QnA investigation is read-only with respect to tracked files: the **only** repository change is the addition/update of this document. The distinction between the two relevant commits is important:

- **Source commit under investigation:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` — the kitty code that all `file:line` citations refer to.
- **Destination delivery branch:** `blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9` — the branch this document is committed to. (The `go build` line at `build.log:319` embeds `-X kitty.VCSRevision=<this delivery commit>`, which is why the launcher's version string reflects the destination commit, not the source commit.)

The three compiled `*.so` are git-ignored generated artifacts (`.gitignore:1` = `*.so`). The two that were temporarily moved during §5 were restored **byte-for-byte**; the SHA-256 values below match the post-build baseline. The captured working-tree state at delivery, the restored-artifact hashes, and the confirmation that all `/tmp` observation scratch was removed:

```console
# Repository state at delivery (destination branch), captured with the commands shown.

$ git rev-parse --abbrev-ref HEAD          # destination delivery branch
blitzy-25deee7b-18fb-49cf-b74b-43873fbe40c9

$ git log -1 --format="%H  %s"             # source commit under investigation is 815df1e210e0…;
                                            # the delivery commit that adds this document is created on the branch above
c732e9295e99e1d7091953f9317997b35b7bc5aa  docs: add kitty C-extension vs test-suite QnA investigation

$ git status --porcelain                    # only the answer document is added/changed; no source file is touched
 M blitzy/documentation/kitty_815df1e210e0.md

$ sha256sum kitty/fast_data_types.so kitty/glfw-x11.so kittens/transfer/rsync.so   # git-ignored artifacts, restored byte-for-byte (== post-build baseline)
1dc3caa73dc80733b5d755ee37cf9261125f660ae551d5419c53549650f2d0ed  kitty/fast_data_types.so
99db5778b34c5370637a2fce743fd6379ebb879f9fbee34bb38f67f0cbd83a56  kitty/glfw-x11.so
c0bf7b038558d5dfabd176602d7a05458459bc635ee8dd869e1125695dea35a8  kittens/transfer/rsync.so

$ ls -d /tmp/blitzy_qa_obs.* /tmp/blitzy_qa_stash.* 2>/dev/null || echo "no temporary observation scratch remains under /tmp"
no temporary observation scratch remains under /tmp
```

### Coverage of the nine sub-questions

| # | Sub-question | Where answered | Key observed evidence |
|---|--------------|----------------|-----------------------|
| 1 | Build kitty from source (canonical) | §1, §1.3 | `CC=gcc-13 python3 setup.py build --verbose`, exit 0, `BUILD_REAL_SECONDS=60.871`; 3 artifacts on disk |
| 2 | Execute the test suite via its real entry point | §2.1, §2.2 | `./test.py` → `Ran 145 tests`, `FAILED (failures=4, skipped=4)`, `All Go tests succeeded` (×2 runs) |
| 3 | Trace the extension ↔ test-execution relationship | §3, §9 | root/discovery/native three-level gating; captured import trace |
| 4 | Which extension modules actually get loaded | §4, §4.1 | bootstrap=1 kitty ext; post-discovery 9 `.so` (2 kitty + 7 stdlib); glfw loaded natively (maps `False`→`True`) |
| 5 | How failures cascade when extensions are unavailable | §5.1–§5.3 | fast_data_types→0 tests; rsync→0 Python tests; glfw→2 tests break |
| 6 | What the output reveals about the dependency structure | §6 | one hard root, one discovery chokepoint, one native leaf; stage encodes coupling |
| 7 | Map extensions → test categories | §7.1, §7.2 | 22 modules; per-extension consumption table |
| 8 | Classify each extension CRITICAL vs OPTIONAL | §8 | fast_data_types & rsync CRITICAL; glfw-x11 OPTIONAL (native lib) |
| 9 | Document the actual import chains | §9.1, §9.2 | two real Python-extension chains + the native GLFW `ctypes` chain |

Named items explicitly covered: the three compiled extensions (`fast_data_types.so`, `glfw-x11.so`, `rsync.so`); the real entry point (`./test.py` → `kitty_tests.main`); `find_all_tests()`; the `itertests()` guard; `check_build` (`test_loading_extensions`, `test_loading_shaders`, `test_glfw_modules`); `file_transmission`; the `glfw.test_utf_8_strndup` native load; CI-vs-non-CI `test_glfw_modules`; and the environmental failures/skips.
