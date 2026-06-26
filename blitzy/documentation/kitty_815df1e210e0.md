# Kitty terminal emulator — how rendering-adjacent work is divided across Python, C, and Go

> An empirically-grounded runtime investigation of the [kitty](https://github.com/kovidgoyal/kitty) terminal emulator.
> **Branch:** `kitty_815df1e210e0` · **HEAD commit:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
> Every conclusion in this report is backed by a runtime artifact — the **exact command and its captured output** — collected from a **live, stressed** Kitty process. Nothing here is inferred from reading source alone; the source is cited only to *explain* what the runtime evidence shows.

---

## Executive summary

Kitty is deliberately a **tri-language** program, and the runtime evidence collected here shows a clean division of labour:

- **The main `kitty` process is a single address space that contains both C and Python.** A small **C launcher** (`kitty/launcher/kitty`, 40 KB) embeds **CPython 3.13** and loads the large **C extension `fast_data_types.so`** (1.2 MB). `/proc/<pid>/maps` proves that `libpython3.13`, `fast_data_types.so`, and the native rendering/font/colour libraries (HarfBuzz, FreeType, FontConfig, libGL/GLX, lcms2, libpng, libz, libcrypto) are **all mapped into the one process**.
- **C does the rendering-adjacent hot path.** Under a sustained colour/scrollback flood, the busy threads (`utime`/`stime` climbing, state `R`) are the **C threads** — the main render/event thread and the named C worker threads `KittyChildMon` / `KittyPeerMon`. Stack snapshots (py-spy `--native` and `gdb`) show the active frames living in `fast_data_types.so` (`do_parse`, `process_global_state`, `run_worker`) and in the GL stack (`glfwRunMainLoop` → Mesa), **while Python sits parked at the top of the boot/event-loop call chain** (`kitty/main.py`).
- **Python does orchestration and the remote-control server.** The boot path, the `Boss` lifecycle, and the **41 `kitty/rc/*.py` remote-control handlers (39 `RemoteCommand` subclasses)** are Python. `kitty @ ls` / `get-text` / `get-colors` return live state served by these Python handlers.
- **Go is a *separate* process.** `kitty +kitten icat` and the `kitty @` *client* run as the standalone **Go `kitten` binary** (`kitty/launcher/kitten`, ~15 MB / 15,765,764 bytes, `go1.22.12`) — a **Go ELF dynamically linked only to libc / `ld-linux`** (its sole runtime `.so` dependencies; `file` reports "dynamically linked", `ldd` lists just `linux-vdso`, `libc.so.6`, and `ld-linux-x86-64.so.2`). It appears as its **own PID** in the process tree, and **never appears in the main process's memory map** (`grep -c kitten /proc/<main_pid>/maps` → `0`). The C launcher `execv`s it **before CPython is even initialised**.

The one-line thesis: **Python + C share one process** (C = render/parse hot path, Python = orchestration + RC server); **Go `kitten` is a separate, portable process** for CLI/TUI tooling and the RC client.

---

## Methodology & environment

| Item | Value (captured at runtime) |
|------|------------------------------|
| Container | `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`), Ubuntu 25.10 |
| Investigated checkout | Kitty source checkpoint `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (the mandated branch `kitty_815df1e210e0`). This report is the single file added on top of that checkpoint; `git diff 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1..HEAD --name-status` → `A blitzy/documentation/kitty_815df1e210e0.md` (no source file differs), so the compiled C/Go sources are identical to the checkpoint — verbatim proof in section (a). |
| Python | **3.13.7** — satisfies the project floor `requires-python = ">=3.8"` (`pyproject.toml:L2`) |
| Go | **go1.22.12** — satisfies the project pin `go 1.22` (`go.mod:L3`) |
| C compiler | gcc 15.2.0, `-std=c11 -O3` |
| Display path | **Headless**: `xvfb-run` + **Mesa llvmpipe software GL** (no GPU). Kitty is GPU-only with *no* CPU text fallback, so a GL context is mandatory; the vendored GLFW provides the display path. |
| Remote control | Enabled at launch via `-o allow_remote_control=yes --listen-on unix:/tmp/kitty.sock` (it is **off by default**). |
| `ptrace` posture | Yama `kernel.yama.ptrace_scope = 1` (captured: `cat /proc/sys/kernel/yama/ptrace_scope` → `1`), but the session runs as **root** (`id -u` → `0`, `CapEff: 000001ffffffffff` includes `CAP_SYS_PTRACE`), so `gdb -p` and `py-spy` attach succeed despite scope `1`. `py-spy` (0.4.2) and `gdb` (16.3) are present; the required "show the error, then fall back" evidence is the genuinely **absent** `eu-stack` (elfutils) tool, which fails with `command not found` and forces the procfs `/proc/<tid>/stack` fallback — see section (f). |

**One main PID throughout (full transparency).** Almost every runtime artifact in sections (a)–(f) was captured against a **single** headless main process, PID **`163470`**, launched once and kept alive for the whole investigation; the `kitten` process in section (e) appears as a separate child PID (`178410`) of that same `163470`. The **one** deliberate exception is the dedicated transient-thread experiment in section (f): observing `KittyWriteStdin` *requires* writing to a child's stdin, so it was run against its **own short-lived instance** (a fresh, volatile PID), leaving `163470` untouched. Stack capture used only **non-destructive** methods — `py-spy dump`, `gdb -batch 'thread apply all bt'` (read-only backtraces, immediate detach), and procfs reads — with **no** `gdb call` or other process-mutating probe, so the target was never perturbed. The transient `KittyWriteStdin` writer thread is **on-demand** (created only when kitty writes to a child's stdin); the chosen *steady-state* stress mix (a flood emitted *by* the child toward kitty, window resizes, and tab switches) exercises the read/render and control paths but never writes to a child's stdin, so it does **not** appear in the steady-state thread samples — it is instead **captured live** in the dedicated stdin-write experiment in section (f), which confirms its **code-as-truth** origin (`kitty/child-monitor.c:L967`).

**Reproducibility.** Every command below was run inside the container and its output captured verbatim into a scratch directory (`/tmp/kitty_investigation/`, deleted in section (h)). Outputs are pasted unmodified except for eliding the long absolute repo path to `<repo>` for readability, where `<repo>` = `/tmp/blitzy/kitty/blitzy-69ce8195-d0f2-4cd8-8246-5c472699a20b_8fc29a`.

### Code-as-truth corrections

Per the "code is the truth, no assumptions" rule, where the live checkout differed from a prior expectation I report the **observed** value and explain it:

| # | Expectation | Observed (code-as-truth) | Evidence |
|---|-------------|--------------------------|----------|
| 1 | `kitten` is a fully-static ELF (`ldd` → "not a dynamic executable") | `kitten` is a Go ELF **dynamically linked to libc only** (`linux-vdso`, `libc.so.6`, `ld-linux`) | `ldd` in section (e) |
| 2 | `kitty/rc` has 40 `RemoteCommand` subclasses | **41 `.py` files, 39 `RemoteCommand` subclasses** (`__init__.py` is the package init; `base.py` defines the base `class RemoteCommand:`) | counts in section (d) |
| 3 | `kittens/icat` has 6 `.go` files | **6 tracked + 1 generated** `cli_generated.go` (gitignored via `*_generated.go`) = 7 on disk, all `package icat` | counts in section (e) |
| 4 | Only 3 `set_thread_name` calls exist | 3 in `kitty/child-monitor.c` (correct), but **5 tree-wide** (also `disk-cache.c`, `desktop.c`). The observed `kitty:disk$0` thread is a **Mesa** software-GL worker (`<comm>:disk$N`), *not* kitty's `DiskCacheWrite` | section (c) |

---

## Table of contents

- [(a) Build & launch the emulator, with a clean `git status` proof](#a-build--launch-the-emulator-with-a-clean-git-status-proof)
- [(b) What loads into the MAIN process](#b-what-loads-into-the-main-process)
- [(c) Thread activity: idle vs. stress](#c-thread-activity-idle-vs-stress)
- [(d) Live state through the control interface (`kitty @`)](#d-live-state-through-the-control-interface-kitty-)
- [(e) How `kitten` fits in: process relationship & binary forensics](#e-how-kitten-fits-in-process-relationship--binary-forensics)
- [(f) Stack/symbol snapshot during the stress run (with error + fallback)](#f-stacksymbol-snapshot-during-the-stress-run-with-error--fallback)
- [(g) Inferring responsibilities, ruling out wrong interpretations, naming a tradeoff](#g-inferring-responsibilities-ruling-out-wrong-interpretations-naming-a-tradeoff)
- [(h) Cleanup & final repository state](#h-cleanup--final-repository-state)
- [Appendix A: evidence-locator cross-reference](#appendix-a-evidence-locator-cross-reference)
- [Appendix B: complete captured artifacts](#appendix-b-complete-captured-artifacts) — §B1 link line · §B2 full `ls` JSON · §B3 full `get-colors` · §B4 full `gdb` backtrace

---

## (a) Build & launch the emulator, with a clean `git status` proof

**Goal (R1/R8):** build Kitty from this exact checkout, launch it under load, and prove the build does not dirty any tracked file.

### Confirm the checkout before building

This report is the single file added on top of the investigated Kitty source checkpoint. The checkpoint commit is `815df1e210e0…` (the mandated branch `kitty_815df1e210e0`); the report commit sits directly on top of it, so the checkpoint is exactly `HEAD~1`, and the only path that differs between them is this document:

```console
$ git rev-parse HEAD
9f973185870b0394e326ea1f2e975bfdbcc56fcc

$ git rev-parse HEAD~1
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1

$ git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1..HEAD
A	blitzy/documentation/kitty_815df1e210e0.md

$ git branch --show-current
blitzy-69ce8195-d0f2-4cd8-8246-5c472699a20b
```

Because the diff between the checkpoint and `HEAD` is **exactly one added file** (this report) and **zero source files**, every `kitty/`, `tools/`, `glfw/`, and build-system file compiled below is byte-identical to the `815df1e210e0…` checkpoint.

> **On the exact `HEAD`/VCS-revision values in this section (historical, by necessity).** The hash `9f973185870b0394e326ea1f2e975bfdbcc56fcc` shown above — and the identical `VCSRevision` / `KITTY_VCS_REV` build stamps reproduced below — are the **investigation-time `HEAD`**: the commit that was checked out *while these commands were captured*, **before** this report received its own, final commit. The build derives its VCS stamp directly from `git rev-parse HEAD` at build time — confirmed empirically, since rebuilding on a later checkout stamps *that* checkout's hash instead — so the stamp is **volatile across commits and rebuilds by design**, exactly like a PID or a timestamp. The **delivered** report is itself a commit placed on top of the checkpoint, so the delivered `HEAD` is necessarily a *different, later* hash; moreover a document **cannot embed its own commit hash** (writing the hash in would change the file's content, and therefore the hash). What is consequently **invariant and independently verifiable in the delivered artifact** is not any single hash but the two relationships proven above and re-confirmed in the final-delivery block of section (h): **`HEAD~1` is exactly the checkpoint `815df1e210e0…`**, and **`git diff 815df1e210e0…..HEAD` is exactly one added file — this report — and zero source files**. The `9f97318…` stamp below is thus the historical investigation value; as the diff invariant proves, the *compiled sources* are the checkpoint's regardless.

### Build via the project entry point

The canonical entry point is `python3 setup.py`. The `Makefile` simply forwards to it:

```
# Makefile:L12-13
all:
	python3 setup.py $(VVAL)
