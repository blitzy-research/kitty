# How kitty handles a ZWJ multi-codepoint emoji in a 1×1 cell — and what a state-report query then says

- **Subject repository:** [kovidgoyal/kitty](https://github.com/kovidgoyal/kitty) @ commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
- **Investigation type:** read-only, *build-and-run-first*. The terminal engine was built and driven through its **real VT parser**; every behavioral claim below is backed by **byte-exact captured output** and a **`file:line` citation** naming the responsible function or struct.
- **Deliverable:** this single Markdown document. No existing repository file was modified, created, or deleted; the temporary observation scripts lived outside the repository tree (`/tmp/kitty_obs/`) and were removed afterward. `git diff --stat` between the subject commit and `HEAD` shows exactly one added file — this document (Section 2.6).

---

## Section 1 — The question, decomposed, with the direct answer up front

### 1.1 The question (verbatim)

> I am trying to get a practical understanding of how kitty handles complex Unicode at runtime, especially in cases where the screen state is hard to reason about just by reading the code. When the terminal receives a stream of zero width joiners that form a multi codepoint emoji, how does the internal screen buffer decide what to keep when there is almost no space available, such as a one by one cell? What does the terminal think is actually present in that cell once everything settles? If the terminal is then asked to report part of its current state through a control sequence query, what response does it generate and how does that reflect the earlier grapheme handling? I want to understand how normalization, grapheme breaking, and state reporting interact when the terminal is under extreme constraints. Temporary scripts may be used for observation, but the repository itself should remain unchanged and anything temporary should be cleaned up afterward.

### 1.2 Four decomposed objectives

- **OBJ-1 — Retention under constraint:** how the internal screen buffer decides *what to keep* when a ZWJ emoji stream arrives into a 1×1 cell.
- **OBJ-2 — Settled cell content:** what the terminal believes is *actually present* in the cell once the stream is fully processed.
- **OBJ-3 — State reporting:** what response a control-sequence state query generates, and *how that response reflects the earlier grapheme handling*.
- **OBJ-4 — Interaction:** how *normalization*, *grapheme breaking*, and *state reporting* interact under extreme constraint.

### 1.3 Direct answer (TL;DR)

**What the *visible* 1×1 cell keeps, and why it is not simple overwrite.** A width-2 emoji can never fit into a single column, so kitty's default autowrap mode (`DECAWM`, on by default — `kitty/screen.c:L33`) fires *before* each base emoji is placed. The autowrap does not silently discard the current line: it runs `continue_to_next_line → screen_linefeed → screen_index`, and because the cursor is already on the only (bottom) row, `screen_index` pushes the **current line into the scrollback history buffer** (`historybuf_add_line`, `kitty/screen.c:L1558`), clears the now-recycled visible row (`linebuf_clear_line`, `kitty/screen.c:L1565`), and only then writes the new base into the freshly cleared cell. Consequently, feeding the canonical family sequence `👨‍👩‍👧‍👦` into a 1×1 grid leaves the **visible cell** holding just the **final base emoji**, Boy `U+1F466` (`👦`) — but the earlier bases (`👨`, `👩`, `👧`), each still carrying its trailing ZWJ, are **retained in the scrollback history**, one per history line, not destroyed. Each ZWJ (`U+200D`) is a codepoint that kitty classifies as *combining/appendable* and attaches to whichever base currently occupies the cell, so a ZWJ never triggers a wrap of its own.

**What the terminal thinks is present once it settles (OBJ-2).** The visible cell reports exactly **one** codepoint — `U+1F466` — because the family sequence ends on a base with no trailing ZWJ. (`str(line)` = `'👦'`, `len == 1`.) The scrollback history simultaneously holds `['👧\u200d', '👩\u200d', '👨\u200d', '']` (Section 4).

**What the state report says, and how it reflects the handling (OBJ-3).** After the stream the cursor sits at **`x=2`** (a pending column one past the single column). A Cursor-Position-Report query `CSI 6 n` replies with the exact bytes **`b'\x1b[1;2R'`**. This is *not* a "clamp to the last real column"; `report_device_status` (`kitty/screen.c:L2188-L2198`) sees the out-of-range pending column (`x >= self->columns`), takes the bottom-row branch that **decrements `x` once** (`2 → 1`), then emits it 1-based as column **2**. The report therefore conveys only *how many columns the last base consumed* (a width consequence); it carries **no** information identifying which codepoints were kept or scrolled into history.

**How normalization, grapheme breaking, and state reporting interact (OBJ-4).** On this path kitty performs **no Unicode normalization** (no NFC/NFD/NFKC/NFKD pass) and runs **no grapheme-segmentation (UAX #29) state machine** — verified by a scoped source search that returns zero matches in `kitty/vt-parser.c`, `kitty/screen.c`, `kitty/line.c` (Section 7). Every decision is made by **width classification** (`wcwidth_std`) plus **combining-character membership** (`is_combining_char`), driven by generated **Unicode 15.0.0** tables. So the family emoji is never assembled into one grapheme in the buffer; it is decomposed by width, the visible cell keeps the last base, history keeps the rest, and the state report merely echoes the resulting cursor column.

> **Observed vs. inferred vs. standards.** The runtime *values* above (settled cell, history contents, cursor column, exact reply bytes) are **observed** — captured byte-for-byte in Sections 4–6 and re-verified by 46 in-script assertions. The *mechanism* statements (autowrap → index → history push → clear → write; the decrement branch; the absence of normalization/segmentation) are **inferred from the cited source** and corroborated by the observed values. The ZWJ/emoji-ZWJ, UAX #29, and DSR/CPR definitions are **external Unicode/VT standards** background. Section 8 classifies every key statement explicitly.

---

## Section 2 — Environment and build

### 2.1 Versions actually observed in this environment

| Item | Value (observed here) | How obtained |
|------|-----------------------|--------------|
| CPython | **3.13.7** | `python3 --version` inside the venv `/tmp/kitty-venv` |
| C compiler | **gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0** | `gcc --version` |
| Unicode data version | **15.0.0** | first line of `kitty/unicode-data.c` (see below) |
| Repository commit | `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` | `git rev-parse HEAD` |
| Intended reproduction image | `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (from `ghcr.io/scaleapi/swe-atlas`) | provided environment |

> **Note on versions (observed vs. planning reference):** the task's planning notes referenced CPython 3.12.3 / gcc 13.3.0. The actual host used for this investigation runs **CPython 3.13.7 / gcc 15.2.0**. Per the rule to *build and run in the default configuration and report the value that produces*, the versions above are the ones actually observed. The behavior under investigation is decided by the C source logic and the **source-baked Unicode 15.0.0 tables** rather than by the interpreter or compiler, so it is not expected to vary with the Python or gcc version; empirically, on **this** observed toolchain the captured output was **byte-identical across two runs** (Section 2.5). This document reports the values produced by this toolchain and does **not** claim to have independently tested other Python or gcc versions.

The Unicode provenance is stated in the generated data file itself — `kitty/unicode-data.c:L1`:

```c
// Unicode data, built from the Unicode Standard 15.0.0
// Code generated by gen-wcwidth.py, DO NOT EDIT.
```

### 2.2 Build command — canonical, and what actually happened here

The documented, **canonical** build command for kitty's C extension is:

```bash
python3 setup.py build
```

**Observed result in this environment: the canonical command SUCCEEDED with exit code `0`.** No warning-suppression workaround was necessary here. The reason is that `wayland-protocols` is **not installed** on this host, so kitty's build system disables the Wayland (GLFW) backend and never compiles the GUI/windowing code that is the source of the warning discussed below. The head of the captured build log:

```text
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
```

The exact clean-build command used here (the gitignored `build/` directory, `kitty/fast_data_types.so`, and `kitty/launcher/kitty` were removed first so the compile is from scratch), and its **complete, unedited** output — captured with `2>&1` and terminated by the shell's `echo "EXIT_CODE=$?"`:

```bash
$ source /tmp/kitty-venv/bin/activate
$ rm -rf build kitty/fast_data_types.so kitty/launcher/kitty \
    && python3 setup.py build > /tmp/kitty_obs/build_canonical.log 2>&1; echo "EXIT_CODE=$?"
```

```text
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
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
kitty/tools/cmd
EXIT_CODE=0
```

The build compiled **85 objects** and linked **4 targets**, ending in `done` with **`EXIT_CODE=0`**. Note that only the **X11** GLFW backend is compiled/linked (`[x11]` steps and `kitty/glfw-x11`); there is **no `glfw-wayland`** step, which follows directly from Wayland being absent here. The trailing `kitty/tools/cmd` line is the Go-tools build step, which is not required by the terminal engine this document exercises. Crucially, the compile ran under kitty's **default `-pedantic-errors -Werror`** (the `--ignore-compiler-warnings` flag was **not** passed — confirmed below), so the terminal-engine translation units (`kitty/screen.c`, `kitty/line.c`, `kitty/unicode-data.c`, `kitty/vt-parser.c`) all compiled warning-clean at the strictest setting.

```bash
$ grep -c 'ignore-compiler-warnings' /tmp/kitty_obs/build_canonical.log
0            # the warning-suppression flag was NOT used; build ran with default -Werror
```

### 2.3 The sandbox-only warning workaround (disclosed for reproducibility)

On a host **where `wayland-protocols` *is* present** (as in the original planning sandbox), the full default build additionally compiles the Wayland GUI code under `-pedantic-errors -Werror`, and that compile can fail on an unrelated `-Werror=switch` warning coming from a newer system copy of `wayland-protocols`:

```text
glfw/wl_window.c:668:9: error: enumeration value 'XDG_TOPLEVEL_STATE_CONSTRAINED_RIGHT' not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value 'XDG_TOPLEVEL_STATE_CONSTRAINED_TOP' not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value 'XDG_TOPLEVEL_STATE_CONSTRAINED_BOTTOM' not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
```

On such a host the documented workaround is:

```bash
export CI=true
python3 setup.py build --debug --ignore-compiler-warnings
```

The `--ignore-compiler-warnings` flag flips exactly one line in the build system — `setup.py:L491`:

```python
werror = '' if ignore_compiler_warnings else '-pedantic-errors -Werror'
```

This flag controls **only whether warnings are fatal**; it does **not** change the terminal engine's logic. The affected file (`glfw/wl_window.c`) is GUI/windowing code — the terminal engine proper (`kitty/screen.c`, `kitty/line.c`, `kitty/unicode-data.c`) compiles cleanly with or without it. **Therefore the observed Unicode/state-report behavior in this document is independent of the warning flag**, whether the build succeeds outright (as here) or requires the workaround (as on a Wayland-equipped host).

### 2.4 Import check (the engine we exercise)

The terminal engine is the CPython extension `kitty.fast_data_types`, which provides the `Screen`, `Line`, and `Cursor` types that this investigation drives:

```bash
$ PYTHONPATH=. python3 -c "import kitty.fast_data_types as f; from kitty.fast_data_types import Screen, Line, Cursor; print('import OK ->', f.__file__)"
import OK -> /tmp/blitzy/kitty/blitzy-8842e805-1a59-4efe-ad77-f5c3cfe88349_b1d24d/kitty/fast_data_types.so
```

### 2.5 Reproducibility — two byte-identical runs

The observation script (Section 10) was executed **twice** from the repository root; the two output files were then compared with `diff` and hashed with both MD5 and SHA-256:

```bash
$ PYTHONPATH=. python3 /tmp/kitty_obs/observe.py > /tmp/kitty_obs/run1.txt 2>&1; echo "EXIT=$?"
EXIT=0
$ PYTHONPATH=. python3 /tmp/kitty_obs/observe.py > /tmp/kitty_obs/run2.txt 2>&1; echo "EXIT=$?"
EXIT=0
$ diff /tmp/kitty_obs/run1.txt /tmp/kitty_obs/run2.txt && echo "IDENTICAL (diff empty)"
IDENTICAL (diff empty)
$ md5sum /tmp/kitty_obs/run1.txt /tmp/kitty_obs/run2.txt
77a2325fbe5315bbbf5ff19ad1d77f57  /tmp/kitty_obs/run1.txt
77a2325fbe5315bbbf5ff19ad1d77f57  /tmp/kitty_obs/run2.txt
$ sha256sum /tmp/kitty_obs/run1.txt /tmp/kitty_obs/run2.txt
d26db8c171e8deef9e0bcdebb7e6a40c8e4c6b7d31bd94e51160c729732b1101  /tmp/kitty_obs/run1.txt
d26db8c171e8deef9e0bcdebb7e6a40c8e4c6b7d31bd94e51160c729732b1101  /tmp/kitty_obs/run2.txt
```

The two runs are **byte-for-byte identical** (empty `diff`; identical MD5 `77a2325f…` and SHA-256 `d26db8c1…`), so every value reported in this document is stable across repeated identical runs. In addition, the script embeds **46 golden assertions** that compare each observed value against a hard-coded expected value and exit non-zero on any mismatch; both runs report `ASSERTIONS: 46 passed, 0 failed (of 46)` (full assertion block in Section 10).

### 2.6 Repository integrity — the source tree is unchanged

The build artifacts (`kitty/fast_data_types.so`, `kitty/launcher/kitty`, the `build/` tree) are gitignored, and the temporary observation scripts live entirely outside the repository under `/tmp/kitty_obs/`. The subject source tree is therefore left byte-for-byte unchanged; the only tracked change relative to the subject commit is this one document:

```bash
$ grep -nE '\.so|/build/|launcher' .gitignore
1:*.so
14:/build/
18:/kitty/launcher/kitt*

$ git merge-base --is-ancestor 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD && echo "subject commit is an ancestor of HEAD"
subject commit is an ancestor of HEAD

$ git diff --stat 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD
 blitzy/documentation/kitty_815df1e210e0.md | 1796 ++++++++++++++++++++++++++++
 1 file changed, 1796 insertions(+)

$ git status --porcelain      # after the rebuild and both observation runs
                              # (empty output = clean working tree; artifacts are gitignored)
```

The temporary scripts and their output are deleted at the end of the investigation:

```bash
$ rm -rf /tmp/kitty_obs
$ ls -d /tmp/kitty_obs 2>&1
ls: cannot access '/tmp/kitty_obs': No such file or directory
```

(The `git diff --stat` line count above reflects this document's final size; the byte counts in `.gitignore` line references are exact at the subject commit.)

---

## Section 3 — Methodology: the canonical entry point

**Direct statement:** every input in this investigation flows into the terminal engine as **UTF-8 bytes through the real VT parser** — never through a convenience/direct-draw API, remote control, or a debug hook. This satisfies the requirement that the observed values be produced by the exact code path a real program would exercise.

### 3.1 The byte path

The driver is `kitty_tests.parse_bytes` — `kitty_tests/__init__.py:L30-L36`:

```python
def parse_bytes(screen, data, dump_callback=None):
    data = memoryview(data)
    while data:
        dest = screen.test_create_write_buffer()
        s = screen.test_commit_write_buffer(data, dest)
        data = data[s:]
        screen.test_parse_written_data(dump_callback)
```

The three `screen.test_*` methods are thin C bindings that push bytes into the genuine parser rather than around it:

- `test_create_write_buffer` — `kitty/screen.c:L4755`
- `test_commit_write_buffer` — `kitty/screen.c:L4762`
- `test_parse_written_data` — `kitty/screen.c:L4772`

The bytes are decoded and dispatched by the real byte-stream state machine in `kitty/vt-parser.c`; drawn text is routed to `screen_draw_text`, which contains the per-codepoint draw loop cited throughout Section 4.

### 3.2 Capturing control-sequence replies

Replies that the engine emits "to the child" (e.g., a Cursor Position Report) are captured by the test `Callbacks` object, whose `write` method accumulates them — `kitty_tests/__init__.py:L50-L51`:

```python
    def write(self, data) -> None:
        self.wtcbuf += bytes(data)
```

A query such as `CSI 6 n` is itself fed **through `parse_bytes`** (i.e., through the real parser), and the reply is read back byte-for-byte from `callbacks.wtcbuf`. This is why the reports in Section 6 are shown as literal `bytes` objects.

### 3.3 Canonical option initialization

A `Screen` reads process-wide options through `get_options()`, which **raises** `RuntimeError('Must call set_options() before using get_options()')` if options were never installed. The canonical way kitty's own test harness installs them is `BaseTest.set_options` — `kitty_tests/__init__.py:L223-L231`:

```python
    def set_options(self, options=None):
        final_options = {'scrollback_pager_history_size': 1024, 'click_interval': 0.5}
        if options:
            final_options.update(options)
        options = Options(merge_result_dicts(defaults._asdict(), final_options))
        finalize_keys(options, {})
        finalize_mouse_mappings(options, {})
        set_options(options)
        return options
```

The observation script mirrors this exactly (it builds `Options` from the shipped `defaults`, finalizes key/mouse mappings, and calls `set_options`) so the engine runs under kitty's **default configuration**. As a guard, the script prints the type of `get_options()` after initialization; the captured output begins with `get_options() type after canonical init = Options`, confirming the default options are installed rather than a stub.

### 3.4 Building a 1×1 (and wider) `Screen`

The `Screen` constructor argument order is mirrored from `kitty_tests.create_screen` — `kitty_tests/__init__.py:L237-L240`:

```python
    def create_screen(self, cols=5, lines=5, scrollback=5, cell_width=10, cell_height=20, options=None):
        self.set_options(options)
        c = Callbacks()
        s = Screen(c, lines, cols, scrollback, cell_width, cell_height, 0, c)
        return s
```

So the order is `Screen(callbacks, LINES, COLS, scrollback, cell_width, cell_height, 0, callbacks)`. This investigation constructs a **1×1** screen as `Screen(cb, 1, 1, 5, 10, 20, 0, cb)` — 1 line, 1 column, **scrollback = 5** (the harness default), cell size 10×20 px. The non-zero scrollback is deliberate and important for OBJ-1: it lets the scrollback **history buffer** retain the lines that autowrap scrolls off the single visible row, so we can observe *where the earlier bases went* rather than assuming they were destroyed (Section 4). The `cell_width=10, cell_height=20` choice is why the pixel-size reports in Section 6 come out as multiples of 10 and 20.

### 3.5 A note on the direct-draw API (non-canonical)

kitty's own existing tests in `kitty_tests/screen.py` frequently call `s.draw(<str>)` for convenience — for example `test_emoji_skin_tone_modifiers`, `test_regional_indicators`, and `test_variation_selectors`. `s.draw(<str>)` is the **direct draw API**: it takes an already-decoded Python `str` and bypasses the UTF-8 byte parser. Any value obtained that way would be **non-canonical** for this question. This document deliberately uses UTF-8 **bytes** through `parse_bytes` instead, so that the observed values are exactly what the byte-stream parser produces.

---

## Section 4 — Part 1: What the buffer keeps (retention under constraint) → OBJ-1

**Direct answer:** retention is governed by **two independent mechanisms**, and it is important not to conflate them:

1. **Within a single line/cell**, capacity is fixed — one base codepoint plus **three** combining slots (`CPUCell.cc_idx[3]`). Any codepoint kitty classifies as *combining/appendable* (which includes ZWJ `U+200D`) is **appended** into the current base's slots rather than being given its own cell.
2. **Across lines**, a width-2 base emoji cannot fit into a single column, so before it is written kitty's default **autowrap** (`DECAWM`) scrolls the current visible line off the screen. On a one-row grid that scroll pushes the line into the **scrollback history buffer** and clears the visible row; the new base is then written into the cleared cell.

So a ZWJ family stream into a **1×1** grid does **not** collapse by in-place overwrite. Each successive base triggers an autowrap that **preserves the prior line in scrollback history**, then occupies the freshly cleared visible cell. When the stream settles, the **visible** cell holds only the **final base** (Boy, `U+1F466`) while the earlier bases — `👨`, `👩`, `👧`, each still carrying the ZWJ that followed it — remain **in history**, one per line. The ZWJ never triggers a wrap of its own; it is appended to whichever base currently occupies the visible cell.

### 4.1 The fixed-capacity cell model

The CPU-side per-cell structure stores one base `ch` plus a **3-element** combining array — `kitty/data-types.h:L223-L228`:

```c
typedef struct {
    char_type ch;
    hyperlink_id_type hyperlink_id;
    combining_type cc_idx[3];
} CPUCell;
static_assert(sizeof(CPUCell) == 12, "Fix the ordering of CPUCell");
```

This is the entire basis of "what can be kept" **within one cell**: **one** base codepoint (`ch`) and **at most three** appended codepoints (`cc_idx[3]`). There is no per-cell growable list. (This is mechanism #1 from the direct answer; mechanism #2 — autowrap into history across lines — is §4.4.)

### 4.2 The retention / overflow decision

The routine that decides how a combining mark is stored is `line_add_combining_char` — `kitty/line.c:L456-L467`:

```c
void
line_add_combining_char(CPUCell *cpu_cells, GPUCell *gpu_cells, uint32_t ch, unsigned int x) {
    CPUCell *cell = cpu_cells + x;
    if (!cell->ch) {
        if (x > 0 && (gpu_cells[x-1].attrs.width) == 2 && cpu_cells[x-1].ch) cell = cpu_cells + x - 1;
        else return; // don't allow adding combining chars to a null cell
    }
    for (unsigned i = 0; i < arraysz(cell->cc_idx); i++) {
        if (!cell->cc_idx[i]) { cell->cc_idx[i] = mark_for_codepoint(ch); return; }
    }
    cell->cc_idx[arraysz(cell->cc_idx) - 1] = mark_for_codepoint(ch);
}
```

Three behaviors matter for OBJ-1:

1. It targets the cell at `cpu_cells + x`.
2. It **refuses to attach an appended codepoint to a null (empty) cell** — *unless* the previous cell is a **width-2 base** (`x > 0 && width == 2 && cpu_cells[x-1].ch`), in which case it retargets onto that wide base. This is exactly how a codepoint following the trailing (right) half of a wide emoji still "belongs" to the base to its left.
3. It **fills the first empty `cc_idx` slot**; and once **all three** are full, the fourth (and any later) appended codepoint **overwrites the last slot** `cc_idx[arraysz(cc_idx)-1]` (i.e. `cc_idx[2]`). This is the overflow rule exercised with an explicit `A + 4 marks` sequence in **Section 7.5**.

> **Terminology (per Unicode).** kitty's "combining" class here is an *engine* classification of codepoints that get **appended** to a base rather than occupying their own cell. It is broader than the Unicode general category *Mark* (Mn/Mc/Me): it also includes ZWJ `U+200D` (general category **Cf**, *Format*), the variation selectors `U+FE0E`/`U+FE0F` (**Mn**), regional-indicator symbols (**So**), and emoji skin-tone modifiers `U+1F3FB–U+1F3FF` (**Sk**). Throughout this document, "appended codepoint" / "stored in a combining slot" refers to this engine class; where a codepoint's Unicode category matters it is named explicitly.

### 4.3 Where a ZWJ is routed (not given its own cell)

In the per-codepoint draw loop (inside `screen_draw_text`), a combining codepoint never occupies a fresh cell — `kitty/screen.c:L805-L812`:

```c
            if (is_ignored_char(ch)) continue;
            if (UNLIKELY(is_combining_char(ch))) {
                if (UNLIKELY(is_flag_codepoint(ch))) {
                    if (draw_second_flag_codepoint(self, ch)) continue;
                } else {
                    draw_combining_char(self, s, ch);
                    continue;
                }
```

- `is_ignored_char(ch)` (`kitty/unicode-data.c:L671`) is the gate that would *drop* a codepoint entirely; ZWJ is **not** in it (see Section 7).
- `is_combining_char(ch)` (`kitty/unicode-data.c:L11`) is **true** for ZWJ `U+200D` (see Section 7), so ZWJ takes the combining branch and is handed to `draw_combining_char` — it is **appended**, never allocated a new cell.

`draw_combining_char` (`kitty/screen.c:L663-L710`) locates the "previous" cell — the cell at `cursor->x - 1` on the current row, or, when the cursor is at column 0, the **last column of the previous row** (`ypos = cursor->y - 1`, `xpos = columns - 1`) — and then calls `line_add_combining_char(cp, gp, ch, xpos)` to append the codepoint into that base's slots. (The complete function body, including the VS16/VS15 width-flip branches, is shown and analysed in **Section 7.2**; it is not excerpted here to avoid eliding any logic.) In the family stream each ZWJ arrives while the cursor is at `x=2` (just past a width-2 base occupying the single column), so `xpos = 1` — the position of the base's *trailing* half. `line_add_combining_char` finds that trailing half empty but sees that the cell to its left is a width-2 base, so it **retargets** the append onto the base itself (the `x > 0 && width == 2 && cpu_cells[x-1].ch` guard, §4.2). This is why the observed cell shows `['U+1F468', 'U+200D']` — the ZWJ landed in the Man base's first combining slot — with the cursor unchanged at `x=2`.

### 4.4 What actually happens to each width-2 base: autowrap into history (not in-place overwrite)

Each **base** emoji in the family sequence (`👨`, `👩`, `👧`, `👦`) is a width-2 codepoint. Its width is computed at the top of the draw loop by `wcwidth_std(ch)` — `kitty/screen.c:L814` (the function itself is defined in `kitty/wcwidth-std.h:L9-L10`; see Section 7). The decisive point is that a width-2 glyph **cannot fit into a one-column grid**, so the "does it fit?" test fires for *every* base — including the very first, because `columns (1) < cursor->x (0) + char_width (2)` is already true. The relevant draw-loop fragment is `kitty/screen.c:L820-L843`:

```c
        self->last_graphic_char = ch;
        if (UNLIKELY(self->columns < self->cursor->x + (unsigned int)char_width)) {
            if (self->modes.mDECAWM) {
                continue_to_next_line(self);
                init_text_loop_line(self, s);
            } else {
                self->cursor->x = self->columns - char_width;
                if (cursor_on_wide_char_trailer(self, s)) move_cursor_off_wide_char_trailer(self, s);
            }
        }
        if (self->modes.mIRM) line_right_shift(self->linebuf->line, self->cursor->x, char_width);
        if (UNLIKELY(!s->image_placeholder_marked && ch == IMAGE_PLACEHOLDER_CHAR)) {
            linebuf_set_line_has_image_placeholders(self->linebuf, self->cursor->y, true);
            s->image_placeholder_marked = true;
        }
        zero_cells(s, s->cp + self->cursor->x, s->gp + self->cursor->x);
        s->cp[self->cursor->x].ch = ch;
        self->cursor->x++;
        if (char_width == 2) {
            s->gp[self->cursor->x-1].attrs.width = 2;
            zero_cells(s, s->cp + self->cursor->x, s->gp + self->cursor->x);
            s->gp[self->cursor->x].attrs.width = 0;
            self->cursor->x++;
        }
```

Because **`DECAWM` is on by default** (`empty_modes = {0, .mDECAWM=true, …}` — `kitty/screen.c:L33`), the `if (self->modes.mDECAWM)` branch is taken, calling **`continue_to_next_line`** (`kitty/screen.c:L823`). That is the entire retention mechanism, and it is worth following to the leaf because it is what preserves the earlier bases:

```c
// kitty/screen.c:L523-L528
static void
continue_to_next_line(Screen *self) {
    linebuf_set_last_char_as_continuation(self->linebuf, self->cursor->y, true);
    self->cursor->x = 0;
    screen_linefeed(self);
}

// kitty/screen.c:L1642-L1648
void
screen_linefeed(Screen *self) {
    bool in_margins = cursor_within_margins(self);
    screen_index(self);
    if (self->modes.mLNM) screen_carriage_return(self);
    screen_ensure_bounds(self, false, in_margins);
}

// kitty/screen.c:L1569-L1577
void
screen_index(Screen *self) {
    // Move cursor down one line, scrolling screen if needed
    unsigned int top = self->margin_top, bottom = self->margin_bottom;
    if (self->cursor->y == bottom) {
        const bool add_to_history = self->linebuf == self->main_linebuf && self->margin_top == 0;
        INDEX_UP(add_to_history);
    } else screen_cursor_down(self, 1);
}
```

On a **one-row** grid the cursor is always on the bottom line (`cursor->y == bottom`), so `screen_index` takes the `INDEX_UP(add_to_history)` path with `add_to_history == true` (main line buffer, no top margin). `INDEX_UP` (`kitty/screen.c:L1552-L1567`) is the macro that both **saves the outgoing line to scrollback** and **clears the recycled visible row**:

```c
// kitty/screen.c:L1552-L1567
#define INDEX_UP(add_to_history) \
    linebuf_index(self->linebuf, top, bottom); \
    INDEX_GRAPHICS(-1) \
    if (add_to_history) { \
        /* Only add to history when no top margin has been set */ \
        linebuf_init_line(self->linebuf, bottom); \
        historybuf_add_line(self->historybuf, self->linebuf->line, &self->as_ansi_buf); \
        self->history_line_added_count++; \
        if (self->last_visited_prompt.is_set) { \
            if (self->last_visited_prompt.scrolled_by < self->historybuf->count) self->last_visited_prompt.scrolled_by++; \
            else self->last_visited_prompt.is_set = false; \
        } \
    } \
    linebuf_clear_line(self->linebuf, bottom, true); \
    self->is_dirty = true; \
    index_selection(self, &self->selections, true);
```

So the sequence for each base is: **autowrap test true → `continue_to_next_line` → `screen_linefeed` → `screen_index` → `INDEX_UP(true)` → `historybuf_add_line` (`kitty/screen.c:L1558`) pushes the current visible line into the scrollback history buffer → `linebuf_clear_line` (`kitty/screen.c:L1565`) blanks the visible row → control returns to the draw loop, which writes the base into the freshly-cleared cell (`s->cp[self->cursor->x].ch = ch` — `kitty/screen.c:L836`) and advances the cursor to `x=2` (`kitty/screen.c:L837-L842`).**

The consequence for OBJ-1: the earlier base+ZWJ content is **not** overwritten in place and is **not** lost — it is moved, intact, into the scrollback history one line at a time. The single **visible** cell always ends up holding just the most-recent base. In the family sequence the final base (Boy, `U+1F466`) has **no** trailing ZWJ, so the settled visible cell holds exactly `U+1F466`, while history holds Girl+ZWJ, Woman+ZWJ, Man+ZWJ (and the initial blank line). Section 4.5 shows this happening step by step.

### 4.5 OBSERVED: the incremental before / during / after trace (1×1), including history

This is the single most important piece of evidence for OBJ-1. The family stream was fed **one codepoint at a time** into a fresh 1×1 `Screen` (**scrollback = 5**), printing the **visible cell and the scrollback history buffer** after each step. Command (the full script is in Section 10):

```text
FAMILY = U+1F468 U+200D U+1F469 U+200D U+1F467 U+200D U+1F466
each chunk fed via: parse_bytes(s, chunk.encode('utf-8'))   # real VT parser
history read via:   s.historybuf.line(i) for i in range(s.historybuf.count)
```

Complete, unedited output:

```text
   [before (empty screen)]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
      historybuf    = count=0  lines=[]
   [after Man U+1F468 (width2 base)  <-- fed bytes b'\xf0\x9f\x91\xa8']
      str(line0)   = '👨'
      codepoints    = ['U+1F468']  (len=1)
      as_ansi()     = '👨'
      cursor        = (x=2, y=0)
      historybuf    = count=1  lines=['']
   [after ZWJ U+200D  <-- fed bytes b'\xe2\x80\x8d']
      str(line0)   = '👨\u200d'
      codepoints    = ['U+1F468', 'U+200D']  (len=2)
      as_ansi()     = '👨\u200d'
      cursor        = (x=2, y=0)
      historybuf    = count=1  lines=['']
   [after Woman U+1F469 (new width2 base)  <-- fed bytes b'\xf0\x9f\x91\xa9']
      str(line0)   = '👩'
      codepoints    = ['U+1F469']  (len=1)
      as_ansi()     = '👩'
      cursor        = (x=2, y=0)
      historybuf    = count=2  lines=['👨\u200d', '']
   [after ZWJ U+200D  <-- fed bytes b'\xe2\x80\x8d']
      str(line0)   = '👩\u200d'
      codepoints    = ['U+1F469', 'U+200D']  (len=2)
      as_ansi()     = '👩\u200d'
      cursor        = (x=2, y=0)
      historybuf    = count=2  lines=['👨\u200d', '']
   [after Girl U+1F467 (new width2 base)  <-- fed bytes b'\xf0\x9f\x91\xa7']
      str(line0)   = '👧'
      codepoints    = ['U+1F467']  (len=1)
      as_ansi()     = '👧'
      cursor        = (x=2, y=0)
      historybuf    = count=3  lines=['👩\u200d', '👨\u200d', '']
   [after ZWJ U+200D  <-- fed bytes b'\xe2\x80\x8d']
      str(line0)   = '👧\u200d'
      codepoints    = ['U+1F467', 'U+200D']  (len=2)
      as_ansi()     = '👧\u200d'
      cursor        = (x=2, y=0)
      historybuf    = count=3  lines=['👩\u200d', '👨\u200d', '']
   [after Boy U+1F466 (final width2 base)  <-- fed bytes b'\xf0\x9f\x91\xa6']
      str(line0)   = '👦'
      codepoints    = ['U+1F466']  (len=1)
      as_ansi()     = '👦'
      cursor        = (x=2, y=0)
      historybuf    = count=4  lines=['👧\u200d', '👩\u200d', '👨\u200d', '']
```

**Reading the trace (this is the "what to keep" decision made visible):**

- **before** — the cell is empty (`len=0`, cursor `x=0`, history empty).
- **first base placed → initial blank line scrolled to history** — after Man, the visible cell holds one codepoint `U+1F468`, the cursor is at `x=2`, and `historybuf.count` has already grown to **1** (`['']`). That single history line is the *empty* row that was displaced when Man's autowrap fired (§4.4): a width-2 base cannot fit column 0, so even the first base scrolls the (blank) current line into history before being written.
- **ZWJ appended (no wrap)** — after the ZWJ, the visible cell holds **two** codepoints `['U+1F468', 'U+200D']` and history is unchanged (`count=1`): the ZWJ was appended into the Man base's first combining slot (§4.2–4.3), not given its own cell, and being zero-width it triggers no autowrap; the cursor stays at `x=2`.
- **next base scrolls the previous line into history** — after Woman, the visible cell shows a **single** codepoint `U+1F469`, and `historybuf` now holds `['👨\u200d', '']`: the Man+ZWJ line was **not** destroyed — it was pushed into scrollback (index 0) by the autowrap that preceded Woman, and the visible row was cleared before Woman was written.
- this base→ZWJ→autowrap cycle repeats for Girl and Boy, each time prepending the outgoing visible line to history.
- **settled** — after the final base (Boy, `U+1F466`) with **no** trailing ZWJ, the **visible** cell keeps exactly `['U+1F466']`, while the **history** holds `['👧\u200d', '👩\u200d', '👨\u200d', '']` — Girl, Woman, Man (each with its trailing ZWJ) plus the original blank line, most-recent first.

So the retention rule under the 1×1 constraint is: *the visible cell keeps the most-recent width-2 base plus any zero-width codepoints appended after it (up to three combining slots); every earlier base+ZWJ line is preserved in the scrollback history buffer via autowrap, not overwritten.* The settled answer to "what does the terminal think is in the cell" (OBJ-2) is therefore about the **visible** cell specifically — one codepoint, `U+1F466` — a distinction made explicit in Section 5.

> **Observed vs. inferred (this section).** The bracketed trace above — every `str(line0)`, `codepoints`, `cursor`, and `historybuf` value — is **observed** (captured byte-for-byte and re-checked by the golden assertions `fam_incr_visible`, `fam_incr_cursor`, `fam_incr_hist` in Section 10). The *explanation* tying each history growth to the autowrap→`screen_index`→`INDEX_UP`→`historybuf_add_line` chain (§4.4) is **inferred from the cited source**, and it is exactly consistent with the observed `historybuf.count` incrementing on each base and never on a ZWJ.


---

## Section 5 — Part 2: Settled cell content → OBJ-2

**Direct answer:** once the family ZWJ stream settles into the 1×1 cell, the terminal believes exactly **one** codepoint is present: **`U+1F466`** (`👦`, Boy). Length is 1.

### 5.1 How the settled content is read (canonical text accessors)

The cell was read through the two text accessors any consumer uses. They are **distinct code paths** — a point worth getting right, because they do *not* share a single serializer:

- **`str(line)`** → `__str__` (`kitty/line.c:L443-L444`) → `line_as_unicode` (`kitty/line.c:L282-L284`) → the shared helper `unicode_in_range` (`kitty/line.c:L253`), whose per-cell step calls **`cell_as_unicode`** at `kitty/line.c:L271`. `cell_as_unicode` (`kitty/line.c:L200-L207`) emits the base `ch` then up to three appended codepoints:

```c
size_t
cell_as_unicode(CPUCell *cell, bool include_cc, Py_UCS4 *buf, char_type zero_char) {
    size_t n = 1;
    buf[0] = cell->ch ? cell->ch : zero_char;
    if (include_cc) {
        for (unsigned i = 0; i < arraysz(cell->cc_idx) && cell->cc_idx[i]; i++) buf[n++] = codepoint_for_mark(cell->cc_idx[i]);
    }
    return n;
}
```

- **`line.as_ansi()`** → `as_ansi` (`kitty/line.c:L416-L424`) → `line_as_ansi` (`kitty/line.c:L338-L410`). This function does **not** call `cell_as_unicode`; it has its **own inline cell loop** that writes the base with `WRITE_CH(ch)` (`kitty/line.c:L392`) and then walks the same three `cc_idx` slots in its own loop (`kitty/line.c:L399-L401`):

```c
        WRITE_CH(ch);
        if (ch == '\t') {
            unsigned num_cells_to_skip_for_tab = self->cpu_cells[pos].cc_idx[0];
            while (num_cells_to_skip_for_tab && pos + 1 < limit && self->cpu_cells[pos+1].ch == ' ') {
                num_cells_to_skip_for_tab--; pos++;
            }
        } else {
            for(unsigned c = 0; c < arraysz(self->cpu_cells[pos].cc_idx) && self->cpu_cells[pos].cc_idx[c]; c++) {
                WRITE_CH(codepoint_for_mark(self->cpu_cells[pos].cc_idx[c]));
            }
        }
```

Both paths independently produce the same "base then whatever appended codepoints are non-empty" form because both stop at the first empty `cc_idx` slot (`… && cell->cc_idx[i]`), which is why `str(line)` and `as_ansi()` agree in the observed output below. (The UTF-8 equivalent used elsewhere is `cell_as_utf8` — `kitty/line.c:L222-L232`.)

### 5.2 OBSERVED: the settled cell

Command:

```text
s, c = new_screen(1, 1)                       # Screen(cb, 1, 1, 5, 10, 20, 0, cb)  scrollback=5
parse_bytes(s, FAMILY.encode('utf-8'))        # whole stream at once, real VT parser
```

Complete, unedited output:

```text
   [before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
      historybuf    = count=0  lines=[]
   [after whole family stream (settled)]
      str(line0)   = '👦'
      codepoints    = ['U+1F466']  (len=1)
      as_ansi()     = '👦'
      cursor        = (x=2, y=0)
      historybuf    = count=4  lines=['👧\u200d', '👩\u200d', '👨\u200d', '']
```

The settled **visible** cell is `'👦'`, its codepoint list is `['U+1F466']`, `len == 1`, and `as_ansi()` agrees. This is exactly what OBJ-2 asks for: what the terminal believes is present in the (one and only) visible cell once everything settles.

Two qualifications keep this answer precise and prevent it from being misread:

- **"Settled cell content" means the *visible* cell, not the whole stream.** The three earlier width-2 bases (`👨`, `👩`, `👧`) were not discarded — they were scrolled into the history buffer by the autowrap mechanism of §4.4, and the `historybuf` snapshot above shows them verbatim (`['👧\u200d', '👩\u200d', '👨\u200d', '']`, most-recent first, each still carrying its trailing ZWJ `U+200D`). Only the *final* base survives in the on-screen cell. See §4.4–§4.5 for the mechanism and the incremental history growth.
- **This is chunking-independent for this unchanged input.** The settled visible cell here (whole stream fed in one `parse_bytes` call) is identical to the end state of the incremental trace in §4.5 (the same bytes fed one codepoint at a time), including the identical four-entry history buffer. Feeding the *same* sequence as one chunk or as several produces the same result. This is not a claim of order-independence — reordering the codepoints would change grapheme membership and therefore the outcome; it is the narrower, observed fact that *how the identical byte sequence is split across parser calls* does not affect the settled state.


---

## Section 6 — Part 3: State reporting → OBJ-3

**Direct answer:** on the settled 1×1 family cell, a Cursor-Position-Report request `CSI 6 n` replies with the exact bytes **`b'\x1b[1;2R'`** — row 1, column 2. The reported column is **2**, not 1. This is *not* a clamp to the last real column (which in a one-column screen would be column 1): after the final width-2 base was placed, the cursor sat at the pending column `x=2` (one past the single column), and `report_device_status` takes the bottom-row branch that **decrements `x` by exactly one** (`2 → 1`) and then emits it 1-based as column `1 + 1 = 2`. The report therefore encodes only **how many columns the last base consumed** — a width consequence of the Part-1 handling — and carries **no** information about which codepoints were kept in the visible cell or scrolled into history. A state query cannot, by itself, identify the retained grapheme; it only reflects the cursor position that the width classification produced.

### 6.1 The DSR/CPR code and the bottom-row column adjustment

`CSI 6 n` (Device Status Report — cursor position) is handled by `report_device_status` — `kitty/screen.c:L2179-L2199`:

```c
report_device_status(Screen *self, unsigned int which, bool private) {
    // We don't implement the private device status codes, since I haven't come
    // across any programs that use them
    unsigned int x, y;
    static char buf[64];
    switch(which) {
        case 5:  // device status
            write_escape_code_to_child(self, ESC_CSI, "0n");
            break;
        case 6:  // cursor position
            x = self->cursor->x; y = self->cursor->y;
            if (x >= self->columns) {
                if (y < self->lines - 1) { x = 0; y++; }
                else x--;
            }
            if (self->modes.mDECOM) y -= MAX(y, self->margin_top);
            // 1-based indexing
            int sz = snprintf(buf, sizeof(buf) - 1, "%s%u;%uR", (private ? "?": ""), y + 1, x + 1);
            if (sz > 0) write_escape_code_to_child(self, ESC_CSI, buf);
            break;
    }
}
```

**Applying it to the settled 1×1 family cell** (`x=2, y=0, columns=1, lines=1`):

- `x >= self->columns` → `2 >= 1` → **true**, so the bottom-row branch is entered.
- `y < self->lines - 1` → `0 < 0` → **false**, so the `else` arm runs: `x--` → `x = 1`. This is a single decrement of the pending (past-the-end) column, **not** a clamp to `columns - 1`; a clamp would set `x = 0` and report column 1, but the code reports column 2.
- `mDECOM` is off, so `y` is unchanged.
- 1-based output: `y + 1 = 1`, `x + 1 = 2` → the formatted body is `"1;2R"`.

`write_escape_code_to_child` prepends the CSI introducer `ESC [`, giving the full reply **`b'\x1b[1;2R'`**.

**How this reflects the earlier grapheme handling:** the report carries no notion of "family emoji" or "grapheme cluster." All it encodes is the **cursor column**, which is a consequence of the width classification from Part 1 — a single width-2 base was placed and advanced the cursor to the pending column `x=2`, which the bottom-row branch then decremented by one before reporting it 1-based. The report is thus a faithful echo of *how many columns the surviving base consumed under the constraint*, not of the codepoints that streamed through — it cannot tell a client that three earlier bases were scrolled into history, only that the cursor advanced by two columns' worth of one width-2 base.

### 6.2 OBSERVED: `CSI 6 n` on the settled 1×1 cell

```text
   [after whole family stream (settled)]
      str(line0)   = '👦'
      codepoints    = ['U+1F466']  (len=1)
      as_ansi()     = '👦'
      cursor        = (x=2, y=0)
      historybuf    = count=4  lines=['👧\u200d', '👩\u200d', '👨\u200d', '']
      CSI 6 n  cmd=parse_bytes(s, b'\x1b[6n')  -> reply = b'\x1b[1;2R'
```

The `historybuf` line makes the scoping concrete: the CPR reply `b'\x1b[1;2R'` reports the cursor position in the *visible* cell (which holds only `👦`), while the three earlier bases sit in the history buffer. The state report is blind to that history — it echoes cursor column only.

### 6.3 OBSERVED: the wide-grid contrast (20×1)

Fed into a **20×1** grid, the very same byte stream is **not** collapsed: there is room, so the whole cluster survives (all 7 codepoints), the cursor lands at `x=8` (four width-2 bases = 8 columns), and `CSI 6 n` reports column 9:

```text
   [20x1 after whole family stream]
      str(line0)   = '👨\u200d👩\u200d👧\u200d👦'
      codepoints    = ['U+1F468', 'U+200D', 'U+1F469', 'U+200D', 'U+1F467', 'U+200D', 'U+1F466']  (len=7)
      as_ansi()     = '👨\u200d👩\u200d👧\u200d👦'
      cursor        = (x=8, y=0)
      historybuf    = count=0  lines=[]
      CSI 6 n reply = b'\x1b[1;9R'
```

Here `x=8 < columns=20`, so the `x >= self->columns` test is false and the bottom-row decrement branch does **not** run; the report is `y+1=1`, `x+1=9` → **`b'\x1b[1;9R'`**. Note also `historybuf count=0`: with 20 columns there was room for all four width-2 bases (8 columns total), so **nothing autowrapped into history** — the entire seven-codepoint cluster stayed in the visible line. Contrast is the point: the identical input yields **`b'\x1b[1;2R'`** with an empty settled cell region plus a *four-entry* history in 1×1 (collapsed), versus **`b'\x1b[1;9R'`** with a *zero-entry* history in 20×1 (whole cluster retained). The difference in the state report — and in the history buffer — is a direct readout of the difference in retention forced by geometry.

### 6.4 OBSERVED: related state reports on the settled 1×1 cell

Each of the following was issued through the real parser against the settled 1×1 family cell, and the reply captured from `wtcbuf`. Complete, unedited output:

```text
   settled cell = '👦'  cursor=(x=2,y=0)
   DSR-5  CSI 5 n     parse_bytes(s, b'\x1b[5n')  -> b'\x1b[0n'
   DA     CSI c       parse_bytes(s, b'\x1b[c')  -> b'\x1b[?62;c'
   DA>    CSI > c     parse_bytes(s, b'\x1b[>c')  -> b'\x1b[>1;4000;35c'
   size14 CSI 14 t    parse_bytes(s, b'\x1b[14t')  -> b'\x1b[4;20;10t'
   size16 CSI 16 t    parse_bytes(s, b'\x1b[16t')  -> b'\x1b[6;20;10t'
   size18 CSI 18 t    parse_bytes(s, b'\x1b[18t')  -> b'\x1b[8;1;1t'
   DECRPM CSI ?25 $p  parse_bytes(s, b'\x1b[?25$p')  -> b'\x1b[?25;1$y'
   DECRQSS DECSCUSR   parse_bytes(s, b'\x1bP$q q\x1b\\')  -> b'\x1bP1$r1 q\x1b\\'
```

With mechanism and citation for each:

- **DSR-5** `CSI 5 n` → **`b'\x1b[0n'`** — the `case 5` branch writes the literal `"0n"` ("device OK") at `kitty/screen.c:L2185-L2186`. It is independent of the cell content.
- **Primary Device Attributes** `CSI c` → **`b'\x1b[?62;c'`** — `report_device_attributes` (`kitty/screen.c:L2121-L2132`) writes `"?62;c"` for the no-modifier case at `kitty/screen.c:L2125` (VT220-class identification).
- **Secondary DA** `CSI > c` → **`b'\x1b[>1;4000;35c'`** — same function, `'>'` case at `kitty/screen.c:L2128`, format `">1;" xstr(PRIMARY_VERSION) ";" xstr(SECONDARY_VERSION) "c"`. `PRIMARY_VERSION` and `SECONDARY_VERSION` are compile-time macros whose values are **derived from the kitty version**: `kitty/constants.py:L25` declares `version = Version(0, 35, 2)`; `setup.py:L605` computes `primary_version = version[0] + 4000 = 4000` (the `+4000` is intentional, per the source comment, so that vim enables SGR mouse mode) and `setup.py:L606` computes `secondary_version = version[1] = 35`; these are emitted as `-D` defines at `setup.py:L730`. So **these two numbers are version-dependent** — observed here as `4000;35`, exactly matching `0 + 4000` and `35` from the version tuple.
- **Size reports** `screen_report_size` — `kitty/screen.c:L2142-L2168` — format `"%u;%u;%ut"` = `code;height;width`:
  - `CSI 14 t` → **`b'\x1b[4;20;10t'`** — text area in pixels: `code=4`, `height = cell_height × lines = 20 × 1 = 20`, `width = cell_width × cols = 10 × 1 = 10`.
  - `CSI 16 t` → **`b'\x1b[6;20;10t'`** — single cell in pixels: `code=6`, `20 × 10`.
  - `CSI 18 t` → **`b'\x1b[8;1;1t'`** — text area in characters: `code=8`, `height = lines = 1`, `width = cols = 1`.
- **Mode status / DECRPM** `CSI ?25 $p` → **`b'\x1b[?25;1$y'`** — `report_mode_status` (`kitty/screen.c:L2203-L2242`) reports mode 25 (cursor visibility) with value `1` (visible); the reply is formatted with `"%s%u;%u$y"` at `kitty/screen.c:L2240`.
- **DECRQSS** (DCS `$ q` … ST) querying DECSCUSR cursor shape → **`b'\x1bP1$r1 q\x1b\\'`** — `screen_request_capabilities` (`kitty/screen.c:L2446-L2482`), `'$'` case at `kitty/screen.c:L2453`, recognizes the `" q"` query (`kitty/screen.c:L2455`) and formats the current cursor shape with `"1$r%d q"` at `kitty/screen.c:L2468`. *(In the Python byte-repr, the trailing `\x1b\\` denotes exactly two bytes — `ESC` (`0x1B`) then a single backslash `\` (`0x5C`) — which together are the String Terminator `ST` = `ESC \`. So the full reply is DCS `ESC P 1 $ r 1 SP q` followed by `ESC \`.)*

None of DSR-5, DA, DA>, the size reports, DECRPM, or DECRQSS depend on the cell's Unicode content — only `CSI 6 n` (§6.1–6.3) reflects the grapheme handling, via the cursor column.


---

## Section 7 — Part 4: How normalization, grapheme breaking, and state reporting interact → OBJ-4

**Direct answer:** they interact only **indirectly**, because kitty implements the first two as a **single width + combining-membership classification** rather than as separate passes, and the third (state reporting) merely reads out the cursor position that classification produced. Specifically: **normalization is absent**, there is **no grapheme-segmentation state machine**, and the `CSI 6 n` report reflects the outcome only through the cursor column.

### 7.1 Normalization: ABSENT (on the parser → screen → line path)

There is **no Unicode normalization pass** on the input path that this investigation exercises — the VT parser (`kitty/vt-parser.c`), the screen draw path (`kitty/screen.c`), and the line/cell storage (`kitty/line.c`, `kitty/data-types.h`). The UTF-8 bytes decoded by the parser are classified by width/combining-membership and stored as-is; no NFC/NFD/NFKC/NFKD transformation is applied before a codepoint is placed in a cell or appended to `cc_idx`. *(This statement is scoped to that path; it is not a claim that the string "normalize" appears nowhere in the entire multi-hundred-file repository.)*

**Source-search evidence (inferred-from-source, shown as commands + output).** Searching the four files on the path for any normalization vocabulary returns nothing:

```text
$ grep -rniE 'normaliz|nfc|nfd|nfkc|nfkd|unicodedata|precompos' \
      kitty/vt-parser.c kitty/screen.c kitty/line.c kitty/data-types.h
$ echo "grep-exit=$?"
grep-exit=1
```

(`grep` exit status `1` means "no lines matched".) There is simply no normalization code to invoke on this path.

**Correcting a subtle point about what normalization *would* do.** It is tempting to say "if kitty normalized, the combining marks in the overflow case (§7.6) would have been reordered or precomposed." That is only *partly* true and depends on the form — so the claim must be made precisely. Running Python's `unicodedata` (a standards-conformant implementation) on the exact overflow input `A U+0300 U+0301 U+0302 U+0303` shows:

```text
unidata_version = 15.1.0
raw   : ['U+0041', 'U+0300', 'U+0301', 'U+0302', 'U+0303']
NFC  : ['U+00C0', 'U+0301', 'U+0302', 'U+0303']
NFD  : ['U+0041', 'U+0300', 'U+0301', 'U+0302', 'U+0303']
NFKC : ['U+00C0', 'U+0301', 'U+0302', 'U+0303']
NFKD : ['U+0041', 'U+0300', 'U+0301', 'U+0302', 'U+0303']
ccc U+0300..0303: [230, 230, 230, 230]
```

So: **NFC and NFKC would precompose** the base and the first mark (`A` + `U+0300` → `À` = `U+00C0`), changing the leading codepoint; **NFD and NFKD would leave the sequence byte-for-byte unchanged** (no reordering, no precomposition), because the four marks all share canonical combining class 230 and equal-class marks are never reordered relative to each other. kitty's observed cell in §7.6 begins with `U+0041` (not `U+00C0`) and preserves the raw order — a result **consistent with NFD/NFKD or with no normalization at all**, and **inconsistent only with NFC/NFKC**. The definitive proof that *no* normalization runs is therefore the source search above, not the overflow codepoints alone. *(The `unidata_version = 15.1.0` here is Python's bundled UCD; kitty's own tables are generated from UCD 15.0.0 — see §7.3. The composition/decomposition of `A`+`U+0300` is identical in both.)*

### 7.2 Grapheme breaking: NO UAX #29 state machine

kitty does **not** run a Unicode Text-Segmentation (UAX #29) extended-grapheme-cluster segmenter on this path. A search of the same four path files for segmentation vocabulary is empty:

```text
$ grep -rniE 'grapheme|UAX.?29|GB[0-9]|cluster_break|segmentation|break_property' \
      kitty/vt-parser.c kitty/screen.c kitty/line.c kitty/unicode-data.c
$ echo "grep-exit=$?"
grep-exit=1
```

Instead each codepoint is classified independently by two properties:

- **width** — computed by `wcwidth_std`, which the draw loop calls at `char_width = wcwidth_std(ch);` (`kitty/screen.c:L814`). `wcwidth_std` is defined `static inline int wcwidth_std(int32_t code)` at `kitty/wcwidth-std.h:L9-L10`, backed by the generated table in the same header.
- **combining membership** — `is_combining_char` (`kitty/unicode-data.c:L11`).

A non-combining codepoint with width > 0 **is placed into a cell** (consuming columns); a combining codepoint is **appended** to the current base's `cc_idx` (§4.2–4.3). "Grapheme handling" in kitty is therefore an *emergent* consequence of these per-codepoint decisions, not the output of a cluster-boundary algorithm. This is exactly why a ZWJ family sequence does **not** stay together in a 1×1 cell: nothing is tracking the cluster, so each new width-2 base needs its own column — and because there is only one column, placing the next base first triggers the pending-wrap path (§4.4) that scrolls the *previous* base into the history buffer before the new base is written into the freshly cleared visible cell. The marks (ZWJ included) ride along on whichever base is current; they are never what "wins" the cell.

### 7.3 Provenance: the classification tables are generated (Unicode 15.0.0)

The classification tables are **generated**, not hand-written. The generator is `gen_ucd()` — `gen/wcwidth.py:L403` — which emits `kitty/unicode-data.c` (via `create_header('kitty/unicode-data.c')` at `gen/wcwidth.py:L405`). The emitted file's first line records the standard version — `kitty/unicode-data.c:L1`: `// Unicode data, built from the Unicode Standard 15.0.0`. The same version is cross-confirmed in the Go width table at `tools/wcswidth/std.go:L3242`: `var UnicodeDatabaseVersion [3]int = [3]int{15, 0, 0}`. There is no runtime normalization layer to generate — only membership/width tables.

### 7.4 Subtle classification facts that make the ZWJ behavior work

These three facts (each grounded in code) are what route a ZWJ into a combining slot rather than dropping it or giving it a cell:

- **`is_combining_char(0x200D)` is TRUE.** ZWJ falls inside `case 0x200b ... 0x200f:` at `kitty/unicode-data.c:L323`, which is a case within `is_combining_char` (`L11`). Hence ZWJ takes the combining branch of the draw loop (§4.3) and is appended, not allocated a new cell. *(Observed indirectly: after a ZWJ the cell length grows by one combining mark on the same base rather than producing a second cell — §4.5.)*
- **`is_ignored_char(0x200D)` is FALSE.** `is_ignored_char` (`kitty/unicode-data.c:L671`, "Control characters and non-characters") does **not** contain the `0x200b..0x200f` range, so the draw-loop gate `if (is_ignored_char(ch)) continue;` (`screen.c:L805`) does **not** drop ZWJ. The identical-looking `case 0x200b ... 0x200f:` at `kitty/unicode-data.c:L753` belongs to a **different** function, `is_non_rendered_char` (`L723`), which is **not** the draw-loop gate. The distinction matters: it is why ZWJ genuinely reaches `draw_combining_char`. *(Observed indirectly via §4.5: the ZWJ demonstrably survives as a combining mark on the base — it is neither dropped nor given its own cell.)*
- **VS15/VS16 are combining.** The variation selectors fall inside `case 0xfe00 ... 0xfe0f:` at `kitty/unicode-data.c:L407` (within `is_combining_char`), so they too are appended to the preceding base — which is what enables the width-flip logic in §7.5.

> **Terminology precision (F15).** "Combining" in this document means *kitty's* `is_combining_char` membership, which is deliberately **broader** than the Unicode "Combining Mark" general categories (`Mn`/`Mc`/`Me`). The codepoints kitty routes into `cc_idx` here span several distinct Unicode general categories: ZWJ `U+200D` is `Cf` (Format); variation selectors `U+FE0E`/`U+FE0F` are `Mn` (Nonspacing Mark); regional-indicator symbols `U+1F1E6..U+1F1FF` are `So` (Other Symbol); and emoji skin-tone modifiers `U+1F3FB..U+1F3FF` are `Sk` (Modifier Symbol). kitty does not consult the Unicode general category on this path; it consults only `is_combining_char` (plus the flag-pair special case), so any codepoint that returns true there is *appended as a stored codepoint in a `cc_idx` slot*, regardless of its formal category. That is why calling every appended codepoint a "combining mark" would be imprecise — several of them are not `M*`-category marks at all.

### 7.5 Variation-selector width flips (and how they change the state report)

Because the variation selectors are combining, they reach `draw_combining_char` (`kitty/screen.c:L663-L710`), which contains two special branches that **mutate the base cell's effective width** — and therefore the cursor column that `CSI 6 n` later echoes:

- **VS16 `U+FE0F` (emoji presentation) widens** a default text-presentation emoji to width 2 (branch `if (ch == 0xfe0f)` at `kitty/screen.c:L679`). Its precondition — `gpu_cell->attrs.width != 2 && cpu_cell->cc_idx[0] == VS16 && is_emoji_presentation_base(...)` — is at `kitty/screen.c:L682`; on success it sets `gpu_cell->attrs.width = 2` (`L683`); if there is a spare column (`xpos + 1 < self->columns`, `L684`) it zeroes the next cell and does `self->cursor->x++` (`L687`), **otherwise** it calls `move_widened_char` (`L688`, defined at `kitty/screen.c:L575`) and the cursor is **not** incremented.
- **VS15 `U+FE0E` (text presentation) narrows** to width 1 (branch `else if (ch == 0xfe0e)` at `kitty/screen.c:L690`). Its precondition — crucially `gpu_cell->attrs.width == 2 && cpu_cell->cc_idx[0] == VS15 && is_emoji_presentation_base(...)` — is at `kitty/screen.c:L696`; on success it sets `attrs.width = 1` (`L697`) and does `self->cursor->x--` (`L698`).

**A note on why the VS15 base must be width-2 (F2).** The VS15 narrowing branch only fires when `gpu_cell->attrs.width == 2` (`L696`). A base that is *already* width 1 (e.g. `U+2764`, whose default `wcwidth_std` is 1) can never satisfy that test, so feeding `U+2764 + VS15` would append the selector but **never execute** the `attrs.width = 1` / `cursor->x--` logic — the narrowing would be untested. To genuinely exercise the branch, the observations below use a **default-width-2** base, `U+1F610` (😐, `wcwidth_std == 2`), so the precondition at `L696` actually holds and the `cursor->x--` at `L698` is observed to fire.

OBSERVED — base `U+2764` (❤, a default text-presentation emoji, `wcwidth_std == 1`) followed by VS16, fed **incrementally** (before / after base / after VS16) into 1×1 and 5×1:

```text
   [1x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
   [1x1 after base U+2764 (width 1)  <-- b'\xe2\x9d\xa4']
      str(line0)   = '❤'
      codepoints    = ['U+2764']  (len=1)
      as_ansi()     = '❤'
      cursor        = (x=1, y=0)
      CSI 6 n reply = b'\x1b[1;1R'
   [1x1 after +VS16 U+FE0F (widen)  <-- b'\xef\xb8\x8f']
      str(line0)   = '❤️'
      codepoints    = ['U+2764', 'U+FE0F']  (len=2)
      as_ansi()     = '❤️'
      cursor        = (x=1, y=0)
      CSI 6 n reply = b'\x1b[1;1R'
   [5x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
   [5x1 after base U+2764 (width 1)  <-- b'\xe2\x9d\xa4']
      str(line0)   = '❤'
      codepoints    = ['U+2764']  (len=1)
      as_ansi()     = '❤'
      cursor        = (x=1, y=0)
      CSI 6 n reply = b'\x1b[1;2R'
   [5x1 after +VS16 U+FE0F (widen)  <-- b'\xef\xb8\x8f']
      str(line0)   = '❤️'
      codepoints    = ['U+2764', 'U+FE0F']  (len=2)
      as_ansi()     = '❤️'
      cursor        = (x=2, y=0)
      CSI 6 n reply = b'\x1b[1;3R'
```

In **1×1**, after the base the cursor is at `x=1` (CPR col 1: `1 >= 1` true → `x--` → `0` → 1-based col 1). Adding VS16 widens the cell to width 2 but there is **no spare column** (`xpos + 1 < self->columns` is `1 < 1`, false), so `move_widened_char` runs and the cursor **stays** at `x=1` → `CSI 6 n` = **`b'\x1b[1;1R'`** (unchanged). In **5×1**, after the base the cursor is `x=1` (CPR col 2). VS16 finds a spare column, zeroes it, and does `self->cursor->x++`, landing at `x=2` → **`b'\x1b[1;3R'`**. So VS16 advanced the reported column by one **only where a spare column existed**; under the 1×1 constraint the widen is absorbed with no cursor advance.

OBSERVED — base `U+1F610` (😐, a default-width-2 emoji, `wcwidth_std == 2`) followed by VS15, fed **incrementally** (before / after base / after VS15) into 1×1 and 5×1:

```text
   [1x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
   [1x1 after base U+1F610 (width 2)  <-- b'\xf0\x9f\x98\x90']
      str(line0)   = '😐'
      codepoints    = ['U+1F610']  (len=1)
      as_ansi()     = '😐'
      cursor        = (x=2, y=0)
      CSI 6 n reply = b'\x1b[1;2R'
   [1x1 after +VS15 U+FE0E (narrow)  <-- b'\xef\xb8\x8e']
      str(line0)   = '😐︎'
      codepoints    = ['U+1F610', 'U+FE0E']  (len=2)
      as_ansi()     = '😐︎'
      cursor        = (x=1, y=0)
      CSI 6 n reply = b'\x1b[1;1R'
   [5x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
   [5x1 after base U+1F610 (width 2)  <-- b'\xf0\x9f\x98\x90']
      str(line0)   = '😐'
      codepoints    = ['U+1F610']  (len=1)
      as_ansi()     = '😐'
      cursor        = (x=2, y=0)
      CSI 6 n reply = b'\x1b[1;3R'
   [5x1 after +VS15 U+FE0E (narrow)  <-- b'\xef\xb8\x8e']
      str(line0)   = '😐︎'
      codepoints    = ['U+1F610', 'U+FE0E']  (len=2)
      as_ansi()     = '😐︎'
      cursor        = (x=1, y=0)
      CSI 6 n reply = b'\x1b[1;2R'
```

Now the narrowing branch genuinely executes. In **5×1**: after the width-2 base the cursor is `x=2` (CPR col 3); adding VS15 satisfies the `attrs.width == 2` precondition (`L696`), sets `attrs.width = 1` (`L697`), and does `self->cursor->x--` (`L698`), pulling the cursor back to `x=1` → **`b'\x1b[1;2R'`**. The one-column drop in the reported column (3 → 2) is the directly observed effect of the `cursor->x--`. In **1×1**: after the base the cursor is `x=2` (CPR col 2 — `2 >= 1` true → `x--` → `1` → col 2); VS15 narrows and does `cursor->x--` → `x=1`, and the report is now `x=1 >= columns=1` true → the bottom-row branch decrements to `x=0` → 1-based column 1 → **`b'\x1b[1;1R'`**. (This is the same single-decrement bottom-row adjustment of §6.1, not a clamp.) The VS15 width flip thus lowers the reported column in both geometries — the mirror image of VS16.

### 7.6 The `cc_idx` overflow rule (more than three combining marks)

To exercise the overflow branch of `line_add_combining_char` (§4.2), a base `A` (`U+0041`) was followed by **four** combining marks `U+0300 U+0301 U+0302 U+0303` in a 5×1 grid, fed one at a time. Complete, unedited output (the codepoint list is the authoritative view, since stacked combining marks may not display cleanly):

```text
   [before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
   [base A]
      str(line0)   = 'A'
      codepoints    = ['U+0041']  (len=1)
      as_ansi()     = 'A'
      cursor        = (x=1, y=0)
   [+U+0300 (fills slot0)]
      str(line0)   = 'À'
      codepoints    = ['U+0041', 'U+0300']  (len=2)
      as_ansi()     = 'À'
      cursor        = (x=1, y=0)
   [+U+0301 (fills slot1)]
      str(line0)   = 'À́'
      codepoints    = ['U+0041', 'U+0300', 'U+0301']  (len=3)
      as_ansi()     = 'À́'
      cursor        = (x=1, y=0)
   [+U+0302 (fills slot2)]
      str(line0)   = 'À́̂'
      codepoints    = ['U+0041', 'U+0300', 'U+0301', 'U+0302']  (len=4)
      as_ansi()     = 'À́̂'
      cursor        = (x=1, y=0)
   [+U+0303 (OVERFLOW -> overwrites last slot)]
      str(line0)   = 'À́̃'
      codepoints    = ['U+0041', 'U+0300', 'U+0301', 'U+0303']  (len=4)
      as_ansi()     = 'À́̃'
      cursor        = (x=1, y=0)
```

The first three marks fill `cc_idx[0..2]` (the codepoint list grows 1 → 2 → 3 → 4). The **fourth** mark `U+0303` does not extend the list further — it **overwrites the last slot**, so `U+0302` (which had been in `cc_idx[2]`) is replaced while `U+0300` and `U+0301` remain. The retained set is `['U+0041','U+0300','U+0301','U+0303']`, exactly as `cell->cc_idx[arraysz(cell->cc_idx) - 1] = mark_for_codepoint(ch);` (`kitty/line.c:L466`) dictates. This is the same fixed-capacity retention rule as Part 1's `CPUCell.cc_idx[3]`, now visible on a single base rather than across autowrapped lines. *(The cursor stays at `x=1` throughout — appended codepoints never advance it, which is also why the overflow case emits no `CSI 6 n` change.)*

### 7.7 Contrast conditions: flag pairs and skin-tone modifiers are retained even in 1×1

Not every "multi-codepoint" sequence collapses the way the ZWJ family does. Two important contrasts, both OBSERVED, show that when the second codepoint is *combining/appendable*, **both** codepoints survive even in a 1×1 cell:

**Regional-indicator flag pair (US) `U+1F1FA U+1F1F8`.** The second regional indicator is appended to the first via `draw_second_flag_codepoint` (defined `kitty/screen.c:L638-L654`, dispatched from the draw loop at `kitty/screen.c:L808`), which calls `line_add_combining_char` at `kitty/screen.c:L652` after confirming the pair with `is_flag_pair` (`kitty/screen.c:L633`). `is_flag_codepoint` is the range `0x1F1E6..0x1F1FF`, defined at `kitty/unicode-data.h:L81-L84` (the range test itself at `L83`). Complete, unedited output (fed incrementally: before / after 1st RI / after 2nd RI):

```text
   [1x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
      historybuf    = count=0  lines=[]
   [1x1 after 1st RI U+1F1FA]
      str(line0)   = '🇺'
      codepoints    = ['U+1F1FA']  (len=1)
      as_ansi()     = '🇺'
      cursor        = (x=2, y=0)
      historybuf    = count=1  lines=['']
      CSI 6 n reply = b'\x1b[1;2R'
   [1x1 after 2nd RI U+1F1F8 (appended)]
      str(line0)   = '🇺🇸'
      codepoints    = ['U+1F1FA', 'U+1F1F8']  (len=2)
      as_ansi()     = '🇺🇸'
      cursor        = (x=2, y=0)
      historybuf    = count=1  lines=['']
      CSI 6 n reply = b'\x1b[1;2R'
   [5x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
      historybuf    = count=0  lines=[]
   [5x1 after 1st RI U+1F1FA]
      str(line0)   = '🇺'
      codepoints    = ['U+1F1FA']  (len=1)
      as_ansi()     = '🇺'
      cursor        = (x=2, y=0)
      historybuf    = count=0  lines=[]
      CSI 6 n reply = b'\x1b[1;3R'
   [5x1 after 2nd RI U+1F1F8 (appended)]
      str(line0)   = '🇺🇸'
      codepoints    = ['U+1F1FA', 'U+1F1F8']  (len=2)
      as_ansi()     = '🇺🇸'
      cursor        = (x=2, y=0)
      historybuf    = count=0  lines=[]
      CSI 6 n reply = b'\x1b[1;3R'
```

**Skin-tone modifier `U+1F44B U+1F3FF`** (waving hand + dark skin tone). The modifier `U+1F3FF` is in `is_combining_char` (skin-tone modifiers `0x1F3FB..0x1F3FF`, `kitty/unicode-data.c:L661`) and is appended to the wave base. Complete, unedited output (before / after base / after modifier):

```text
   [1x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
      historybuf    = count=0  lines=[]
   [1x1 after base wave U+1F44B]
      str(line0)   = '👋'
      codepoints    = ['U+1F44B']  (len=1)
      as_ansi()     = '👋'
      cursor        = (x=2, y=0)
      historybuf    = count=1  lines=['']
      CSI 6 n reply = b'\x1b[1;2R'
   [1x1 after +skin-tone U+1F3FF (appended)]
      str(line0)   = '👋🏿'
      codepoints    = ['U+1F44B', 'U+1F3FF']  (len=2)
      as_ansi()     = '👋🏿'
      cursor        = (x=2, y=0)
      historybuf    = count=1  lines=['']
      CSI 6 n reply = b'\x1b[1;2R'
   [5x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
      historybuf    = count=0  lines=[]
   [5x1 after base wave U+1F44B]
      str(line0)   = '👋'
      codepoints    = ['U+1F44B']  (len=1)
      as_ansi()     = '👋'
      cursor        = (x=2, y=0)
      historybuf    = count=0  lines=[]
      CSI 6 n reply = b'\x1b[1;3R'
   [5x1 after +skin-tone U+1F3FF (appended)]
      str(line0)   = '👋🏿'
      codepoints    = ['U+1F44B', 'U+1F3FF']  (len=2)
      as_ansi()     = '👋🏿'
      cursor        = (x=2, y=0)
      historybuf    = count=0  lines=[]
      CSI 6 n reply = b'\x1b[1;3R'
```

In both cases the 1×1 **visible** cell keeps **two** codepoints (`len=2`) — unlike the ZWJ family, where the visible cell keeps **one** (`len=1`). The distinction is entirely explained by classification, and it shows up in the history buffer too: the *second* codepoint here is **appended** to the first base (flag pair via `draw_second_flag_codepoint`; skin tone via the combining path), so it does **not** consume a new column and does **not** trigger another autowrap — the 1×1 history count stays at **1** (the single empty line left by the first width-2 base) rather than growing. In the family sequence, by contrast, each element after a ZWJ is a **new width-2 base**, so each one autowraps the previous base into history (§4.4) and the 1×1 history count grows to **4**. The `CSI 6 n` column (2 in 1×1, 3 in 5×1) echoes the *single* width-2 base's cursor advance under the geometry — the appended second codepoint adds nothing to the column because appended codepoints never move the cursor.

### 7.8 Putting the three mechanisms together

- **Normalization** never runs on this path (§7.1), so nothing recomposes or reorders the stream — the raw codepoints are what get classified.
- **Grapheme breaking** is not a segmentation pass but the emergent result of per-codepoint width + combining decisions; under a 1×1 constraint this means "last base wins **in the visible cell**," with appended codepoints (ZWJ included) riding along on whatever base is current. The earlier bases are not destroyed — each new width-2 base triggers the pending-wrap path (§4.4) that scrolls the previous base into the **history buffer** before the new base is written into the cleared visible cell.
- **State reporting** does not observe graphemes at all; `CSI 6 n` reports the **cursor column**, which is the only place the earlier handling leaves a visible trace — and only because width classification moved the cursor. When the cursor is one past the last column, `report_device_status` takes its bottom-row branch and **decrements the column by exactly one** (`x--`, §6.1) before emitting it 1-based — it does **not** clamp to the last real column, and it conveys nothing about which codepoints were kept in the cell or scrolled into history.


---

## Section 8 — Observed vs. inferred

Every behavioral claim in this document falls into exactly one of three evidentiary categories. This section states, category by category, which is which — so no source-derived or standards-background statement is misread as directly observed.

**Category A — RUNTIME-OBSERVED** (real captured output from the built engine, byte-identical across two runs; see §2.5):

- Every `str(line)`, codepoint list, `as_ansi()`, cursor `(x,y)`, and `historybuf` value shown in the output blocks of Sections 4–7.
- The settled 1×1 family cell = `'👦'` / `['U+1F466']` / `len=1`, with the four-entry history buffer `['👧\u200d', '👩\u200d', '👨\u200d', '']`.
- All control-sequence replies shown as literal `bytes`: `CSI 6 n` = `b'\x1b[1;2R'` (1×1) and `b'\x1b[1;9R'` (20×1); DSR-5 `b'\x1b[0n'`; DA `b'\x1b[?62;c'`; DA> `b'\x1b[>1;4000;35c'`; size `b'\x1b[4;20;10t'`, `b'\x1b[6;20;10t'`, `b'\x1b[8;1;1t'`; DECRPM `b'\x1b[?25;1$y'`; DECRQSS `b'\x1bP1$r1 q\x1b\\'`.
- The VS16/VS15 cursor and CPR differences between 1×1 and 5×1; the flag-pair and skin-tone 1×1 retention (`len=2`, history count stays 1); the `cc_idx` overflow codepoint lists.
- **Chunking-independence:** feeding the identical family byte sequence as one `parse_bytes` call (§5.2) versus one codepoint at a time (§4.5) produces the identical settled cell *and* the identical four-entry history buffer. (Observed for this unchanged input; this is *not* a claim of order-independence — see §5.2.)
- The two source-search commands and their `grep-exit=1` results (§7.1, §7.2), and the `unicodedata` NFC/NFD/NFKC/NFKD output (§7.1) — these are observed *command outputs*.
- The build result in this environment (canonical `python3 setup.py build` → exit 0, Wayland disabled, 85 objects, 4 links) and the successful import.

**Category A′ — OBSERVED but VERSION-DEPENDENT:**

- The Secondary-DA numbers `4000;35` in `b'\x1b[>1;4000;35c'` are compile-time version constants derived from the kitty version (`constants.py:L25` → `setup.py:L605-L606`, emitted at `setup.py:L730`); a different kitty version would report different numbers. The *shape* of the reply (`ESC [ > 1 ; … ; … c`) is stable.

**Category B — INFERRED from reading the source** (entailed by the observations plus the code, but not a distinct line in the output):

- That the ZWJ family's earlier bases are **scrolled into the history buffer** (rather than destroyed) by the pending-wrap → `screen_index` → `INDEX_UP`/`historybuf_add_line` → `linebuf_clear_line` → base-write sequence (§4.4) is the code-level explanation for the directly-observed facts that the visible cell collapses to `len=1` while `historybuf` grows to four entries. The *sequence of internal calls* is source-derived; the *cell and history values it produces* are observed.
- That the VS16 widen in the **1×1** case takes the `move_widened_char` path rather than `cursor->x++` is inferred from `kitty/screen.c:L682-L688` together with the observed fact that the cursor stayed at `x=1` (had the `cursor->x++` branch run, `x` would have become 2).
- The exact internal slot that each mark occupies (`cc_idx[0]`, `[1]`, `[2]`) is inferred from `line_add_combining_char` (`kitty/line.c:L456-L467`) plus the observed codepoint-list growth and the overflow overwrite; the public accessors expose the resulting *list*, not the raw slot indices.
- "No Unicode normalization runs at runtime on this path" is a source-derived *generalization*: the observed `grep-exit=1` command output shows the vocabulary is absent from the four path files, and from that (plus reading those files) we conclude no normalization executes. The overflow codepoints alone are only *consistent* with this (§7.1), not proof of it.

**Category C — EXTERNAL-STANDARD BACKGROUND** (not observed here and not from kitty's source; drawn from the Unicode Standard and VT/xterm control-sequence specifications, used only to describe the observations in standards-correct language):

- ZWJ is `U+200D`; an emoji-ZWJ sequence is intended (per Unicode UAX #29 / UTS #51) to form a single extended grapheme cluster; the family `👨‍👩‍👧‍👦` is the seven codepoints `U+1F468 U+200D U+1F469 U+200D U+1F467 U+200D U+1F466`.
- Variation selectors `U+FE0F` (emoji presentation) and `U+FE0E` (text presentation) alter presentation/width; regional-indicator pairs form flags; skin-tone modifiers are `U+1F3FB..U+1F3FF`.
- The DSR/CPR request `CSI 6 n` elicits a reply of the form `CSI Pl ; Pc R`; `CSI c` elicits a primary Device Attributes reply identifying a VT220-class terminal.
- The Unicode general categories cited in §7.4 (`Cf`, `Mn`, `So`, `Sk`) are properties defined by the Unicode Standard, not values read out of kitty at runtime.

**NON-CANONICAL (explicitly avoided in the answer):** any value from `s.draw(<str>)` (the direct draw API), remote control, or a debug hook. Every value in Category A came through `parse_bytes` → the real VT parser (§3). If one wished to check `is_combining_char`/`is_ignored_char` by calling the classifier bindings directly, that would be a non-canonical direct-API check; this document instead demonstrates their effect **indirectly** through the observed cell contents.

---

## Section 9 — Coverage-pass checklist

Every named item the question implies, addressed by name:

| Item | Where addressed | Key observed evidence |
|------|-----------------|-----------------------|
| **Normalization** (interaction) | §7.1, §7.8 | ABSENT — raw codepoints preserved (overflow list unreordered) |
| **Grapheme breaking** (interaction) | §7.2, §7.8 | NO UAX #29 state machine; width + combining membership |
| **State reporting** (interaction) | §6, §7.8 | `CSI 6 n` → `b'\x1b[1;2R'` (1×1) |
| **1×1 constraint** | §4, §5, §6.1–6.2 | settled visible cell `len=1`; cursor `x=2`; bottom-row `x--` decrement (not a clamp) → col 2 |
| **Primary family ZWJ emoji** | §4.5, §5.2 | incremental trace; settled visible `'👦'` `['U+1F466']`; history retains Man/Woman/Girl+ZWJ |
| **VS15 (narrow)** | §7.5 | width-2 base `U+1F610` genuinely narrowed (`x--` fires): 5×1 `x` 2→1 `b'\x1b[1;2R'`; 1×1 `b'\x1b[1;1R'` |
| **VS16 (widen)** | §7.5 | base `U+2764`: 1×1 `b'\x1b[1;1R'` (move_widened_char, no advance); 5×1 `b'\x1b[1;3R'` (`x++`) |
| **Regional-indicator flag pair** | §7.7 | 1×1 `'🇺🇸'` `len=2`, `b'\x1b[1;2R'` |
| **Skin-tone modifier** | §7.7 | 1×1 `'👋🏿'` `len=2`, `b'\x1b[1;2R'` |
| **>3-mark `cc_idx` overflow** | §7.6 | 4th mark overwrites last slot → `['U+0041','U+0300','U+0301','U+0303']` |
| **Wider-grid contrast** | §6.3, §7.5, §7.7 | 20×1 whole cluster survives (history count=0), `b'\x1b[1;9R'`; 5×1 columns |
| **DSR/CPR `CSI 6 n`** | §6.1–6.3 | `b'\x1b[1;2R'` / `b'\x1b[1;9R'` |
| **DSR-5 `CSI 5 n`** | §6.4 | `b'\x1b[0n'` |
| **Device Attributes** | §6.4 | DA `b'\x1b[?62;c'`; DA> `b'\x1b[>1;4000;35c'` |
| **Size reports** | §6.4 | `b'\x1b[4;20;10t'`, `b'\x1b[6;20;10t'`, `b'\x1b[8;1;1t'` |
| **DECRPM (mode status)** | §6.4 | `b'\x1b[?25;1$y'` |
| **DECRQSS** | §6.4 | `b'\x1bP1$r1 q\x1b\\'` |
| **Canonical entry point** | §3 | all input via `parse_bytes` → real VT parser |
| **Reproducibility / stability** | §2.5 | 2/2 byte-identical runs (identical MD5) |
| **Repository integrity / cleanup** | §2.5, §10 | build artifacts gitignored; temp script outside repo, removed; `git status` clean |


---

## Section 10 — Appendix: the temporary observation script and its complete output

### 10.1 The observation script (verbatim)

This script lived **outside** the repository tree at `/tmp/kitty_obs/observe.py` and was **removed** afterward (§8). It drives the **real VT parser** (the canonical entry point) via `kitty_tests.parse_bytes` (`kitty_tests/__init__.py:L30-L36`), and captures every control-sequence reply from `Callbacks.write` → `self.wtcbuf` (`kitty_tests/__init__.py:L50-L51`). Before constructing any `Screen` it initializes the global `Options` exactly as `kitty_tests.BaseTest.set_options` does (via `merge_result_dicts(defaults._asdict(), …)` then `set_options(...)`), so that `get_options()` succeeds (§3.3, finding F3). Every observed value is checked against a golden expected value; any mismatch raises `SystemExit(1)`. It was executed from the repository root as `source /tmp/kitty-venv/bin/activate && PYTHONPATH=. python3 /tmp/kitty_obs/observe.py`.

```python
# TEMPORARY observation script (lives OUTSIDE the kitty repo tree at /tmp/kitty_obs).
# Drives the REAL VT parser (canonical entry point) via kitty_tests.parse_bytes:
#   screen.test_create_write_buffer() -> test_commit_write_buffer(data,dest) -> test_parse_written_data()
# Captures control-sequence replies from Callbacks.write -> self.wtcbuf.
#
# It initializes the global Options exactly as kitty_tests.BaseTest.set_options does,
# so the Screen runs under kitty's canonical default configuration (get_options() works).
# Every observed value is checked against an expected "golden" value at the end; the
# script exits non-zero if ANY assertion fails, so silent behavioral drift cannot pass.
from kitty.fast_data_types import Screen, get_options, set_options
from kitty.config import finalize_keys, finalize_mouse_mappings
from kitty.options.parse import merge_result_dicts
from kitty.options.types import Options, defaults
from kitty_tests import Callbacks, parse_bytes  # canonical byte-parser template


def init_canonical_options():
    # Mirror kitty_tests.BaseTest.set_options: build Options from the shipped defaults,
    # finalize key/mouse mappings, and install them as the process-wide options.
    final_options = {'scrollback_pager_history_size': 1024, 'click_interval': 0.5}
    opts = Options(merge_result_dicts(defaults._asdict(), final_options))
    finalize_keys(opts, {})
    finalize_mouse_mappings(opts, {})
    set_options(opts)
    return opts


# Collected (name -> observed) pairs, asserted against EXPECTED at the end.
OBS = {}
def rec(name, value):
    OBS[name] = value
    return value


def new_screen(lines, cols, scrollback=5):
    c = Callbacks()
    # Mirror kitty_tests.create_screen arg order:
    #   Screen(callbacks, LINES, COLS, scrollback, cell_width, cell_height, 0, callbacks)
    s = Screen(c, lines, cols, scrollback, 10, 20, 0, c)
    return s, c


def cps(text):
    return [f'U+{ord(ch):04X}' for ch in text]


def hist_lines(s):
    return [str(s.historybuf.line(i)) for i in range(s.historybuf.count)]


def show(s, label, y=0, with_hist=False):
    line = s.line(y)
    text = str(line)
    print(f'   [{label}]')
    print(f'      str(line{y})   = {text!r}')
    print(f'      codepoints    = {cps(text)}  (len={len(text)})')
    print(f'      as_ansi()     = {line.as_ansi()!r}')
    print(f'      cursor        = (x={s.cursor.x}, y={s.cursor.y})')
    if with_hist:
        print(f'      historybuf    = count={s.historybuf.count}  lines={hist_lines(s)!r}')


def query(s, c, q):
    c.wtcbuf = b''                 # reset reply capture
    parse_bytes(s, q)              # drive query through the REAL parser
    return bytes(c.wtcbuf)


def hdr(t):
    print('\n' + '=' * 78 + f'\n{t}\n' + '=' * 78)


CSI6N = b'\x1b[6n'

# ---------------------------------------------------------------------------
init_canonical_options()
# Prove the canonical options are installed (raises RuntimeError if not set).
print(f'get_options() type after canonical init = {type(get_options()).__name__}')

FAMILY = '\U0001F468\u200D\U0001F469\u200D\U0001F467\u200D\U0001F466'
print(f'FAMILY emoji codepoints = {cps(FAMILY)} (Python len={len(FAMILY)})')
print(f'FAMILY UTF-8 bytes      = {FAMILY.encode("utf-8")!r}')

hdr('PRIMARY: family ZWJ emoji -> 1x1 Screen (INCREMENTAL: before/during/after + history)')
s, c = new_screen(1, 1)
show(s, 'before (empty screen)', with_hist=True)
chunks = ['\U0001F468', '\u200D', '\U0001F469', '\u200D', '\U0001F467', '\u200D', '\U0001F466']
labels = ['after Man U+1F468 (width2 base)', 'after ZWJ U+200D', 'after Woman U+1F469 (new width2 base)',
          'after ZWJ U+200D', 'after Girl U+1F467 (new width2 base)', 'after ZWJ U+200D',
          'after Boy U+1F466 (final width2 base)']
for chunk, label in zip(chunks, labels):
    parse_bytes(s, chunk.encode('utf-8'))
    show(s, label + f'  <-- fed bytes {chunk.encode("utf-8")!r}', with_hist=True)
rec('fam_incr_visible', str(s.line(0)))
rec('fam_incr_cursor', (s.cursor.x, s.cursor.y))
rec('fam_incr_hist', hist_lines(s))

hdr('PRIMARY (canonical single-stream): whole FAMILY at once -> 1x1, then CSI 6 n')
s, c = new_screen(1, 1)
show(s, 'before', with_hist=True)
parse_bytes(s, FAMILY.encode('utf-8'))
show(s, 'after whole family stream (settled)', with_hist=True)
r6 = query(s, c, CSI6N)
print(f'      CSI 6 n  cmd=parse_bytes(s, {CSI6N!r})  -> reply = {r6!r}')
rec('fam_visible', str(s.line(0)))
rec('fam_cps', cps(str(s.line(0))))
rec('fam_cursor', (s.cursor.x, s.cursor.y))
rec('fam_hist', hist_lines(s))
rec('fam_cpr', r6)

hdr('SECONDARY: VS16 widen  base U+2764 (width 1) + U+FE0F  into 1x1 and 5x1 (INCREMENTAL)')
for cols in (1, 5):
    s, c = new_screen(1, cols)
    show(s, f'{cols}x1 before', with_hist=False)
    parse_bytes(s, '\u2764'.encode('utf-8'))
    show(s, f'{cols}x1 after base U+2764 (width 1)  <-- b{chr(39)}\\xe2\\x9d\\xa4{chr(39)}')
    print(f'      CSI 6 n reply = {query(s, c, CSI6N)!r}')
    parse_bytes(s, '\uFE0F'.encode('utf-8'))
    show(s, f'{cols}x1 after +VS16 U+FE0F (widen)  <-- b{chr(39)}\\xef\\xb8\\x8f{chr(39)}')
    r = query(s, c, CSI6N)
    print(f'      CSI 6 n reply = {r!r}')
    rec(f'vs16_{cols}_cps', cps(str(s.line(0))))
    rec(f'vs16_{cols}_cursor_x', s.cursor.x)
    rec(f'vs16_{cols}_cpr', r)

hdr('SECONDARY: VS15 narrow  base U+1F610 (width 2) + U+FE0E  into 1x1 and 5x1 (INCREMENTAL)')
# NOTE: base MUST be a default-width-2 emoji so the narrowing precondition
# (gpu_cell->attrs.width == 2) at kitty/screen.c:L696 can actually execute.
for cols in (1, 5):
    s, c = new_screen(1, cols)
    show(s, f'{cols}x1 before')
    parse_bytes(s, '\U0001F610'.encode('utf-8'))
    show(s, f'{cols}x1 after base U+1F610 (width 2)  <-- b{chr(39)}\\xf0\\x9f\\x98\\x90{chr(39)}')
    print(f'      CSI 6 n reply = {query(s, c, CSI6N)!r}')
    parse_bytes(s, '\uFE0E'.encode('utf-8'))
    show(s, f'{cols}x1 after +VS15 U+FE0E (narrow)  <-- b{chr(39)}\\xef\\xb8\\x8e{chr(39)}')
    r = query(s, c, CSI6N)
    print(f'      CSI 6 n reply = {r!r}')
    rec(f'vs15_{cols}_cps', cps(str(s.line(0))))
    rec(f'vs15_{cols}_cursor_x', s.cursor.x)
    rec(f'vs15_{cols}_cpr', r)

hdr('SECONDARY: regional-indicator flag pair (US)  U+1F1FA U+1F1F8  into 1x1 and 5x1 (INCREMENTAL)')
for cols in (1, 5):
    s, c = new_screen(1, cols)
    show(s, f'{cols}x1 before', with_hist=True)
    parse_bytes(s, '\U0001F1FA'.encode('utf-8'))
    show(s, f'{cols}x1 after 1st RI U+1F1FA', with_hist=True)
    print(f'      CSI 6 n reply = {query(s, c, CSI6N)!r}')
    parse_bytes(s, '\U0001F1F8'.encode('utf-8'))
    show(s, f'{cols}x1 after 2nd RI U+1F1F8 (appended)', with_hist=True)
    r = query(s, c, CSI6N)
    print(f'      CSI 6 n reply = {r!r}')
    rec(f'flag_{cols}_cps', cps(str(s.line(0))))
    rec(f'flag_{cols}_cursor_x', s.cursor.x)
    rec(f'flag_{cols}_cpr', r)

hdr('SECONDARY: skin-tone modifier  U+1F44B U+1F3FF  into 1x1 and 5x1 (INCREMENTAL)')
for cols in (1, 5):
    s, c = new_screen(1, cols)
    show(s, f'{cols}x1 before', with_hist=True)
    parse_bytes(s, '\U0001F44B'.encode('utf-8'))
    show(s, f'{cols}x1 after base wave U+1F44B', with_hist=True)
    print(f'      CSI 6 n reply = {query(s, c, CSI6N)!r}')
    parse_bytes(s, '\U0001F3FF'.encode('utf-8'))
    show(s, f'{cols}x1 after +skin-tone U+1F3FF (appended)', with_hist=True)
    r = query(s, c, CSI6N)
    print(f'      CSI 6 n reply = {r!r}')
    rec(f'skin_{cols}_cps', cps(str(s.line(0))))
    rec(f'skin_{cols}_cursor_x', s.cursor.x)
    rec(f'skin_{cols}_cpr', r)

hdr('SECONDARY: cc_idx OVERFLOW  A + U+0300 U+0301 U+0302 U+0303  (5x1, >3 marks, INCREMENTAL)')
s, c = new_screen(1, 5)
show(s, 'before')
seq = ['A', '\u0300', '\u0301', '\u0302', '\u0303']
lbl = ['base A', '+U+0300 (fills slot0)', '+U+0301 (fills slot1)', '+U+0302 (fills slot2)',
       '+U+0303 (OVERFLOW -> overwrites last slot)']
for ch, l in zip(seq, lbl):
    parse_bytes(s, ch.encode('utf-8'))
    show(s, l)
rec('overflow_cps', cps(str(s.line(0))))
rec('overflow_cursor_x', s.cursor.x)

hdr('CONTRAST: whole FAMILY into WIDE grid 20x1 (vs 1x1 collapse), then CSI 6 n')
s, c = new_screen(1, 20)
show(s, '20x1 before', with_hist=True)
parse_bytes(s, FAMILY.encode('utf-8'))
show(s, '20x1 after whole family stream', with_hist=True)
r = query(s, c, CSI6N)
print(f'      CSI 6 n reply = {r!r}')
rec('wide_cps', cps(str(s.line(0))))
rec('wide_cursor_x', s.cursor.x)
rec('wide_hist_count', s.historybuf.count)
rec('wide_cpr', r)

hdr('RELATED REPORTS on the settled 1x1 family cell (all via the REAL parser)')
s, c = new_screen(1, 1)
parse_bytes(s, FAMILY.encode('utf-8'))
print(f'   settled cell = {str(s.line(0))!r}  cursor=(x={s.cursor.x},y={s.cursor.y})')
QUERIES = [
    ('DSR-5  CSI 5 n   ', b'\x1b[5n'),
    ('DA     CSI c     ', b'\x1b[c'),
    ('DA>    CSI > c   ', b'\x1b[>c'),
    ('size14 CSI 14 t  ', b'\x1b[14t'),
    ('size16 CSI 16 t  ', b'\x1b[16t'),
    ('size18 CSI 18 t  ', b'\x1b[18t'),
    ('DECRPM CSI ?25 $p', b'\x1b[?25$p'),
    ('DECRQSS DECSCUSR ', b'\x1bP$q q\x1b\\'),
]
reports = {}
for name, q in QUERIES:
    r = query(s, c, q)
    reports[name.strip()] = r
    print(f'   {name}  parse_bytes(s, {q!r})  -> {r!r}')
rec('rep_dsr5', reports['DSR-5  CSI 5 n'])
rec('rep_da', reports['DA     CSI c'])
rec('rep_da_secondary', reports['DA>    CSI > c'])
rec('rep_size14', reports['size14 CSI 14 t'])
rec('rep_size16', reports['size16 CSI 16 t'])
rec('rep_size18', reports['size18 CSI 18 t'])
rec('rep_decrpm', reports['DECRPM CSI ?25 $p'])
rec('rep_decrqss', reports['DECRQSS DECSCUSR'])

# ---------------------------------------------------------------------------
# GOLDEN ASSERTIONS: every observed value is checked against its expected value.
# A byte-for-byte mismatch here fails loudly (SystemExit 1) rather than passing silently.
hdr('GOLDEN ASSERTIONS (observed vs expected)')
EXPECTED = {
    # Primary family, 1x1 (visible cell keeps only the last base; earlier lines go to history)
    'fam_incr_visible': '\U0001F466',
    'fam_incr_cursor': (2, 0),
    'fam_incr_hist': ['\U0001F467\u200d', '\U0001F469\u200d', '\U0001F468\u200d', ''],
    'fam_visible': '\U0001F466',
    'fam_cps': ['U+1F466'],
    'fam_cursor': (2, 0),
    'fam_hist': ['\U0001F467\u200d', '\U0001F469\u200d', '\U0001F468\u200d', ''],
    'fam_cpr': b'\x1b[1;2R',
    # VS16 widen (base width 1 -> width 2)
    'vs16_1_cps': ['U+2764', 'U+FE0F'], 'vs16_1_cursor_x': 1, 'vs16_1_cpr': b'\x1b[1;1R',
    'vs16_5_cps': ['U+2764', 'U+FE0F'], 'vs16_5_cursor_x': 2, 'vs16_5_cpr': b'\x1b[1;3R',
    # VS15 narrow (base width 2 -> width 1) -- requires width-2 base U+1F610
    'vs15_1_cps': ['U+1F610', 'U+FE0E'], 'vs15_1_cursor_x': 1, 'vs15_1_cpr': b'\x1b[1;1R',
    'vs15_5_cps': ['U+1F610', 'U+FE0E'], 'vs15_5_cursor_x': 1, 'vs15_5_cpr': b'\x1b[1;2R',
    # Regional-indicator flag pair (second RI appended -> both retained even in 1x1)
    'flag_1_cps': ['U+1F1FA', 'U+1F1F8'], 'flag_1_cursor_x': 2, 'flag_1_cpr': b'\x1b[1;2R',
    'flag_5_cps': ['U+1F1FA', 'U+1F1F8'], 'flag_5_cursor_x': 2, 'flag_5_cpr': b'\x1b[1;3R',
    # Skin-tone modifier appended -> both retained even in 1x1
    'skin_1_cps': ['U+1F44B', 'U+1F3FF'], 'skin_1_cursor_x': 2, 'skin_1_cpr': b'\x1b[1;2R',
    'skin_5_cps': ['U+1F44B', 'U+1F3FF'], 'skin_5_cursor_x': 2, 'skin_5_cpr': b'\x1b[1;3R',
    # cc_idx overflow: 4th mark overwrites the last slot (U+0302 replaced by U+0303)
    'overflow_cps': ['U+0041', 'U+0300', 'U+0301', 'U+0303'], 'overflow_cursor_x': 1,
    # Wide grid retains whole cluster
    'wide_cps': ['U+1F468', 'U+200D', 'U+1F469', 'U+200D', 'U+1F467', 'U+200D', 'U+1F466'],
    'wide_cursor_x': 8, 'wide_hist_count': 0, 'wide_cpr': b'\x1b[1;9R',
    # Related reports
    'rep_dsr5': b'\x1b[0n',
    'rep_da': b'\x1b[?62;c',
    'rep_da_secondary': b'\x1b[>1;4000;35c',
    'rep_size14': b'\x1b[4;20;10t',
    'rep_size16': b'\x1b[6;20;10t',
    'rep_size18': b'\x1b[8;1;1t',
    'rep_decrpm': b'\x1b[?25;1$y',
    'rep_decrqss': b'\x1bP1$r1 q\x1b\\',
}
passed = failed = 0
for name, expected in EXPECTED.items():
    observed = OBS.get(name, '<MISSING>')
    ok = observed == expected
    passed += ok
    failed += (not ok)
    print(f'   [{"PASS" if ok else "FAIL"}] {name}: observed={observed!r}' + ('' if ok else f'  expected={expected!r}'))
print(f'\nASSERTIONS: {passed} passed, {failed} failed (of {len(EXPECTED)})')

print('\n[OBSERVE-DONE]')
if failed:
    raise SystemExit(1)
```

### 10.2 Complete, unedited captured output

The following is the **complete, unedited** output of one run (no ellipses, no truncation). Running the identical script a second time produced **byte-identical** output, confirming stability (§2.5). Hashes of the captured output: MD5 `77a2325fbe5315bbbf5ff19ad1d77f57`, SHA-256 `d26db8c171e8deef9e0bcdebb7e6a40c8e4c6b7d31bd94e51160c729732b1101`. The final line `ASSERTIONS: 46 passed, 0 failed (of 46)` is the script's own golden self-check (finding F10).

```text
get_options() type after canonical init = Options
FAMILY emoji codepoints = ['U+1F468', 'U+200D', 'U+1F469', 'U+200D', 'U+1F467', 'U+200D', 'U+1F466'] (Python len=7)
FAMILY UTF-8 bytes      = b'\xf0\x9f\x91\xa8\xe2\x80\x8d\xf0\x9f\x91\xa9\xe2\x80\x8d\xf0\x9f\x91\xa7\xe2\x80\x8d\xf0\x9f\x91\xa6'

==============================================================================
PRIMARY: family ZWJ emoji -> 1x1 Screen (INCREMENTAL: before/during/after + history)
==============================================================================
   [before (empty screen)]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
      historybuf    = count=0  lines=[]
   [after Man U+1F468 (width2 base)  <-- fed bytes b'\xf0\x9f\x91\xa8']
      str(line0)   = '👨'
      codepoints    = ['U+1F468']  (len=1)
      as_ansi()     = '👨'
      cursor        = (x=2, y=0)
      historybuf    = count=1  lines=['']
   [after ZWJ U+200D  <-- fed bytes b'\xe2\x80\x8d']
      str(line0)   = '👨\u200d'
      codepoints    = ['U+1F468', 'U+200D']  (len=2)
      as_ansi()     = '👨\u200d'
      cursor        = (x=2, y=0)
      historybuf    = count=1  lines=['']
   [after Woman U+1F469 (new width2 base)  <-- fed bytes b'\xf0\x9f\x91\xa9']
      str(line0)   = '👩'
      codepoints    = ['U+1F469']  (len=1)
      as_ansi()     = '👩'
      cursor        = (x=2, y=0)
      historybuf    = count=2  lines=['👨\u200d', '']
   [after ZWJ U+200D  <-- fed bytes b'\xe2\x80\x8d']
      str(line0)   = '👩\u200d'
      codepoints    = ['U+1F469', 'U+200D']  (len=2)
      as_ansi()     = '👩\u200d'
      cursor        = (x=2, y=0)
      historybuf    = count=2  lines=['👨\u200d', '']
   [after Girl U+1F467 (new width2 base)  <-- fed bytes b'\xf0\x9f\x91\xa7']
      str(line0)   = '👧'
      codepoints    = ['U+1F467']  (len=1)
      as_ansi()     = '👧'
      cursor        = (x=2, y=0)
      historybuf    = count=3  lines=['👩\u200d', '👨\u200d', '']
   [after ZWJ U+200D  <-- fed bytes b'\xe2\x80\x8d']
      str(line0)   = '👧\u200d'
      codepoints    = ['U+1F467', 'U+200D']  (len=2)
      as_ansi()     = '👧\u200d'
      cursor        = (x=2, y=0)
      historybuf    = count=3  lines=['👩\u200d', '👨\u200d', '']
   [after Boy U+1F466 (final width2 base)  <-- fed bytes b'\xf0\x9f\x91\xa6']
      str(line0)   = '👦'
      codepoints    = ['U+1F466']  (len=1)
      as_ansi()     = '👦'
      cursor        = (x=2, y=0)
      historybuf    = count=4  lines=['👧\u200d', '👩\u200d', '👨\u200d', '']

==============================================================================
PRIMARY (canonical single-stream): whole FAMILY at once -> 1x1, then CSI 6 n
==============================================================================
   [before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
      historybuf    = count=0  lines=[]
   [after whole family stream (settled)]
      str(line0)   = '👦'
      codepoints    = ['U+1F466']  (len=1)
      as_ansi()     = '👦'
      cursor        = (x=2, y=0)
      historybuf    = count=4  lines=['👧\u200d', '👩\u200d', '👨\u200d', '']
      CSI 6 n  cmd=parse_bytes(s, b'\x1b[6n')  -> reply = b'\x1b[1;2R'

==============================================================================
SECONDARY: VS16 widen  base U+2764 (width 1) + U+FE0F  into 1x1 and 5x1 (INCREMENTAL)
==============================================================================
   [1x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
   [1x1 after base U+2764 (width 1)  <-- b'\xe2\x9d\xa4']
      str(line0)   = '❤'
      codepoints    = ['U+2764']  (len=1)
      as_ansi()     = '❤'
      cursor        = (x=1, y=0)
      CSI 6 n reply = b'\x1b[1;1R'
   [1x1 after +VS16 U+FE0F (widen)  <-- b'\xef\xb8\x8f']
      str(line0)   = '❤️'
      codepoints    = ['U+2764', 'U+FE0F']  (len=2)
      as_ansi()     = '❤️'
      cursor        = (x=1, y=0)
      CSI 6 n reply = b'\x1b[1;1R'
   [5x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
   [5x1 after base U+2764 (width 1)  <-- b'\xe2\x9d\xa4']
      str(line0)   = '❤'
      codepoints    = ['U+2764']  (len=1)
      as_ansi()     = '❤'
      cursor        = (x=1, y=0)
      CSI 6 n reply = b'\x1b[1;2R'
   [5x1 after +VS16 U+FE0F (widen)  <-- b'\xef\xb8\x8f']
      str(line0)   = '❤️'
      codepoints    = ['U+2764', 'U+FE0F']  (len=2)
      as_ansi()     = '❤️'
      cursor        = (x=2, y=0)
      CSI 6 n reply = b'\x1b[1;3R'

==============================================================================
SECONDARY: VS15 narrow  base U+1F610 (width 2) + U+FE0E  into 1x1 and 5x1 (INCREMENTAL)
==============================================================================
   [1x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
   [1x1 after base U+1F610 (width 2)  <-- b'\xf0\x9f\x98\x90']
      str(line0)   = '😐'
      codepoints    = ['U+1F610']  (len=1)
      as_ansi()     = '😐'
      cursor        = (x=2, y=0)
      CSI 6 n reply = b'\x1b[1;2R'
   [1x1 after +VS15 U+FE0E (narrow)  <-- b'\xef\xb8\x8e']
      str(line0)   = '😐︎'
      codepoints    = ['U+1F610', 'U+FE0E']  (len=2)
      as_ansi()     = '😐︎'
      cursor        = (x=1, y=0)
      CSI 6 n reply = b'\x1b[1;1R'
   [5x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
   [5x1 after base U+1F610 (width 2)  <-- b'\xf0\x9f\x98\x90']
      str(line0)   = '😐'
      codepoints    = ['U+1F610']  (len=1)
      as_ansi()     = '😐'
      cursor        = (x=2, y=0)
      CSI 6 n reply = b'\x1b[1;3R'
   [5x1 after +VS15 U+FE0E (narrow)  <-- b'\xef\xb8\x8e']
      str(line0)   = '😐︎'
      codepoints    = ['U+1F610', 'U+FE0E']  (len=2)
      as_ansi()     = '😐︎'
      cursor        = (x=1, y=0)
      CSI 6 n reply = b'\x1b[1;2R'

==============================================================================
SECONDARY: regional-indicator flag pair (US)  U+1F1FA U+1F1F8  into 1x1 and 5x1 (INCREMENTAL)
==============================================================================
   [1x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
      historybuf    = count=0  lines=[]
   [1x1 after 1st RI U+1F1FA]
      str(line0)   = '🇺'
      codepoints    = ['U+1F1FA']  (len=1)
      as_ansi()     = '🇺'
      cursor        = (x=2, y=0)
      historybuf    = count=1  lines=['']
      CSI 6 n reply = b'\x1b[1;2R'
   [1x1 after 2nd RI U+1F1F8 (appended)]
      str(line0)   = '🇺🇸'
      codepoints    = ['U+1F1FA', 'U+1F1F8']  (len=2)
      as_ansi()     = '🇺🇸'
      cursor        = (x=2, y=0)
      historybuf    = count=1  lines=['']
      CSI 6 n reply = b'\x1b[1;2R'
   [5x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
      historybuf    = count=0  lines=[]
   [5x1 after 1st RI U+1F1FA]
      str(line0)   = '🇺'
      codepoints    = ['U+1F1FA']  (len=1)
      as_ansi()     = '🇺'
      cursor        = (x=2, y=0)
      historybuf    = count=0  lines=[]
      CSI 6 n reply = b'\x1b[1;3R'
   [5x1 after 2nd RI U+1F1F8 (appended)]
      str(line0)   = '🇺🇸'
      codepoints    = ['U+1F1FA', 'U+1F1F8']  (len=2)
      as_ansi()     = '🇺🇸'
      cursor        = (x=2, y=0)
      historybuf    = count=0  lines=[]
      CSI 6 n reply = b'\x1b[1;3R'

==============================================================================
SECONDARY: skin-tone modifier  U+1F44B U+1F3FF  into 1x1 and 5x1 (INCREMENTAL)
==============================================================================
   [1x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
      historybuf    = count=0  lines=[]
   [1x1 after base wave U+1F44B]
      str(line0)   = '👋'
      codepoints    = ['U+1F44B']  (len=1)
      as_ansi()     = '👋'
      cursor        = (x=2, y=0)
      historybuf    = count=1  lines=['']
      CSI 6 n reply = b'\x1b[1;2R'
   [1x1 after +skin-tone U+1F3FF (appended)]
      str(line0)   = '👋🏿'
      codepoints    = ['U+1F44B', 'U+1F3FF']  (len=2)
      as_ansi()     = '👋🏿'
      cursor        = (x=2, y=0)
      historybuf    = count=1  lines=['']
      CSI 6 n reply = b'\x1b[1;2R'
   [5x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
      historybuf    = count=0  lines=[]
   [5x1 after base wave U+1F44B]
      str(line0)   = '👋'
      codepoints    = ['U+1F44B']  (len=1)
      as_ansi()     = '👋'
      cursor        = (x=2, y=0)
      historybuf    = count=0  lines=[]
      CSI 6 n reply = b'\x1b[1;3R'
   [5x1 after +skin-tone U+1F3FF (appended)]
      str(line0)   = '👋🏿'
      codepoints    = ['U+1F44B', 'U+1F3FF']  (len=2)
      as_ansi()     = '👋🏿'
      cursor        = (x=2, y=0)
      historybuf    = count=0  lines=[]
      CSI 6 n reply = b'\x1b[1;3R'

==============================================================================
SECONDARY: cc_idx OVERFLOW  A + U+0300 U+0301 U+0302 U+0303  (5x1, >3 marks, INCREMENTAL)
==============================================================================
   [before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
   [base A]
      str(line0)   = 'A'
      codepoints    = ['U+0041']  (len=1)
      as_ansi()     = 'A'
      cursor        = (x=1, y=0)
   [+U+0300 (fills slot0)]
      str(line0)   = 'À'
      codepoints    = ['U+0041', 'U+0300']  (len=2)
      as_ansi()     = 'À'
      cursor        = (x=1, y=0)
   [+U+0301 (fills slot1)]
      str(line0)   = 'À́'
      codepoints    = ['U+0041', 'U+0300', 'U+0301']  (len=3)
      as_ansi()     = 'À́'
      cursor        = (x=1, y=0)
   [+U+0302 (fills slot2)]
      str(line0)   = 'À́̂'
      codepoints    = ['U+0041', 'U+0300', 'U+0301', 'U+0302']  (len=4)
      as_ansi()     = 'À́̂'
      cursor        = (x=1, y=0)
   [+U+0303 (OVERFLOW -> overwrites last slot)]
      str(line0)   = 'À́̃'
      codepoints    = ['U+0041', 'U+0300', 'U+0301', 'U+0303']  (len=4)
      as_ansi()     = 'À́̃'
      cursor        = (x=1, y=0)

==============================================================================
CONTRAST: whole FAMILY into WIDE grid 20x1 (vs 1x1 collapse), then CSI 6 n
==============================================================================
   [20x1 before]
      str(line0)   = ''
      codepoints    = []  (len=0)
      as_ansi()     = ''
      cursor        = (x=0, y=0)
      historybuf    = count=0  lines=[]
   [20x1 after whole family stream]
      str(line0)   = '👨\u200d👩\u200d👧\u200d👦'
      codepoints    = ['U+1F468', 'U+200D', 'U+1F469', 'U+200D', 'U+1F467', 'U+200D', 'U+1F466']  (len=7)
      as_ansi()     = '👨\u200d👩\u200d👧\u200d👦'
      cursor        = (x=8, y=0)
      historybuf    = count=0  lines=[]
      CSI 6 n reply = b'\x1b[1;9R'

==============================================================================
RELATED REPORTS on the settled 1x1 family cell (all via the REAL parser)
==============================================================================
   settled cell = '👦'  cursor=(x=2,y=0)
   DSR-5  CSI 5 n     parse_bytes(s, b'\x1b[5n')  -> b'\x1b[0n'
   DA     CSI c       parse_bytes(s, b'\x1b[c')  -> b'\x1b[?62;c'
   DA>    CSI > c     parse_bytes(s, b'\x1b[>c')  -> b'\x1b[>1;4000;35c'
   size14 CSI 14 t    parse_bytes(s, b'\x1b[14t')  -> b'\x1b[4;20;10t'
   size16 CSI 16 t    parse_bytes(s, b'\x1b[16t')  -> b'\x1b[6;20;10t'
   size18 CSI 18 t    parse_bytes(s, b'\x1b[18t')  -> b'\x1b[8;1;1t'
   DECRPM CSI ?25 $p  parse_bytes(s, b'\x1b[?25$p')  -> b'\x1b[?25;1$y'
   DECRQSS DECSCUSR   parse_bytes(s, b'\x1bP$q q\x1b\\')  -> b'\x1bP1$r1 q\x1b\\'

==============================================================================
GOLDEN ASSERTIONS (observed vs expected)
==============================================================================
   [PASS] fam_incr_visible: observed='👦'
   [PASS] fam_incr_cursor: observed=(2, 0)
   [PASS] fam_incr_hist: observed=['👧\u200d', '👩\u200d', '👨\u200d', '']
   [PASS] fam_visible: observed='👦'
   [PASS] fam_cps: observed=['U+1F466']
   [PASS] fam_cursor: observed=(2, 0)
   [PASS] fam_hist: observed=['👧\u200d', '👩\u200d', '👨\u200d', '']
   [PASS] fam_cpr: observed=b'\x1b[1;2R'
   [PASS] vs16_1_cps: observed=['U+2764', 'U+FE0F']
   [PASS] vs16_1_cursor_x: observed=1
   [PASS] vs16_1_cpr: observed=b'\x1b[1;1R'
   [PASS] vs16_5_cps: observed=['U+2764', 'U+FE0F']
   [PASS] vs16_5_cursor_x: observed=2
   [PASS] vs16_5_cpr: observed=b'\x1b[1;3R'
   [PASS] vs15_1_cps: observed=['U+1F610', 'U+FE0E']
   [PASS] vs15_1_cursor_x: observed=1
   [PASS] vs15_1_cpr: observed=b'\x1b[1;1R'
   [PASS] vs15_5_cps: observed=['U+1F610', 'U+FE0E']
   [PASS] vs15_5_cursor_x: observed=1
   [PASS] vs15_5_cpr: observed=b'\x1b[1;2R'
   [PASS] flag_1_cps: observed=['U+1F1FA', 'U+1F1F8']
   [PASS] flag_1_cursor_x: observed=2
   [PASS] flag_1_cpr: observed=b'\x1b[1;2R'
   [PASS] flag_5_cps: observed=['U+1F1FA', 'U+1F1F8']
   [PASS] flag_5_cursor_x: observed=2
   [PASS] flag_5_cpr: observed=b'\x1b[1;3R'
   [PASS] skin_1_cps: observed=['U+1F44B', 'U+1F3FF']
   [PASS] skin_1_cursor_x: observed=2
   [PASS] skin_1_cpr: observed=b'\x1b[1;2R'
   [PASS] skin_5_cps: observed=['U+1F44B', 'U+1F3FF']
   [PASS] skin_5_cursor_x: observed=2
   [PASS] skin_5_cpr: observed=b'\x1b[1;3R'
   [PASS] overflow_cps: observed=['U+0041', 'U+0300', 'U+0301', 'U+0303']
   [PASS] overflow_cursor_x: observed=1
   [PASS] wide_cps: observed=['U+1F468', 'U+200D', 'U+1F469', 'U+200D', 'U+1F467', 'U+200D', 'U+1F466']
   [PASS] wide_cursor_x: observed=8
   [PASS] wide_hist_count: observed=0
   [PASS] wide_cpr: observed=b'\x1b[1;9R'
   [PASS] rep_dsr5: observed=b'\x1b[0n'
   [PASS] rep_da: observed=b'\x1b[?62;c'
   [PASS] rep_da_secondary: observed=b'\x1b[>1;4000;35c'
   [PASS] rep_size14: observed=b'\x1b[4;20;10t'
   [PASS] rep_size16: observed=b'\x1b[6;20;10t'
   [PASS] rep_size18: observed=b'\x1b[8;1;1t'
   [PASS] rep_decrpm: observed=b'\x1b[?25;1$y'
   [PASS] rep_decrqss: observed=b'\x1bP1$r1 q\x1b\\'

ASSERTIONS: 46 passed, 0 failed (of 46)

[OBSERVE-DONE]
```

> **Rendering note on the overflow block (§7.6):** in the `cc_idx` overflow section the combining marks stack visually on the base `A`, and different fonts/terminals may render the stacked glyphs differently. The authoritative representation is the **codepoint list**, which is stable and shown above: after the fourth mark the retained codepoints are `['U+0041','U+0300','U+0301','U+0303']` — the last slot's `U+0302` was overwritten by `U+0303` while `U+0300` and `U+0301` remain, exactly as `kitty/line.c:L466` dictates.

---

*End of document. All observed values above were produced by the built `kitty.fast_data_types` engine driven through the real VT parser at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`; the repository working tree was left unchanged and the temporary observation script was deleted.*