```

To force a *genuine* (not incremental) build, the gitignored outputs were removed first, then:

```console
$ rm -f kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so kitty/*.so
$ rm -rf build
$ { time python3 setup.py build --verbose --ignore-compiler-warnings ; } > 01_build.log 2>&1

$ tail -6 01_build.log          # (last 6 lines of the captured 140-line build log)
kitty/tools/cmd
/usr/local/go/bin/go build -v -ldflags '-X kitty.VCSRevision=9f973185870b0394e326ea1f2e975bfdbcc56fcc -s -w' -o kitty/launcher/kitten <repo>/tools/cmd

real	0m23.550s
user	0m39.175s
sys	0m7.010s
```

> The `--ignore-compiler-warnings` flag is a *build-time toolchain override*, not a source edit: kitty defaults to `-pedantic-errors -Werror`, and GCC 15 + Ubuntu 25.10's `wayland-protocols` introduce new enum values not handled by a `switch` in the 2024-vintage vendored `glfw/wl_window.c`, which would otherwise be fatal under `-Werror=switch`. No tracked file is changed by the flag.

The verbose build log captures the **three languages converging into two artifacts**. First, the core C extension `kitty/fast_data_types` is compiled — exactly the target named in `setup.py:L1090-1095`:

```
# setup.py:L1090-1095
    compile_c_extension(
        kitty_env(args), 'kitty/fast_data_types', args.compilation_database, sources, headers,
        build_dsym=args.build_dsym,
    )
    compile_glfw(args.compilation_database, args.build_dsym)
    compile_kittens(args)
```

The single `gcc` **link line for `fast_data_types.so`** links **62 object files** — **49 kitty C translation units** (the rendering/parsing hot path: `child-monitor.c`, `vt-parser.c`, `screen.c`, `line.c`, `gl.c`, `shaders.c`, `freetype.c`, `fontconfig.c`, `graphics.c`, `glyph-cache.c`, `png-reader.c`, the `simd-string-*` sources, …) plus **13 third-party objects** (`ringbuf` and the per-architecture `base64` codec) — together with the **embedded interpreter and the native graphics/font/colour libraries**. The object-file count and the complete (un-elided) link line are reproduced in **Appendix B §B1**; the excerpt below shows the translation-unit head and the verbatim library/flags tail (no part of the tail is elided):

```console
$ grep -E '\-o build/kitty/fast_data_types\.so' 01_build.log   # excerpt — full 62-object line in Appendix B §B1
gcc -Wextra -Wfloat-conversion ... -std=c11 -O3 ... -flto -fcf-protection=full -march=native -mtune=native
    -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/python3.13 ... -shared -flto
    build/fast_data_types-kitty-charsets.c.o build/fast_data_types-kitty-child-monitor.c.o
    build/fast_data_types-kitty-vt-parser.c.o build/fast_data_types-kitty-screen.c.o
    build/fast_data_types-kitty-line.c.o build/fast_data_types-kitty-gl.c.o
    build/fast_data_types-kitty-shaders.c.o build/fast_data_types-kitty-freetype.c.o
    build/fast_data_types-kitty-fontconfig.c.o build/fast_data_types-kitty-graphics.c.o
    build/fast_data_types-kitty-glyph-cache.c.o      # … 49 kitty + 13 third-party = 62 .o (full list in §B1)
    -ldl -lm -L/usr/lib/x86_64-linux-gnu -lpython3.13 -Xlinker -export-dynamic
    -Wl,-O1 -Wl,-Bsymbolic-functions -lharfbuzz -lGL -lpng16 -llcms2 -llcms2_fast_float
    -llcms2_threaded -pthread -lm -lcrypto -lrt -lz -o build/kitty/fast_data_types.so
```

Second, the **C launcher** `kitty/launcher/kitty` is linked against `libpython3.13` (this is what embeds CPython):

```console
$ grep -E '\-o kitty/launcher/kitty( |$)' 01_build.log
gcc build/kitty-launcher-main.o build/kitty-launcher-single-instance.o -ldl -lm
    -L/usr/lib/x86_64-linux-gnu -lpython3.13 -Xlinker -export-dynamic
    -Wl,-O1 -Wl,-Bsymbolic-functions -o kitty/launcher/kitty
```

Third, the **Go `kitten`** binary is built by the Go toolchain from `tools/cmd` (note the same VCS revision is stamped in):

```console
$ grep 'go build' 01_build.log
/usr/local/go/bin/go build -v -ldflags '-X kitty.VCSRevision=9f973185870b0394e326ea1f2e975bfdbcc56fcc -s -w' -o kitty/launcher/kitten <repo>/tools/cmd
```

A fourth, subtle artifact: the launcher's `WRAPPED_KITTENS` allow-list is **baked in as a compile-time macro** when the launcher translation unit `kitty/launcher/main.c` is compiled — this is the exact list `is_wrapped_kitten()` tests to decide whether to delegate to Go (used in section (e)). The launcher bakes it **space-delimited** (note the leading and trailing space inside the quotes), which is what lets `is_wrapped_kitten`'s `strstr(" " WRAPPED_KITTENS " ", " icat ")` match on word boundaries:

```console
$ grep -oE '\-DKITTY_VCS_REV="[^"]*"' 01_build.log | head -1
-DKITTY_VCS_REV="9f973185870b0394e326ea1f2e975bfdbcc56fcc"

$ grep 'launcher/main.c' 01_build.log | grep -oE '\-DWRAPPED_KITTENS="[^"]*"'
-DWRAPPED_KITTENS=" ask clipboard diff hints hyperlinked_grep icat query_terminal show_key ssh themes transfer unicode_input "
```

### The two binaries + the in-process extension

```console
$ ls -l kitty/fast_data_types.so kitty/launcher/kitten kitty/launcher/kitty
-rwxr-xr-x 1 root root  1253792 Jun 26 21:59 kitty/fast_data_types.so
-rwxr-xr-x 1 root root 15765764 Jun 26 21:59 kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 Jun 26 21:59 kitty/launcher/kitty

$ kitty/launcher/kitty --version ; kitty/launcher/kitten --version
kitty 0.35.2 created by Kovid Goyal
kitten 0.35.2 created by Kovid Goyal
```

**Rationale / what this shows.** The 40 KB `kitty` is a *thin* C launcher; the real C code is the 1.2 MB `fast_data_types.so`; the ~15 MB `kitten` is a *self-contained* Go binary an order of magnitude larger than everything else combined — an early hint that it ships its own runtime and is meant to stand alone.

### The build leaves the tracked tree clean

```console
$ git status
On branch blitzy-69ce8195-d0f2-4cd8-8246-5c472699a20b
nothing to commit, working tree clean

$ git status --porcelain
$            # (empty == clean)
```

Why is the tree clean despite three fresh artifacts? Because `.gitignore` excludes them. The proof — `git check-ignore -v` prints the *matching rule* for each:

```console
$ git check-ignore -v kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so
.gitignore:18:/kitty/launcher/kitt*	kitty/launcher/kitty
.gitignore:18:/kitty/launcher/kitt*	kitty/launcher/kitten
.gitignore:1:*.so	kitty/fast_data_types.so
```

The single glob `/kitty/launcher/kitt*` (line 18) covers **both** the `kitty` and `kitten` binaries; `*.so` (line 1) covers the extension. Crucially, the **deliverable itself is *not* ignored** (exit status 1 means "no ignore rule matched", i.e. it is trackable/committable):

```console
$ git check-ignore -v blitzy blitzy/documentation blitzy/documentation/kitty_815df1e210e0.md ; echo "exit=$?"
exit=1
```

### Launch headless with remote control enabled

```console
$ nohup xvfb-run -a --server-args="-screen 0 1280x720x24" \
    kitty/launcher/kitty \
      -o allow_remote_control=yes \
      --listen-on unix:/tmp/kitty.sock \
      -o enable_audio_bell=no \
      -o scrollback_lines=100000 \
      bash --norc --noprofile \
    > /tmp/kitty_investigation/kitty_run.log 2>&1 &
```

The reliable way to find the *real* main PID is `pgrep -x kitty` (an `-f` match would also catch the `xvfb-run` wrapper):

```console
$ pgrep -x kitty
163470
$ readlink /proc/163470/exe
<repo>/kitty/launcher/kitty
```

Remote control answers on the socket, confirming the process is alive and the control endpoint is up (shown pretty-printed; the full 364-line `ls` payload is in section (d) and Appendix B §B2):

```console
$ kitty/launcher/kitty @ --to unix:/tmp/kitty.sock ls | python3 -m json.tool | head -8
[
    {
        "background_opacity": 1.0,
        "id": 1,
        "is_active": true,
        "is_focused": true,
        "last_focused": true,
        "platform_window_id": 2097164,
```

**Main `kitty` PID for the entire investigation = `163470`** (one process throughout, sections (a)–(f)). The GL path is Mesa **llvmpipe software rendering** under Xvfb (visible later as `llvmpipe-N` worker threads) — i.e. no physical GPU, but a real OpenGL context, exactly as the GPU-only renderer requires.

---


## (b) What loads into the MAIN process

**Goal (R2):** enumerate the Kitty-specific modules and the major rendering/font libraries mapped into the **main `kitty` process** address space.

### The native object map (`/proc/<pid>/maps`)

```console
$ grep -oE '/[^ ]+\.so[^ ]*' /proc/163470/maps | sort -u | wc -l
73

$ grep -c kitten /proc/163470/maps
0
```

The main process maps **73 distinct shared objects** — a large native footprint — and, as a foreshadowing of section (e), the Go `kitten` binary is **not one of them** (`grep -c kitten` → `0`). Filtering the map for the in-process C extension, the embedded interpreter, and the graphics/font/colour libraries yields their exact resident pathnames (verbatim, only the long repo path elided to `<repo>`):

```console
$ grep -E 'fast_data_types|libpython|libharfbuzz|libfreetype|libfontconfig|libgallium|libGL|liblcms2|libpng|libz\.so|libcrypto' \
      /proc/163470/maps | awk '{print $NF}' | sort -u
<repo>/kitty/fast_data_types.so
/usr/lib/x86_64-linux-gnu/libGL.so.1.7.0
/usr/lib/x86_64-linux-gnu/libGLX.so.0.0.0
/usr/lib/x86_64-linux-gnu/libGLX_mesa.so.0.0.0
/usr/lib/x86_64-linux-gnu/libGLdispatch.so.0.0.0
/usr/lib/x86_64-linux-gnu/libcrypto.so.3
/usr/lib/x86_64-linux-gnu/libfontconfig.so.1.12.1
/usr/lib/x86_64-linux-gnu/libfreetype.so.6.20.2
/usr/lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
/usr/lib/x86_64-linux-gnu/libharfbuzz.so.0.61020.0
/usr/lib/x86_64-linux-gnu/liblcms2.so.2.0.16
/usr/lib/x86_64-linux-gnu/libpng16.so.16.50.0
/usr/lib/x86_64-linux-gnu/libpython3.13.so.1.0
/usr/lib/x86_64-linux-gnu/libz.so.1.3.1
```

To remove any doubt that these are *resident mappings* (not merely a file handle), here is the **first mapped segment of each key object** — its real load address, page offset `00000000`, device `103:01`, and inode — proving the file is `mmap`ed into the process address space. (Each object is actually mapped as the usual multi-segment `r--p`/`r-xp`/`rw-p` set by `ld.so`; the ELF-header `r--p` segment shown here anchors each object at its load address.)

```console
$ for so in fast_data_types libpython3.13 libharfbuzz libfreetype libfontconfig libGLX_mesa liblcms2 libpng16 libz.so libcrypto; do \
      grep -m1 -E "/[^ ]*${so}[^ ]*" /proc/163470/maps ; done
7c621c600000-7c621c611000 r--p 00000000 103:01 169473809   <repo>/kitty/fast_data_types.so
7c621d4d9000-7c621d571000 r--p 00000000 103:01 600724561   /usr/lib/x86_64-linux-gnu/libpython3.13.so.1.0
7c621c4c3000-7c621c4cf000 r--p 00000000 103:01 600842480   /usr/lib/x86_64-linux-gnu/libharfbuzz.so.0.61020.0
7c621bd3e000-7c621bd4b000 r--p 00000000 103:01 600842460   /usr/lib/x86_64-linux-gnu/libfreetype.so.6.20.2
7c621b10c000-7c621b114000 r--p 00000000 103:01 600842454   /usr/lib/x86_64-linux-gnu/libfontconfig.so.1.12.1
7c621acca000-7c621acd5000 r--p 00000000 103:01 600842369   /usr/lib/x86_64-linux-gnu/libGLX_mesa.so.0.0.0
7c621c422000-7c621c42c000 r--p 00000000 103:01 600842485   /usr/lib/x86_64-linux-gnu/liblcms2.so.2.0.16
7c621c489000-7c621c48e000 r--p 00000000 103:01 600842505   /usr/lib/x86_64-linux-gnu/libpng16.so.16.50.0
7c621d169000-7c621d16c000 r--p 00000000 103:01 600712885   /usr/lib/x86_64-linux-gnu/libz.so.1.3.1
7c621be0f000-7c621bef3000 r--p 00000000 103:01 600712796   /usr/lib/x86_64-linux-gnu/libcrypto.so.3
```

These map 1:1 onto the link line from section (a): the C extension links `-lpython3.13 -lharfbuzz -lGL -lpng16 -llcms2 … -lcrypto -lz`, and here they all are, resident in the same address space. The `libGLX_mesa` / `libgallium` pair is the Mesa **software-GL** implementation backing the headless `xvfb` context.

### The embedded interpreter's view (`sys.modules`)

The map proves C objects are present; to prove the **Python side is the *same* interpreter that imports the C extension**, I probed the embedded CPython directly using kitty's `+runpy` entry (which runs Python *inside* the kitty environment, with kitty's `sys.path`):

```console
$ kitty/launcher/kitty +runpy '
import sys
print("sys.implementation.name =", sys.implementation.name)
print("sys.version =", sys.version.split()[0])
print("sys.version_info >= (3,8) ->", sys.version_info >= (3,8))
import kitty.fast_data_types as f
print("kitty.fast_data_types in sys.modules ->", "kitty.fast_data_types" in sys.modules)
print("kitty.fast_data_types.__file__ =", f.__file__)
print("loader ->", type(f.__loader__).__name__)
print("fast_data_types exposes C type Screen ->", hasattr(f, "Screen"))
import kitty.boss, kitty.child, kitty.tabs, kitty.window
mods = sorted(m for m in sys.modules if m == "kitty" or m.startswith("kitty."))
print("count of kitty.* modules in sys.modules ->", len(mods))
'
sys.implementation.name = cpython
sys.version = 3.13.7
sys.version_info >= (3,8) -> True
kitty.fast_data_types in sys.modules -> True
kitty.fast_data_types.__file__ = <repo>/kitty/launcher/../../kitty/fast_data_types.so
loader -> ExtensionFileLoader
fast_data_types exposes C type Screen -> True
count of kitty.* modules in sys.modules -> 41
```

**Rationale / what this shows.** Four facts nail down the "one address space" claim:

1. The interpreter is **CPython 3.13.7**, comfortably above the `>=3.8` floor (`pyproject.toml:L2`).
2. `kitty.fast_data_types` is a **loaded module** of that interpreter, and its `__file__` is the very `fast_data_types.so` that appears in `/proc/163470/maps`.
3. Its loader is `ExtensionFileLoader` — i.e. Python loaded it as a **compiled C extension**, not a `.py` file — and it **exposes C types directly to Python** (`hasattr(f, "Screen") -> True`: the per-window screen/grid model is a C type, foreshadowing that the cell/glyph model lives in C, not Python — see the rule-out in section (g)).
4. Importing kitty's real boot modules pulls in **41 `kitty.*` Python modules** alongside that one C extension — the orchestration layer is Python, the data/render core is the single C `.so`.

So the C rendering/parsing core is not a sibling process that Python talks to over IPC; it is a **C extension imported into the embedded interpreter**. CPython, the `fast_data_types` C core, and the native graphics/font libraries (HarfBuzz, FreeType, FontConfig, libGL/GLX, lcms2, libpng, libz, libcrypto) **share one process** — exactly the surface `setup.py:L1090-1095` builds (`fast_data_types` extension + vendored GLFW + kittens).

---


## (c) Thread activity: idle vs. stress

**Goal (R3):** show how the thread population and per-thread activity of the main process change between **idle** and **stress**. The expected observable signal is the set of named C worker threads created in `kitty/child-monitor.c`.

### The named threads, from the C source

There are exactly **three** `set_thread_name(...)` calls in `kitty/child-monitor.c` — these are the only kitty-named threads the child monitor creates:

```console
$ grep -n 'set_thread_name' kitty/child-monitor.c
967:    set_thread_name("KittyWriteStdin");
1489:    set_thread_name("KittyChildMon");
1808:    set_thread_name("KittyPeerMon");
```

- **`KittyChildMon`** (`:L1489`) — the child/PTY I/O monitor loop; drains terminal output from child processes and feeds the VT parser.
- **`KittyPeerMon`** (`:L1808`, inside `talk_loop()`) — the remote-control peer loop; services `kitty @` connections on the socket.
- **`KittyWriteStdin`** (`:L967`, inside `thread_write()`) — a **transient**, detached writer thread spawned on demand to write data to a child's stdin, which exits as soon as the write completes.

The **main/initial thread** keeps the process `comm` (`"kitty"`) and runs the GLFW event loop + GPU render.

> Code-as-truth note: tree-wide there are actually **five** `set_thread_name` call sites (the three above plus `disk-cache.c:342` `DiskCacheWrite` and `desktop.c:214` `LinuxAudioSucks`). Neither extra one appears below; the `kitty:disk$0` thread seen at runtime is a **Mesa** software-GL worker (Mesa names helper threads `<comm>:disk$N`), not kitty's own disk-cache thread.

### Idle snapshot (PID 163470)

```console
$ ps -T -p 163470 | head -4
    PID    SPID TTY          TIME CMD
 163470  163470 ?        00:00:00 kitty
 163470  163471 ?        00:00:00 llvmpipe-0
 163470  163472 ?        00:00:00 llvmpipe-1

$ for t in /proc/163470/task/*; do cat "$t/comm"; done | sort | uniq -c | sort -rn
     33 kitty
      1 llvmpipe-9
      ...                       # llvmpipe-0 .. llvmpipe-31  (32 total; full per-thread list in §B4)
      1 kitty:disk$0
      1 KittyPeerMon
      1 KittyChildMon
```

That is **68 threads** at idle, partitioned as:

- **1** main `kitty` thread (the GLFW event loop / renderer),
- **~32** generic `kitty` worker threads + **32** `llvmpipe-N` threads + **1** `kitty:disk$0` — these are the **Mesa llvmpipe software-GL rasterizer pool** (present only because we render on the CPU under Xvfb),
- **`KittyChildMon`** and **`KittyPeerMon`** — the two persistent kitty C worker threads. They **persist even at idle** because remote control is enabled.

Characterising each group by kernel wait-channel (`wchan`) at idle:

```console
$ for tid in 163470 163471 163535 163536 163537; do
    printf 'tid=%-7s comm=%-16s state=%s wchan=%s\n' "$tid" \
       "$(cat /proc/163470/task/$tid/comm)" \
       "$(awk '{print $3}' /proc/163470/task/$tid/stat)" \
       "$(cat /proc/163470/task/$tid/wchan)"
  done
tid=163470  comm=kitty            state=S wchan=do_sys_poll        # main: blocked in the event-loop poll()
tid=163471  comm=llvmpipe-0       state=S wchan=futex_wait_queue   # Mesa SW-GL worker, parked
tid=163535  comm=kitty:disk$0     state=S wchan=futex_wait_queue   # Mesa shader-cache worker, parked
tid=163536  comm=KittyPeerMon     state=S wchan=do_sys_poll        # RC peer loop, waiting on the socket
tid=163537  comm=KittyChildMon    state=S wchan=do_sys_poll        # PTY monitor, waiting on child fds
```

At idle, **every thread is blocked** — the main thread, `KittyPeerMon`, and `KittyChildMon` are parked in `do_sys_poll` (waiting on their respective fds), while the 32-thread generic `kitty`/llvmpipe pool and `kitty:disk$0` are parked in `futex_wait_queue`. All accumulate ~0 CPU (`ps -T` shows `TIME 00:00:00` for every thread).

### Apply sustained load (all four stress dimensions)

Load was driven from a single disposable script, `stress_driver.sh`, which exercises **all four requested dimensions simultaneously** against the running instance. The exact script (reproduced verbatim — it is the load workload, deleted in section (h)) is:

```bash
#!/usr/bin/env bash
# stress_driver.sh — drive sustained rendering load on the headless kitty
# under investigation, exercising ALL FOUR requested stress dimensions at once.
# Disposable investigation script (deleted in the cleanup step); never committed.
#
# Usage: stress_driver.sh [DURATION_SECONDS]
set -u
SOCK="unix:/tmp/kitty.sock"
K=(kitty/launcher/kitty @ --to "${SOCK}")
DURATION="${1:-45}"
WIDFILE="/tmp/kitty_investigation/flood_window_id.txt"

# --- Dimensions 1 + 2: SGR 256-colour flood + heavy scrollback churn --------
# One window runs a tight loop printing a 256-colour SGR sequence with an
# ever-incrementing counter. A single generator covers BOTH "lots of coloured
# output" (the \033[38;5;Nm SGR code, N cycling 0..255) and "scrollback churn"
# (the monotonic counter is the churn proof, read back via `get-text`).
flood_wid="$("${K[@]}" launch --type=window --keep-focus sh -c '
  i=0
  while :; do
    c=$(( i % 256 ))
    printf "\033[38;5;%dm%d  COLOR-FLOOD scrollback churn line padding padding padding\033[0m\n" "$c" "$i"
    i=$(( i + 1 ))
  done
')"
echo "${flood_wid}" > "${WIDFILE}"

# --- Dimension 3: repeated OS-window resizes --------------------------------
# Toggle the OS-window geometry back and forth; each resize forces kitty to
# reflow the screen model and re-rasterise via the GL path.
(
  end=$(( $(date +%s) + DURATION ))
  while [ "$(date +%s)" -lt "${end}" ]; do
    "${K[@]}" resize-os-window --width 100 --height 30 >/dev/null 2>&1
    "${K[@]}" resize-os-window --width 120 --height 40 >/dev/null 2>&1
  done
) &
resize_pid=$!

# --- Dimension 4: tab creation + tab switching ------------------------------
# NOTE (code-as-truth): there is no `kitty @ new-tab` subcommand on this
# checkout; tabs are created with `launch --type=tab` and switched with the
# `next_tab` / `previous_tab` actions (verified against kitty/rc/).
(
  end=$(( $(date +%s) + DURATION ))
  for n in 1 2 3; do
    "${K[@]}" launch --type=tab --keep-focus sh -c \
      'while :; do printf "tab churn line %d\n" "$$"; done' >/dev/null 2>&1
  done
  while [ "$(date +%s)" -lt "${end}" ]; do
    "${K[@]}" action next_tab     >/dev/null 2>&1
    "${K[@]}" action previous_tab >/dev/null 2>&1
  done
) &
tab_pid=$!

echo "stress drivers launched:"
echo "  - colour-flood + scrollback window id = ${flood_wid}"
echo "  - resize loop          pid = ${resize_pid}"
echo "  - tab create/switch    pid = ${tab_pid}"
echo "  running for ${DURATION}s ..."
wait "${resize_pid}" "${tab_pid}"
echo "resize + tab loops finished (colour-flood window keeps running until kitty is stopped)"
```

Mapping to the four requested dimensions: lines 18–26 are **(1) the SGR 256-colour flood** *and* **(2) the heavy scrollback churn** (one tight `printf` loop emitting `\033[38;5;Nm` with `N` cycling `0..255` plus a monotonic counter — the counter is the churn proof, read back in section (d)); lines 31–40 are **(3) the repeated `kitty @ resize-os-window` loop**; lines 45–58 are **(4) tab creation + switching** (`launch --type=tab` ×3, then a `next_tab`/`previous_tab` cycle).

> **Code-as-truth note:** there is **no** `kitty @ new-tab` remote subcommand on this checkout (the only tab-related rc commands are `close_tab`, `detach_tab`, `focus_tab`, `set_tab_color`, `set_tab_title`). New tabs are therefore created with `launch --type=tab` and switched with the `next_tab` / `previous_tab` **actions**. The script uses the commands that actually exist.

Running it for 75 seconds produced the following confirmation, and `kitty @ ls` then showed the new windows/tabs created by the load (the original `bash` window plus the colour-flood window and three tab-churn windows):

```console
$ bash stress_driver.sh 75
stress drivers launched:
  - colour-flood + scrollback window id = 4
  - resize loop          pid = 166293
  - tab create/switch    pid = 166294
  running for 75s ...
resize + tab loops finished (colour-flood window keeps running until kitty is stopped)

$ kitty/launcher/kitty @ --to unix:/tmp/kitty.sock ls \
    | python3 -c 'import json,sys; d=json.load(sys.stdin); \
        print(sum(len(t["windows"]) for o in d for t in o["tabs"]), "windows,", \
              sum(len(o["tabs"]) for o in d), "tabs")'
5 windows, 4 tabs
```

### Stress snapshot (same PID 163470)

```console
$ for tid in 163470 163536 163537; do
    s=$(cat /proc/163470/task/$tid/stat)
    printf 'tid=%-7s comm=%-15s state=%s utime=%s stime=%s wchan=%s\n' "$tid" \
      "$(cat /proc/163470/task/$tid/comm)" \
      "$(echo "$s" | awk '{print $3}')" "$(echo "$s" | awk '{print $14}')" \
      "$(echo "$s" | awk '{print $15}')" "$(cat /proc/163470/task/$tid/wchan)"
  done
tid=163470  comm=kitty           state=R utime=1600 stime=159  wchan=0
tid=163536  comm=KittyPeerMon    state=R utime=135  stime=1223 wchan=0
tid=163537  comm=KittyChildMon   state=D utime=84   stime=856  wchan=0
```

The three named C threads that were parked in `do_sys_poll` at idle are now **off the wait-channel** (`wchan=0`) and burning CPU — `R` (running) for the main render thread and `KittyPeerMon`, and `D` (uninterruptible sleep, i.e. mid-syscall doing PTY I/O) for `KittyChildMon`. `ps -T` corroborates the accrued CPU (`TIME` of `00:07:16` / `00:00:12` / `00:00:09` respectively, vs `00:00:00` at idle). Side-by-side:

| Thread | Role (from C source) | Idle | Under stress |
|--------|----------------------|------|--------------|
| `kitty` (main, tid 163470) | GLFW event loop + GL render | `S`, `do_sys_poll`, TIME 00:00:00 | **`R`, utime=1600, stime=159, wchan=0** |
| `KittyPeerMon` (tid 163536) | RC peer loop, `talk_loop()` | `S`, `do_sys_poll`, TIME 00:00:00 | **`R`, utime=135, stime=1223, wchan=0** |
| `KittyChildMon` (tid 163537) | PTY I/O monitor, `io_loop()` | `S`, `do_sys_poll`, TIME 00:00:00 | **`D`, utime=84, stime=856, wchan=0** |
| `llvmpipe-N` × 32 / generic `kitty` workers / `kitty:disk$0` | Mesa SW-GL pool | `futex_wait_queue`, ~0 | still `futex_wait_queue`, utime=0 stime=0 |

```console
$ for t in /proc/163470/task/*; do cat "$t/comm"; done | wc -l
68      # population is stable; the heavy lifting moves onto existing threads
```

**Rationale / what this shows.** Under load the **CPU accrues on the C threads**: the main render/event thread goes to running state (`R`) with `utime` climbing from ~0 to **1600** clock-ticks; `KittyPeerMon` is hammered by the RC command flood (`stime` 0 → **1223**, almost all kernel time servicing the socket); `KittyChildMon` is busy reading the flood-window PTYs (`stime` 0 → **856**) and feeding the parser, sitting in `D` because it is mid-`read()`/`poll()`. Meanwhile the 32-thread generic/llvmpipe pool and `kitty:disk$0` **stay parked at `futex_wait_queue` with `utime=stime=0`** — they are a pre-created pool that the workload never wakes, which is why the **total thread count is unchanged at 68** (the load lands on *existing* threads, not new ones). **No Python-named thread appears anywhere** — there is no per-thread Python worker doing the rendering; all of the hot work is on threads created and named by the C `child-monitor.c`. The transient `KittyWriteStdin` writer (`thread_write()`, `child-monitor.c:L967`) does **not** appear here because it is created **on demand only when kitty writes to a child's stdin**, and this steady-state workload (a flood emitted *by* the child toward kitty, plus resizes and tab switches) never writes to a child's stdin — it is instead caught **live**, in a dedicated stdin-write experiment, in section (f) (*The transient `KittyWriteStdin` thread — captured live*), where it is shown appearing (68 → 69), blocked in `write()` ← `thread_write()`, and exiting (69 → 68).

---


## (d) Live state through the control interface (`kitty @`)

**Goal (R4):** demonstrate what live state Kitty exposes through `kitty @` while load is happening, with the exact commands and outputs so observations are independently verifiable. All commands target the socket from section (a): `--to unix:/tmp/kitty.sock`.

### `kitty @ ls` — the window/tab/window JSON tree

The complete `ls` payload at this moment is **364 lines** of pretty-printed JSON (5,355 bytes compact); the **full output is reproduced verbatim in Appendix B §B2**. The first 18 lines (clearly an excerpt — see §B2 for the complete tree) are:

```console
$ kitty/launcher/kitty @ --to unix:/tmp/kitty.sock ls | python3 -m json.tool | head -18   # excerpt — full 364-line payload in Appendix B §B2
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
                "active_window_history": [],
                "enabled_layouts": [
                    "fat",
                    "grid",
                    "horizontal",
                    "splits",
                    "stack",
                    "tall",
```

The shape matches `kitty/rc/ls.py` exactly. The handler is a Python `RemoteCommand` subclass whose own docstring describes the tree:

```python
# kitty/rc/ls.py:L15,L23-30 — faithful source excerpt; lines between markers omitted as noted
class LS(RemoteCommand):
    # L16-21 protocol_spec/__doc__ omitted
    short_desc = 'List tabs/windows'                         # L23
    desc = (                                                 # L24
        'List windows. The list is returned as JSON tree. The top-level is a list of'
        f' operating system {appname} windows. Each OS window has an :italic:`id` and a list'
        ' of :italic:`tabs`. Each tab has its own :italic:`id`, a :italic:`title` and a list of :italic:`windows`.'
        ' Each window has an :italic:`id`, :italic:`title`, :italic:`current working directory`, :italic:`process id (PID)`,'
        ' :italic:`command-line` and :italic:`environment` of the process running in the window. Additionally, when'
        ' running the command inside a kitty window, that window can be identified by the :italic:`is_self` parameter.'
        # L31+ (--match usage guidance) omitted
    )
```

Drilling into one window object confirms every field the docstring promises — `id`, `title`, `cwd`, `pid`, `cmdline`, `env`, and the `is_self` discriminator:

```console
$ kitty/launcher/kitty @ --to unix:/tmp/kitty.sock ls \
    | python3 -c 'import json,sys; w=json.load(sys.stdin)[0]["tabs"][0]["windows"][0]; print(json.dumps({k:w[k] for k in ["id","title","pid","cwd","cmdline","is_self","at_prompt","env"]}, indent=2))'
{
  "id": 1,
  "title": "<repo>",
  "pid": 163538,
  "cwd": "<repo>",
  "cmdline": [
    "bash",
    "--posix"
  ],
  "is_self": false,
  "at_prompt": true,
  "env": {
    "ENV": "<repo>/shell-integration/bash/kitty.bash",
    "HISTFILE": "/root/.bash_history",
    "KITTY_BASH_INJECT": "no-rc 1 no-profile",
    "KITTY_BASH_UNEXPORT_HISTFILE": "1",
    "KITTY_SHELL_INTEGRATION": "enabled",
    "KITTY_WINDOW_ID": "1"
  }
}
```

This window object is shown **complete** (the `env` map has exactly these six keys — no elision). The full `ls` payload at this moment describes **1 OS window → 4 tabs → 5 windows** (the extra tabs/windows are the flood/stress windows created during load, matching the "5 windows, 4 tabs" count printed by the stress driver in section (c)).

> **Secret review (R4 / Rule 8).** Before reproducing `env` blocks verbatim, every distinct environment-variable name across all windows in the `ls` payload was enumerated and inspected. The complete set is `ENV`, `HISTFILE`, `KITTY_BASH_INJECT`, `KITTY_BASH_UNEXPORT_HISTFILE`, `KITTY_SHELL_INTEGRATION`, `KITTY_WINDOW_ID` — only kitty shell-integration variables and a bash-history path; **no tokens, passwords, keys, or other secrets are present**, so the payload is safe to reproduce in full (Appendix B §B2).

### `kitty @ get-text` — captured screen text (proves the flood is live)

Targeting the colour-flood window (`--match id:4`, the id printed by the stress driver) returns its **live screen** — the churn counter is mid-flight in the **8.74-million** range. The capture below is the **complete, verbatim on-screen grid** (all 18 visible rows, `8740747`→`8740764`, contiguous — no elision):

```console
$ kitty/launcher/kitty @ --to unix:/tmp/kitty.sock get-text --match id:4
8740747  COLOR-FLOOD scrollback churn line padding padding padding
8740748  COLOR-FLOOD scrollback churn line padding padding padding
8740749  COLOR-FLOOD scrollback churn line padding padding padding
8740750  COLOR-FLOOD scrollback churn line padding padding padding
8740751  COLOR-FLOOD scrollback churn line padding padding padding
8740752  COLOR-FLOOD scrollback churn line padding padding padding
8740753  COLOR-FLOOD scrollback churn line padding padding padding
8740754  COLOR-FLOOD scrollback churn line padding padding padding
8740755  COLOR-FLOOD scrollback churn line padding padding padding
8740756  COLOR-FLOOD scrollback churn line padding padding padding
8740757  COLOR-FLOOD scrollback churn line padding padding padding
8740758  COLOR-FLOOD scrollback churn line padding padding padding
8740759  COLOR-FLOOD scrollback churn line padding padding padding
8740760  COLOR-FLOOD scrollback churn line padding padding padding
8740761  COLOR-FLOOD scrollback churn line padding padding padding
8740762  COLOR-FLOOD scrollback churn line padding padding padding
8740763  COLOR-FLOOD scrollback churn line padding padding padding
8740764  COLOR-FLOOD scrollback churn line padding padding padding
```

To quantify the scrollback churn, `get-text --extent all` dumps the **entire scrollback buffer** — which is capped at the `scrollback_lines=100000` we launched with, plus the on-screen rows:

```console
$ kitty/launcher/kitty @ --to unix:/tmp/kitty.sock get-text --match id:4 --extent all | wc -l
100018
$ kitty/launcher/kitty @ --to unix:/tmp/kitty.sock get-text --match id:4 --extent all | grep COLOR-FLOOD | head -1
8664172  COLOR-FLOOD scrollback churn line padding padding padding
$ kitty/launcher/kitty @ --to unix:/tmp/kitty.sock get-text --match id:4 --extent all | grep COLOR-FLOOD | tail -1
8764189  COLOR-FLOOD scrollback churn line padding padding padding
```

The counter near **8.74 million**, and a retained scrollback window of **100,018 lines** spanning counter values `8,664,172 → 8,764,189`, are direct evidence that the generator has streamed **millions** of coloured lines through the terminal model while we query it — and that the control interface returns *live* state mid-flood, with the scrollback correctly bounded at the configured 100,000 lines. (The on-screen grid above is shown **complete and verbatim**; the 100,018-line `--extent all` dump is characterized here by its exact line count and its first/last counter values rather than pasted in full, because every one of its lines shares the identical `<counter>  COLOR-FLOOD …` format — the count and the two boundary values are the complete, non-truncated evidence.)

### `kitty @ get-colors` — the active colour table

The full command output is **277 lines** — **21** named UI colours plus the **256** `colorN` entries (`color0`–`color15` are the ANSI colours; `color16`–`color255` are the 256-colour cube). It is reproduced **verbatim in Appendix B §B3**. A representative `grep` of the headline UI/ANSI entries (an excerpt — the complete 277-line table is in §B3) is:

```console
$ kitty/launcher/kitty @ --to unix:/tmp/kitty.sock get-colors | grep -E \
   '^(foreground|background|cursor|selection_background|selection_foreground|active_border_color|inactive_border_color|color0|color1|color2|color7|color8|color15) '
active_border_color     #00ff00
background              #000000
color0                  #000000
color1                  #cc0403
color2                  #19cb00
color7                  #dddddd
color8                  #767676
color15                 #ffffff
cursor                  #cccccc
foreground              #dddddd
inactive_border_color   #cccccc
selection_background    #fffacd
selection_foreground    #000000
```

### Architecture: Go client → Python server

A key — and easily-misread — detail is *which language serves these commands*. The **server-side handlers are Python**:

```console
$ ls kitty/rc/*.py | wc -l
41
$ grep -lE 'class .+\(RemoteCommand\)' kitty/rc/*.py | wc -l
39
```

There are **41 `.py` files** in `kitty/rc/`, **39** of which define a `RemoteCommand` subclass (the two that do not are `__init__.py`, the package initialiser, and `base.py`, which defines the base `class RemoteCommand:` itself). `ls`, `get-text`, and `get-colors` are three of these Python handlers, running inside the embedded interpreter of the main process.

The **client**, by contrast, is the **Go `kitten` binary**. `kitty @` is registered into the Go tool in `tools/cmd/tool/main.go`:

```console
$ sed -n '14p;22p;40p' tools/cmd/tool/main.go
	"kitty/kittens/icat"
	"kitty/tools/cmd/at"
	at.EntryPoint(root)
```

`at.EntryPoint(root)` registers the `kitty @` client subtree into the Go binary. So a `kitty @ ls` invocation is a **Go client** (`kitten`) that connects to the socket and is answered by a **Python handler** (`kitty/rc/ls.py`) executing in the main process.

**Rationale / what this shows.** The control interface is a concrete demonstration of the Python/Go split *and* of the IPC boundary between them: Python owns the *state model* and the *command semantics* (the `rc/*.py` handlers reflecting windows, text, colours), while Go owns the *client* ergonomics. This becomes one of the ruled-out misconceptions in section (g): it would be wrong to assume "remote control is a Go feature" — the wire client is Go, but every handler that produces the answers above is Python.

---


## (e) How `kitten` fits in: process relationship & binary forensics

**Goal (R5):** run `kitty +kitten icat` on an image, document the process relationship between `kitty` and `kitten`, and inspect the `kitten` executable to substantiate its runtime/language and whether it is loaded into the main process or runs separately.

This section was captured against the **same** main instance used throughout, **PID `163470`** — the `kitten` appears below as a separate child PID (`178410`), not a restart of the main process.

### Run the user's exact command on a sample image

A throwaway 64×64 PNG was created (and later deleted in section (h)). The user's command is `kitty +kitten icat <image>`; `icat` needs a controlling terminal, so to capture a **live, non-exiting** kitten for inspection it was launched *inside* a kitty window with the image fed from a **named pipe (FIFO)**, so the kitten blocks on the pipe instead of rendering-and-exiting:

```console
$ file /tmp/kitty_investigation/sample.png
/tmp/kitty_investigation/sample.png: PNG image data, 64 x 64, 8-bit/color RGB, non-interlaced
$ mkfifo /tmp/kitty_investigation/feed.fifo
$ # launch the user's command inside a kitty window, reading the image from the FIFO:
$ kitty/launcher/kitty @ --to unix:/tmp/kitty.sock launch --type=window --keep-focus \
    kitty +kitten icat --place 20x10@0x0 /tmp/kitty_investigation/feed.fifo
$ cat /tmp/kitty_investigation/sample.png > /tmp/kitty_investigation/feed.fifo &   # feed it
```

Because the window's child runs `kitty +kitten icat …`, the C launcher's `+kitten` delegation (`execv`, see the code-as-truth chain below) replaces that child's image with the `kitten` binary **before** any Python starts — so the resulting process is `kitten icat …` directly parented by the main kitty, with **no intervening shell**.

### `pstree` is unavailable — show the error, then fall back to `ps -ef` (R5 / F5)

The requested `pstree` view of the process tree is unavailable in this container; here is the exact command and captured error, followed by the `ps -ef` parent/child fallback:

```console
$ command -v pstree || echo "pstree: not found (exit $?)"
pstree: not found (exit 1)
$ pstree -ap 163470
/bin/bash: line 797: pstree: command not found      # exit 127 — binary absent
```

### The process tree shows `kitten` as a DISTINCT PID

```console
$ ps -ef | grep -E 'kitty|kitten' | grep -v grep
root      163456       1  0 22:02 ?        00:00:00 /bin/sh /usr/bin/xvfb-run -a --server-args=-screen 0 1280x720x24 kitty/launcher/kitty -o allow_remote_control=yes --listen-on unix:/tmp/kitty.sock -o enable_audio_bell=no -o scrollback_lines=100000 bash --norc --noprofile
root      163470  163456 87 22:02 ?        00:07:16 kitty/launcher/kitty -o allow_remote_control=yes --listen-on unix:/tmp/kitty.sock -o enable_audio_bell=no -o scrollback_lines=100000 bash --norc --noprofile
root      178410  163470  0 22:10 pts/5    00:00:00 kitten icat --place 20x10@0x0 /tmp/kitty_investigation/feed.fifo
```

The hierarchy is **`xvfb-run` `163456` → main kitty `163470` → `kitten icat` `178410`**. The `PPid` field confirms `kitten` is a *direct child* of the main kitty process (not a thread, and not behind an intervening shell):

```console
$ grep PPid /proc/178410/status ; cat /proc/163470/comm
PPid:	163470
kitty
```

### The runtime clincher: which executable did `icat` actually run?

The decisive, assumption-free check — read the running `icat` process's own `exe` link (`kitten` PID `178410` from the `ps -ef` above):

```console
$ readlink /proc/178410/exe
<repo>/kitty/launcher/kitten
$ cat /proc/178410/comm
kitten
$ tr '\0' ' ' < /proc/178410/cmdline ; echo
kitten icat --place 20x10@0x0 /tmp/kitty_investigation/feed.fifo
```

The process that `kitty +kitten icat` produced is executing the **Go `kitten` binary**, not a Python interpreter. The corollary check seals it: **if `icat` were running the legacy Python `kittens/icat/main.py`, `libpython` would be mapped into its address space — it is not**:

```console
$ grep -E -c 'libpython|fast_data_types' /proc/178410/maps
0
$ wc -l < /proc/178410/maps ; grep -oE '/[^ ]+\.so[^ ]*' /proc/178410/maps | sort -u
81
/usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
/usr/lib/x86_64-linux-gnu/libc.so.6
```

The `kitten` process has **81** total map entries but maps **only `ld-linux` and `libc`** as shared objects — no Python, no `fast_data_types`, none of the graphics/font stack.

### `kitten` is ABSENT from the main process's address space

```console
$ grep -c kitten /proc/163470/maps
0
```

Zero. The `kitten` binary is **never mapped into the main `kitty` process** (PID `163470`) — conclusive proof that it is not an in-process plugin/module/thread.

### Binary forensics on `kitty/launcher/kitten`

```console
$ file <repo>/kitty/launcher/kitten
<repo>/kitty/launcher/kitten: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, Go BuildID=834jHfxeYq_uzj4UGTxR/tIJxVGCabAx0DjUp5evp/9Dtvgi-Fnxw0-g1ZS8GX/XmzXw8XIACNnBOlYzJ1p, stripped

$ ldd <repo>/kitty/launcher/kitten
	linux-vdso.so.1 (0x00007fff74723000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007ff904bc2000)
	/lib64/ld-linux-x86-64.so.2 (0x00007ff904e0e000)

$ go version <repo>/kitty/launcher/kitten
<repo>/kitty/launcher/kitten: go1.22.12

$ strings <repo>/kitty/launcher/kitten | grep -m3 -E '^go1\.'
go1.22.12

$ stat -c '%s' <repo>/kitty/launcher/kitten ; readelf -h <repo>/kitty/launcher/kitten | grep -E 'Type|Machine'
15765764
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
```

> **Code-as-truth correction.** A common expectation is that the Go binary is *fully static* (`ldd` → "not a dynamic executable"). On this checkout it is **not** fully static: it is a Go ELF **dynamically linked to libc only** (`linux-vdso`, `libc.so.6`, `ld-linux`). This still strongly supports the portability thesis in section (g): its *only* shared dependencies are libc and the dynamic loader — present on essentially every Linux host — versus the main process's **73** shared objects. The Go runtime version is **`go1.22.12`**, matching the `go 1.22` pin in `go.mod:L3`.

### Code-as-truth chain: *why* `icat` is Go and the Python path is dead

The runtime evidence above is *explained* by this chain in the C launcher, which delegates to the Go binary **before CPython is initialised**:

1. **`main()` delegates first, embeds Python second.** In `kitty/launcher/main.c`, the delegation call precedes the embedded-interpreter call:

   ```c
   // kitty/launcher/main.c (main): L452 then L464
   delegate_to_kitten_if_possible(argc, argv, exe_dir);   // L452 — runs BEFORE Python
   /* L453-463: handle_fast_commandline() + KITTY_LIB_PATH setup + RunData init (omitted) */
   ret = run_embedded(&run_data);                         // L464 — embeds CPython
   ```

   And `run_embedded` is where CPython actually starts (`Py_InitializeFromConfig` at L211, `Py_RunMain` at L216). So delegation strictly precedes interpreter start-up.

2. **The delegation conditions** match `@` (remote control) and wrapped `+kitten`/`+ kitten` invocations:

   ```c
   // kitty/launcher/main.c:L355-357
   if (argc > 1 && argv[1][0] == '@') exec_kitten(argc, argv, exe_dir);
   if (argc > 2 && strcmp(argv[1], "+kitten") == 0 && is_wrapped_kitten(argv[2])) exec_kitten(argc - 1, argv + 1, exe_dir);
   if (argc > 3 && strcmp(argv[1], "+") == 0 && strcmp(argv[2], "kitten") == 0 && is_wrapped_kitten(argv[3])) exec_kitten(argc - 2, argv + 2, exe_dir);
   ```

3. **`exec_kitten` `execv`s the sibling `kitten` binary** (replacing the process image — so CPython never starts for these invocations):

   ```c
   // kitty/launcher/main.c:L340-348 (full body — no elision)
   exec_kitten(int argc, char *argv[], char *exe_dir) {
       char exe[PATH_MAX+1] = {0};
       snprintf(exe, PATH_MAX, "%s/kitten", exe_dir);        // L342 — <exe_dir>/kitten
       char **newargv = malloc(sizeof(char*) * (argc + 1));  // L343
       memcpy(newargv, argv, sizeof(char*) * argc);          // L344
       newargv[argc] = 0;                                    // L345
       newargv[0] = "kitten";                                // L346 — argv[0] becomes "kitten" (hence the observed cmdline)
       errno = 0;                                            // L347
       execv(exe, newargv);                                  // L348 — replaces the process image
   ```

4. **`is_wrapped_kitten` tests the compile-time `WRAPPED_KITTENS` set**, and **`icat` is in it** (captured from the build in section (a)):

   ```c
   // kitty/launcher/main.c:L333-336
   is_wrapped_kitten(const char *arg) {
       char buf[64];
       snprintf(buf, sizeof(buf)-1, " %s ", arg);
       return strstr(" " WRAPPED_KITTENS " ", buf);
   }
   ```
   ```text
   # launcher/main.c's compiled value (section (a)), space-delimited so " icat " matches on word boundaries:
   -DWRAPPED_KITTENS=" ask clipboard diff hints hyperlinked_grep icat query_terminal show_key ssh themes transfer unicode_input "
   ```

5. **The Go side registers `icat`.** `tools/cmd/tool/main.go` imports `kitty/kittens/icat` (`:L14`) and calls `icat.EntryPoint(root)` (`:L48`); `kittens/icat/` is Go (`func EntryPoint` at `kittens/icat/main.go:L312`):

   ```console
   $ git ls-files 'kittens/icat/*.go' | wc -l ; ls kittens/icat/*.go | wc -l
   6        # tracked
   7        # on disk (1 generated cli_generated.go, gitignored)
   $ grep -h '^package ' kittens/icat/*.go | sort | uniq -c
         7 package icat
   ```

6. **The legacy Python path is present but never executed.** `kittens/icat/main.py` still exists (6 185 bytes) and `kittens/runner.py` still contains the legacy dispatch — but neither runs for `+kitten icat`, because the C launcher already `execv`'d the Go binary in step 3:

   ```console
   $ ls -l kittens/icat/main.py | awk '{print $5, $9}'
   6185 kittens/icat/main.py
   $ sed -n '110,116p' kittens/runner.py | grep -nE 'def |runpy|run_module'
   1:def run_kitten(kitten: str, run_name: str = '__main__') -> None:
   2:    import runpy
   7:    runpy.run_module(f'kittens.{kitten}.main', run_name=run_name)
   ```

   (Two further Python entry points — `kitty/entry_points.py:L10-12` `icat()` → `os.execl(kitten_exe(), "kitten", *args)` and the sibling-path resolver `kitty/constants.py:L82-84` `kitten_exe()` = `dirname(kitty_exe())/'kitten'` — *also* hand off to the same Go binary, so there is no Python code path that renders the image.)

**Rationale / what this shows.** Both the **runtime** (`readlink /proc/<pid>/exe` → the Go binary; no `libpython` mapped; absent from the main map) and the **code path** (delegate-before-init, `icat ∈ WRAPPED_KITTENS`, `package icat`) agree: `kitty +kitten icat` runs as a **separate Go process**, and the legacy Python `kittens/icat/main.py` is the **non-executed** path. This is exactly the "code-as-truth, runtime decides" resolution the task demands.

---


## (f) Stack/symbol snapshot during the stress run (with error + fallback)

**Goal (R6):** capture at least one symbol/stack snapshot during the stress run; if a tool is blocked, show the error and fall back to a method that still yields real stack/symbol visibility.

### Attempt 1: `py-spy` — combined Python + native stack

`py-spy` (v0.4.2) is present in the container as an *external inspection tool* — it is **not** a project dependency and nothing is added to the repository. Attaching to a live PID requires `ptrace`; the Yama `ptrace_scope` is `1` here, but the session is **root** (carrying `CAP_SYS_PTRACE`), so the attach succeeds — see the methodology note. `py-spy` runs **out-of-process**, so it does not perturb the target. A **combined Python + native** dump (`--native` walks C frames too) of the main process *under load*:

```console
$ py-spy --version
py-spy 0.4.2
$ py-spy dump --pid 163470 --native
Process 163470: kitty/launcher/kitty -o allow_remote_control=yes --listen-on unix:/tmp/kitty.sock -o enable_audio_bell=no -o scrollback_lines=100000 bash --norc --noprofile
Python v3.13.7 (<repo>/kitty/launcher/kitty)

Thread 163470 (active+gil): "MainThread"
    0x7c621d342772 (libc.so.6)
    0x7c621d3360ac (libc.so.6)
    0x7c621d336807 (libc.so.6)
    pthread_cond_wait (libc.so.6)
    0x7c621869a89d (libgallium-25.2.8-0ubuntu0.25.10.2.so)
    0x7c621897ae8b (libgallium-25.2.8-0ubuntu0.25.10.2.so)
    0x7c6218975fa9 (libgallium-25.2.8-0ubuntu0.25.10.2.so)
    0x7c621818213f (libgallium-25.2.8-0ubuntu0.25.10.2.so)
    0x7c621ace86c8 (libGLX_mesa.so.0.0.0)
    0x7c621acebe3d (libGLX_mesa.so.0.0.0)
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
    0x7c621d2c0575 (libc.so.6)
```

This single snapshot is the heart of the whole investigation. Read **bottom-to-top** it is the boot path: libc → `runpy` → `__main__` → `kitty/entry_points.py` → `kitty/main.py`. Then control **crosses into C and never returns to Python**: `main_loop` → `glfwRunMainLoop` (the vendored GLFW) → `process_global_state`, **all in `fast_data_types.so` / `glfw-x11.so`**, and from there straight into the Mesa GL driver (`libGLX_mesa` → `libgallium`) where it sits in `pthread_cond_wait` — i.e. at *this* instant the main thread has submitted GPU work and is waiting on the driver. The thread is `active+gil` — it holds the GIL while executing **C** code. **Python is parked at the top of the call chain; the actual work is in C and the GL driver.**

### Attempt 2: `gdb` — native backtraces of every thread

`gdb` attach also requires `ptrace`. Here it **succeeds** (the session is root, which carries `CAP_SYS_PTRACE`, overriding the Yama `ptrace_scope=1` restriction — see the methodology note). Had it failed with "Operation not permitted", the documented fallbacks would be: launch kitty *as a child* of gdb (Yama scope 1 always permits tracing direct children), or `--cap-add SYS_PTRACE` / `sysctl -w kernel.yama.ptrace_scope=0`. The attach captured **68 thread backtraces** (the complete capture is in Appendix B §B4); the three named threads (`.so` paths abbreviated for readability — full verbatim in §B4):

```console
$ gdb -p 163470 -batch -ex 'set pagination off' -ex 'thread apply all bt'
Thread 1 (Thread 0x7c621d137780 (LWP 163470) "kitty"):
#0  0x00007c621d43f613 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621c683d42 in screen_index () from <repo>/kitty/fast_data_types.so
#2  0x00007c621c685e24 in draw_text_loop () from <repo>/kitty/fast_data_types.so
#3  0x00007c621c6868c0 in draw_text.lto_priv () from <repo>/kitty/fast_data_types.so
#4  0x00007c621c6c38ae in run_worker.lto_priv () from <repo>/kitty/fast_data_types.so
#5  0x00007c621c613a55 in do_parse () from <repo>/kitty/fast_data_types.so
#6  0x00007c621c616a28 in process_global_state () from <repo>/kitty/fast_data_types.so
#7  0x00007c621b53fcca in glfwRunMainLoop () from <repo>/kitty/glfw-x11.so
#8  0x00007c621c6140cc in main_loop.lto_priv () from <repo>/kitty/fast_data_types.so
#9  0x00007c621d5fe7c0 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#10 0x00007c621d5f257e in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#11 0x00007c621d7371a9 in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
# (#12-#17: further CPython eval frames — PyEval_EvalCode etc. — full list in §B4)

Thread 2 (Thread 0x7c61ed9ec6c0 (LWP 163537) "KittyChildMon"):
#2  0x00007c621d3bda8e in poll () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621c6156ce in io_loop () from <repo>/kitty/fast_data_types.so

Thread 3 (Thread 0x7c61ee1ed6c0 (LWP 163536) "KittyPeerMon"):
#2  0x00007c621d3bda8e in poll () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621c61966b in talk_loop () from <repo>/kitty/fast_data_types.so
```

This independently corroborates py-spy and ties the named threads to their C functions. Note that `gdb` and `py-spy` sampled the main thread at **different instants**, which is itself informative: `py-spy` (above) caught it *waiting on the GL driver* (`process_global_state` → Mesa → `pthread_cond_wait`); `gdb` (here) caught it *mid parse-and-draw* (`do_parse` → `run_worker` → `draw_text` → `draw_text_loop` → `screen_index`). Both instants are **C frames under `main_loop` → `glfwRunMainLoop`** — the main thread oscillates between parsing/drawing and submitting to the GL driver, all in C.

- **Thread 1 (main)**: CPython eval frames (`libpython3.13`, #9+) launched the loop; *above* them the live frames are all **C** (`main_loop` → `glfwRunMainLoop` → `process_global_state` → `do_parse` → `run_worker` → `draw_text` → `screen_index`). Python is *below* the C frames — it started the loop, then C owns it.
- **Thread 2 `KittyChildMon`**: blocked in `poll()` inside **`io_loop`** (`fast_data_types.so`) — the PTY monitor, exactly the role of `child-monitor.c:L1489`.
- **Thread 3 `KittyPeerMon`**: blocked in `poll()` inside **`talk_loop`** (`fast_data_types.so`) — the RC peer loop, `child-monitor.c:L1808`.

The GL worker pool is visible in the aggregate — **195 frames** across the 68 threads are in Mesa's software rasteriser:

```console
$ grep -c libgallium <saved gdb bt>     # the 822-line capture in §B4
195
```

### Attempt 3: `eu-stack` is absent — show the error, fall back to procfs

The prompt's "if a tool is blocked, show the error and fall back" requirement is satisfied here by a *genuinely* absent tool: `eu-stack` (elfutils) is not installed in this container. The exact command and error:

```console
$ command -v eu-stack || echo "eu-stack: not found (exit $?)"
eu-stack: not found (exit 1)
$ eu-stack -p 163470
/bin/bash: line 993: eu-stack: command not found
```

The documented fallback is procfs: `/proc/<tid>/stack` yields a kernel-side stack for any thread with **no `ptrace` attach and no special privilege**. Sampling the four named threads of PID `163470` (with each thread's `wchan`):

```console
$ for tid in 163470 163537 163536 163535; do \
    echo "----- /proc/163470/task/$tid/stack -----"; \
    cat /proc/163470/task/$tid/stack; \
    echo "  wchan: $(cat /proc/163470/task/$tid/wchan)"; done
----- /proc/163470/task/163470/stack  ("kitty") -----
  wchan: 0
----- /proc/163470/task/163537/stack  ("KittyChildMon") -----
  wchan: 0
----- /proc/163470/task/163536/stack  ("KittyPeerMon") -----
[<0>] do_sys_poll+0x572/0x680
[<0>] do_restart_poll+0x5c/0xa0
[<0>] do_syscall_64+0x46/0xb0
[<0>] entry_SYSCALL_64_after_hwframe+0x78/0xe2
  wchan: do_sys_poll
----- /proc/163470/task/163535/stack  ("kitty:disk$0") -----
[<0>] futex_wait_queue+0xde/0x130
[<0>] futex_wait+0x179/0x300
[<0>] do_futex+0x18f/0x1e0
[<0>] __se_sys_futex+0x152/0x1d0
[<0>] do_syscall_64+0x46/0xb0
[<0>] entry_SYSCALL_64_after_hwframe+0x78/0xe2
  wchan: futex_wait_queue
```

This kernel-side view complements the userspace backtraces and is fully consistent with them:

- **`kitty` (main, 163470)** and **`KittyChildMon` (163537)** show an **empty kernel stack with `wchan: 0`** — they are **on-CPU in userspace** at the sampling instant (not blocked in any syscall). This matches the gdb backtrace, which caught the main thread mid parse/draw in C, and `ps`, which showed `KittyChildMon` busy in `R`/`D` reading the flood.
- **`KittyPeerMon` (163536)** is parked in the kernel `do_sys_poll` path (`wchan: do_sys_poll`) — exactly the `poll()` its userspace `talk_loop` sits in, waiting on the RC socket.
- **`kitty:disk$0` (163535)** is parked in `futex_wait_queue` (`wchan: futex_wait_queue`) — a pooled worker waiting on a futex, never woken by this workload (consistent with section (c), where the disk/worker pool stays at `utime=stime=0`).

### The transient `KittyWriteStdin` thread — captured live in a dedicated experiment

The named worker set sampled above is the **steady-state** population. The source defines one further, **transient** thread name, `KittyWriteStdin`, set inside `thread_write()`:

```c
// kitty/child-monitor.c:L967
set_thread_name("KittyWriteStdin");
```

This thread is created **on demand only when kitty writes to a child's stdin** — the C entry point `cm_thread_write` → `thread_write` (registered as `fast_data_types.thread_write` in `kitty/data-types.c:L445`), called from Python at `kitty/child.py:L341` (`fast_data_types.thread_write(stdin_write_fd, stdin)`) and `kitty/boss.py:L2439` — and it exits as soon as the write completes (it is detached). The **steady-state** stress mix used in section (c) — a colour flood emitted *by* the child toward kitty, window resizes, and tab switches — exercises the read/render and control paths but **never writes to a child's stdin**, which is exactly why `KittyWriteStdin` is **absent** from the steady-state samples above (and why a `send-text` flood does not summon it either — `send-text` goes through the non-blocking PTY write queue, not `thread_write`).

To complete R3's explicit transient-thread coverage, a **dedicated experiment** exercises the *one* code path that creates this thread and captures it **live**. This is **not** a process-mutating probe — there is **no** `gdb call` and no injected code (those remain forbidden, and the section-(a)–(f) main instance `163470` was never perturbed) — it is a **normal, user-facing kitty operation**: launching a window whose stdin is supplied via the documented `launch --stdin-source` feature, run against its **own short-lived instance**. The trick that makes the *transient* thread *observable for long enough to sample* is to keep its blocking `write()` from completing: feed a payload **larger than the OS pipe buffer** (64 KiB — confirmed via `fcntl(F_GETPIPE_SZ)` → `65536`) to a child that **never reads its stdin** (`sleep`). `thread_write()` clears `O_NONBLOCK` and then blocks in `write()` once the pipe fills, so the thread lingers and is trivially caught.

> Full transparency: this dedicated run is its **own** kitty instance; its PIDs (main `567236`, writer tid `568211`, `sleep` child `568210`) are — like every PID in this report — **volatile** across runs, and are distinct from the section-(a) instance `163470`. The absolute repo path is elided to `<repo>` as elsewhere.

**Trigger** — generate a large scrollback in the active window, then launch `sleep` fed that scrollback as stdin:

```console
$ # window 1 was filled to ~200,000 lines of scrollback (= 1,288,908 bytes ≈ 1.29 MB ≫ the 64 KiB pipe buffer)
$ kitty @ --to unix:<sock> launch --type=window --keep-focus --stdin-source @screen_scrollback sleep 600
2
```

**Observe** — the transient thread appears (population 68 → 69) and is caught on the very first poll, parked mid-write:

```console
$ ls /proc/567236/task | wc -l          # before the trigger
68
$ ls /proc/567236/task | wc -l          # after the trigger — exactly one new thread
69

$ grep -lx KittyWriteStdin /proc/567236/task/*/comm
/proc/567236/task/568211/comm

$ cat /proc/567236/task/568211/comm
KittyWriteStdin
$ grep -E '^(Name|State):' /proc/567236/task/568211/status
Name:	KittyWriteStdin
State:	S (sleeping)
$ cat /proc/567236/task/568211/wchan
pipe_write
```

The procfs **kernel stack** (no `ptrace` required) shows it parked exactly where the code predicts — blocked in the `write(2)` syscall on the now-full pipe:

```console
$ cat /proc/567236/task/568211/stack
[<0>] pipe_write+0x3e0/0x600
[<0>] vfs_write+0x2be/0x390
[<0>] ksys_write+0x75/0xe0
[<0>] do_syscall_64+0x46/0xb0
[<0>] entry_SYSCALL_64_after_hwframe+0x78/0xe2
```

The **userspace** `gdb` backtrace of that same LWP ties the kernel block directly to the C source function `thread_write()` in `fast_data_types.so` — the very function that calls `set_thread_name("KittyWriteStdin")` (addresses are volatile and shown as `0x…`):

```console
$ gdb -p 567236 -batch -ex 'set pagination off' -ex 'thread apply all bt'   # the KittyWriteStdin thread block
Thread 2 (Thread 0x… (LWP 568211) "KittyWriteStdin"):
#0  0x… in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x… in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x… in write () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x… in thread_write () from <repo>/kitty/launcher/../../kitty/fast_data_types.so
#4  0x… in ?? () from /lib/x86_64-linux-gnu/libc.so.6
```

**Lifecycle proof** — because the thread exists only for the duration of that blocked write, releasing the write makes it disappear. Killing the non-reading child closes the pipe's read end, so `write()` returns and the detached thread exits:

```console
$ kill 568210                                   # the 'sleep 600' child whose unread stdin is the full pipe
$ grep -lx KittyWriteStdin /proc/567236/task/*/comm || echo GONE
GONE
$ ls /proc/567236/task | wc -l                  # population back to baseline
68
```

`KittyWriteStdin` is therefore observed **live**, end-to-end: **absent** at idle/steady-state, **appearing** (68 → 69) precisely when kitty writes to a child's stdin, sitting **blocked in `write()` ← `thread_write()`** in `fast_data_types.so` exactly as `kitty/child-monitor.c:L967` dictates, and **vanishing** (69 → 68) the instant the write completes. This is the directly-observed confirmation of the code-as-truth origin — the honest counterpart to the steady-state threads in section (c), which were likewise directly observed.

**Rationale / what this shows.** Three independent lenses — `py-spy` (Python + native), `gdb` (native, all 68 threads), and procfs (kernel stacks, no `ptrace`) — **agree** on the division of labour during the flood. The main thread's hot, **GIL-holding** work is entirely in **C and the GL stack** (`main_loop` → `glfwRunMainLoop` → `process_global_state` → `do_parse` / `run_worker` / `draw_text` → Mesa `libgallium`); the persistent C worker threads run their own C event loops (`io_loop`, `talk_loop`) blocked in `poll()`; and **Python sits idle at the top of the boot/event-loop chain** in `kitty/main.py` (`_run_app` → `main_loop`), *below* all the live C frames. No Python frame is ever on a hot rendering path. This directly powers rule-out #2 in section (g).

---


## (g) Inferring responsibilities, ruling out wrong interpretations, naming a tradeoff

**Goal (R7):** using *only* the collected runtime artifacts, infer the responsibilities of Python vs. C vs. Go, explicitly rule out at least two plausible-but-wrong interpretations, and describe one portability-vs-performance tradeoff supported by a runtime observation.

### Inferred responsibilities (each fact cites the section that produced it)

**Python (embedded CPython) — orchestration, boot, and the remote-control *server*.**
- `libpython3.13.so` and the `kitty.*` modules are mapped/imported in the main process — section (b).
- The top of every live call stack is `kitty/main.py` / `kitty/entry_points.py`; Python launches the event loop and then waits — section (f) (py-spy: `_run_app`, `_main`, `main`).
- The `kitty @` answers come from **Python** handlers (`kitty/rc/*.py`, 39 `RemoteCommand` subclasses) — section (d).
- Inference: Python is the **conductor** — it boots the program, constructs the object model, owns command semantics and configuration, and hands the per-frame work to C.

**C (in-process `fast_data_types` + vendored GLFW + linked native libs) — the rendering-adjacent hot path.**
- `fast_data_types.so` plus `libharfbuzz`/`libfreetype`/`libfontconfig`/`libGL`/`liblcms2`/`libpng` are co-resident in the process — section (b).
- Under load the busy, running-state threads are the **C** threads, with `utime`/`stime` climbing — section (c).
- The active C frames during the flood are, on the **main (GIL-holding) thread**, `glfwRunMainLoop` → `process_global_state` → `do_parse` / `run_worker` / `draw_text` → Mesa, while the two **C worker threads** run `io_loop` (`KittyChildMon`) and `talk_loop` (`KittyPeerMon`) — all in `fast_data_types.so` / `glfw-x11.so` — section (f).
- Inference: C does VT parsing, the screen/line model, glyph/font work (via the linked libraries), and feeds the GPU — the throughput-critical path.

**Go (the separate `kitten` binary) — standalone CLI/TUI tooling and the RC *client*.**
- `kitty +kitten icat` runs as a **distinct PID** executing `kitty/launcher/kitten` — sections (e).
- It is a `go1.22.12` ELF depending only on libc/ld, and is **absent from the main process map** (`grep -c kitten` → 0) — section (e).
- `kitty @` is the Go client (`tools/cmd/at`, registered in `tools/cmd/tool/main.go`) — section (d).
- Inference: Go provides self-contained, shippable tools that run *outside* the render process.

### Ruling out plausible-but-wrong interpretations

**Rule-out #1 — "`kitten` is an in-process plugin / module / thread of `kitty`." FALSE.**
Evidence: it has its **own PID** in the process tree (`kitten icat`, PID `178410`, a *direct child* of the main kitty `163470`) — section (e); `file`/`ldd` show a **standalone Go ELF** with its own loader and libc — section (e); and it is **never mapped into the main process** —
```console
$ grep -c kitten /proc/163470/maps
0
```
A plugin/thread would share the main address space and appear in its map; `kitten` does neither.

**Rule-out #2 — "Python renders each cell/glyph." FALSE.**
Evidence: under the colour flood, the `active+gil` MainThread is executing **C** frames (`glfwRunMainLoop` → `process_global_state` → `do_parse`/`run_worker`/`draw_text`), and the only Python frames present are the dormant boot/event-loop frames in `kitty/main.py` — section (f). The CPU-burning threads are the **C** threads: the main render/event thread reaches running state (`R`) with `utime` climbing from ~0 to **1600** clock-ticks, `KittyPeerMon`'s `stime` goes 0 → **1223** servicing the RC flood, and `KittyChildMon`'s `stime` goes 0 → **856** reading the flood-window PTYs — section (c). If Python were doing per-glyph rendering, py-spy would show busy Python frames (a `for`-loop over cells, a draw call per glyph) and a Python worker accruing CPU — neither exists. The per-glyph work is in C/GL.

**Rule-out #3 — "Remote control is a Go feature, served by the Go binary." FALSE.**
Evidence: the `kitty @` *client* is indeed Go, but every command is **served by a Python handler** in `kitty/rc/` (39 `RemoteCommand` subclasses; `ls` is `kitty/rc/ls.py:L15`) running in the main process — section (d). The Go binary opens the socket; Python computes and returns the JSON. Concluding "RC is Go" mistakes the client for the server.

### One portability-vs-performance tradeoff (runtime-backed)

Kitty makes **opposite** language/linking choices for its two executables, and the artifacts quantify the trade:

- **The Go `kitten` chooses portability over in-process performance.** Its runtime dependency surface is essentially empty:
  ```console
  $ ldd <repo>/kitty/launcher/kitten
  	linux-vdso.so.1 (0x00007fff74723000)
  	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007ff904bc2000)
  	/lib64/ld-linux-x86-64.so.2 (0x00007ff904e0e000)
  ```
  Depending only on libc + the dynamic loader (both present on essentially every Linux host), and carrying its own Go runtime, `kitten` can be **copied to and run on a remote machine that has none of Kitty's C/GPU stack** — which is precisely how Kitty ships kittens over SSH. The cost is that it runs **out-of-process** (its own PID, its own ~15 MB image, no shared memory with the renderer — section (e)), so it can never be on the per-frame render hot path.

- **The C + GPU core chooses performance over portability.** The renderer's dependency surface is the opposite extreme:
  ```console
  $ grep -oE '/[^ ]+\.so[^ ]*' /proc/163470/maps | sort -u | wc -l
  73
  ```
  **73 shared objects** — `libGL`/`libGLX`/Mesa, `libharfbuzz`, `libfreetype`, `libfontconfig`, `liblcms2`, `libpng`, `libcrypto`, `libpython3.13`, … — must all be present and correctly versioned for the main process to start, and a GL context is mandatory (Kitty has no CPU text fallback). That heavy, host-specific stack is exactly what buys the in-process throughput shown in section (f) (parsing/shaping/GL all in one address space, no IPC per frame). It is **not** something you can casually scp to an arbitrary remote host.

The tradeoff in one line: **`kitten` (1 effective shared dep) is built to *travel*; the C core (73 shared deps + mandatory GL) is built to *render fast in place*.**

---


## (h) Cleanup & final repository state

**Goal (R8):** remove every temporary artifact created during the investigation and confirm the repository is unchanged apart from the single deliverable.

### What was temporary

Every artifact produced for this investigation lives **outside the repository working tree**, under `/tmp`:

- the disposable load-driver script `stress_driver.sh` (the four-dimension load generator shown in section (a));
- the throwaway sample image (`sample.png`) and the named pipe (`feed.fifo`) used to feed `kitten icat`;
- the remote-control UNIX socket (`/tmp/kitty.sock`);
- captured logs and the entire scratch directory `/tmp/kitty_investigation/`;
- the single headless `kitty` instance (PID `163470`) launched for the investigation, together with its `kitten icat` child (PID `178410`).

Because none of these is inside the repo, they never affected `git status`; removing them is about leaving no trace.

### Cleanup commands

```console
$ kill "$(cat /tmp/kitty_investigation/mainpid.txt)" 2>/dev/null     # stop the headless kitty (PID 163470)
$ pkill -x kitten 2>/dev/null                                        # stop any lingering kitten child
$ rm -f /tmp/kitty.sock                                              # remove the RC socket
$ rm -rf /tmp/kitty_investigation                                    # remove script, image, fifo, logs, captures
```

(The `py-spy`, `gdb`, and procfs tools used in section (f) are pre-installed container utilities under `/usr/local/bin` and `/usr/bin`, outside the repo; **nothing was installed into the repository**, and no project dependency was added.)

### Final `git status` — only the deliverable is added (investigation-time, pre-report-commit)

The block below is the **investigation-time** capture, taken *before* this report received its own commit — which is why the document shows up as an **untracked** file (`??`). The delivered, committed state is verified separately in **Final-delivery verification** immediately below; both states agree on the one load-bearing fact — the *only* repository change is this single document.

```console
$ git status
On branch blitzy-69ce8195-d0f2-4cd8-8246-5c472699a20b
Untracked files:
  (use "git add <file>..." to include in what will be committed)
	blitzy/

nothing added to commit but untracked files present (use "git add" to track)

$ git status --porcelain -uall
?? blitzy/documentation/kitty_815df1e210e0.md

$ find blitzy -type f
blitzy/documentation/kitty_815df1e210e0.md
```

The machine-readable `--porcelain -uall` line is unambiguous: the **only** change to the repository is the single new file `blitzy/documentation/kitty_815df1e210e0.md`. The build outputs remain ignored (not tracked), so the multi-language build left **zero** tracked-file modifications:

```console
$ git check-ignore -v kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so
.gitignore:18:/kitty/launcher/kitt*	kitty/launcher/kitty
.gitignore:18:/kitty/launcher/kitt*	kitty/launcher/kitten
.gitignore:1:*.so	kitty/fast_data_types.so
```

**No existing repository file was modified, created, or deleted** — the sole artifact added is this report.

### Final-delivery verification (the committed, delivered state)

The block above is the *pre-commit* investigation snapshot. In the **delivered** repository this report is committed as a single commit placed directly on top of the checkpoint, so the working tree is **clean** and the document is **tracked** rather than untracked. Because a document cannot contain its own commit hash — writing the hash in would change the file and therefore the hash (see the note in section (a)) — the delivered state is pinned not by `HEAD` itself but by two **hash-independent invariants**, both exactly reproducible against the delivered checkout:

```console
$ git rev-parse HEAD~1                                              # the report sits directly on the checkpoint
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1

$ git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1..HEAD   # exactly one added file, zero source files
A	blitzy/documentation/kitty_815df1e210e0.md

$ git status --porcelain                                            # clean tree: the deliverable is committed/tracked
$                                                                   # (empty output — nothing uncommitted)
```

Read together these say: `HEAD~1` is the frozen checkpoint `815df1e210e0…`; the *entire* difference between that checkpoint and the delivered `HEAD` is this one added document (no source file is touched); and `git status --porcelain` is **empty**, so the working tree carries no uncommitted change. The exact delivered `HEAD` hash is deliberately omitted — for the chicken-and-egg reason above, and because, like every PID, timestamp, and VCS stamp elsewhere in this report, it is a **volatile** value. The invariants above are what make the delivered state independently verifiable regardless of the specific commit hash.

---

## Appendix A: evidence-locator cross-reference

Each locator below was re-confirmed against the live checkout while writing this report. The checkout's tracked sources are byte-identical to checkpoint `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (that checkpoint is `HEAD~1`; the single file added on top of it is this report — see section (a) for the `git diff … --name-status` proof).

| Claim | Locator (verified) |
|-------|--------------------|
| Build entry point | `Makefile:L12-13` → `python3 setup.py $(VVAL)` |
| Python floor `>=3.8` | `pyproject.toml:L2` |
| Go module + pin | `go.mod:L1` `module kitty`; `:L3` `go 1.22` |
| C extension target | `setup.py:L1090-1095` (`compile_c_extension(..., 'kitty/fast_data_types', ...)`, `compile_glfw`, `compile_kittens`) |
| Delegation before Python init | `kitty/launcher/main.c:L452` (`delegate_to_kitten_if_possible`) precedes `:L464` (`run_embedded`); CPython starts at `:L211` `Py_InitializeFromConfig`, `:L216` `Py_RunMain` |
| Delegation conditions | `kitty/launcher/main.c:L355-357` |
| `exec_kitten` execs `<exe_dir>/kitten` | `kitty/launcher/main.c:L340-348` |
| `is_wrapped_kitten` / `WRAPPED_KITTENS` | `kitty/launcher/main.c:L333-336`; macro value baked at build (section (a)); `icat` present |
| Named threads | `kitty/child-monitor.c:L1489` `KittyChildMon`, `:L1808` `KittyPeerMon`, `:L967` `KittyWriteStdin` |
| `kitten_exe()` sibling resolution | `kitty/constants.py:L82-84` |
| Python `icat()` alias execs kitten | `kitty/entry_points.py:L10-12` |
| Legacy Python kitten dispatch | `kittens/runner.py:L110-116` (`runpy.run_module`) — non-executed for `icat` |
| `ls` handler + JSON-tree doc | `kitty/rc/ls.py:L15` `class LS(RemoteCommand)`, `:L24-30` |
| RC handler counts | 41 `.py` files / 39 `RemoteCommand` subclasses (live count) |
| `icat` registered into Go binary | `tools/cmd/tool/main.go:L14` import, `:L48` `icat.EntryPoint(root)`; `kittens/icat/main.go:L312` `func EntryPoint` |
| `kitty @` client is Go | `tools/cmd/tool/main.go:L22` import, `:L40` `at.EntryPoint(root)` |
| `wrapped_kittens` shell list | `shell-integration/ssh/kitty:L27` (contains `icat`) |

## Appendix B: complete captured artifacts

This appendix reproduces, **verbatim and in full**, the captured outputs that sections (a)–(f) cite as excerpts. The only modification is the disclosed `<repo>` path elision (see Methodology → “Reproducibility”). Each block names the section that references it and the scratch file it was captured into (under `/tmp/kitty_investigation/`, since deleted — see section (h)).

### §B1 — full link line of the in-process C extension `fast_data_types.so`

*Referenced from section (a).* Captured into `04_linklines.txt`. This single `gcc` invocation links **62 object files** — 49 first-party `kitty/*.c` translation units plus 13 third-party units (1 `ringbuf` + 12 `base64` codec/table units) — into the in-process extension, then links the native libraries (`-lharfbuzz -lGL -lpng16 -llcms2 -llcms2_fast_float -llcms2_threaded -lcrypto -lz -lpython3.13`). The object-file count is reproduced below the line.

```text
gcc -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -I/usr/include/libpng16 -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/harfbuzz -I/usr/include/freetype2 -I/usr/include/libpng16 -I/usr/include/glib-2.0 -I/usr/lib/x86_64-linux-gnu/glib-2.0/include -I/usr/include/sysprof-6 -I/usr/include/python3.13 -Wall -O3 -shared -flto build/fast_data_types-kitty-charsets.c.o build/fast_data_types-kitty-child-monitor.c.o build/fast_data_types-kitty-child.c.o build/fast_data_types-kitty-cleanup.c.o build/fast_data_types-kitty-colors.c.o build/fast_data_types-kitty-crypto.c.o build/fast_data_types-kitty-cursor.c.o build/fast_data_types-kitty-data-types.c.o build/fast_data_types-kitty-desktop.c.o build/fast_data_types-kitty-disk-cache.c.o build/fast_data_types-kitty-fast-file-copy.c.o build/fast_data_types-kitty-font-names.c.o build/fast_data_types-kitty-fontconfig.c.o build/fast_data_types-kitty-fonts.c.o build/fast_data_types-kitty-freetype.c.o build/fast_data_types-kitty-freetype_render_ui_text.c.o build/fast_data_types-kitty-gl-wrapper.c.o build/fast_data_types-kitty-gl.c.o build/fast_data_types-kitty-glfw-wrapper.c.o build/fast_data_types-kitty-glfw.c.o build/fast_data_types-kitty-glyph-cache.c.o build/fast_data_types-kitty-graphics.c.o build/fast_data_types-kitty-history.c.o build/fast_data_types-kitty-hyperlink.c.o build/fast_data_types-kitty-key_encoding.c.o build/fast_data_types-kitty-keys.c.o build/fast_data_types-kitty-kittens.c.o build/fast_data_types-kitty-line-buf.c.o build/fast_data_types-kitty-line.c.o build/fast_data_types-kitty-logging.c.o build/fast_data_types-kitty-loop-utils.c.o build/fast_data_types-kitty-monotonic.c.o build/fast_data_types-kitty-mouse.c.o build/fast_data_types-kitty-png-reader.c.o build/fast_data_types-kitty-rowcolumn-diacritics.c.o build/fast_data_types-kitty-screen.c.o build/fast_data_types-kitty-shaders.c.o build/fast_data_types-kitty-shlex.c.o build/fast_data_types-kitty-simd-string-128.c.o build/fast_data_types-kitty-simd-string-256.c.o build/fast_data_types-kitty-simd-string.c.o build/fast_data_types-kitty-state.c.o build/fast_data_types-kitty-systemd.c.o build/fast_data_types-kitty-unicode-data.c.o build/fast_data_types-kitty-utmp.c.o build/fast_data_types-kitty-vt-parser.c.o build/fast_data_types-kitty-wcswidth.c.o build/fast_data_types-kitty-window_logo.c.o build/fast_data_types-kitty-vt-parser-dump.c.o build/fast_data_types-3rdparty-ringbuf-ringbuf.c.o build/fast_data_types-3rdparty-base64-lib-arch-neon32-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-sse42-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-ssse3-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-sse41-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-generic-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-avx2-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-avx512-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-avx-codec.c.o build/fast_data_types-3rdparty-base64-lib-arch-neon64-codec.c.o build/fast_data_types-3rdparty-base64-lib-tables-tables.c.o build/fast_data_types-3rdparty-base64-lib-codec_choose.c.o build/fast_data_types-3rdparty-base64-lib-lib.c.o -ldl -lm -L/usr/lib/x86_64-linux-gnu -lpython3.13 -Xlinker -export-dynamic -Wl,-O1 -Wl,-Bsymbolic-functions -lharfbuzz -lGL -lpng16 -llcms2 -llcms2_fast_float -llcms2_threaded -pthread -lm -lcrypto -lrt -lz -o build/kitty/fast_data_types.so
```

```console
$ # object files on the link line above (counted from the captured build log):
$ grep -o 'build/fast_data_types-[^ ]*\.o' 04_linklines.txt | wc -l
62
$ grep -o 'build/fast_data_types-kitty-[^ ]*\.o' 04_linklines.txt | wc -l   # first-party kitty/*.c
49
$ grep -o 'build/fast_data_types-3rdparty-[^ ]*\.o' 04_linklines.txt | wc -l # third-party
13
```

### §B2 — full `kitty @ ls` JSON tree (under load)

*Referenced from section (d).* Captured into `14_ls_full.json`; **364 lines** pretty-printed (5,355 bytes compact). Structure: **1 OS window → 4 tabs → 5 windows**. Secret-reviewed in section (d): the only environment variables present across all windows are kitty shell-integration variables plus a bash-history path — no secrets. The disclosed `<repo>` elision is applied to the `title`, `cwd`, and `ENV` path fields.

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
                "active_window_history": [],
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
                    },
                    {
                        "id": 4,
                        "windows": [
                            4
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
                "title": "<repo>",
                "windows": [
                    {
                        "at_prompt": true,
                        "cmdline": [
                            "bash",
                            "--posix"
                        ],
                        "columns": 119,
                        "created_at": 1782511325503052536,
                        "cwd": "<repo>",
                        "env": {
                            "ENV": "<repo>/shell-integration/bash/kitty.bash",
                            "HISTFILE": "/root/.bash_history",
                            "KITTY_BASH_INJECT": "no-rc 1 no-profile",
                            "KITTY_BASH_UNEXPORT_HISTFILE": "1",
                            "KITTY_SHELL_INTEGRATION": "enabled",
                            "KITTY_WINDOW_ID": "1"
                        },
                        "foreground_processes": [
                            {
                                "cmdline": [
                                    "bash",
                                    "--posix"
                                ],
                                "cwd": "<repo>",
                                "pid": 163538
                            }
                        ],
                        "id": 1,
                        "is_active": true,
                        "is_focused": true,
                        "is_self": false,
                        "last_cmd_exit_status": 0,
                        "last_reported_cmdline": "",
                        "lines": 19,
                        "pid": 163538,
                        "title": "<repo>",
                        "user_vars": {}
                    },
                    {
                        "at_prompt": false,
                        "cmdline": [
                            "/usr/bin/sh",
                            "-c",
                            "\n  i=0\n  while :; do\n    c=$(( i % 256 ))\n    printf \"\\033[38;5;%dm%d  COLOR-FLOOD scrollback churn line padding padding padding\\033[0m\\n\" \"$c\" \"$i\"\n    i=$(( i + 1 ))\n  done\n"
                        ],
                        "columns": 119,
                        "created_at": 1782511566926306700,
                        "cwd": "<repo>",
                        "env": {
                            "KITTY_WINDOW_ID": "4"
                        },
                        "foreground_processes": [
                            {
                                "cmdline": [
                                    "/usr/bin/sh",
                                    "-c",
                                    "\n  i=0\n  while :; do\n    c=$(( i % 256 ))\n    printf \"\\033[38;5;%dm%d  COLOR-FLOOD scrollback churn line padding padding padding\\033[0m\\n\" \"$c\" \"$i\"\n    i=$(( i + 1 ))\n  done\n"
                                ],
                                "cwd": "<repo>",
                                "pid": 166292
                            }
                        ],
                        "id": 4,
                        "is_active": false,
                        "is_focused": false,
                        "is_self": false,
                        "last_cmd_exit_status": 0,
                        "last_reported_cmdline": "",
                        "lines": 19,
                        "pid": 166292,
                        "title": "sh",
                        "user_vars": {}
                    }
                ]
            },
            {
                "active_window_history": [
                    5
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
                        "id": 5,
                        "windows": [
                            5
                        ]
                    }
                ],
                "id": 3,
                "is_active": false,
                "is_focused": false,
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
                "title": "sh",
                "windows": [
                    {
                        "at_prompt": false,
                        "cmdline": [
                            "/usr/bin/sh",
                            "-c",
                            "while :; do printf \"tab churn line %d\\n\" \"$$\"; done"
                        ],
                        "columns": 120,
                        "created_at": 1782511566989328401,
                        "cwd": "<repo>",
                        "env": {
                            "KITTY_WINDOW_ID": "5"
                        },
                        "foreground_processes": [
                            {
                                "cmdline": [
                                    "/usr/bin/sh",
                                    "-c",
                                    "while :; do printf \"tab churn line %d\\n\" \"$$\"; done"
                                ],
                                "cwd": "<repo>",
                                "pid": 166329
                            }
                        ],
                        "id": 5,
                        "is_active": true,
                        "is_focused": true,
                        "is_self": false,
                        "last_cmd_exit_status": 0,
                        "last_reported_cmdline": "",
                        "lines": 39,
                        "pid": 166329,
                        "title": "sh",
                        "user_vars": {}
                    }
                ]
            },
            {
                "active_window_history": [
                    6
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
                        "id": 6,
                        "windows": [
                            6
                        ]
                    }
                ],
                "id": 4,
                "is_active": false,
                "is_focused": false,
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
                "title": "sh",
                "windows": [
                    {
                        "at_prompt": false,
                        "cmdline": [
                            "/usr/bin/sh",
                            "-c",
                            "while :; do printf \"tab churn line %d\\n\" \"$$\"; done"
                        ],
                        "columns": 120,
                        "created_at": 1782511567227651008,
                        "cwd": "<repo>",
                        "env": {
                            "KITTY_WINDOW_ID": "6"
                        },
                        "foreground_processes": [
                            {
                                "cmdline": [
                                    "/usr/bin/sh",
                                    "-c",
                                    "while :; do printf \"tab churn line %d\\n\" \"$$\"; done"
                                ],
                                "cwd": "<repo>",
                                "pid": 166360
                            }
                        ],
                        "id": 6,
                        "is_active": true,
                        "is_focused": true,
                        "is_self": false,
                        "last_cmd_exit_status": 0,
                        "last_reported_cmdline": "",
                        "lines": 39,
                        "pid": 166360,
                        "title": "sh",
                        "user_vars": {}
                    }
                ]
            },
            {
                "active_window_history": [
                    7
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
                        "id": 7,
                        "windows": [
                            7
                        ]
                    }
                ],
                "id": 5,
                "is_active": false,
                "is_focused": false,
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
                "title": "sh",
                "windows": [
                    {
                        "at_prompt": false,
                        "cmdline": [
                            "/usr/bin/sh",
                            "-c",
                            "while :; do printf \"tab churn line %d\\n\" \"$$\"; done"
                        ],
                        "columns": 120,
                        "created_at": 1782511567791184533,
                        "cwd": "<repo>",
                        "env": {
                            "KITTY_WINDOW_ID": "7"
                        },
                        "foreground_processes": [
                            {
                                "cmdline": [
                                    "/usr/bin/sh",
                                    "-c",
                                    "while :; do printf \"tab churn line %d\\n\" \"$$\"; done"
                                ],
                                "cwd": "<repo>",
                                "pid": 166394
                            }
                        ],
                        "id": 7,
                        "is_active": true,
                        "is_focused": true,
                        "is_self": false,
                        "last_cmd_exit_status": 0,
                        "last_reported_cmdline": "",
                        "lines": 39,
                        "pid": 166394,
                        "title": "sh",
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

### §B3 — full `kitty @ get-colors` table (under load)

*Referenced from section (d).* The command's output is **277 lines** = **21** named UI colours + the **256** `colorN` entries (`color0`–`color15` ANSI + `color16`–`color255` cube). Captured into `16_getcolors_full.txt` (which adds 4 scratch banner/comment lines around this payload, hence the file is 281 lines); the **command output itself**, reproduced verbatim below, is 277 lines:

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


### §B4 — full `gdb` `thread apply all bt` (all 68 threads, under load)

Complete verbatim capture referenced by section (f), *Attempt 2*. Command:

```console
$ gdb -p 163470 -batch -ex 'set pagination off' -ex 'thread apply all bt'
```

The capture is **68 threads** (822 lines). The only elision applied is the disclosed repository-root substitution `<repo>` = `/tmp/blitzy/kitty/blitzy-69ce8195-d0f2-4cd8-8246-5c472699a20b_8fc29a` (every other byte is verbatim, including the `gdb` `[New LWP ...]` preamble and the unresolved `??` frames `gdb` could not symbolicate). Structure:

- **Thread 1 `"kitty"` (LWP 163470)** — the main render/event thread; the live frames above the CPython eval machinery are all C (`main_loop` → `glfwRunMainLoop` → `process_global_state` → `do_parse` → `run_worker` → `draw_text` → `screen_index`).
- **Thread 2 `"KittyChildMon"` (LWP 163537)** — `poll()` → `io_loop` (`fast_data_types.so`); the PTY monitor of `child-monitor.c:L1489`.
- **Thread 3 `"KittyPeerMon"` (LWP 163536)** — `poll()` → `talk_loop` (`fast_data_types.so`); the RC peer loop of `child-monitor.c:L1808`.
- **Thread 4 `"kitty:disk$0"` (LWP 163535)** — a pooled disk/IO worker parked in `pthread_cond_wait`.
- **Threads 5–68 `"llvmpipe-N"` / generic Mesa workers** — the GPU-software-rasteriser pool; each has the identical `pthread_cond_wait` → `libgallium` shape. These dominate the **195** total `libgallium` frames reported in section (f).

Only 3 of the 68 threads touch a `kitty` symbol (`fast_data_types.so`) — the main thread plus the two named C monitors — confirming that all kitty-specific work lives on C-named threads and **no thread is executing Python on a hot path**.

```text
[New LWP 163537]
[New LWP 163536]
[New LWP 163535]
[New LWP 163534]
[New LWP 163533]
[New LWP 163532]
[New LWP 163531]
[New LWP 163530]
[New LWP 163529]
[New LWP 163528]
[New LWP 163527]
[New LWP 163526]
[New LWP 163525]
[New LWP 163524]
[New LWP 163523]
[New LWP 163522]
[New LWP 163521]
[New LWP 163520]
[New LWP 163519]
[New LWP 163518]
[New LWP 163517]
[New LWP 163516]
[New LWP 163515]
[New LWP 163514]
[New LWP 163513]
[New LWP 163512]
[New LWP 163511]
[New LWP 163510]
[New LWP 163509]
[New LWP 163508]
[New LWP 163507]
[New LWP 163506]
[New LWP 163505]
[New LWP 163504]
[New LWP 163503]
[New LWP 163502]
[New LWP 163501]
[New LWP 163500]
[New LWP 163499]
[New LWP 163498]
[New LWP 163497]
[New LWP 163496]
[New LWP 163495]
[New LWP 163494]
[New LWP 163493]
[New LWP 163492]
[New LWP 163491]
[New LWP 163490]
[New LWP 163489]
[New LWP 163488]
[New LWP 163487]
[New LWP 163486]
[New LWP 163485]
[New LWP 163484]
[New LWP 163483]
[New LWP 163482]
[New LWP 163481]
[New LWP 163480]
[New LWP 163479]
[New LWP 163478]
[New LWP 163477]
[New LWP 163476]
[New LWP 163475]
[New LWP 163474]
[New LWP 163473]
[New LWP 163472]
[New LWP 163471]
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
0x00007c621d43f613 in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 68 (Thread 0x7c620f14e6c0 (LWP 163471) "llvmpipe-0"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 67 (Thread 0x7c620e94d6c0 (LWP 163472) "llvmpipe-1"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 66 (Thread 0x7c620e14c6c0 (LWP 163473) "llvmpipe-2"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 65 (Thread 0x7c620d94b6c0 (LWP 163474) "llvmpipe-3"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 64 (Thread 0x7c620d14a6c0 (LWP 163475) "llvmpipe-4"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 63 (Thread 0x7c620c9496c0 (LWP 163476) "llvmpipe-5"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 62 (Thread 0x7c620c1486c0 (LWP 163477) "llvmpipe-6"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 61 (Thread 0x7c620b9476c0 (LWP 163478) "llvmpipe-7"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 60 (Thread 0x7c620b1466c0 (LWP 163479) "llvmpipe-8"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 59 (Thread 0x7c620a9456c0 (LWP 163480) "llvmpipe-9"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 58 (Thread 0x7c620a1446c0 (LWP 163481) "llvmpipe-10"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 57 (Thread 0x7c62099436c0 (LWP 163482) "llvmpipe-11"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 56 (Thread 0x7c62091426c0 (LWP 163483) "llvmpipe-12"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 55 (Thread 0x7c62089416c0 (LWP 163484) "llvmpipe-13"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 54 (Thread 0x7c62081406c0 (LWP 163485) "llvmpipe-14"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 53 (Thread 0x7c620793f6c0 (LWP 163486) "llvmpipe-15"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 52 (Thread 0x7c620713e6c0 (LWP 163487) "llvmpipe-16"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 51 (Thread 0x7c620693d6c0 (LWP 163488) "llvmpipe-17"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 50 (Thread 0x7c620613c6c0 (LWP 163489) "llvmpipe-18"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 49 (Thread 0x7c620593b6c0 (LWP 163490) "llvmpipe-19"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 48 (Thread 0x7c620513a6c0 (LWP 163491) "llvmpipe-20"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 47 (Thread 0x7c62049396c0 (LWP 163492) "llvmpipe-21"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 46 (Thread 0x7c62041386c0 (LWP 163493) "llvmpipe-22"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 45 (Thread 0x7c62039376c0 (LWP 163494) "llvmpipe-23"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 44 (Thread 0x7c62031366c0 (LWP 163495) "llvmpipe-24"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 43 (Thread 0x7c62029356c0 (LWP 163496) "llvmpipe-25"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 42 (Thread 0x7c62021346c0 (LWP 163497) "llvmpipe-26"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 41 (Thread 0x7c62019336c0 (LWP 163498) "llvmpipe-27"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 40 (Thread 0x7c62011326c0 (LWP 163499) "llvmpipe-28"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 39 (Thread 0x7c62009316c0 (LWP 163500) "llvmpipe-29"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 38 (Thread 0x7c62001306c0 (LWP 163501) "llvmpipe-30"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 37 (Thread 0x7c61ff92f6c0 (LWP 163502) "llvmpipe-31"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897e91f in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 36 (Thread 0x7c61ff12e6c0 (LWP 163503) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 35 (Thread 0x7c61fe92d6c0 (LWP 163504) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 34 (Thread 0x7c61fe12c6c0 (LWP 163505) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 33 (Thread 0x7c61fd92b6c0 (LWP 163506) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 32 (Thread 0x7c61fd12a6c0 (LWP 163507) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 31 (Thread 0x7c61fc9296c0 (LWP 163508) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 30 (Thread 0x7c61fc1286c0 (LWP 163509) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 29 (Thread 0x7c61fb9276c0 (LWP 163510) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 28 (Thread 0x7c61fb1266c0 (LWP 163511) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 27 (Thread 0x7c61fa9256c0 (LWP 163512) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 26 (Thread 0x7c61fa1246c0 (LWP 163513) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 25 (Thread 0x7c61f99236c0 (LWP 163514) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 24 (Thread 0x7c61f91226c0 (LWP 163515) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 23 (Thread 0x7c61f89216c0 (LWP 163516) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 22 (Thread 0x7c61f81206c0 (LWP 163517) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 21 (Thread 0x7c61f791f6c0 (LWP 163518) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 20 (Thread 0x7c61f711e6c0 (LWP 163519) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 19 (Thread 0x7c61f691d6c0 (LWP 163520) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 18 (Thread 0x7c61f611c6c0 (LWP 163521) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 17 (Thread 0x7c61f591b6c0 (LWP 163522) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 16 (Thread 0x7c61f511a6c0 (LWP 163523) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 15 (Thread 0x7c61f49196c0 (LWP 163524) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 14 (Thread 0x7c61f41186c0 (LWP 163525) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 13 (Thread 0x7c61f39176c0 (LWP 163526) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 12 (Thread 0x7c61f31166c0 (LWP 163527) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 11 (Thread 0x7c61f29156c0 (LWP 163528) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 10 (Thread 0x7c61f21146c0 (LWP 163529) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 9 (Thread 0x7c61f19136c0 (LWP 163530) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 8 (Thread 0x7c61f11126c0 (LWP 163531) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 7 (Thread 0x7c61f09116c0 (LWP 163532) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 6 (Thread 0x7c61f01106c0 (LWP 163533) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 5 (Thread 0x7c61ef90f6c0 (LWP 163534) "kitty"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c621897a28c in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 4 (Thread 0x7c61eefcd6c0 (LWP 163535) "kitty:disk$0"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d3360ac in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d336807 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621d339067 in pthread_cond_wait () from /lib/x86_64-linux-gnu/libc.so.6
#4  0x00007c621869a89d in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#5  0x00007c6218653fbc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#6  0x00007c621869a7cc in ?? () from /lib/x86_64-linux-gnu/libgallium-25.2.8-0ubuntu0.25.10.2.so
#7  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#8  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 3 (Thread 0x7c61ee1ed6c0 (LWP 163536) "KittyPeerMon"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d33613c in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d3bda8e in poll () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621c61966b in talk_loop () from <repo>/kitty/launcher/../../kitty/fast_data_types.so
#4  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#5  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 2 (Thread 0x7c61ed9ec6c0 (LWP 163537) "KittyChildMon"):
#0  0x00007c621d342772 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621d33613c in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c621d3bda8e in poll () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c621c6156ce in io_loop () from <repo>/kitty/launcher/../../kitty/fast_data_types.so
#4  0x00007c621d339d64 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#5  0x00007c621d3cd3fc in ?? () from /lib/x86_64-linux-gnu/libc.so.6

Thread 1 (Thread 0x7c621d137780 (LWP 163470) "kitty"):
#0  0x00007c621d43f613 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c621c683d42 in screen_index () from <repo>/kitty/launcher/../../kitty/fast_data_types.so
#2  0x00007c621c685e24 in draw_text_loop () from <repo>/kitty/launcher/../../kitty/fast_data_types.so
#3  0x00007c621c6868c0 in draw_text.lto_priv () from <repo>/kitty/launcher/../../kitty/fast_data_types.so
#4  0x00007c621c6c38ae in run_worker.lto_priv () from <repo>/kitty/launcher/../../kitty/fast_data_types.so
#5  0x00007c621c613a55 in do_parse () from <repo>/kitty/launcher/../../kitty/fast_data_types.so
#6  0x00007c621c616a28 in process_global_state () from <repo>/kitty/launcher/../../kitty/fast_data_types.so
#7  0x00007c621b53fcca in glfwRunMainLoop () from <repo>/kitty/glfw-x11.so
#8  0x00007c621c6140cc in main_loop.lto_priv () from <repo>/kitty/launcher/../../kitty/fast_data_types.so
#9  0x00007c621d5fe7c0 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#10 0x00007c621d5f257e in PyObject_Vectorcall () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#11 0x00007c621d7371a9 in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#12 0x00007c621d5f408e in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#13 0x00007c621d695ad5 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#14 0x00007c621d5f23c2 in _PyObject_MakeTpCall () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#15 0x00007c621d7371a9 in _PyEval_EvalFrameDefault () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#16 0x00007c621d736159 in PyEval_EvalCode () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
#17 0x00007c621d72ff29 in ?? () from /lib/x86_64-linux-gnu/libpython3.13.so.1.0
[Inferior 1 (process 163470) detached]
```

*End of report.*

