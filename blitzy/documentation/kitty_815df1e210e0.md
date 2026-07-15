# Kitty's scrollback under extreme write pressure: a runtime-observed characterization

This document answers, from **observed runtime behavior** rather than code reading alone, what
happens inside kitty's history/scrollback subsystem when a command "pours out an enormous amount
of text in a very short time." Every behavioral claim below is paired with the exact command that
produced it and the actual, unedited output, plus a `file:line` citation into the source. The
question decomposes into five named things, each answered explicitly:

1. **Fill / stretch / carve** — what unfolds inside the `HistoryBuf` as it fills toward capacity and
   "carves out new segments" (§4).
2. **Segmented storage ↔ pager ring** — the "quiet interaction" between the segmented line store and
   the pager-style ring buffer, and how it holds up under sustained pressure (§5).
3. **Smoothness vs. hesitation** — whether transitions are smooth as segments reach their limits, or
   whether the system "hesitates" at subtle edges (§6).
4. **Concurrent scroll + ingest** — what changes when someone scrolls through old output while new
   data keeps arriving at full speed (§7).
5. **Allocation, wrapping, retention** — how these behave at runtime and how the memory structures
   evolve as pressure builds (§8).

Short answer, up front: under a fast burst the segmented store fills its circular slot array and
**carves out one fresh 2048-line segment at a time** (a `realloc` of the segment pointer array plus a
~5 MiB per-segment `calloc`); once `count` reaches `ynum` the store stops growing and every new line
**evicts the oldest** one, optionally handing it to the pager ring. Transitions are *mostly* smooth,
with two genuine, reproducible hesitation points — the per-segment allocation at each 2048-line
boundary, and the pager ring's `≥1 MiB` growth copies as it extends toward its cap. Scrolling while
ingesting re-anchors the viewport by the number of newly added lines each frame, until it clamps at
the retained-line count. The large (~1–2 ms) pauses one might *guess* are the data structure turn out,
on investigation, to be the embedded interpreter's cyclic garbage collector — a harness artifact, not
the subsystem.

---

## 1. Scope and method

- **Read-only investigation.** No product source, header, test, configuration, or build file was
  modified. The only artifact created is this document. A clean-tree verification is shown in §12.
- **Run-first.** Each scenario was scripted against kitty's real ingest path, executed, and its output
  captured *before* any conclusion was drawn. Each scenario was run **at least twice** with identical
  input; where a value is deterministic across runs this is stated, and where it is not (e.g. wall-clock
  timings) the run-to-run spread is shown rather than hidden.
- **Canonical entry point only.** Bytes are fed to a real `Screen` through the real VT parser; no
  remote-control bypass, debug hook, or synthetic poke of the buffer is used as a basis for any claim.
  The one non-canonical primitive that exists (`HistoryBuf.push`) is not used here; §8.4 additionally
  shows Valgrind's own call stack proving the ingest path is the VT parser → screen → history chain.
- **Observed vs. inferred.** Anything asserted from reading the source rather than observing it at
  runtime is labeled *(inferred)* at the point of use and collected in §10.

---

## 2. Environment, build, and invocation

### 2.1 Reported versions

Every command below was run from the repository root (shown by `pwd`); each `$`-prefixed line is the
exact command and the line(s) beneath it are its complete, unedited output:

```text
$ pwd
/tmp/blitzy/kitty/blitzy-37fb2915-992a-43f5-a568-f9834bbb3df4_a66137

$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal

$ ./kitty/launcher/kitty +runpy 'import sys; print(sys.version)'
3.14.6 (main, Jun 23 2026, 03:01:40) [GCC 11.4.0]

$ go version
go version go1.22.12 linux/amd64

$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0

$ valgrind --version
valgrind-3.25.1
```

The runnable launcher embeds its **own** Python interpreter (**3.14.6**, full build string above)
alongside the compiled `fast_data_types` C extension; that embedded interpreter — not the system
Python — runs every observation script below via `kitty +launch`.

**Source commit.** These observations characterize the history/scrollback subsystem at the required
source commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (branch `kitty_815df1e210e0`). The acceptance
checkout reports `git rev-parse HEAD` = `e6486adb6975d2c063659f6a17d6068bc55a157d`; the required commit
is an **ancestor** of that HEAD, and the two trees are **byte-identical across the entire product**
(`git diff 815df1e2..HEAD -- . ':(exclude)blitzy/**'` is empty — the only difference is this
deliverable under `blitzy/`). Every cited `file:line` and every observed value therefore applies
unchanged. The exact verification is shown in §12.

### 2.2 Canonical build (and an unrelated build-warning caveat)

The documented build is `./dev.sh build`, producing `kitty/launcher/kitty` (`docs/build.rst:19`,
`docs/build.rst:22`); `dev.sh` execs the Go build driver (`dev.sh:9`). In this environment the bare
command **fails**, but for a reason unrelated to the history subsystem — a newer `wayland-protocols`
adds enum values the vendored Wayland backend's `switch` does not list, and the default
`-Werror` promotes that to an error:

```text
$ ./dev.sh build
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
[118 further "[N/122] Compiling <file>" progress lines omitted for length — all succeeded]
[122/122] Compiling kitty/gl-wrapper.c ...
glfw/wl_window.c: In function 'xdgToplevelHandleConfigure':
glfw/wl_window.c:668:9: error: enumeration value 'XDG_TOPLEVEL_STATE_CONSTRAINED_LEFT' not handled in switch [-Werror=switch]
  668 |         switch (*state) {
      |         ^~~~~~
glfw/wl_window.c:668:9: error: enumeration value 'XDG_TOPLEVEL_STATE_CONSTRAINED_RIGHT' not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value 'XDG_TOPLEVEL_STATE_CONSTRAINED_TOP' not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value 'XDG_TOPLEVEL_STATE_CONSTRAINED_BOTTOM' not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
The following build command failed: /tmp/blitzy/kitty/blitzy-37fb2915-992a-43f5-a568-f9834bbb3df4_a66137/dependencies/linux-amd64/bin/python setup.py develop
exit status 1
```

The failing file is `glfw/wl_window.c` (the Wayland **windowing** backend); the history subsystem
(`kitty/history.c`, `kitty/screen.c`, the SIMD string helpers) compiled cleanly in the files just
before it. kitty ships an official flag for exactly this situation, so the canonical build used here is:

```text
$ ./dev.sh build --ignore-compiler-warnings
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
[118 further "[N/122] Compiling <file>" progress lines omitted for length — all succeeded]
[122/122] Compiling kitty/gl-wrapper.c ...
 done
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
Build successful. Run kitty as: kitty/launcher/kitty
```

This changes no runtime behavior of the subsystem under study; it only relaxes `-Werror` for the
unrelated Wayland enum-skew. (A separate `--debug` profiling build is described in §8.4.)

**Supply-chain caveat (build provenance).** On a fresh, cacheless checkout, `./dev.sh build`
(`dev.sh:9` → `bypy/devenv.go`) first fetches a *prebuilt* dependency bundle: `devenv.go` extracts
`BUNDLE_URL` from `ci.py` (`bypy/devenv.go:257`) and downloads the archive with `cached_download`
(`bypy/devenv.go:150-186`). That function validates only the HTTP status and an `ETag` used for
caching — there is **no repository-pinned cryptographic digest or signature check** of the downloaded
archive (a full-file scan of `devenv.go` finds zero `sha256`/`checksum`/`digest`/`signature`
verification). The archive is then `tar xf`-extracted (`bypy/devenv.go:286-291`) and *its* bundled
Python runs `setup.py develop` to build the extension — i.e. downloaded code is executed locally. Bundle
integrity therefore rests on TLS and the upstream host, not on an in-repo pin, so a substituted or
tampered bundle would not be detected by the build itself. **Mitigation:** build inside a
digest-pinned container image, or from a pre-provisioned and independently verified dependency bundle,
and/or record an approved archive digest and verify it before extraction. **No fetch occurred during
this investigation:** the container ships the dependency bundle pre-provisioned (the §2.1 toolchain was
already present), so the build reused that existing cache and made no network download while observing;
the release artifacts were additionally `sha256`-verified before and after the profiling build (§12),
so the measurements reported here are unaffected by this caveat.

### 2.3 Canonical ingest path

A child's output bytes traverse: PTY byte stream → VT parser (`kitty/vt-parser.c`) → screen
operations (`kitty/screen.c`) → line-buffer update → the scroll decision → the history buffer
(`kitty/history.c`). For **live child output** — the path this investigation exercises — the writer
into scrollback is `historybuf_add_line` (`kitty/history.c:286-291`), and on that canonical ingest
path it is reached *only* through the `INDEX_UP` macro in `kitty/screen.c:1552-1567` (which calls
`historybuf_add_line` at `screen.c:1558` and bumps `history_line_added_count` at `screen.c:1559`),
gated by `add_to_history` at `screen.c:1574-1575`. `historybuf_add_line` does have two other callers,
neither of which is a live-ingest path: the Python `HistoryBuf.push` method (`kitty/history.c:342`),
a *non-canonical* scripting entry used here only as an explicitly labelled cross-check, and the
rewrap/resize path (`kitty/rewrap.h:32`), which *re-flows existing* history lines on a size change
rather than ingesting new bytes. Every script below reproduces the "enormous output, very short time"
command by feeding a large, fast block of newline-terminated lines into a real `Screen` via
`parse_bytes` (`kitty_tests/__init__.py:30-36`), which routes the bytes through that exact parser and
therefore through `INDEX_UP`.

### 2.4 Observation harness and a disclosed default-option override

Scripts use two in-process primitives from kitty's own test harness:
`create_screen(cols, lines, scrollback, options=<mapping>)` (`kitty_tests/__init__.py:237-241`) and
`parse_bytes(screen, data)` (`kitty_tests/__init__.py:30-36`).

**Disclosed override (important).** `BaseTest.set_options` seeds a default option dictionary that
includes `scrollback_pager_history_size = 1024` and only *updates* it when the caller passes a
**truthy** `options` mapping (`kitty_tests/__init__.py:223-231`). Because an empty dict `{}` is falsy,
passing `options={}` does **not** disable the pager — it silently leaves a 1024-**byte** pager ring
enabled. Every script here therefore passes an **explicit** `{'scrollback_pager_history_size': 0}`
when it intends the product default (pager disabled), and an explicit nonzero byte value when it
intends the pager enabled. Note the harness value is a **raw byte count**, whereas the user-facing
option `scrollback_pager_history_size` is expressed in **megabytes** and converted by
`options/utils.py:564-566`; the product default is `0` (disabled) at `options/definition.py:406-407`,
and `scrollback_lines` defaults to `2000` at `options/definition.py:372-373`. The units difference is
demonstrated directly in §5.

### 2.5 What is and isn't observable from Python

`HistoryBuf` exposes `xnum`, `ynum`, and `count` as read-only members
(`kitty/history.c:554-559`); it does **not** expose `num_segments` or `start_of_data`. Segment count is
therefore **derived** as `ceil(min(count, ynum) / 2048)` *(inferred from `SEGMENT_SIZE`,
`kitty/history.c:15)`* and independently corroborated by process RSS and by Valgrind Massif (§8.4).
Pager size is read exactly via `pagerhist_as_bytes()`/`pagerhist_as_text()`
(`kitty/history.c:460-494`). For §7, `Screen.scrolled_by` is read-only (`kitty/screen.c:4903`) and
`Screen.history_line_added_count` is readable (`kitty/screen.c:4908`).

**Interpreter optimization level (why the scripts use `print`, not `assert`).** The interpreter behind
`kitty +launch` runs **optimized** — `sys.flags.optimize == 2`, so `__debug__` is `False` and every
bare `assert` is *stripped at compile time* and can never fire:

```text
$ ./kitty/launcher/kitty +runpy 'import sys; print("__debug__ =", __debug__, "; optimize =", sys.flags.optimize)'
__debug__ = False ; optimize = 2
```

Consequently the observation scripts below **do not use `assert`** for any verification; they check by
printing the actual values (and, where a mismatch must fail loudly, by an explicit `if …: raise`). Every
`True`/"confirmed" in the captured output is therefore a real computed comparison that survived
optimization, not an assertion that could have been silently elided.

### 2.6 Temporary workspace and cleanup

All observation scripts and profiler outputs live in a **private** directory made with `mktemp -d`
and locked to the owner with `chmod 700` (an unpredictable name, not a guessable `/tmp/obs_*.py`), and
removed afterward; the repository tree is left untouched. The workspace `$WORK` is created **before any
script is written or run** — every `$WORK/<file>` path in the sections that follow refers to it. Because
one profiling detour (§8.4) rebuilds the C extension as a debug variant, the release artifacts are
**backed up into `$WORK` first** so they can be restored bit-for-bit afterward:

```text
$ WORK=$(mktemp -d /tmp/kitty_obs.XXXXXX); chmod 700 "$WORK"; stat -c '%A %n' "$WORK"
drwx--S--- /tmp/kitty_obs.iPXoYt

$ mkdir -p "$WORK/release_backup/launcher"
$ cp kitty/fast_data_types.so "$WORK/release_backup/"
$ cp kitty/launcher/kitty kitty/launcher/kitten "$WORK/release_backup/launcher/"
$ sha256sum kitty/fast_data_types.so kitty/launcher/kitty kitty/launcher/kitten
c314243b974fc0d132cf55e8d7362f013c74e964935dd0d07e6b277c31d67888  kitty/fast_data_types.so
05fc54d4e44f1615b61f00970ed2b30fcf32eb628d6feeba64978455e4c3da12  kitty/launcher/kitty
93f3eaae4f1f4a61d6b254f2c024a0b37574b188cb4dcec9c30dd6f714febde4  kitty/launcher/kitten
```

Owner-only `rwx` (group/other have no access). Every scenario script
(`scenarioA.py` … `scenarioF.py`, `units.py`, `mallocinfo.py`, `massif_target.py`) and the
`sizeof_probe.c` helper (§3.1), plus all captured outputs, were written under `$WORK`, never inside the
repository. The `mktemp` suffix is random by design, so its exact value differs run to run; the full
restore/verify and final clean-tree check are in §12.

---

## 3. The two structures under study

Under stress, exactly **two** memory structures inside `HistoryBuf` participate
(`kitty/data-types.h:282-290`):

- **The segmented line store** — an array of `HistoryBufSegment` (`kitty/data-types.h:262-266`), each
  segment holding `SEGMENT_SIZE = 2048` lines (`kitty/history.c:15`). Lines occupy a **circular** slot
  space indexed `(start_of_data + i) % ynum`; the physical backing is grown one segment at a time by
  `add_segment` (`kitty/history.c:17-29`), which `realloc`s the segment pointer array
  (`kitty/history.c:20`) and `calloc`s a fresh per-segment block (`kitty/history.c:25`). Segments are
  added lazily, only when a line index demands one, by `segment_for` (`kitty/history.c:36-42`, growth
  condition at `history.c:39`).
- **The pager ring** — an optional `PagerHistoryBuf` (`kitty/data-types.h:268-272`) wrapping the
  vendored byte ring buffer in `3rdparty/ringbuf/ringbuf.c`. It is allocated by `alloc_pagerhist`
  (`kitty/history.c:69-80`) only when the configured size is nonzero (returns without allocating at
  `history.c:72` when zero); its initial capacity is `MIN(1 MiB, configured)` (`kitty/history.c:66-67`),
  and it grows in `≥1 MiB` steps via `pagerhist_extend` (`kitty/history.c:89-101`) up to its cap.

The per-segment allocation size matters throughout, so it is pinned down exactly next.

### 3.1 Exact per-segment allocation size

`add_segment` performs one `calloc` (`kitty/history.c:25`) sized as
`xnum * 2048 * sizeof(CPUCell)  +  xnum * 2048 * sizeof(GPUCell)  +  2048 * sizeof(LineAttrs)`.
The cell sizes are pinned by `static_assert`: `sizeof(GPUCell) == 20` (`kitty/data-types.h:221`) and
`sizeof(CPUCell) == 12` (`kitty/data-types.h:228`). `LineAttrs` has **no** `static_assert`, and it is
**not** one byte: it is a union (`kitty/data-types.h:231-239`) whose bitfield is declared with the
`PromptKind` **enum** type (`kitty/data-types.h:230`), which forces 4-byte storage for the whole union.
The self-contained probe below copies those two declarations **verbatim** from the header, so it builds
with a plain C11 toolchain (no kitty headers) and isolates the one non-obvious size. It is created in
`$WORK` (created in §2.6) with a heredoc, `cat > "$WORK/sizeof_probe.c" << 'EOF' … EOF`:

```c
/* sizeof_probe.c — self-contained probe reproducing the ONE non-obvious size in
 * kitty/data-types.h: sizeof(LineAttrs) is 4, not 1. The GPUCell (20) and CPUCell
 * (12) sizes are pinned by static_assert in the real header (data-types.h:221,228)
 * and are cited directly; this probe isolates LineAttrs, whose bitfield is declared
 * with the PromptKind *enum* type (data-types.h:230), forcing int-width (4-byte)
 * storage for the whole union. Definitions copied verbatim from
 * kitty/data-types.h:230-239 so the probe compiles with a plain C11 toolchain. */
#include <stdio.h>
#include <stdint.h>

typedef enum { UNKNOWN_PROMPT_KIND = 0, PROMPT_START = 1, SECONDARY_PROMPT = 2, OUTPUT_START = 3 } PromptKind;
typedef union LineAttrs {
    struct {
        uint8_t is_continued : 1;
        uint8_t has_dirty_text : 1;
        uint8_t has_image_placeholders : 1;
        PromptKind prompt_kind : 2;
    };
    uint8_t val;
} LineAttrs;

int main(void) {
    printf("sizeof(LineAttrs)=%zu  sizeof(PromptKind)=%zu\n",
           sizeof(LineAttrs), sizeof(PromptKind));
    return 0;
}
```

```text
$ gcc -std=c11 -O2 "$WORK/sizeof_probe.c" -o "$WORK/sizeof_probe" && "$WORK/sizeof_probe"
sizeof(LineAttrs)=4  sizeof(PromptKind)=4
```

As a cross-check, a second probe that `#include`s the **real** `kitty/data-types.h` (using the embedded
interpreter's own include path) confirms all four sizes at once — `LineAttrs` is indeed 4, and
`GPUCell`/`CPUCell` match their `static_assert`s:

```text
$ PYINC=$(./kitty/launcher/kitty +runpy 'import sysconfig;print(sysconfig.get_path("include"))')
$ printf '%s\n' '#include "kitty/data-types.h"' '#include <stdio.h>' \
    'int main(void){printf("LineAttrs=%zu GPUCell=%zu CPUCell=%zu PromptKind=%zu\n",' \
    'sizeof(LineAttrs),sizeof(GPUCell),sizeof(CPUCell),sizeof(PromptKind));return 0;}' > "$WORK/sizeof_real.c"
$ gcc -std=c11 -I. -I"$PYINC" "$WORK/sizeof_real.c" -o "$WORK/sizeof_real" && "$WORK/sizeof_real"
LineAttrs=4 GPUCell=20 CPUCell=12 PromptKind=4
```

So at `xnum = 80` the per-segment request is exactly:

```text
CPU cells : 2048 * 80 * 12 = 1,966,080
GPU cells : 2048 * 80 * 20 = 3,276,800
LineAttrs : 2048 *      4  =     8,192
per-segment calloc         = 5,251,072 bytes  (= 5128 KiB)
```

This 5,251,072-byte figure is confirmed independently by Valgrind Massif in §8.4 (which reports
`5,251,072B` at `add_segment (history.c:25)`), and it matches the per-segment RSS step measured in §4.

---

## 4. REQ-1 — Fill, stretch, and carve

**Mechanism (cause → effect).** With the pager disabled, ingest reaches `historybuf_add_line`
(`kitty/history.c:286-291`) → `historybuf_push` (`kitty/history.c:275-284`). While `count < ynum`,
`historybuf_push` computes the target slot `(start_of_data + count) % ynum` (`kitty/history.c:277`),
calls `init_line` on it, and increments `count` (`kitty/history.c:282`). `init_line` reaches the slot
through `segment_for` (`kitty/history.c:36-42`), which — when the requested line lands in a segment
that does not physically exist yet — calls `add_segment` (growth condition at `kitty/history.c:39`).
`add_segment` `realloc`s the segment pointer array (`kitty/history.c:20`) and `calloc`s a new
5,251,072-byte block (`kitty/history.c:25`). That is the literal "carving out of a new segment":
one fresh 2048-line block appears each time `count` crosses a 2048 multiple. Once `count == ynum`
the store is full and `historybuf_push` stops growing (the `count++` branch is no longer taken);
this is the fill→saturate boundary examined further in §6.

**Scenario A** drives a fast burst through the canonical path, explicitly disabling the pager
(§2.4), and samples `count`, derived segment count, and process RSS at each 2048-line boundary, then
far past capacity. RSS is read **once** per row from `/proc/self/status`:

```python
# scenarioA.py — REQ-1 fill/stretch/carve.
# Pager EXPLICITLY DISABLED (product default) so we isolate the segmented store.
import math
from kitty_tests import BaseTest, parse_bytes

def rss_kb():
    with open('/proc/self/status') as f:
        for ln in f:
            if ln.startswith('VmRSS:'):
                return int(ln.split()[1])
    return -1

def derived_segments(count, ynum):
    return math.ceil(min(count, ynum) / 2048)

class T(BaseTest):
    def run(self):
        cols, lines, scrollback = 80, 24, 20000
        s = self.create_screen(cols, lines, scrollback,
                               options={'scrollback_pager_history_size': 0})
        hb = s.historybuf
        ynum, xnum = hb.ynum, hb.xnum
        base = rss_kb()
        print(f"ynum={ynum} xnum={xnum}  pager_disabled  BASELINE: count={hb.count} "
              f"derived_segments={derived_segments(hb.count, ynum)} RSS={base} kB")
        print(f"{'target':<8} {'count':<8} {'segments':<9} {'RSS_kB':<10} {'dRSS_kB':<9}")
        fed = 0
        for target in (2048, 4096, 6144, 8192, 10240, 12288, 14336, 16384, 18432, 20000):
            need = (target + (lines - 1)) - fed
            parse_bytes(s, ("".join(f"L{fed+i:08d}\r\n" for i in range(need))).encode())
            fed += need
            r = rss_kb()                      # single snapshot per row
            print(f"{target:<8} {hb.count:<8} {derived_segments(hb.count, ynum):<9} "
                  f"{r:<10} {r-base:<9}")
        # push far beyond capacity to demonstrate saturation
        parse_bytes(s, ("".join(f"X{i:08d}\r\n" for i in range(80000))).encode())
        r = rss_kb()
        print(f"SATURATION: requested target=80000, actual count={hb.count} (ynum={ynum}) "
              f"segments={derived_segments(hb.count, ynum)} RSS={r} kB dRSS={r-base} kB")
        print(f"count==ynum ? {hb.count==ynum} ; count never exceeded ynum ? {hb.count<=ynum} ; "
              f"pager_bytes={len(hb.pagerhist_as_bytes())}")

T().run()
```

Command and complete output (two identical runs):

```text
$ for r in 1 2; do echo "=== Scenario A / RUN $r ==="; \
    ./kitty/launcher/kitty +launch "$WORK/scenarioA.py"; done
=== Scenario A / RUN 1 ===
ynum=20000 xnum=80  pager_disabled  BASELINE: count=0 derived_segments=0 RSS=25928 kB
target   count    segments  RSS_kB     dRSS_kB
2048     2048     1         31532      5604
4096     4096     2         36664      10736
6144     6144     3         41796      15868
8192     8192     4         46928      21000
10240    10240    5         52064      26136
12288    12288    6         57196      31268
14336    14336    7         62328      36400
16384    16384    8         67460      41532
18432    18432    9         72592      46664
20000    20000    10        76524      50596
SATURATION: requested target=80000, actual count=20000 (ynum=20000) segments=10 RSS=80760 kB dRSS=54832 kB
count==ynum ? True ; count never exceeded ynum ? True ; pager_bytes=0
=== Scenario A / RUN 2 ===
ynum=20000 xnum=80  pager_disabled  BASELINE: count=0 derived_segments=0 RSS=26052 kB
target   count    segments  RSS_kB     dRSS_kB
2048     2048     1         31660      5608
4096     4096     2         36792      10740
6144     6144     3         41924      15872
8192     8192     4         47056      21004
10240    10240    5         52192      26140
12288    12288    6         57324      31272
14336    14336    7         62456      36404
16384    16384    8         67588      41536
18432    18432    9         72720      46668
20000    20000    10        76652      50600
SATURATION: requested target=80000, actual count=20000 (ynum=20000) segments=10 RSS=82196 kB dRSS=56144 kB
count==ynum ? True ; count never exceeded ynum ? True ; pager_bytes=0
```

**Before / during / after.**
- *Before:* `BASELINE: count=0 derived_segments=0` — an empty history (one segment is pre-allocated at
  construction, `create_historybuf` → `add_segment`, `kitty/history.c:127`, but holds no lines yet).
- *During (stretch/carve):* as the burst crosses each 2048 multiple, `count` tracks the fed line count
  and the derived segment count steps `1 → 2 → 3 → … → 10`. RSS climbs in near-constant steps —
  `dRSS_kB` = 5604, 10736, 15868, 21000, 26136, …, 50596 — i.e. **≈5132 kB per new segment**. The
  increment is *not* perfectly uniform: reading the `dRSS_kB` column as consecutive differences gives
  the gap sequence `5132, 5132, 5132, 5136, 5132, 5132, 5132, 5132, 3932` — eight full-segment gaps of
  5132 kB, a **single 5136 kB step into the `count = 10240` boundary** (a one-page, ≈4 kB, first-touch
  jitter, visible directly in the `dRSS_kB` column: `26136 − 21000 = 5136`), and a smaller **3932 kB
  final step** because the tenth segment is only partially faulted (`20000 − 18432 = 1568` of its 2048
  rows). The ≈5132 kB step matches the 5,251,072-byte (5128 KiB) per-segment `calloc` above (the small
  excess is page-table and first-touch overhead). Every **structural value** (`count`, the
  `1 → 2 → … → 10` segment progression, `count == ynum`, `pager_bytes = 0`) was **byte-identical across
  both runs**, and so was the *gap sequence* above — including the 5136 kB outlier, which reproduced at
  the same `count = 10240` boundary in both runs; what drifted was only the *absolute* `dRSS`, by ≈4 kB
  (one page) run-to-run (RUN 2's first step read 5608 vs RUN 1's 5604), reflecting ordinary first-touch
  / baseline-RSS noise, not any change in allocation behaviour. RSS is a process-wide figure and is
  used here as corroboration of the segmented store's growth, not as its exclusive measure (§8.4/§8.5
  give allocator-level attribution).
- *After (saturate):* requesting 80,000 more lines pins `count` at `ynum = 20000` with `segments = 10`;
  `count == ynum` is `True` and `count` never exceeds `ynum`. `pager_bytes = 0` throughout, confirming
  the pager truly stayed disabled.

**What this shows for REQ-1.** The buffer does not "stretch" elastically; it **carves discrete
2048-line segments** on demand — a pointer-array `realloc` plus a ~5 MiB `calloc` at each boundary —
until the circular slot space is full at `ynum`, after which it holds steady and recycles slots.

---

## 5. REQ-2 — The segmented store ↔ pager ring relationship

**Mechanism (cause → effect).** The two structures interact at exactly one place: the eviction gate in
`historybuf_push` (`kitty/history.c:275-284`). While `count < ynum` the pager is untouched. The moment
a new line arrives with `count == ynum`, `historybuf_push` takes its other branch — it calls
`pagerhist_push` on the line about to be overwritten (`kitty/history.c:280`) and only then advances
`start_of_data` (`kitty/history.c:281`), so the oldest scrollback line is **serialized into the pager
ring at the instant it is evicted from the segmented store**. `pagerhist_push`
(`kitty/history.c:258-273`) writes an SGR reset `"\x1b[m"` (`history.c:266`), the line's UCS4 text, a
carriage return (`history.c:269`), and a newline **only if the line did not wrap** (`history.c:270`).
The segmented store is thus the bounded "live" window; the pager ring is the optional overflow that
catches what the window drops.

**Units caveat (raw bytes here, megabytes in config).** Through `create_screen(options=<mapping>)` the value
is a **raw byte count**. This is verified directly — a tiny cap truncates the ring to that many bytes,
a large cap leaves the same 6800 serialized bytes (400 records × 17 B) untouched:

```python
# units.py — verify that a raw int passed as scrollback_pager_history_size
# through create_screen(options=...) is used as RAW BYTES (not megabytes).
from kitty_tests import BaseTest, parse_bytes

class T(BaseTest):
    def run(self):
        cols, lines, sb = 80, 24, 100
        # Feed K lines past saturation so K serialized records (17 B each) are evicted.
        K = 400
        total = (sb + (lines - 1)) + K  # reach count==sb then evict K more
        payload = "".join(f"LINE{i:08d}\r\n" for i in range(total)).encode()
        for cap in (8, 64, 1048576, 4194304):
            s = self.create_screen(cols, lines, sb,
                                   options={'scrollback_pager_history_size': cap})
            parse_bytes(s, payload)
            pb = len(s.historybuf.pagerhist_as_bytes())
            print(f"cap={cap:<10} -> pager_bytes={pb}")

T().run()
```
```text
$ for r in 1 2; do echo "=== units / RUN $r ==="; \
    ./kitty/launcher/kitty +launch "$WORK/units.py"; done
=== units / RUN 1 ===
cap=8          -> pager_bytes=8
cap=64         -> pager_bytes=64
cap=1048576    -> pager_bytes=6800
cap=4194304    -> pager_bytes=6800
=== units / RUN 2 ===
cap=8          -> pager_bytes=8
cap=64         -> pager_bytes=64
cap=1048576    -> pager_bytes=6800
cap=4194304    -> pager_bytes=6800
```

**Scenario B** enables the pager (16 MiB) and watches the hand-off, plus a Part 0 that isolates the
ring's initial allocation and a Part 2 that confirms the disabled case:

```python
# scenarioB.py — REQ-2 segmented store <-> pager ring relationship.
from kitty_tests import BaseTest, parse_bytes

def rss_kb():
    with open('/proc/self/status') as f:
        for ln in f:
            if ln.startswith('VmRSS:'):
                return int(ln.split()[1])
    return -1

def feed(s, n, tag):
    parse_bytes(s, ("".join(f"{tag}{i:08d}\r\n" for i in range(n))).encode())

class T(BaseTest):
    def run(self):
        cols, lines = 80, 24
        CAP = 16 * 1024 * 1024  # 16 MiB, raw bytes

        print("===== PART 0: initial ring capacity = MIN(1 MiB, cap) allocated at construction =====")
        # Construction allocates segment 1 (~5 MiB, lazily touched) + the pager ring.
        # Compare pager=0 vs pager=16MiB to isolate the ring's initial allocation.
        base = rss_kb()
        s0 = self.create_screen(cols, lines, 100, options={'scrollback_pager_history_size': 0})
        rss0 = rss_kb()
        s1 = self.create_screen(cols, lines, 100, options={'scrollback_pager_history_size': CAP})
        rss1 = rss_kb()
        print(f"construct pager=0     : dRSS={rss0-base} kB  pager_bytes={len(s0.historybuf.pagerhist_as_bytes())}")
        print(f"construct pager=16MiB : dRSS_over_pager0={rss1-rss0} kB  pager_bytes={len(s1.historybuf.pagerhist_as_bytes())} (used=0 while capacity=MIN(1MiB,16MiB))")

        print("\n===== PART 1: pager ENABLED (cap = 16 MiB = %d bytes) =====" % CAP)
        s = self.create_screen(cols, lines, 100, options={'scrollback_pager_history_size': CAP})
        hb = s.historybuf
        ynum = hb.ynum
        # Fill toward ynum but stay below it.
        feed(s, 50 + (lines - 1), "LINE")      # count -> 50
        print(f"count={hb.count:<4} ynum={ynum}  pager_bytes={len(hb.pagerhist_as_bytes()):<7} (count<ynum -> ring empty)")
        feed(s, 49, "LINE")                     # count -> 99
        print(f"count={hb.count:<4} ynum={ynum}  pager_bytes={len(hb.pagerhist_as_bytes()):<7} (count<ynum)")
        feed(s, 1, "LINE")                      # count -> 100 == ynum
        print(f"count={hb.count:<4} ynum={ynum}  pager_bytes={len(hb.pagerhist_as_bytes()):<7} <== count JUST reached ynum, still no eviction")
        for add in (1, 10, 100, 1000):
            feed(s, add, "LINE")
            print(f"+{add:<5} lines -> count={hb.count:<4} (pinned==ynum? {hb.count==ynum}) "
                  f"pager_bytes={len(hb.pagerhist_as_bytes()):<7}")
        full = hb.pagerhist_as_bytes()
        print(f"pager head (oldest serialized) = {full[:40]!r}")
        print(f"pager tail (newest serialized) = {full[-40:]!r}")

        print("\n===== PART 2: pager DISABLED (cap = 0) =====")
        s2 = self.create_screen(cols, lines, 100, options={'scrollback_pager_history_size': 0})
        hb2 = s2.historybuf
        feed(s2, 100 + (lines - 1), "LINE")     # count -> 100 == ynum
        feed(s2, 5000, "GONE")                  # evict 5000
        pb2 = len(hb2.pagerhist_as_bytes())
        print(f"count={hb2.count} ynum={hb2.ynum} pager_bytes={pb2} (disabled -> alloc_pagerhist returns NULL, ring never exists)")

T().run()
```
```text
$ for r in 1 2; do echo "================= RUN $r ================="; \
    ./kitty/launcher/kitty +launch "$WORK/scenarioB.py"; echo "exit=$?"; done
================= RUN 1 =================
===== PART 0: initial ring capacity = MIN(1 MiB, cap) allocated at construction =====
construct pager=0     : dRSS=1308 kB  pager_bytes=0
construct pager=16MiB : dRSS_over_pager0=1220 kB  pager_bytes=0 (used=0 while capacity=MIN(1MiB,16MiB))

===== PART 1: pager ENABLED (cap = 16 MiB = 16777216 bytes) =====
count=50   ynum=100  pager_bytes=0       (count<ynum -> ring empty)
count=99   ynum=100  pager_bytes=0       (count<ynum)
count=100  ynum=100  pager_bytes=0       <== count JUST reached ynum, still no eviction
+1     lines -> count=100  (pinned==ynum? True) pager_bytes=17
+10    lines -> count=100  (pinned==ynum? True) pager_bytes=187
+100   lines -> count=100  (pinned==ynum? True) pager_bytes=1887
+1000  lines -> count=100  (pinned==ynum? True) pager_bytes=18887
pager head (oldest serialized) = b'\x1b[mLINE00000000\r\n\x1b[mLINE00000001\r\n\x1b[mLIN'
pager tail (newest serialized) = b'0874\r\n\x1b[mLINE00000875\r\n\x1b[mLINE00000876\r\n'

===== PART 2: pager DISABLED (cap = 0) =====
count=100 ynum=100 pager_bytes=0 (disabled -> alloc_pagerhist returns NULL, ring never exists)
exit=0
================= RUN 2 =================
===== PART 0: initial ring capacity = MIN(1 MiB, cap) allocated at construction =====
construct pager=0     : dRSS=1304 kB  pager_bytes=0
construct pager=16MiB : dRSS_over_pager0=1220 kB  pager_bytes=0 (used=0 while capacity=MIN(1MiB,16MiB))

===== PART 1: pager ENABLED (cap = 16 MiB = 16777216 bytes) =====
count=50   ynum=100  pager_bytes=0       (count<ynum -> ring empty)
count=99   ynum=100  pager_bytes=0       (count<ynum)
count=100  ynum=100  pager_bytes=0       <== count JUST reached ynum, still no eviction
+1     lines -> count=100  (pinned==ynum? True) pager_bytes=17
+10    lines -> count=100  (pinned==ynum? True) pager_bytes=187
+100   lines -> count=100  (pinned==ynum? True) pager_bytes=1887
+1000  lines -> count=100  (pinned==ynum? True) pager_bytes=18887
pager head (oldest serialized) = b'\x1b[mLINE00000000\r\n\x1b[mLINE00000001\r\n\x1b[mLIN'
pager tail (newest serialized) = b'0874\r\n\x1b[mLINE00000875\r\n\x1b[mLINE00000876\r\n'

===== PART 2: pager DISABLED (cap = 0) =====
count=100 ynum=100 pager_bytes=0 (disabled -> alloc_pagerhist returns NULL, ring never exists)
exit=0
```

**Before / during / after (the hand-off).**
- *Before eviction:* at `count = 50` and `count = 99` (`ynum = 100`), `pager_bytes = 0` — while the
  segmented store is still filling, the pager stays empty. Even at `count = 100` (`count` *just* equals
  `ynum`) `pager_bytes` is still `0`: the line that filled the last slot was added, not evicted.
- *During eviction:* the very next lines cross into the eviction branch and the pager begins to grow in
  lockstep — `+1 → 17 B`, `+10 → 187 B`, `+100 → 1887 B`, `+1000 → 18887 B` (17 bytes per evicted
  record: `"\x1b[m"` = 3, `"LINE00000000"` = 12, `"\r\n"` = 2). The ring is FIFO: its **head** holds the
  oldest evicted line (`\x1b[mLINE00000000…`) and its **tail** the newest.
- *After (disabled):* Part 2 shows that with an explicit `0`, feeding 5000 evictions still yields
  `pager_bytes = 0` — `alloc_pagerhist` returned without allocating (`kitty/history.c:72`) and the ring
  never exists.

**Initial ring capacity vs. used bytes (Part 0).** Constructing a screen with a 16 MiB pager costs
**≈1220 kB more RSS** than constructing with the pager disabled, *while `pager_bytes` is still 0*.
*Per source*, the ring's **initial capacity** is `MIN(1 MiB, configured)` (`kitty/history.c:66-67`),
here 1 MiB, allocated up front by `ringbuf_new` (`kitty/history.c:76`) even though nothing has been
serialized yet — and `pagerhist_as_bytes()` only ever reports *used* bytes (here 0), never that
allocated capacity (§2.5). The ≈1220 kB construction-time RSS delta therefore *corroborates* that
source-specified ~1 MiB reservation rather than measuring it directly: enabling the pager reserves the
capacity immediately, while "used" bytes (`pager_bytes`) only grow later, once eviction begins. This
up-front reservation read 1220 kB in both runs; as an RSS-derived delta it carries the ~one-page wobble
noted in §9, so it is treated as corroboration of — not an exact measurement of — the ~1 MiB initial
capacity (the capacity value itself is the source constant, not a directly read runtime quantity).

**How the relationship holds up under pressure.** Steady and simple: the segmented store's size is
fixed at saturation, and each additional evicted line adds one serialized record to the ring — a bounded
`memcpy` into the ring's tail — until the ring itself reaches its cap (§6.2). There is no coupling that
degrades; the only cost that scales is the ring's occasional capacity growth, examined next.

---

## 6. REQ-3 — Smooth transitions vs. subtle hesitation

Most of the burst is smooth: the common case is a slot write and a `count++`. There are exactly **two**
genuine, reproducible hesitation points, and one *tempting-but-wrong* candidate that investigation
attributes to the harness rather than the subsystem.

### 6.1 Edge (a): the 2048-line segment boundary

**Claim.** The hesitation is not *at* the 2048 multiple but on the **very next line** (`count = 2048k+1`),
which is the first push that needs a not-yet-existing segment and therefore triggers `segment_for` →
`add_segment` (pointer-array `realloc` at `kitty/history.c:20` + 5,251,072-byte `calloc` at
`kitty/history.c:25`). **Scenario C** times every single line, isolates the C cost by toggling Python's
cyclic GC, checks reproducibility over 5 fresh trials, and directly tests whether the *large* pauses are
GC:

```python
# scenarioC.py — REQ-3 edge (a): per-line timing across 2048-line segment
# boundaries, plus a controlled investigation of off-boundary spikes.
import time, gc, statistics
from kitty_tests import BaseTest, parse_bytes

NLINES = 10000

def one_trial(bt):
    # fresh screen each trial; pager disabled to isolate the segmented store
    s = bt.create_screen(80, 24, 20000, options={'scrollback_pager_history_size': 0})
    hb = s.historybuf
    times = []
    for i in range(NLINES):
        data = (f"L{i:08d}\r\n").encode()
        t0 = time.perf_counter_ns()
        parse_bytes(s, data)
        times.append((hb.count, (time.perf_counter_ns() - t0) / 1000.0))
    return times

def detail(times, label):
    us = [t for _, t in times]
    print(f"--- {label}: fed {len(times)} single lines; final count={times[-1][0]} ---")
    print(f"per-line us: mean={statistics.mean(us):.3f} median={statistics.median(us):.3f} "
          f"p99={sorted(us)[int(0.99*len(us))]:.3f} max={max(us):.3f} min={min(us):.3f}")
    print("12 slowest (count, us, dist-to-nearest-2048-multiple):")
    for c, t in sorted(times, key=lambda x: -x[1])[:12]:
        print(f"   count={c:<6} {t:9.3f} us  dist={min(c % 2048, 2048 - (c % 2048))}")
    for m in (2048, 4096, 6144, 8192):
        d = dict(times)
        cells = "  ".join(f"c{c}={d.get(c, float('nan')):.2f}" for c in range(m-3, m+4))
        print(f"  boundary ~{m}: {cells}")

class T(BaseTest):
    def run(self):
        print("########## PART A: GC ENABLED (realistic harness) — single fresh trial ##########")
        gc.enable(); detail(one_trial(self), "GC-ON")
        print("\n########## PART B: GC DISABLED (isolates the C subsystem) — single fresh trial ##########")
        gc.disable(); detail(one_trial(self), "GC-OFF"); gc.enable()
        print("\n########## PART C: reproducibility of the boundary spikes (GC OFF, 5 fresh trials) ##########")
        gc.disable()
        trials = [one_trial(self) for _ in range(5)]
        gc.enable()
        print("segment-boundary first-push counts (2048k+1) — us per trial:")
        for c in (2049, 4097, 6145, 8193):
            print(f"   count={c:<6} -> [" + ", ".join(f"{dict(tr).get(c, float('nan')):.1f}" for tr in trials) + "]")
        print("off-boundary count=1918 (originally reported ~27us) — us per trial:")
        print("   count=1918   -> [" + ", ".join(f"{dict(tr).get(1918, float('nan')):.2f}" for tr in trials) + "]")
        print("\n########## PART D: are the large off-boundary maxima caused by the cyclic GC? ##########")
        for label, dis in (("GC ON ", False), ("GC OFF", True)):
            (gc.disable() if dis else gc.enable())
            ncoll = [0]
            def cb(phase, info, _n=ncoll):
                if phase == 'stop':
                    _n[0] += 1
            gc.callbacks.append(cb)
            worst_off, worst_c = 0.0, -1
            for _ in range(5):
                for c, t in one_trial(self):
                    if min(c % 2048, 2048 - (c % 2048)) > 8 and t > worst_off:
                        worst_off, worst_c = t, c
            gc.callbacks.remove(cb)
            print(f"   {label}: worst OFF-boundary per-line over 5 trials = {worst_off:8.1f}us "
                  f"(count={worst_c})  gc_collections={ncoll[0]}")
        gc.enable()

T().run()
```
```text
$ for r in 1 2; do echo "================= RUN $r ================="; \
    ./kitty/launcher/kitty +launch "$WORK/scenarioC.py"; echo "exit=$?"; done
================= RUN 1 =================
########## PART A: GC ENABLED (realistic harness) — single fresh trial ##########
--- GC-ON: fed 10000 single lines; final count=9977 ---
per-line us: mean=2.452 median=2.526 p99=8.023 max=880.348 min=1.060
12 slowest (count, us, dist-to-nearest-2048-multiple):
   count=7719     880.348 us  dist=473
   count=8751      90.370 us  dist=559
   count=300       75.874 us  dist=300
   count=0         32.379 us  dist=0
   count=1892      22.861 us  dist=156
   count=8752      16.460 us  dist=560
   count=4195      16.419 us  dist=99
   count=1995      16.100 us  dist=53
   count=2049      15.580 us  dist=1
   count=4097      15.055 us  dist=1
   count=6411      14.840 us  dist=267
   count=8146      14.817 us  dist=46
  boundary ~2048: c2045=3.83  c2046=3.05  c2047=1.29  c2048=1.44  c2049=15.58  c2050=1.38  c2051=2.79
  boundary ~4096: c4093=3.76  c4094=7.50  c4095=1.29  c4096=1.54  c4097=15.05  c4098=1.34  c4099=2.65
  boundary ~6144: c6141=3.92  c6142=2.64  c6143=1.28  c6144=1.52  c6145=13.30  c6146=1.34  c6147=3.00
  boundary ~8192: c8189=4.05  c8190=2.69  c8191=1.33  c8192=1.57  c8193=14.36  c8194=1.44  c8195=2.94

########## PART B: GC DISABLED (isolates the C subsystem) — single fresh trial ##########
--- GC-OFF: fed 10000 single lines; final count=9977 ---
per-line us: mean=2.502 median=2.558 p99=6.878 max=63.520 min=1.198
12 slowest (count, us, dist-to-nearest-2048-multiple):
   count=2049      63.520 us  dist=1
   count=4097      54.345 us  dist=1
   count=6145      49.527 us  dist=1
   count=8193      41.435 us  dist=1
   count=0         23.069 us  dist=0
   count=4778      20.687 us  dist=682
   count=1888      18.450 us  dist=160
   count=5207      17.776 us  dist=937
   count=0         17.623 us  dist=0
   count=708       17.111 us  dist=708
   count=1105      16.445 us  dist=943
   count=4990      16.242 us  dist=894
  boundary ~2048: c2045=1.47  c2046=1.46  c2047=1.38  c2048=1.27  c2049=63.52  c2050=1.44  c2051=15.22
  boundary ~4096: c4093=1.27  c4094=2.66  c4095=1.25  c4096=1.52  c4097=54.34  c4098=1.62  c4099=2.66
  boundary ~6144: c6141=2.55  c6142=1.29  c6143=1.27  c6144=1.61  c6145=49.53  c6146=1.46  c6147=2.73
  boundary ~8192: c8189=1.27  c8190=3.00  c8191=1.25  c8192=9.46  c8193=41.44  c8194=2.98  c8195=1.32

########## PART C: reproducibility of the boundary spikes (GC OFF, 5 fresh trials) ##########
segment-boundary first-push counts (2048k+1) — us per trial:
   count=2049   -> [51.4, 438.0, 433.5, 410.0, 455.0]
   count=4097   -> [51.2, 466.7, 422.5, 417.0, 456.8]
   count=6145   -> [44.6, 442.0, 431.6, 427.7, 456.5]
   count=8193   -> [38.4, 46.8, 656.4, 423.3, 459.9]
off-boundary count=1918 (originally reported ~27us) — us per trial:
   count=1918   -> [1.29, 1.38, 1.49, 1.45, 1.41]

########## PART D: are the large off-boundary maxima caused by the cyclic GC? ##########
   GC ON : worst OFF-boundary per-line over 5 trials =   1815.1us (count=1753)  gc_collections=21
   GC OFF: worst OFF-boundary per-line over 5 trials =     69.5us (count=8805)  gc_collections=0
exit=0
================= RUN 2 =================
########## PART A: GC ENABLED (realistic harness) — single fresh trial ##########
--- GC-ON: fed 10000 single lines; final count=9977 ---
per-line us: mean=2.421 median=2.524 p99=8.852 max=34.957 min=1.076
12 slowest (count, us, dist-to-nearest-2048-multiple):
   count=0         34.957 us  dist=0
   count=4698      26.310 us  dist=602
   count=9426      26.177 us  dist=814
   count=1892      25.942 us  dist=156
   count=6569      25.269 us  dist=425
   count=29        23.935 us  dist=29
   count=5815      21.828 us  dist=329
   count=3563      21.470 us  dist=533
   count=1321      19.724 us  dist=727
   count=9421      17.077 us  dist=819
   count=6905      16.973 us  dist=761
   count=2248      16.972 us  dist=200
  boundary ~2048: c2045=4.25  c2046=3.42  c2047=1.35  c2048=1.44  c2049=15.57  c2050=1.45  c2051=2.68
  boundary ~4096: c4093=4.83  c4094=2.83  c4095=1.31  c4096=1.49  c4097=15.55  c4098=1.36  c4099=2.73
  boundary ~6144: c6141=4.33  c6142=2.73  c6143=1.35  c6144=1.57  c6145=14.21  c6146=1.40  c6147=2.84
  boundary ~8192: c8189=3.88  c8190=2.72  c8191=1.28  c8192=1.51  c8193=15.16  c8194=1.46  c8195=2.68

########## PART B: GC DISABLED (isolates the C subsystem) — single fresh trial ##########
--- GC-OFF: fed 10000 single lines; final count=9977 ---
per-line us: mean=2.650 median=2.589 p99=7.560 max=74.369 min=1.077
12 slowest (count, us, dist-to-nearest-2048-multiple):
   count=8193      74.369 us  dist=1
   count=2049      66.202 us  dist=1
   count=4097      58.242 us  dist=1
   count=6145      51.186 us  dist=1
   count=8194      40.828 us  dist=2
   count=1568      39.195 us  dist=480
   count=6121      27.876 us  dist=23
   count=3469      26.894 us  dist=627
   count=6120      26.306 us  dist=24
   count=4776      25.578 us  dist=680
   count=8204      25.418 us  dist=12
   count=3454      25.204 us  dist=642
  boundary ~2048: c2045=1.32  c2046=1.39  c2047=1.39  c2048=1.29  c2049=66.20  c2050=3.05  c2051=4.30
  boundary ~4096: c4093=1.26  c4094=2.57  c4095=1.23  c4096=1.51  c4097=58.24  c4098=1.40  c4099=2.55
  boundary ~6144: c6141=2.74  c6142=1.30  c6143=1.23  c6144=1.52  c6145=51.19  c6146=2.41  c6147=4.57
  boundary ~8192: c8189=1.32  c8190=2.70  c8191=1.30  c8192=1.50  c8193=74.37  c8194=40.83  c8195=14.30

########## PART C: reproducibility of the boundary spikes (GC OFF, 5 fresh trials) ##########
segment-boundary first-push counts (2048k+1) — us per trial:
   count=2049   -> [59.8, 428.5, 499.4, 415.5, 437.3]
   count=4097   -> [61.5, 433.5, 432.0, 410.7, 422.4]
   count=6145   -> [54.2, 426.2, 419.4, 473.2, 435.6]
   count=8193   -> [50.0, 48.1, 643.9, 424.3, 431.1]
off-boundary count=1918 (originally reported ~27us) — us per trial:
   count=1918   -> [1.26, 1.41, 2.16, 1.45, 1.44]

########## PART D: are the large off-boundary maxima caused by the cyclic GC? ##########
   GC ON : worst OFF-boundary per-line over 5 trials =   1733.3us (count=1753)  gc_collections=21
   GC OFF: worst OFF-boundary per-line over 5 trials =     40.3us (count=720)  gc_collections=0
exit=0
```

**Before / during / after at a boundary (from the `boundary ~2048` rows, GC-OFF, RUN 1).**
- *Before:* `c2045..c2048` are ordinary, ~1.3–1.5 µs each — including the line that fills the boundary
  slot (`c2048 = 1.27`).
- *During (carve):* `c2049 = 63.52 µs` — the first push into the newly needed segment, ~40× a normal
  line. The same spike recurs at every boundary: `c4097 = 54.34`, `c6145 = 49.53`, `c8193 = 41.44`.
- *After:* `c2050`, `c2051`, … immediately return to ~1.4 µs. The stall is a single line wide.

**Reproducibility (Part C).** With GC disabled, the four `2048k+1` counts are the four slowest lines in
the whole burst, and they spike in **every** one of five fresh trials — e.g. `count=2049 →
[51.4, 438.0, 433.5, 410.0, 455.0] µs` (RUN 1). The magnitude **grows** across trials within a process
(≈50 µs on the first fresh trial, ≈410–460 µs later) *(inferred: accumulated heap makes the
`realloc`/`calloc` and first-touch page faults progressively costlier)*. The direction — always the
`+1` line, always reproducible — is the observed, reliable signal.

**The tempting-but-wrong candidate (investigated, not hand-waved).** A prior look reported an
off-boundary "spike" around `count=1918` at ~27 µs. Re-running the exact input shows `count=1918` is
**1.26–2.16 µs in all five trials** (Part C) — it does **not** reproduce; the earlier number was
run-specific noise. More generally, the eye-catching ~1–2 ms maxima in the realistic (GC-on) column are
**the cyclic garbage collector**, not the buffer: Part D pins the worst off-boundary line at
**1815.1 µs / 1733.3 µs with GC on (21 collections)** versus **69.5 µs / 40.3 µs with GC off
(0 collections)**. So the subsystem's own hesitation at a boundary is tens of microseconds; the
millisecond pauses are an artifact of the embedded interpreter's GC running during the burst.

### 6.2 Edge (b): the pager ring's growth to its cap

**Claim.** The pager ring is cheap to append to, except when it must **grow**: `pagerhist_write_bytes`
extends the ring when a write would exceed free space (`kitty/history.c:222-223`), and
`pagerhist_extend` (`kitty/history.c:89-101`) allocates a new ring at least 1 MiB larger
(`kitty/history.c:93-94`) and **copies the whole ring across** (`kitty/history.c:97`). So the cost of the
extending line scales with the current ring size. Once the ring reaches its cap, `pagerhist_extend`
refuses to grow (`kitty/history.c:92`) and writes instead **overwrite the oldest bytes** via
`ringbuf_memcpy_into` (`3rdparty/ringbuf/ringbuf.c:211-238`; tail advance at `ringbuf.c:233`).

**Scenario C2** fills a 4 MiB pager past its cap, timing every line, and — critically — computes **all**
statistics from a **single** sample set so a per-band maximum can never exceed the overall maximum
(a self-consistency check is printed). It runs once with GC disabled (to isolate the C-level copy) and
once with GC enabled (realistic):

```python
# scenarioC2.py — REQ-3 edge (b): pager-ring growth/extend timing to the cap,
# then plateau + overwrite. All statistics computed from ONE sample set.
import time, gc, statistics
from kitty_tests import BaseTest, parse_bytes

CAP = 4 * 1024 * 1024          # 4 MiB raw bytes
PAST_SAT = 100000              # lines fed past saturation
PAYLOAD = "B{:010d} " + "p" * 39   # -> 12 + 39 = 51 visible chars -> 54-byte record

def burst_timed(bt):
    s = bt.create_screen(80, 24, 100, options={'scrollback_pager_history_size': CAP})
    hb = s.historybuf
    parse_bytes(s, ("".join(f"F{i:08d}\r\n" for i in range(100 + 23))).encode())  # count -> 100 == ynum
    samples = []                # (pager_bytes_after, microseconds)
    for i in range(PAST_SAT):
        data = (PAYLOAD.format(i) + "\r\n").encode()
        t0 = time.perf_counter_ns()
        parse_bytes(s, data)
        dt = (time.perf_counter_ns() - t0) / 1000.0
        samples.append((len(hb.pagerhist_as_bytes()), dt))
    return hb, samples

def report(samples, hb, label):
    us = [t for _, t in samples]
    print(f"--- {label}: final pager_bytes={samples[-1][0]} (cap={CAP}) ---")
    overall_max = max(us)
    print(f"per-line us: mean={statistics.mean(us):.3f} median={statistics.median(us):.3f} "
          f"p99={sorted(us)[int(0.99*len(us))]:.3f} max={overall_max:.3f} (min={min(us):.3f})")
    print("10 slowest lines (pager_bytes, MiB, us):")
    for pb, t in sorted(samples, key=lambda x: -x[1])[:10]:
        print(f"   pager_bytes={pb:<9} {pb/1048576:5.3f} MiB  {t:9.3f} us")
    print("max per-line us within each 0.25 MiB band of pager_bytes (from the SAME sample set):")
    band_overall = 0.0
    for b in range(0, 18):
        lo, hi = b * 262144, (b + 1) * 262144
        band = [t for pb, t in samples if lo <= pb < hi]
        if band:
            bm = max(band)
            band_overall = max(band_overall, bm)
            print(f"   [{lo/1048576:4.2f}-{hi/1048576:4.2f} MiB) max={bm:8.3f} us  (n={len(band)})")
    print(f"CONSISTENCY CHECK: max(all band maxima)={band_overall:.3f} us  <=  overall max={overall_max:.3f} us  -> {band_overall <= overall_max}")

class T(BaseTest):
    def run(self):
        print("########## PART 1: GC DISABLED — isolates the C-level ringbuf extend cost ##########")
        gc.disable()
        hb, samples = burst_timed(self)
        report(samples, hb, "GC-OFF, cap=4MiB")
        gc.enable()
        print("\n########## PART 2: GC ENABLED — realistic; shows extra GC pauses at non-extend positions ##########")
        gc.enable()
        ncoll = [0]
        def cb(phase, info, _n=ncoll):
            if phase == 'stop':
                _n[0] += 1
        gc.callbacks.append(cb)
        hb2, samples2 = burst_timed(self)
        gc.callbacks.remove(cb)
        report(samples2, hb2, "GC-ON, cap=4MiB")
        print(f"gc_collections during PART 2 = {ncoll[0]}")

T().run()
```
```text
$ for r in 1 2; do echo "================= RUN $r ================="; \
    ./kitty/launcher/kitty +launch "$WORK/scenarioC2.py"; echo "exit=$?"; done
================= RUN 1 =================
########## PART 1: GC DISABLED — isolates the C-level ringbuf extend cost ##########
--- GC-OFF, cap=4MiB: final pager_bytes=4194304 (cap=4194304) ---
per-line us: mean=5.746 median=4.767 p99=14.412 max=1991.901 (min=1.475)
10 slowest lines (pager_bytes, MiB, us):
   pager_bytes=3145730   3.000 MiB   1991.901 us
   pager_bytes=2097186   2.000 MiB   1101.244 us
   pager_bytes=1048586   1.000 MiB    692.177 us
   pager_bytes=3949554   3.767 MiB    130.620 us
   pager_bytes=3952410   3.769 MiB     59.196 us
   pager_bytes=4194304   4.000 MiB     53.305 us
   pager_bytes=3984106   3.800 MiB     52.694 us
   pager_bytes=4194304   4.000 MiB     52.550 us
   pager_bytes=3951458   3.768 MiB     51.696 us
   pager_bytes=4194304   4.000 MiB     44.444 us
max per-line us within each 0.25 MiB band of pager_bytes (from the SAME sample set):
   [0.00-0.25 MiB) max=  34.279 us  (n=4773)
   [0.25-0.50 MiB) max=  18.991 us  (n=4681)
   [0.50-0.75 MiB) max=  20.015 us  (n=4681)
   [0.75-1.00 MiB) max=  30.077 us  (n=4681)
   [1.00-1.25 MiB) max= 692.177 us  (n=4681)
   [1.25-1.50 MiB) max=  37.449 us  (n=4682)
   [1.50-1.75 MiB) max=  31.966 us  (n=4681)
   [1.75-2.00 MiB) max=  25.996 us  (n=4681)
   [2.00-2.25 MiB) max=1101.244 us  (n=4681)
   [2.25-2.50 MiB) max=  26.610 us  (n=4681)
   [2.50-2.75 MiB) max=  33.040 us  (n=4681)
   [2.75-3.00 MiB) max=  28.420 us  (n=4681)
   [3.00-3.25 MiB) max=1991.901 us  (n=4682)
   [3.25-3.50 MiB) max=  25.267 us  (n=4681)
   [3.50-3.75 MiB) max=  30.505 us  (n=4681)
   [3.75-4.00 MiB) max= 130.620 us  (n=4681)
   [4.00-4.25 MiB) max=  53.305 us  (n=25010)
CONSISTENCY CHECK: max(all band maxima)=1991.901 us  <=  overall max=1991.901 us  -> True

########## PART 2: GC ENABLED — realistic; shows extra GC pauses at non-extend positions ##########
--- GC-ON, cap=4MiB: final pager_bytes=4194304 (cap=4194304) ---
per-line us: mean=5.300 median=4.745 p99=13.432 max=3000.578 (min=1.484)
10 slowest lines (pager_bytes, MiB, us):
   pager_bytes=766290    0.731 MiB   3000.578 us
   pager_bytes=3145730   3.000 MiB    209.800 us
   pager_bytes=3691618   3.521 MiB    163.983 us
   pager_bytes=2097186   2.000 MiB    138.583 us
   pager_bytes=1048586   1.000 MiB     71.859 us
   pager_bytes=4194304   4.000 MiB     61.440 us
   pager_bytes=4194304   4.000 MiB     54.410 us
   pager_bytes=4194304   4.000 MiB     51.375 us
   pager_bytes=3878154   3.698 MiB     44.142 us
   pager_bytes=95354     0.091 MiB     41.402 us
max per-line us within each 0.25 MiB band of pager_bytes (from the SAME sample set):
   [0.00-0.25 MiB) max=  41.402 us  (n=4773)
   [0.25-0.50 MiB) max=  15.919 us  (n=4681)
   [0.50-0.75 MiB) max=3000.578 us  (n=4681)
   [0.75-1.00 MiB) max=  24.190 us  (n=4681)
   [1.00-1.25 MiB) max=  71.859 us  (n=4681)
   [1.25-1.50 MiB) max=  22.039 us  (n=4682)
   [1.50-1.75 MiB) max=  24.804 us  (n=4681)
   [1.75-2.00 MiB) max=  23.851 us  (n=4681)
   [2.00-2.25 MiB) max= 138.583 us  (n=4681)
   [2.25-2.50 MiB) max=  31.715 us  (n=4681)
   [2.50-2.75 MiB) max=  31.544 us  (n=4681)
   [2.75-3.00 MiB) max=  29.459 us  (n=4681)
   [3.00-3.25 MiB) max= 209.800 us  (n=4682)
   [3.25-3.50 MiB) max=  29.119 us  (n=4681)
   [3.50-3.75 MiB) max= 163.983 us  (n=4681)
   [3.75-4.00 MiB) max=  26.737 us  (n=4681)
   [4.00-4.25 MiB) max=  61.440 us  (n=25010)
CONSISTENCY CHECK: max(all band maxima)=3000.578 us  <=  overall max=3000.578 us  -> True
gc_collections during PART 2 = 51
exit=0
================= RUN 2 =================
########## PART 1: GC DISABLED — isolates the C-level ringbuf extend cost ##########
--- GC-OFF, cap=4MiB: final pager_bytes=4194304 (cap=4194304) ---
per-line us: mean=4.822 median=4.272 p99=12.089 max=1588.492 (min=1.511)
10 slowest lines (pager_bytes, MiB, us):
   pager_bytes=3145730   3.000 MiB   1588.492 us
   pager_bytes=2097186   2.000 MiB   1211.246 us
   pager_bytes=1048586   1.000 MiB    583.421 us
   pager_bytes=1666938   1.590 MiB     55.557 us
   pager_bytes=4194304   4.000 MiB     41.329 us
   pager_bytes=2916634   2.782 MiB     40.302 us
   pager_bytes=4194304   4.000 MiB     35.714 us
   pager_bytes=4153338   3.961 MiB     35.518 us
   pager_bytes=4194304   4.000 MiB     32.874 us
   pager_bytes=3962994   3.779 MiB     29.473 us
max per-line us within each 0.25 MiB band of pager_bytes (from the SAME sample set):
   [0.00-0.25 MiB) max=  21.288 us  (n=4773)
   [0.25-0.50 MiB) max=  24.864 us  (n=4681)
   [0.50-0.75 MiB) max=  20.268 us  (n=4681)
   [0.75-1.00 MiB) max=  19.435 us  (n=4681)
   [1.00-1.25 MiB) max= 583.421 us  (n=4681)
   [1.25-1.50 MiB) max=  25.359 us  (n=4682)
   [1.50-1.75 MiB) max=  55.557 us  (n=4681)
   [1.75-2.00 MiB) max=  28.862 us  (n=4681)
   [2.00-2.25 MiB) max=1211.246 us  (n=4681)
   [2.25-2.50 MiB) max=  25.762 us  (n=4681)
   [2.50-2.75 MiB) max=  25.430 us  (n=4681)
   [2.75-3.00 MiB) max=  40.302 us  (n=4681)
   [3.00-3.25 MiB) max=1588.492 us  (n=4682)
   [3.25-3.50 MiB) max=  22.208 us  (n=4681)
   [3.50-3.75 MiB) max=  21.892 us  (n=4681)
   [3.75-4.00 MiB) max=  35.518 us  (n=4681)
   [4.00-4.25 MiB) max=  41.329 us  (n=25010)
CONSISTENCY CHECK: max(all band maxima)=1588.492 us  <=  overall max=1588.492 us  -> True

########## PART 2: GC ENABLED — realistic; shows extra GC pauses at non-extend positions ##########
--- GC-ON, cap=4MiB: final pager_bytes=4194304 (cap=4194304) ---
per-line us: mean=4.848 median=4.312 p99=12.856 max=2686.990 (min=1.492)
10 slowest lines (pager_bytes, MiB, us):
   pager_bytes=766290    0.731 MiB   2686.990 us
   pager_bytes=3145730   3.000 MiB    199.343 us
   pager_bytes=2097186   2.000 MiB    142.102 us
   pager_bytes=1048586   1.000 MiB     63.153 us
   pager_bytes=95354     0.091 MiB     56.903 us
   pager_bytes=3901338   3.721 MiB     40.224 us
   pager_bytes=4194304   4.000 MiB     36.298 us
   pager_bytes=4194304   4.000 MiB     35.729 us
   pager_bytes=2823226   2.692 MiB     32.599 us
   pager_bytes=4194304   4.000 MiB     32.043 us
max per-line us within each 0.25 MiB band of pager_bytes (from the SAME sample set):
   [0.00-0.25 MiB) max=  56.903 us  (n=4773)
   [0.25-0.50 MiB) max=  17.987 us  (n=4681)
   [0.50-0.75 MiB) max=2686.990 us  (n=4681)
   [0.75-1.00 MiB) max=  22.584 us  (n=4681)
   [1.00-1.25 MiB) max=  63.153 us  (n=4681)
   [1.25-1.50 MiB) max=  24.724 us  (n=4682)
   [1.50-1.75 MiB) max=  21.530 us  (n=4681)
   [1.75-2.00 MiB) max=  20.645 us  (n=4681)
   [2.00-2.25 MiB) max= 142.102 us  (n=4681)
   [2.25-2.50 MiB) max=  24.960 us  (n=4681)
   [2.50-2.75 MiB) max=  32.599 us  (n=4681)
   [2.75-3.00 MiB) max=  30.024 us  (n=4681)
   [3.00-3.25 MiB) max= 199.343 us  (n=4682)
   [3.25-3.50 MiB) max=  31.940 us  (n=4681)
   [3.50-3.75 MiB) max=  40.224 us  (n=4681)
   [3.75-4.00 MiB) max=  27.355 us  (n=4681)
   [4.00-4.25 MiB) max=  36.298 us  (n=25010)
CONSISTENCY CHECK: max(all band maxima)=2686.990 us  <=  overall max=2686.990 us  -> True
gc_collections during PART 2 = 51
exit=0
```

**Before / during / after (GC-OFF, the clean C-level view).**
- *Before any extend:* per-line cost is ~2–5 µs (median 4.27–4.77 µs).
- *During each extend:* the three slowest lines land **exactly** where the *used* byte count
  (`pager_bytes` — the only pager quantity Python exposes, §2.5) crosses each ~1 MiB mark, at positions
  **identical across all runs** — `pager_bytes = 1048586` (1.000 MiB), `2097186` (2.000 MiB),
  `3145730` (3.000 MiB). These positions mark the ring's *capacity* extends (`pagerhist_extend`,
  `history.c:89-101` — the capacity growth itself is *inferred* from source, since only used bytes are
  observable), and the cost **grows with ring size copied**: ≈583–692 µs (copy ~1 MiB) → ≈1101–1211 µs
  (~2 MiB) → ≈1588–1992 µs (~3 MiB). The per-band maxima confirm this: the only bands with large maxima
  are `[1.00-1.25)`, `[2.00-2.25)`, `[3.00-3.25) MiB`.
- *After the cap:* beyond 3 MiB the ring is at its 4 MiB cap; the `[4.00-4.25 MiB)` band holds ~25,010
  samples with a maximum of only ~40–53 µs — steady **overwrite-oldest** with no further growth.

**Direct content evidence — the plateau really is overwrite-oldest (Scenario F).** The timing plateau
above shows the ring *stops growing*, but not that its *oldest bytes are discarded*. Scenario F proves
the latter directly: it sets a tiny **4096-byte** pager cap and feeds **unique, fixed-width** tokens
(`T000000`, `T000001`, … — each a 12-byte pager record: `ESC[m` + 7 chars + `CRLF`) through the same
canonical `parse_bytes` path, then reads `pagerhist_as_bytes()` before the cap, at the cap, and well
past it. Because every token is distinct, a specific record's disappearance is unambiguous. With a
4096-byte configured size the ring is at its cap from construction (`MIN(1 MiB, 4096) = 4096`,
`kitty/history.c:66-67`), so `pagerhist_extend` refuses to grow from the first eviction
(`kitty/history.c:92`) and every push overwrites:

```python
# scenarioF.py — REQ-2/3/5 DIRECT runtime evidence of pager-ring overwrite-oldest at
# the cap. A small 4096-byte pager cap plus unique fixed-width tokens lets us watch
# specific records survive, then disappear as the full ring overwrites its oldest bytes.
# Canonical ingest only: create_screen + parse_bytes (real VT parser -> screen -> history).
import re
from kitty_tests import BaseTest, parse_bytes

CAP = 4096                      # pager cap in BYTES (raw harness units, see doc s2.4)
COLS, LINES, YNUM = 80, 24, 100

def feed_tokens(s, first, n):
    # token 'T%06d' -> a 7-char history line; serialized pager record =
    # ESC[m (3) + 'T000000' (7) + CRLF (2) = 12 bytes
    parse_bytes(s, ("".join(f"T{first+i:06d}\r\n" for i in range(n))).encode())

def token_range(b):
    toks = sorted(int(m) for m in re.findall(rb'T(\d{6})', b))
    return (toks[0], toks[-1], len(toks)) if toks else (None, None, 0)

class T(BaseTest):
    def run(self):
        s = self.create_screen(COLS, LINES, YNUM,
                               options={'scrollback_pager_history_size': CAP})
        hb = s.historybuf
        # ---- saturate the segmented store so the NEXT pushed line begins evicting ----
        feed_tokens(s, 0, YNUM + (LINES - 1))     # first LINES-1 fill screen; YNUM -> history
        fed = YNUM + (LINES - 1)
        FIRST = 0                                 # T000000 is the oldest line, first to be evicted
        print(f"config: pager cap={CAP} B  ynum={hb.ynum}  xnum={hb.xnum}")
        print(f"after fill: count={hb.count} (==ynum? {hb.count==hb.ynum})  "
              f"pager_bytes={len(hb.pagerhist_as_bytes())}  (no eviction yet)")

        def snap(label):
            b = hb.pagerhist_as_bytes()
            lo, hi, n = token_range(b)
            print(f"  [{label}] pager_bytes={len(b):>5}  intact_records={n:>4}  "
                  f"oldest_intact=T{lo:06d}  newest=T{hi:06d}  "
                  f"first-evicted T{FIRST:06d} present? {(f'T{FIRST:06d}'.encode() in b)}")
            print(f"       head 40B = {b[:40]!r}")
            print(f"       tail 40B = {b[-40:]!r}")

        # ---- PRE-CAP: evict ~200 records; 200*12=2400 B < 4096 so nothing overwritten ----
        feed_tokens(s, fed, 200); fed += 200
        snap("PRE-CAP ")
        # ---- AT-CAP: evict one-at-a-time until used bytes reach the ring capacity ----
        while len(hb.pagerhist_as_bytes()) < CAP - 12:
            feed_tokens(s, fed, 1); fed += 1
        snap("AT-CAP  ")
        # ---- POST-CAP: keep evicting far past the cap ----
        feed_tokens(s, fed, 600); fed += 600
        snap("POST-CAP")

        b = hb.pagerhist_as_bytes()
        print(f"  RESULT: first-ever-evicted token T{FIRST:06d} still present? "
              f"{(f'T{FIRST:06d}'.encode() in b)}  (expect False: overwritten)")
        print(f"  RESULT: pager plateaued at {len(b)} B (never exceeds the {CAP} B cap)")
        print(f"  RESULT: head begins MID-record (no leading ESC[m SGR reset): first3={b[:3]!r} "
              f"(an intact record starts with b'\\x1b[m')")

T().run()
```

```text
$ for r in 1 2; do echo "================= RUN $r ================="; \
    ./kitty/launcher/kitty +launch "$WORK/scenarioF.py"; echo "exit=$?"; done
================= RUN 1 =================
config: pager cap=4096 B  ynum=100  xnum=80
after fill: count=100 (==ynum? True)  pager_bytes=0  (no eviction yet)
  [PRE-CAP ] pager_bytes= 2400  intact_records= 200  oldest_intact=T000000  newest=T000199  first-evicted T000000 present? True
       head 40B = b'\x1b[mT000000\r\n\x1b[mT000001\r\n\x1b[mT000002\r\n\x1b[mT'
       tail 40B = b'96\r\n\x1b[mT000197\r\n\x1b[mT000198\r\n\x1b[mT000199\r\n'
  [AT-CAP  ] pager_bytes= 4092  intact_records= 341  oldest_intact=T000000  newest=T000340  first-evicted T000000 present? True
       head 40B = b'\x1b[mT000000\r\n\x1b[mT000001\r\n\x1b[mT000002\r\n\x1b[mT'
       tail 40B = b'37\r\n\x1b[mT000338\r\n\x1b[mT000339\r\n\x1b[mT000340\r\n'
  [POST-CAP] pager_bytes= 4096  intact_records= 341  oldest_intact=T000600  newest=T000940  first-evicted T000000 present? False
       head 40B = b'99\r\n\x1b[mT000600\r\n\x1b[mT000601\r\n\x1b[mT000602\r\n'
       tail 40B = b'37\r\n\x1b[mT000938\r\n\x1b[mT000939\r\n\x1b[mT000940\r\n'
  RESULT: first-ever-evicted token T000000 still present? False  (expect False: overwritten)
  RESULT: pager plateaued at 4096 B (never exceeds the 4096 B cap)
  RESULT: head begins MID-record (no leading ESC[m SGR reset): first3=b'99\r' (an intact record starts with b'\x1b[m')
exit=0
================= RUN 2 =================
config: pager cap=4096 B  ynum=100  xnum=80
after fill: count=100 (==ynum? True)  pager_bytes=0  (no eviction yet)
  [PRE-CAP ] pager_bytes= 2400  intact_records= 200  oldest_intact=T000000  newest=T000199  first-evicted T000000 present? True
       head 40B = b'\x1b[mT000000\r\n\x1b[mT000001\r\n\x1b[mT000002\r\n\x1b[mT'
       tail 40B = b'96\r\n\x1b[mT000197\r\n\x1b[mT000198\r\n\x1b[mT000199\r\n'
  [AT-CAP  ] pager_bytes= 4092  intact_records= 341  oldest_intact=T000000  newest=T000340  first-evicted T000000 present? True
       head 40B = b'\x1b[mT000000\r\n\x1b[mT000001\r\n\x1b[mT000002\r\n\x1b[mT'
       tail 40B = b'37\r\n\x1b[mT000338\r\n\x1b[mT000339\r\n\x1b[mT000340\r\n'
  [POST-CAP] pager_bytes= 4096  intact_records= 341  oldest_intact=T000600  newest=T000940  first-evicted T000000 present? False
       head 40B = b'99\r\n\x1b[mT000600\r\n\x1b[mT000601\r\n\x1b[mT000602\r\n'
       tail 40B = b'37\r\n\x1b[mT000938\r\n\x1b[mT000939\r\n\x1b[mT000940\r\n'
  RESULT: first-ever-evicted token T000000 still present? False  (expect False: overwritten)
  RESULT: pager plateaued at 4096 B (never exceeds the 4096 B cap)
  RESULT: head begins MID-record (no leading ESC[m SGR reset): first3=b'99\r' (an intact record starts with b'\x1b[m')
exit=0
```

**Before / during / after (content).**
- *Before the cap* (`[PRE-CAP]`, 2400 B < 4096): 200 intact records; `oldest_intact = T000000` — the
  first-ever-evicted token — is **present**, and the head is a clean full record
  (`b'\x1b[mT000000\r\n…'`). Nothing discarded yet.
- *At the cap* (`[AT-CAP]`, 4092 B ≈ 4096): 341 records; `oldest_intact` is **still** `T000000` — the
  ring is full but the oldest survivor has not yet been overwritten.
- *After the cap* (`[POST-CAP]`, exactly 4096 B): the ring **plateaus at the 4096-byte cap and never
  exceeds it**; `oldest_intact` jumps forward to `T000600` and `T000000` is **gone** (`present? False`).
  The head now begins **mid-record** — `b'99\r\n\x1b[mT000600…'`, the trailing `99\r\n` of a
  half-overwritten token rather than a leading `ESC[m` — the signature of a **byte-level** overwrite
  (`ringbuf_memcpy_into` advancing the tail into the middle of the oldest record,
  `3rdparty/ringbuf/ringbuf.c:233`), not a line-atomic drop.

Both runs are byte-identical, so the disappearance of `T000000` and the mid-record head are stable, not
sampling artifacts. This is the **direct** confirmation of the overwrite-oldest behavior that the timing
plateau only implied.

Each run prints `CONSISTENCY CHECK: max(all band maxima) == overall max -> True`, i.e. the reported
per-band maxima are internally consistent with the overall maximum (they are computed from the same
sample set, not mixed across runs).

**The GC again (GC-ON column).** With GC enabled the single largest pause is **not** an extend: it sits
at a non-extend position `pager_bytes = 766290` (0.731 MiB) at ≈2687–3001 µs, deterministic in both
runs, with `gc_collections = 51`. With GC **off**, that same 0.50–0.75 MiB band's maximum is only
~19–20 µs. This is the same lesson as §6.1: a reproducible **off-extend** spike in the realistic column
is the cyclic GC, whereas the genuine ring-growth hesitation is the 1/2/3 MiB copy. (This also explains a
previously puzzling ~0.77 MiB "event": it is a GC pause at a deterministic allocation count, not a ring
operation.)

**Net answer to REQ-3.** Transitions are smooth except at two edges, both of which the buffer *does*
hesitate at, briefly and reproducibly: the per-segment `calloc` at each 2048-line boundary (tens of µs,
scaling with heap pressure), and the pager ring's `≥1 MiB` copy at each extend (hundreds of µs to ~2 ms,
scaling with ring size), after which the ring plateaus into constant-time overwrite at its cap. The much
larger, sporadic pauses one might mistake for the data structure are the interpreter's GC.

---

## 7. REQ-4 — Scrolling through old output while new data floods in

**Mechanism (cause → effect).** The scroll position is `Screen.scrolled_by` (0 = at the bottom).
Scrolling up is clamped so you cannot scroll past the oldest retained line: `screen_history_scroll`
sets `new_scroll = MIN(scrolled_by + amt, count)` (`kitty/screen.c:4091`, clamp at
`kitty/screen.c:4111`). Pure ingest does **not** move `scrolled_by`; each added history line only
increments `history_line_added_count` (`kitty/screen.c:1559`). The re-anchoring happens at **draw
time**: both the cell-data path (`kitty/screen.c:2761`) and the graphics-only path
(`kitty/screen.c:2716`) apply, when `scrolled_by` is nonzero,

```
scrolled_by = MIN(scrolled_by + history_line_added_count, count)
```

and then reset the counter via `screen_reset_dirty` (`kitty/screen.c:2600`). Effect: on each rendered
frame the viewport is bumped up by exactly the number of lines added since the last frame, so the same
old content stays under your eyes — **until** the sum would exceed `count`, where it clamps.

**Scenario D** reproduces "actively scrolling while new data arrives." Because a real terminal redraws
every frame (zeroing the counter each frame), the script renders once after setup to mimic continuous
drawing, then reports `scrolled_by`, `history_line_added_count` (`hlac`), and `count` **before /
during / after**:

```python
# scenarioD.py — REQ-4: concurrent scroll + ingest. A real terminal redraws every
# frame, which resets history_line_added_count (screen_reset_dirty, screen.c:L2600).
# We therefore render once after setup to zero the counter, mimicking continuous
# frame drawing, so each interval's anchor delta is isolated. Anchor formula:
#   scrolled_by = MIN(scrolled_by + history_line_added_count, count)  (screen.c:L2716)
from kitty_tests import BaseTest, parse_bytes

def feed(s, n):
    parse_bytes(s, ("".join(f"L{i:08d}\r\n" for i in range(n))).encode())

def render(s):                                   # one frame: applies the anchor, resets counter
    s.update_only_line_graphics_data()

def show(s, label):
    print(f"  {label:34s} scrolled_by={s.scrolled_by:<6} "
          f"hlac={s.history_line_added_count:<6} count={s.historybuf.count}")

class T(BaseTest):
    def run(self):
        print("##### PART 1: per-frame anchor, scrolled up, NO eviction (scrollback huge) #####")
        s = self.create_screen(80, 24, 100000, options={'scrollback_pager_history_size': 0})
        feed(s, 3000); render(s)                 # continuous drawing zeroes the counter
        show(s, "after 3000 lines + 1 frame")
        s.scroll(500, True)
        show(s, "BEFORE: scrolled up by 500")
        feed(s, 1000)
        show(s, "DURING ingest (before next frame)")
        render(s)
        show(s, "AFTER frame (anchored)")
        print(f"  -> scrolled_by = MIN(500+1000, count={s.historybuf.count}) = {min(1500, s.historybuf.count)}")

        print("\n##### PART 2: view tracks old output across many frames, then CLAMPS (scrollback=2000) #####")
        s2 = self.create_screen(80, 24, 2000, options={'scrollback_pager_history_size': 0})
        feed(s2, 2100); render(s2)               # count saturated at ynum=2000
        show(s2, "after 2100 lines + 1 frame")
        s2.scroll(1900, True)
        show(s2, "BEFORE: scrolled up by 1900")
        for f in range(1, 5):                    # 4 frames, 200 new lines each, at full speed
            feed(s2, 200)
            render(s2)
            show(s2, f"AFTER frame {f} (+200 lines)")
        print(f"  -> anchor rises 1900->2000 then CLAMPS at count={s2.historybuf.count}; "
              f"once clamped the oldest viewed lines are evicted out from under the viewport")

        print("\n##### PART 3: WITHOUT redraw, scrolled_by is frozen; one late frame folds in everything #####")
        s3 = self.create_screen(80, 24, 100000, options={'scrollback_pager_history_size': 0})
        feed(s3, 1000); render(s3); s3.scroll(200, True)
        show(s3, "BEFORE: scrolled up by 200")
        for k in range(3):
            feed(s3, 300)
            show(s3, f"DURING ingest burst #{k+1} (no frame)")
        render(s3)
        show(s3, "AFTER single late frame")
        print(f"  -> one frame folds in ALL 900 accumulated lines: MIN(200+900, count) "
              f"= {min(1100, s3.historybuf.count)}")

T().run()
```
```text
$ for r in 1 2; do echo "================= RUN $r ================="; \
    ./kitty/launcher/kitty +launch "$WORK/scenarioD.py"; echo "exit=$?"; done
================= RUN 1 =================
##### PART 1: per-frame anchor, scrolled up, NO eviction (scrollback huge) #####
  after 3000 lines + 1 frame         scrolled_by=0      hlac=0      count=2977
  BEFORE: scrolled up by 500         scrolled_by=500    hlac=0      count=2977
  DURING ingest (before next frame)  scrolled_by=500    hlac=1000   count=3977
  AFTER frame (anchored)             scrolled_by=1500   hlac=0      count=3977
  -> scrolled_by = MIN(500+1000, count=3977) = 1500

##### PART 2: view tracks old output across many frames, then CLAMPS (scrollback=2000) #####
  after 2100 lines + 1 frame         scrolled_by=0      hlac=0      count=2000
  BEFORE: scrolled up by 1900        scrolled_by=1900   hlac=0      count=2000
  AFTER frame 1 (+200 lines)         scrolled_by=2000   hlac=0      count=2000
  AFTER frame 2 (+200 lines)         scrolled_by=2000   hlac=0      count=2000
  AFTER frame 3 (+200 lines)         scrolled_by=2000   hlac=0      count=2000
  AFTER frame 4 (+200 lines)         scrolled_by=2000   hlac=0      count=2000
  -> anchor rises 1900->2000 then CLAMPS at count=2000; once clamped the oldest viewed lines are evicted out from under the viewport

##### PART 3: WITHOUT redraw, scrolled_by is frozen; one late frame folds in everything #####
  BEFORE: scrolled up by 200         scrolled_by=200    hlac=0      count=977
  DURING ingest burst #1 (no frame)  scrolled_by=200    hlac=300    count=1277
  DURING ingest burst #2 (no frame)  scrolled_by=200    hlac=600    count=1577
  DURING ingest burst #3 (no frame)  scrolled_by=200    hlac=900    count=1877
  AFTER single late frame            scrolled_by=1100   hlac=0      count=1877
  -> one frame folds in ALL 900 accumulated lines: MIN(200+900, count) = 1100
exit=0
================= RUN 2 =================
##### PART 1: per-frame anchor, scrolled up, NO eviction (scrollback huge) #####
  after 3000 lines + 1 frame         scrolled_by=0      hlac=0      count=2977
  BEFORE: scrolled up by 500         scrolled_by=500    hlac=0      count=2977
  DURING ingest (before next frame)  scrolled_by=500    hlac=1000   count=3977
  AFTER frame (anchored)             scrolled_by=1500   hlac=0      count=3977
  -> scrolled_by = MIN(500+1000, count=3977) = 1500

##### PART 2: view tracks old output across many frames, then CLAMPS (scrollback=2000) #####
  after 2100 lines + 1 frame         scrolled_by=0      hlac=0      count=2000
  BEFORE: scrolled up by 1900        scrolled_by=1900   hlac=0      count=2000
  AFTER frame 1 (+200 lines)         scrolled_by=2000   hlac=0      count=2000
  AFTER frame 2 (+200 lines)         scrolled_by=2000   hlac=0      count=2000
  AFTER frame 3 (+200 lines)         scrolled_by=2000   hlac=0      count=2000
  AFTER frame 4 (+200 lines)         scrolled_by=2000   hlac=0      count=2000
  -> anchor rises 1900->2000 then CLAMPS at count=2000; once clamped the oldest viewed lines are evicted out from under the viewport

##### PART 3: WITHOUT redraw, scrolled_by is frozen; one late frame folds in everything #####
  BEFORE: scrolled up by 200         scrolled_by=200    hlac=0      count=977
  DURING ingest burst #1 (no frame)  scrolled_by=200    hlac=300    count=1277
  DURING ingest burst #2 (no frame)  scrolled_by=200    hlac=600    count=1577
  DURING ingest burst #3 (no frame)  scrolled_by=200    hlac=900    count=1877
  AFTER single late frame            scrolled_by=1100   hlac=0      count=1877
  -> one frame folds in ALL 900 accumulated lines: MIN(200+900, count) = 1100
exit=0
```

**Before / during / after (Part 1, no eviction).**
- *Before:* user scrolls up → `scrolled_by=500, hlac=0, count=2977`.
- *During ingest, before the next frame:* 1000 lines arrive; `scrolled_by` is **frozen at 500** while
  `hlac` rises to 1000 and `count` to 3977. The viewport has not moved yet.
- *After the frame:* `scrolled_by = MIN(500 + 1000, 3977) = 1500`, `hlac` resets to 0. The view jumped
  up by exactly the 1000 newly added lines, keeping the same old content in place.

**The boundary / hesitation (Part 2, with eviction).** With `scrollback=2000` and `count` saturated at
`ynum=2000`, the user scrolls up near the top (`scrolled_by=1900`). Now each frame's anchor target
(`1900 + 200 = 2100`, then more) is **clamped to `count=2000`**: after frame 1 `scrolled_by=2000`, and
it stays `2000` through frames 2–4. This is the concurrent-scroll edge — once the anchor hits `count`,
it can rise no further, and because old lines are being evicted at the top, the content the user was
viewing scrolls out from under the viewport (retention, not anchoring, is now the limit).

**Frozen-view corollary (Part 3).** Without an intervening redraw, `scrolled_by` stays fixed while
`hlac` accumulates across multiple ingest bursts (300 → 600 → 900); a single later frame folds all of
them in at once: `scrolled_by = MIN(200 + 900, 1877) = 1100`. So the anchor is applied lazily, per
frame, in proportion to lines added since the previous frame. All Part-1/2/3 values were identical
across both runs.

---

## 8. REQ-5 — Allocation, wrapping, and retention

**Scenario E** captures all three at once: a two-regime footprint measurement, a wrapping
before/during/after with logical-line grouping, and a retention contrast (pager off vs on):

```python
# scenarioE.py — REQ-5: allocation, wrapping, retention (before/during/after).
from kitty_tests import BaseTest, parse_bytes

LINES = 24                         # screen rows; first (LINES-1) fed lines fill the screen
                                   # before INDEX_UP begins pushing lines into history

def rss_kb():
    with open("/proc/self/status") as f:
        for ln in f:
            if ln.startswith("VmRSS:"):
                return int(ln.split()[1])
    return -1

def feed(s, n, tag="L"):
    parse_bytes(s, ("".join(f"{tag}{i:08d}\r\n" for i in range(n))).encode())

def segs(count, ynum):
    eff = min(count, ynum)
    return (eff + 2047) // 2048

class T(BaseTest):
    def run(self):
        YNUM = 20000
        print("########## PART 1: FOOTPRINT = segmented store + pager ring, as pressure builds ##########")
        print(f"(scrollback={YNUM}, pager cap=64 MiB so it never caps during this run)")
        s = self.create_screen(80, LINES, YNUM, options={'scrollback_pager_history_size': 64 * 1024 * 1024})
        hb = s.historybuf
        base = rss_kb()
        print(f"  {'lines_fed':>10} {'count':>6} {'segments':>8} {'pager_bytes':>12} {'RSS_kB':>9} {'dRSS_kB':>9}")
        fed = 0
        # fill phase: drive `count` to each round target exactly by compensating for the
        # (LINES-1) lines that fill the screen before any line is pushed into history.
        for target in (2048, 4096, 8192, 16384, YNUM):
            need = (target + (LINES - 1)) - fed
            feed(s, need); fed += need
            print(f"  {fed:>10} {hb.count:>6} {segs(hb.count,YNUM):>8} {len(hb.pagerhist_as_bytes()):>12} {rss_kb():>9} {rss_kb()-base:>9}")
        print(f"  -- saturation reached (count==ynum? {hb.count==hb.ynum}; segment count fixed); now ONLY the pager grows --")
        for extra in (10000, 20000, 40000):
            feed(s, extra)
            fed += extra
            print(f"  {fed:>10} {hb.count:>6} {segs(hb.count,YNUM):>8} {len(hb.pagerhist_as_bytes()):>12} {rss_kb():>9} {rss_kb()-base:>9}")
        print("  NOTE: RSS is a process-wide figure (interpreter + arenas + buffers); it corroborates,")
        print("        but does not exclusively measure, the two HistoryBuf structures.")

        print("\n########## PART 2: WRAPPING — a wide logical line occupies MULTIPLE physical rows ##########")
        w = self.create_screen(80, 24, 100000, options={'scrollback_pager_history_size': 0})
        hw = w.historybuf
        print(f"  xnum(cols)={hw.xnum}")
        print(f"  BEFORE: history count={hw.count} (empty); cursor.y={w.cursor.y}")
        parse_bytes(w, ("W" * 200).encode())     # 200 chars, no newline -> autowrap on-screen
        print(f"  DURING: fed one 200-char logical line (no newline). "
              f"ceil(200/80)=3 rows -> cursor.y={w.cursor.y} (advanced by 2 wraps); history count={hw.count} (still on-screen)")
        parse_bytes(w, ("\r\n" + "".join(f"s{i:03d}\r\n" for i in range(40))).encode())  # push into history
        print(f"  AFTER : pushed into history; count={hw.count}")
        print("  grouping of the wide line in history (oldest rows; idx 0 = newest):")
        for idx in range(hw.count - 3, hw.count):
            ln = hw.line(idx)
            role = "continues->" if ln.last_char_has_wrapped_flag() else "END of logical line"
            print(f"     line({idx}) wrapped_last={ln.last_char_has_wrapped_flag()!s:5} [{role:19}] text={ln.as_ansi()[:16]!r}")
        print("  -> a 200-col logical line = 3 physical history rows; the first two carry the")
        print("     next_char_was_wrapped continuation flag (screen.c:L521-527, line.c:L426-431).")
        # non-wrapped accounting: 3000 short lines -> count 2977
        n = self.create_screen(80, 24, 100000, options={'scrollback_pager_history_size': 0})
        feed(n, 3000)
        print(f"  non-wrapped accounting: fed 3000 short lines -> history count={n.historybuf.count} "
              f"(= 3000 - (lines-1=23); first 23 fill the screen before INDEX_UP pushes any line)")

        print("\n########## PART 3: RETENTION — pager OFF (lost) vs ON (retained until cap) ##########")
        off = self.create_screen(80, 24, 2000, options={'scrollback_pager_history_size': 0})
        feed(off, 4000)
        oldest_off = off.historybuf.line(off.historybuf.count - 1).as_ansi()[:12]
        print(f"  pager OFF: fed 4000 lines -> count={off.historybuf.count} (saturated at ynum=2000), "
              f"pager_bytes={len(off.historybuf.pagerhist_as_bytes())}")
        print(f"             oldest RETAINED history line = {oldest_off!r} (L00000000..~L00001976 were EVICTED and LOST)")
        on = self.create_screen(80, 24, 2000, options={'scrollback_pager_history_size': 64 * 1024 * 1024})
        feed(on, 4000)
        pht = on.historybuf.pagerhist_as_text()
        print(f"  pager ON : fed 4000 lines -> count={on.historybuf.count} (same saturation), "
              f"pager_bytes={len(on.historybuf.pagerhist_as_bytes())}")
        print(f"             pager text length={len(pht)} chars; earliest evicted content preserved? "
              f"{'L00000000' in pht} (first evicted line still in the ring)")

T().run()
```
```text
$ for r in 1 2; do echo "================= RUN $r ================="; \
    ./kitty/launcher/kitty +launch "$WORK/scenarioE.py"; echo "exit=$?"; done
================= RUN 1 =================
########## PART 1: FOOTPRINT = segmented store + pager ring, as pressure builds ##########
(scrollback=20000, pager cap=64 MiB so it never caps during this run)
   lines_fed  count segments  pager_bytes    RSS_kB   dRSS_kB
        2071   2048        1            0     31760      5604
        4119   4096        2            0     36892     10736
        8215   8192        4            0     47372     21216
       16407  16384        8            0     68396     42240
       20023  20000       10            0     77460     51304
  -- saturation reached (count==ynum? True; segment count fixed); now ONLY the pager grows --
       30023  20000       10       140000     77800     51644
       50023  20000       10       420000     79200     53044
       90023  20000       10       980000     81076     54920
  NOTE: RSS is a process-wide figure (interpreter + arenas + buffers); it corroborates,
        but does not exclusively measure, the two HistoryBuf structures.

########## PART 2: WRAPPING — a wide logical line occupies MULTIPLE physical rows ##########
  xnum(cols)=80
  BEFORE: history count=0 (empty); cursor.y=0
  DURING: fed one 200-char logical line (no newline). ceil(200/80)=3 rows -> cursor.y=2 (advanced by 2 wraps); history count=0 (still on-screen)
  AFTER : pushed into history; count=20
  grouping of the wide line in history (oldest rows; idx 0 = newest):
     line(17) wrapped_last=False [END of logical line] text='WWWWWWWWWWWWWWWW'
     line(18) wrapped_last=True  [continues->        ] text='WWWWWWWWWWWWWWWW'
     line(19) wrapped_last=True  [continues->        ] text='WWWWWWWWWWWWWWWW'
  -> a 200-col logical line = 3 physical history rows; the first two carry the
     next_char_was_wrapped continuation flag (screen.c:L521-527, line.c:L426-431).
  non-wrapped accounting: fed 3000 short lines -> history count=2977 (= 3000 - (lines-1=23); first 23 fill the screen before INDEX_UP pushes any line)

########## PART 3: RETENTION — pager OFF (lost) vs ON (retained until cap) ##########
  pager OFF: fed 4000 lines -> count=2000 (saturated at ynum=2000), pager_bytes=0
             oldest RETAINED history line = 'L00001977' (L00000000..~L00001976 were EVICTED and LOST)
  pager ON : fed 4000 lines -> count=2000 (same saturation), pager_bytes=27678
             pager text length=27678 chars; earliest evicted content preserved? True (first evicted line still in the ring)
exit=0
================= RUN 2 =================
########## PART 1: FOOTPRINT = segmented store + pager ring, as pressure builds ##########
(scrollback=20000, pager cap=64 MiB so it never caps during this run)
   lines_fed  count segments  pager_bytes    RSS_kB   dRSS_kB
        2071   2048        1            0     31548      5604
        4119   4096        2            0     36680     10736
        8215   8192        4            0     47160     21216
       16407  16384        8            0     68184     42240
       20023  20000       10            0     77248     51304
  -- saturation reached (count==ynum? True; segment count fixed); now ONLY the pager grows --
       30023  20000       10       140000     77588     51644
       50023  20000       10       420000     78992     53048
       90023  20000       10       980000     80852     54908
  NOTE: RSS is a process-wide figure (interpreter + arenas + buffers); it corroborates,
        but does not exclusively measure, the two HistoryBuf structures.

########## PART 2: WRAPPING — a wide logical line occupies MULTIPLE physical rows ##########
  xnum(cols)=80
  BEFORE: history count=0 (empty); cursor.y=0
  DURING: fed one 200-char logical line (no newline). ceil(200/80)=3 rows -> cursor.y=2 (advanced by 2 wraps); history count=0 (still on-screen)
  AFTER : pushed into history; count=20
  grouping of the wide line in history (oldest rows; idx 0 = newest):
     line(17) wrapped_last=False [END of logical line] text='WWWWWWWWWWWWWWWW'
     line(18) wrapped_last=True  [continues->        ] text='WWWWWWWWWWWWWWWW'
     line(19) wrapped_last=True  [continues->        ] text='WWWWWWWWWWWWWWWW'
  -> a 200-col logical line = 3 physical history rows; the first two carry the
     next_char_was_wrapped continuation flag (screen.c:L521-527, line.c:L426-431).
  non-wrapped accounting: fed 3000 short lines -> history count=2977 (= 3000 - (lines-1=23); first 23 fill the screen before INDEX_UP pushes any line)

########## PART 3: RETENTION — pager OFF (lost) vs ON (retained until cap) ##########
  pager OFF: fed 4000 lines -> count=2000 (saturated at ynum=2000), pager_bytes=0
             oldest RETAINED history line = 'L00001977' (L00000000..~L00001976 were EVICTED and LOST)
  pager ON : fed 4000 lines -> count=2000 (same saturation), pager_bytes=27678
             pager text length=27678 chars; earliest evicted content preserved? True (first evicted line still in the ring)
exit=0
```

### 8.1 Allocation footprint — two regimes

- *During fill:* segment count steps `1 → 2 → 4 → 8 → 10` as `count` crosses 2048 multiples, and RSS
  rises in matching steps: `dRSS` = 5604, 10736, 21216, 42240, 51304 kB. The increment per 2048-line
  segment is ≈5132–5256 kB, corroborating the 5,251,072-byte (5128 KiB) per-segment `calloc` of §3.1.
- *After saturation:* once `count == ynum = 20000` (segments fixed at 10), the segmented store stops
  growing; feeding tens of thousands more lines leaves segments at 10 and moves RSS only slightly, while
  `pager_bytes` climbs 140000 → 420000 → 980000. So under sustained pressure the footprint evolves in
  two phases: **segment carving during fill, then pager-ring growth after saturation**. The fill-phase
  `dRSS` steps (5604, 10736, 21216, 42240, 51304) were byte-identical across both runs; the
  post-saturation (pager-phase) `dRSS` carried a few-kB run-to-run wobble (RUN 1: 51644, 53044, 54920 vs
  RUN 2: 51644, 53048, 54908), while the *structural* `pager_bytes` figures (140000 → 420000 → 980000)
  were byte-identical. (RSS is a process-wide figure — interpreter, arenas, and buffers included — so it
  corroborates rather than exclusively measures the two structures; §8.4/§8.5 give allocator-level
  attribution.)

### 8.2 Wrapping

**Mechanism.** Autowrap (DECAWM) is on by default (`empty_modes` with `.mDECAWM=true`,
`kitty/screen.c:33`; wrap trigger at `kitty/screen.c:820-822`). When a glyph would overflow the right
margin, `continue_to_next_line` (`kitty/screen.c:521-527`) marks the current row's last cell as a
continuation via `linebuf_set_last_char_as_continuation` (`kitty/line-buf.c:194-197`, which sets
`next_char_was_wrapped` at `line-buf.c:196`) and line-feeds. That per-cell flag is what
`Line.last_char_has_wrapped_flag()` reads (`kitty/line.c:426-431`, checking
`gpu_cells[xnum-1].attrs.next_char_was_wrapped` at `line.c:429`).

**Before / during / after (Part 2).**
- *Before:* empty history, `count=0`, `cursor.y=0`.
- *During:* feeding a single **200-column** logical line (no newline) at `xnum=80` autowraps into
  `ceil(200/80)=3` physical rows — `cursor.y=2` — while `count` is still 0 (the rows are on-screen, not
  yet in history).
- *After:* pushing it into history makes it occupy **3 consecutive physical history rows**. Reading them
  oldest-first, the grouping is unambiguous: the first two rows carry the continuation flag
  (`wrapped_last=True`, "continues →") and the third does not (`wrapped_last=False`, "END of logical
  line"). A logical line is therefore stored as *N* physical rows where the first *N-1* are flagged
  continued.

**Physical-vs-logical accounting.** Retention is counted in **physical rows**, not logical lines: a wide
logical line consumes several scrollback slots, so `scrollback_lines` holds fewer logical lines when
output is wide. The non-wrapped case makes the base accounting explicit — feeding 3000 short lines at
`lines=24` yields `count=2977`, i.e. `3000 - (lines-1) = 3000 - 23`, because the first 23 lines fill the
on-screen rows before `INDEX_UP` begins pushing any line into history (`kitty/screen.c:1552-1567`).

### 8.3 Retention

**Mechanism.** At `count == ynum`, `historybuf_push` overwrites the oldest slot (advancing
`start_of_data`, `kitty/history.c:281`). With the pager **disabled** that oldest line is simply lost;
with the pager **enabled** it is first serialized into the ring (`pagerhist_push`,
`kitty/history.c:280`) and retained until the ring reaches its cap, after which the ring's own
oldest bytes are overwritten. That final overwrite-at-cap step is demonstrated **directly** in §6.2
(Scenario F): with a tiny 4096-byte cap and unique tokens, the earliest-evicted token `T000000`
disappears once the ring is full while newer tokens survive, and the buffer is left beginning
mid-record — so this is **observed**, not merely inferred from the source.

**Observed contrast (Part 3, `scrollback=2000`, 4000 lines fed).**
- *Pager off:* `count=2000`, `pager_bytes=0`; the oldest **retained** line is `L00001977` — lines
  `L00000000 … L00001976` were evicted and are **gone**.
- *Pager on (64 MiB):* `count=2000` (same live window), but `pager_bytes=27678` (the 1977 evicted
  records × 14 B), and the earliest evicted line `L00000000` is still present in the ring
  (`'L00000000' in pager text -> True`). Retention is thus **bounded live window + optional serialized
  overflow**, and the overflow is what preserves content beyond `scrollback_lines`.

### 8.4 Heap attribution via Valgrind Massif

To attribute allocations to their call sites, heap profiling needs a symbolized build. Getting Massif
to run required solving two real obstacles, both shown here. (In the Valgrind excerpts below, the only
edit to the tool's output is cosmetic — the trailing space on Valgrind's blank `==PID==` separator lines
has been trimmed so the document contains no trailing whitespace; no reported value, address, or count
is altered.)

**Obstacle 1 — Massif SIGILLs on the default build.** The default build compiles with `-march=native`,
which emits AVX-512 (EVEX-prefixed) instructions that Valgrind's decoder does not handle. Running Massif
on the release build dies before any history work, in the **option-conversion** code — not in the
history subsystem:

```text
$ ( ulimit -c 0; valgrind --tool=massif --massif-out-file="$WORK/massif.sigill.out" \
      ./kitty/launcher/kitty +launch "$WORK/massif_target.py" )
==85957== Massif, a heap profiler
==85957== Copyright (C) 2003-2024, and GNU GPL'd, by Nicholas Nethercote et al.
==85957== Using Valgrind-3.25.1 and LibVEX; rerun with -h for copyright info
==85957== Command: ./kitty/launcher/kitty +launch /tmp/kitty_obs.iPXoYt/massif_target.py
==85957==
vex amd64->IR: unhandled instruction bytes: 0x62 0xF2 0xFD 0x8 0x3B 0xC1 0xC5 0xFA 0x7E 0xD
vex amd64->IR:   REX=0 REX.W=0 REX.R=0 REX.X=0 REX.B=0
vex amd64->IR:   VEX=0 VEX.L=0 VEX.nVVVV=0x0 ESC=NONE
vex amd64->IR:   PFX.66=0 PFX.F2=0 PFX.F3=0
==85957== valgrind: Unrecognised instruction at address 0x5e9a99d.
==85957==    at 0x5E9A99D: convert_opts_from_python_opts.constprop.0 (in /tmp/blitzy/kitty/blitzy-37fb2915-992a-43f5-a568-f9834bbb3df4_a66137/kitty/fast_data_types.so)
==85957==    by 0x5E9D8CE: pyset_options.lto_priv.0 (in /tmp/blitzy/kitty/blitzy-37fb2915-992a-43f5-a568-f9834bbb3df4_a66137/kitty/fast_data_types.so)
==85957==    by 0x4A94DC9: cfunction_call.lto_priv.0 (methodobject.c:575)
==85957==    by 0x4A4A4FB: _PyObject_MakeTpCall (call.c:242)
==85957==    by 0x4A7188A: _PyEval_EvalFrameDefault (generated_cases.c.h:1621)
==85957==    by 0x4BABB54: UnknownInlinedFun (pycore_ceval.h:120)
==85957==    by 0x4BABB54: UnknownInlinedFun (ceval.c:2110)
==85957==    by 0x4BABB54: PyEval_EvalCode (ceval.c:982)
==85957==    by 0x4BC36DE: UnknownInlinedFun (bltinmodule.c:1183)
==85957==    by 0x4BC36DE: builtin_exec.lto_priv.0 (bltinmodule.c.h:573)
==85957==    by 0x4A71FE0: _PyEval_EvalFrameDefault (generated_cases.c.h:2385)
==85957==    by 0x4BABB54: UnknownInlinedFun (pycore_ceval.h:120)
==85957==    by 0x4BABB54: UnknownInlinedFun (ceval.c:2110)
==85957==    by 0x4BABB54: PyEval_EvalCode (ceval.c:982)
==85957==    by 0x4BC36DE: UnknownInlinedFun (bltinmodule.c:1183)
==85957==    by 0x4BC36DE: builtin_exec.lto_priv.0 (bltinmodule.c.h:573)
==85957==    by 0x4A4D1A4: UnknownInlinedFun (pycore_call.h:177)
==85957==    by 0x4A4D1A4: PyObject_Vectorcall (call.c:327)
==85957==    by 0x4A662F7: _PyEval_EvalFrameDefault (generated_cases.c.h:1621)
==85957==    by 0x4AA9CB1: UnknownInlinedFun (pycore_ceval.h:120)
==85957==    by 0x4AA9CB1: UnknownInlinedFun (ceval.c:2110)
==85957==    by 0x4AA9CB1: _PyFunction_Vectorcall (call.c:413)
==85957==    by 0x4BE7458: pymain_run_module.lto_priv.0 (main.c:353)
==85957==    by 0x48F229D: UnknownInlinedFun (main.c:692)
==85957==    by 0x48F229D: Py_RunMain.cold (main.c:776)
==85957==    by 0x40031E0: main (in /tmp/blitzy/kitty/blitzy-37fb2915-992a-43f5-a568-f9834bbb3df4_a66137/kitty/launcher/kitty)
==85957== Your program just tried to execute an instruction that Valgrind
==85957== did not recognise.  There are two possible reasons for this.
==85957== 1. Your program has a bug and erroneously jumped to a non-code
==85957==    location.  If you are running Memcheck and you just saw a
==85957==    warning about a bad jump, it's probably your program's fault.
==85957== 2. The instruction is legitimate but Valgrind doesn't handle it,
==85957==    i.e. it's Valgrind's fault.  If you think this is the case or
==85957==    you are not sure, please let us know and we'll try to fix it.
==85957== Either way, Valgrind will now raise a SIGILL signal which will
==85957== probably kill your program.
==85957==
==85957== Process terminating with default action of signal 4 (SIGILL)
==85957==  Illegal opcode at address 0x5E9A99D
==85957==    at 0x5E9A99D: convert_opts_from_python_opts.constprop.0 (in /tmp/blitzy/kitty/blitzy-37fb2915-992a-43f5-a568-f9834bbb3df4_a66137/kitty/fast_data_types.so)
==85957==    by 0x5E9D8CE: pyset_options.lto_priv.0 (in /tmp/blitzy/kitty/blitzy-37fb2915-992a-43f5-a568-f9834bbb3df4_a66137/kitty/fast_data_types.so)
==85957==    by 0x4A94DC9: cfunction_call.lto_priv.0 (methodobject.c:575)
==85957==    by 0x4A4A4FB: _PyObject_MakeTpCall (call.c:242)
==85957==    by 0x4A7188A: _PyEval_EvalFrameDefault (generated_cases.c.h:1621)
==85957==    by 0x4BABB54: UnknownInlinedFun (pycore_ceval.h:120)
==85957==    by 0x4BABB54: UnknownInlinedFun (ceval.c:2110)
==85957==    by 0x4BABB54: PyEval_EvalCode (ceval.c:982)
==85957==    by 0x4BC36DE: UnknownInlinedFun (bltinmodule.c:1183)
==85957==    by 0x4BC36DE: builtin_exec.lto_priv.0 (bltinmodule.c.h:573)
==85957==    by 0x4A71FE0: _PyEval_EvalFrameDefault (generated_cases.c.h:2385)
==85957==    by 0x4BABB54: UnknownInlinedFun (pycore_ceval.h:120)
==85957==    by 0x4BABB54: UnknownInlinedFun (ceval.c:2110)
==85957==    by 0x4BABB54: PyEval_EvalCode (ceval.c:982)
==85957==    by 0x4BC36DE: UnknownInlinedFun (bltinmodule.c:1183)
==85957==    by 0x4BC36DE: builtin_exec.lto_priv.0 (bltinmodule.c.h:573)
==85957==    by 0x4A4D1A4: UnknownInlinedFun (pycore_call.h:177)
==85957==    by 0x4A4D1A4: PyObject_Vectorcall (call.c:327)
==85957==    by 0x4A662F7: _PyEval_EvalFrameDefault (generated_cases.c.h:1621)
==85957==    by 0x4AA9CB1: UnknownInlinedFun (pycore_ceval.h:120)
==85957==    by 0x4AA9CB1: UnknownInlinedFun (ceval.c:2110)
==85957==    by 0x4AA9CB1: _PyFunction_Vectorcall (call.c:413)
==85957==    by 0x4BE7458: pymain_run_module.lto_priv.0 (main.c:353)
==85957==    by 0x48F229D: UnknownInlinedFun (main.c:692)
==85957==    by 0x48F229D: Py_RunMain.cold (main.c:776)
==85957==    by 0x40031E0: main (in /tmp/blitzy/kitty/blitzy-37fb2915-992a-43f5-a568-f9834bbb3df4_a66137/kitty/launcher/kitty)
==85957==
```

The unhandled bytes begin `0x62` (the EVEX/AVX-512 prefix); the illegal opcode is in
`convert_opts_from_python_opts` called by `pyset_options` — i.e. the code that converts the Python
options object into the C options struct at screen setup, reached before any line is ingested. This is
purely a profiler-decoding limitation, not a defect. (`ulimit -c 0` suppressed the core dump.)

**Obstacle 2 — the AVX-512-off build flag drops the Python include path.** The fix is to compile the
extension without AVX-512 (`-mno-avx512f`), which kitty appends *after* `-march=native` via
`--python-compiler-flags`. But `get_python_flags` uses an if/else: when `--python-compiler-flags` is
supplied it uses those flags **instead of** the auto-added Python include paths (`setup.py`,
`get_python_flags`). So the naive command drops the auto-added Python include path (`-I$PYINC`, shown
resolved in the next block) and fails:

```text
$ ./dev.sh build --debug --ignore-compiler-warnings --python-compiler-flags="-mno-avx512f"
[1/65] Compiling kitty/screen.c ...
In file included from kitty/state.h:8,
                 from kitty/screen.c:14:
kitty/data-types.h:11:10: fatal error: Python.h: No such file or directory
   11 | #include <Python.h>
      |          ^~~~~~~~~~
The following build command failed: /tmp/blitzy/kitty/blitzy-37fb2915-992a-43f5-a568-f9834bbb3df4_a66137/dependencies/linux-amd64/bin/python setup.py develop --debug --ignore-compiler-warnings --python-compiler-flags=-mno-avx512f
exit status 1
```

The correct profiling build therefore **re-adds** the Python include path alongside `-mno-avx512f`
(the include path is obtained from `sysconfig.get_path('include')`):

```text
$ PYINC=$(./kitty/launcher/kitty +runpy 'import sysconfig;print(sysconfig.get_path("include"))')
$ ./dev.sh build --debug --ignore-compiler-warnings --python-compiler-flags="-I$PYINC -mno-avx512f"
[1/65] Compiling kitty/screen.c ...
[64 further "[N/65] Compiling …" / "[N/M] Linking …" progress lines omitted for length — all succeeded]
Build successful. Run kitty as: kitty/launcher/kitty
```

The debug build carries `-g3 -Og -DKITTY_DEBUG_BUILD -fno-omit-frame-pointer -march=native` (plus the
usual `-march=native`-derived ISA flags) and `-mno-avx512f`; the exact compile line, echoed by the
failing attempt above, differs only by the added `-I$PYINC` Python include path. After profiling, the
release artifacts are restored from a backup and verified (§12).

The Massif **target script** feeds the same canonical `parse_bytes` burst used throughout, filling
the segmented store to **ten segments** — one allocated at construction (`create_historybuf` →
`add_segment`, `history.c:127`) and nine more carved lazily as `count` crosses the nine 2048-line
allocation boundaries (rows 2049, 4097, …, 18433; the eleventh boundary at row 20481 is never reached,
since `count` saturates at `ynum = 20000`) — with the pager explicitly disabled so the profile
isolates segment growth (it is created in `$WORK` with `cat > "$WORK/massif_target.py"` like the
others):

```python
# massif_target.py — canonical burst for heap profiling: fill segmented store to
# ynum crossing multiple 2048 segment boundaries (pager OFF to isolate segments).
from kitty_tests import BaseTest, parse_bytes
class T(BaseTest):
    def run(self):
        s = self.create_screen(80, 24, 20000, options={'scrollback_pager_history_size': 0})
        parse_bytes(s, ("".join(f"L{i:08d}\r\n" for i in range(20050))).encode())
        print("count=", s.historybuf.count, "segments=", (min(s.historybuf.count,20000)+2047)//2048)
T().run()
```

**Massif runs (≥2×).** The target fills the segmented store to `ynum=20000` (10 segments), pager
disabled, over the canonical `parse_bytes` path:

```text
$ for r in 1 2; do ( ulimit -c 0; valgrind --tool=massif --time-unit=B \
      --massif-out-file="$WORK/massif.out.$r" \
      ./kitty/launcher/kitty +launch "$WORK/massif_target.py" ); echo "exit=$?"; done
count= 20000 segments= 10
exit=0
count= 20000 segments= 10
exit=0
```

Both runs completed with no SIGILL and produced identical profiles; the peak snapshot is #76 in both.

**Extraction (documented, reproducible).** Render the report and read the peak snapshot's summary row
and allocation tree:

```text
$ ms_print "$WORK/massif.out.1" > "$WORK/msprint.1.txt"
$ grep -nE '^ 7[4-6] ' "$WORK/msprint.1.txt"      # snapshot table rows around the peak
8564: 74    104,209,136       53,685,920       53,607,692        78,228            0
8740: 75    109,464,264       58,941,048       58,858,788        82,260            0
8741: 76    109,464,264       58,941,048       58,858,788        82,260            0

$ sed -n '8742,8916p' "$WORK/msprint.1.txt" | grep -E '^->'   # top-level branches at peak #76
->89.09% (52,510,720B) 0x5E70920: add_segment (history.c:25)
->03.44% (2,026,012B) in 138 places, all below massif's threshold (1.00%)
->01.78% (1,050,176B) 0x5EC9FE1: alloc_vt_parser (vt-parser.c:1565)
->01.51% (891,712B) 0x5EAD8D0: utf8_decoder_ensure_capacity (simd-string.h:31)
->01.47% (868,296B) 0x4A38075: UnknownInlinedFun (obmalloc.c:63)
->01.35% (794,544B) 0x4A590D1: UnknownInlinedFun (obmalloc.c:63)
->01.22% (717,328B) 0x4A351B5: UnknownInlinedFun (obmalloc.c:63)

$ # the add_segment subtree (its two calloc call sites and their call chains):
->89.09% (52,510,720B) 0x5E70920: add_segment (history.c:25)
| ->80.18% (47,259,648B) 0x5E709BF: segment_for (history.c:39)
| | ->80.18% (47,259,648B) 0x5E709F2: cpu_lineptr (history.c:52)
| |   ->80.18% (47,259,648B) 0x5E70AAF: init_line (history.c:164)
| |     ->80.18% (47,259,648B) 0x5E70FDB: historybuf_push (history.c:278)
| |       ->80.18% (47,259,648B) 0x5E72386: historybuf_add_line (history.c:288)
| |         ->80.18% (47,259,648B) 0x5E9E90C: screen_index (screen.c:1575)
| |           ->80.18% (47,259,648B) 0x5E9EDE5: screen_linefeed (screen.c:1645)
| |             ->80.18% (47,259,648B) 0x5EA0275: draw_text_loop (screen.c:795)
| |               ->80.18% (47,259,648B) 0x5EA05DD: draw_text (screen.c:862)
| |                 ->80.18% (47,259,648B) 0x5EA0655: screen_draw_text (screen.c:868)
| |                   ->80.18% (47,259,648B) 0x5EC664A: consume_normal (vt-parser.c:236)
| |                     ->80.18% (47,259,648B) 0x5EC8C33: consume_input (vt-parser.c:1377)
| |                       ->80.18% (47,259,648B) 0x5EC8DF2: run_worker (vt-parser.c:1432)
| |                         ->80.18% (47,259,648B) 0x5EC9F88: parse_worker (vt-parser.c:1496)
| |                           ->80.18% (47,259,648B) 0x5E9C79D: test_parse_written_data (screen.c:4776)
| ->08.91% (5,251,072B) 0x5E7143E: create_historybuf (history.c:127)
|   ->08.91% (5,251,072B) 0x5E7258F: alloc_historybuf (history.c:578)
|     ->08.91% (5,251,072B) 0x5E9A1B3: new_screen_object (screen.c:130)
```

**Reading the peak (#76).** Total heap at peak is **58,941,048 B**, which is useful heap
**58,858,788 B (56.13 MiB)** plus `82,260 B` of allocator extra (peak-table row #76 above). Massif's
allocation-tree percentages are computed against that total heap, and attribute it as follows:
- **89.09% (52,510,720 B) = `add_segment` (`history.c:25`)** — the entire segmented store, and exactly
  `10 × 5,251,072`, independently confirming the per-segment size of §3.1. It splits into two `calloc`
  call sites:
  - **80.18% (47,259,648 B) via `segment_for` (`history.c:39`)** — the **9 lazily-added** segments
    (`9 × 5,251,072`), reached through the full canonical chain that Massif itself records:
    `segment_for ← cpu_lineptr ← init_line ← historybuf_push (history.c:278) ← historybuf_add_line
    (history.c:288) ← screen_index (screen.c:1575) ← screen_linefeed ← draw_text_loop → draw_text →
    screen_draw_text (screen.c:868) ← consume_normal (vt-parser.c:236) ← consume_input ← run_worker →
    parse_worker (vt-parser.c:1496) ← test_parse_written_data (screen.c:4776)`. That stack is direct
    proof the segments were carved by the real **VT-parser → screen → history** ingest path.
  - **8.91% (5,251,072 B) via `create_historybuf` (`history.c:127`)** — the **one** segment allocated at
    screen construction (`alloc_historybuf`, `history.c:578` ← `new_screen_object`, `screen.c:130`).
- The remaining **≈10.8% (≈6,348,068 B)** is **not** the two history structures: `alloc_vt_parser`
  (`vt-parser.c:1565`) 1.78%, `utf8_decoder_ensure_capacity` (`simd-string.h:31`) 1.51%, the Python
  object allocator (`obmalloc.c`) ≈4%, and 138 sub-threshold sites 3.44%. So while the segmented store
  dominates the heap, it is ~89% — not the whole process — which is why RSS in §8.1 is treated as
  corroboration, not an exclusive measurement.

### 8.5 A second heap cross-check: glibc `malloc_info(3)`

Massif requires a special build; `malloc_info(3)` needs none, so it is a useful independent
cross-check on the *release* build. It reports the process-wide allocator state as XML. Because each
per-segment `calloc` (5,251,072 B) and the grown pager ring far exceed glibc's mmap threshold
(128 KiB), they are served by `mmap`, so the script tracks `<total type="mmap">` plus the arena's
`<system type="current">`. The script and its (byte-identical, ≥2×) output:

```python
# mallocinfo.py — glibc malloc_info(3) cross-check (process-wide allocator view).
# The large per-segment calloc (>128 KiB) and the grown pager ring are served via
# mmap, so we track <total type="mmap"> (large allocs) plus <system current> (arena).
# This corroborates the per-segment size (5,251,072 B) and the pager growth measured
# elsewhere; it is a process-wide figure, not an exclusive measurement.
import ctypes, re, gc
from kitty_tests import BaseTest, parse_bytes

libc = ctypes.CDLL("libc.so.6", use_errno=True); libc.open_memstream.restype = ctypes.c_void_p

def snapshot():
    buf = ctypes.c_char_p(); size = ctypes.c_size_t()
    ms = libc.open_memstream(ctypes.byref(buf), ctypes.byref(size))
    libc.malloc_info(0, ctypes.c_void_p(ms)); libc.fflush(ctypes.c_void_p(ms)); libc.fclose(ctypes.c_void_p(ms))
    x = ctypes.string_at(buf, size.value).decode(); libc.free(ctypes.cast(buf, ctypes.c_void_p))
    arena = int(re.findall(r'<system type="current" size="(\d+)"/>', x)[-1])
    mm = re.findall(r'<total type="mmap" count="\d+" size="(\d+)"/>', x)
    mmap_total = int(mm[-1]) if mm else 0
    return arena, mmap_total, arena + mmap_total

PER_SEG = 2048 * 80 * (12 + 20) + 2048 * 4   # 5,251,072 B (CPUCell12 + GPUCell20 + LineAttrs4)

def feed(s, n):
    parse_bytes(s, ("".join(f"L{i:08d}\r\n" for i in range(n))).encode())

class T(BaseTest):
    def run(self):
        gc.disable()
        print(f"per-segment allocation request (arithmetic) = {PER_SEG} B = {PER_SEG/1024:.1f} KiB")

        print("\n--- SEGMENT growth: 1 (at construction) -> 3 full segments (scrollback=6144, pager OFF) ---")
        s = self.create_screen(80, 24, 6144, options={'scrollback_pager_history_size': 0})
        a0, mm0, t0 = snapshot()
        feed(s, 6200)                          # count -> 6144 == ynum, 3 segments
        a1, mm1, t1 = snapshot()
        segn = (min(s.historybuf.count, 6144) + 2047) // 2048
        print(f"  count={s.historybuf.count} segments={segn} pager_bytes={len(s.historybuf.pagerhist_as_bytes())}")
        print(f"  arena system-current: {a0} -> {a1}  (delta {a1-a0} B)")
        print(f"  mmap total:           {mm0} -> {mm1}  (delta {mm1-mm0} B = {(mm1-mm0)/PER_SEG:.2f} x per-segment)")
        print(f"  total footprint:      {t0} -> {t1}  (delta {t1-t0} B = {(t1-t0)/1048576:.3f} MiB)")
        print(f"  -> +2 segments added after construction; mmap delta {mm1-mm0} vs 2*per-segment={2*PER_SEG} B")

        print("\n--- PAGER growth: known evictions past saturation (scrollback=2000, pager 8 MiB) ---")
        p = self.create_screen(80, 24, 2000, options={'scrollback_pager_history_size': 8 * 1024 * 1024})
        feed(p, 2100)                          # saturate segmented store
        pb0 = len(p.historybuf.pagerhist_as_bytes()); a0, mm0, t0 = snapshot()
        feed(p, 300000)                        # many evictions -> serialized into pager; ring extends in >=1 MiB steps
        pb1 = len(p.historybuf.pagerhist_as_bytes()); a1, mm1, t1 = snapshot()
        print(f"  pager_bytes serialized: {pb0} -> {pb1}  (delta {pb1-pb0} B = {(pb1-pb0)/1048576:.3f} MiB)")
        print(f"  arena system-current:   {a0} -> {a1}  (delta {a1-a0} B)")
        print(f"  mmap total:             {mm0} -> {mm1}  (delta {mm1-mm0} B = {(mm1-mm0)/1048576:.3f} MiB)")
        print(f"  total footprint:        {t0} -> {t1}  (delta {t1-t0} B = {(t1-t0)/1048576:.3f} MiB)")
        print("  -> serialized bytes are the exact ring contents; allocator delta reflects ring")
        print("     CAPACITY, which pagerhist_extend grows in >=1 MiB steps (history.c:L89-101).")
        gc.enable()

T().run()
```

```text
$ for r in 1 2; do echo "================= RUN $r ================="; \
    ./kitty/launcher/kitty +launch "$WORK/mallocinfo.py"; echo "exit=$?"; done
================= RUN 1 =================
per-segment allocation request (arithmetic) = 5251072 B = 5128.0 KiB

--- SEGMENT growth: 1 (at construction) -> 3 full segments (scrollback=6144, pager OFF) ---
  count=6144 segments=3 pager_bytes=0
  arena system-current: 3809280 -> 3866624  (delta 57344 B)
  mmap total:           7127040 -> 17920000  (delta 10792960 B = 2.06 x per-segment)
  total footprint:      10936320 -> 21786624  (delta 10850304 B = 10.348 MiB)
  -> +2 segments added after construction; mmap delta 10792960 vs 2*per-segment=10502144 B

--- PAGER growth: known evictions past saturation (scrollback=2000, pager 8 MiB) ---
  pager_bytes serialized: 1078 -> 4201078  (delta 4200000 B = 4.005 MiB)
  arena system-current:   4210688 -> 13549568  (delta 9338880 B)
  mmap total:             25280512 -> 24227840  (delta -1052672 B = -1.004 MiB)
  total footprint:        29491200 -> 37777408  (delta 8286208 B = 7.902 MiB)
  -> serialized bytes are the exact ring contents; allocator delta reflects ring
     CAPACITY, which pagerhist_extend grows in >=1 MiB steps (history.c:L89-101).
exit=0
================= RUN 2 =================
per-segment allocation request (arithmetic) = 5251072 B = 5128.0 KiB

--- SEGMENT growth: 1 (at construction) -> 3 full segments (scrollback=6144, pager OFF) ---
  count=6144 segments=3 pager_bytes=0
  arena system-current: 3809280 -> 3866624  (delta 57344 B)
  mmap total:           7127040 -> 17920000  (delta 10792960 B = 2.06 x per-segment)
  total footprint:      10936320 -> 21786624  (delta 10850304 B = 10.348 MiB)
  -> +2 segments added after construction; mmap delta 10792960 vs 2*per-segment=10502144 B

--- PAGER growth: known evictions past saturation (scrollback=2000, pager 8 MiB) ---
  pager_bytes serialized: 1078 -> 4201078  (delta 4200000 B = 4.005 MiB)
  arena system-current:   4210688 -> 13549568  (delta 9338880 B)
  mmap total:             25280512 -> 24227840  (delta -1052672 B = -1.004 MiB)
  total footprint:        29491200 -> 37777408  (delta 8286208 B = 7.902 MiB)
  -> serialized bytes are the exact ring contents; allocator delta reflects ring
     CAPACITY, which pagerhist_extend grows in >=1 MiB steps (history.c:L89-101).
exit=0
```

**Segment cross-check.** Filling to 3 full segments (1 built at construction + 2 added) grows the mmap
total by **10,792,960 B = 2.06 × per-segment**, i.e. essentially the two post-construction segments
(`2 × 5,251,072 = 10,502,144 B`). The 2.06× (rather than an exact 2.00×) is allocator rounding and
mmap granularity — which is the point: `malloc_info` corroborates the per-segment size but is a
*process-wide* figure, not an exact per-structure count. This is why the earlier claim of "exactly N
segments" from `malloc_info` is unsound; the sound statement is that the mmap delta ≈ 2× the arithmetic
per-segment size.

**Pager cross-check — and why one must read the *total*, not a single line item.** After saturation,
driving 300,000 more lines serializes **4,200,000 B (4.005 MiB)** of exact record bytes into the ring
(`pager_bytes: 1078 → 4,201,078`). But over the same interval the **mmap sub-total went *down* by
1.004 MiB** while the **arena grew by 9.34 MiB**, for a **total-footprint delta of 7.902 MiB**. The
allocator relocated large blocks between arena and mmap, so no single line item equals "the pager."
The honest reading is the **total-footprint delta (7.902 MiB)**, which reflects the ring's grown
*capacity* (extended in ≥1 MiB steps, `history.c:89-101`), and is necessarily larger than the
4.005 MiB of serialized content it now holds. This is the concrete reason the document reports pager
growth as a total-footprint delta with explicit units, rather than pinning it to one allocator field.

## 9. Stability across runs

Rule 1 requires every magnitude/timing claim to be confirmed stable across at least two identical
runs, and any run-to-run inconsistency to be reproduced with the *same* input rather than smoothed
over. Each scenario above was captured `≥2×`; this section states, per scenario, exactly what was
stable and where variance appeared and why.

**Byte-identical across runs (structural/allocation quantities).** The following outputs were
*character-for-character identical* between run 1 and run 2:
- `units.py` — the raw-bytes-vs-MiB semantics (§5).
- Scenario A — the saturation triple `count=20000, segments=10, pager_bytes=0`, the `1 → 2 → … → 10`
  segment progression, and the per-segment RSS increment *sequence*
  `5132, 5132, 5132, 5136, 5132, 5132, 5132, 5132, 3932` kB — ≈5132 kB per full segment, with one
  reproducible 5136 kB step into the `count=10240` boundary and a partial 3932 kB final step — which
  was itself byte-identical across both runs (§4). *(Only the absolute `dRSS` values, not this gap
  sequence, drift run-to-run — see the RSS-noise note below.)*
- Scenario B — the eviction hand-off byte counts `17/187/1887/18887` and the ~1220 kB
  construction-time RSS delta that corroborates the ~1 MiB initial ring reservation (§5).
- Scenario C2 — the three *used*-byte thresholds at which the extend-cost spikes land,
  `1,048,586 / 2,097,186 / 3,145,730 B`, and `CONSISTENCY CHECK = True` in **both** runs (§6.2).
- Scenario D — the anchor/clamp/frozen-view values `500→1500`, clamp at `2000`, `1100` (§7).
- Scenario E — the **fill-phase** `dRSS` steps `5604 / 10736 / 21216 / 42240 / 51304`, the 3-row wrap,
  and the retention contrast (§8.1–8.3). *(The post-saturation pager-phase `dRSS` carries a few-kB
  wobble; the structural `pager_bytes` figures `140000 → 420000 → 980000` do not — see the RSS-noise
  note below.)*
- `mallocinfo.py` — the segment and pager deltas (§8.5).
- Massif — identical profiles, peak snapshot **#76** with useful heap `58,858,788 B` in both runs
  (§8.4).

**Reproducible in character, not to the microsecond (raw timing).** The per-line *timings* in
Scenario C/C2 are wall-clock and therefore not bit-identical run to run, but their *structure* is
fully reproducible and was confirmed so:
- Segment-boundary spikes (Scenario C) land on the `+1` line past each 2048 multiple
  (`count = 2049, 4097, 6145, 8193`) in **every** trial; only the absolute microseconds drift, and
  they tend to grow with accumulated heap pressure within a process — though not strictly monotonically
  (e.g. `count=2049 → [51.4, 438.0, 433.5, 410.0, 455.0] µs` rises, then wobbles) — reproduced across
  five back-to-back trials (§6.1).
- The one off-boundary `count=1918` anomaly the earlier document flagged was **not** reproducible as a
  spike: re-running the exact input put that line at `1.26–2.16 µs` (ordinary), so it was a one-off
  scheduling blip, not a buffer event (§6.1, Part C).
- The large *off-edge* timing spikes in the GC-on parts of Scenario C/C2 are reproducible **as GC
  events**: disabling the cyclic collector removes them entirely and leaves only the true
  segment/pager edges. The evidence is at the aggregate level — with GC **on**, Part D counts many
  collections (`gc_collections = 21`) alongside millisecond off-boundary maxima; with GC **off**, the
  same burst records `gc_collections = 0` and no such maxima (§6.1 Part D, §6.2). (This is an
  aggregate correlation across the run — GC-on has both collections and large maxima, GC-off has
  neither — not a per-line proof that each individual spike is a collection event.) This is the
  "reproduce it with the same input" discipline in action — the variance had a definite, demonstrable
  cause (Python's cyclic GC), not the history buffer.

**Reproducible increment sequence, noisy absolute (process RSS).** `RSS`/`dRSS` are process-wide
figures, so their *absolute* values carry a small run-to-run wobble (typically one page, ≈4 kB) from
baseline drift and first-touch page accounting. This showed up concretely in Scenario A, where run 1's
first fill step read `5604 kB` and run 2's read `5608 kB`; in Scenario E's post-saturation steps
(`51644/53044/54920` vs `51644/53048/54908`); and in Scenario B's Part 0 construction cost
(`1308/1304 kB`). What is stable is the *increment sequence*, not a single uniform number: across
Scenario A's nine full-segment boundaries the per-segment step was `5132 kB` at eight of them, with a
single `5136 kB` step into `count=10240` and a partial `3932 kB` final step, and that whole sequence
`5132, 5132, 5132, 5136, 5132, 5132, 5132, 5132, 3932` was byte-identical across both runs — the lone
5136 outlier is a one-page first-touch jitter that reproduced, not evidence of uniform behaviour. The
structural quantities layered on top of RSS (`count`, `segments`, `pager_bytes`) were byte-identical
throughout. RSS is therefore used as corroboration of the two structures' growth, never as their
exclusive measure — §8.4/§8.5 give the allocator-level attribution that does not depend on RSS.

In short: every **structural** quantity (counts, segment totals, byte sizes, extend points, heap peak,
and the per-segment RSS *increment sequence* — including its lone 5136 kB outlier) is exactly
reproducible. Two things vary run-to-run, both
benignly: raw microsecond timings (whose *shape* is reproducible and whose outliers were traced to a
specific, demonstrable cause — Python's cyclic GC), and the *absolute* process-RSS figures (which wobble
by about one page for the ordinary reasons above).

## 10. Observed vs. inferred

This section separates what was **directly observed at runtime** from what is **inferred from the
source** (labeled *inferred* where it appears above, and listed here in full). The rule is: a claim is
"observed" only if a captured value demonstrates it; everything else is "inferred" and is flagged as
such, with the reason it could not be observed directly and any indirect corroboration.

### 10.1 Directly observed at runtime

- **Fill and saturation (REQ-1).** `HistoryBuf.count` rising with ingest and saturating exactly at
  `ynum` — Scenario A (`count=20000, segments=10`).
- **Per-segment allocation size.** `5,251,072 B` per segment — observed three independent ways: the
  compiled `sizeof` probe (§3.1), the `dRSS` step per segment (§4, §8.1), and Massif's
  `add_segment (history.c:25)` node (§8.4).
- **Lazy, incremental carving via the real ingest path.** Segments appearing one at a time as `count`
  grows, allocated from `segment_for` — Massif records the full
  `parse_worker → screen_index → historybuf_add_line → historybuf_push → segment_for` stack (§8.4).
- **Eviction hand-off to the pager (REQ-2).** Bytes appearing in `pagerhist_as_bytes()` only after
  saturation, `17 B` per evicted line — Scenario B; and *no* pager bytes when the pager is disabled.
- **Overwrite-oldest at the pager cap (REQ-2/3b/5).** Once the ring is full its oldest bytes are
  overwritten as the tail advances (`ringbuf_memcpy_into`, `3rdparty/ringbuf/ringbuf.c:233`): with a
  4096-byte cap and unique tokens, the earliest-evicted `T000000` disappears while newer tokens remain
  and the buffer is left beginning **mid-record** — Scenario F (§6.2), byte-identical across runs.
- **Segment-boundary timing edge (REQ-3a).** The per-line cost spike on the line just past each 2048
  multiple — Scenario C, reproduced across five trials.
- **The GC provenance of off-edge spikes.** Disabling the cyclic collector removes the large off-edge
  spikes: with GC on, Part D records many collections (`gc_collections = 21`) alongside the
  millisecond off-boundary maxima; with GC off, the same burst records `gc_collections = 0` and no such
  maxima — Scenario C Part D, Scenario C2. *(Aggregate correlation across the run, not a per-line
  attribution of each individual spike.)*
- **Concurrent scroll anchoring and clamp (REQ-4).** `scrolled_by` frozen during ingest then advanced,
  and clamped to `count` — Scenario D (`500→1500`, clamp at `2000`, `1100`).
- **Wrapping (REQ-5).** A 200-column logical line occupying 3 physical rows, with the wrapped-flag set
  on the non-final rows — Scenario E, via `Line.last_char_has_wrapped_flag()`.
- **Retention off vs. on.** Oldest line lost with the pager disabled vs. preserved (serialized) with it
  enabled — Scenario E.
- **Total heap peak and its attribution.** Useful heap `58,858,788 B`; segmented store `89.09%` —
  Massif (§8.4); corroborated process-wide by `malloc_info` (§8.5).
- **Build/tooling facts.** The default `-Werror` build failure, the Massif SIGILL location, and the
  corrected profiling build — §2.2, §8.4 (all with captured command output).

### 10.2 Inferred from source (not directly observed), with corroboration

Each item below is derived from reading the code because the relevant internal is **not exposed to
Python** (§2.5) or is **below the profiler's measurement threshold**. Where possible, an *indirect*
runtime corroboration is noted.

- **Initial pager-ring *capacity* `MIN(1 MiB, cap)`.** The ring's *allocated capacity* is fixed at
  construction by `alloc_pagerhist` (`history.c:66–67`, via `ringbuf_new`). This is *inferred* from
  source because `pagerhist_as_bytes()` exposes only the *used* byte count, never the allocated
  capacity (§2.5). *Corroboration:* with the pager enabled but still empty, Scenario B Part 0 shows a
  one-time construction cost in process memory (RSS/`malloc_info`) consistent with a ~1 MiB allocation
  standing ready before any line is evicted.
- **Pager *capacity* extends in ≥1 MiB steps (REQ-3b).** That the ring grows its *allocated capacity*
  in ≥1 MiB increments (`pagerhist_extend`, `history.c:89–101`) is *inferred* from source, since the
  capacity itself is hidden. *Corroboration:* Scenario C2 directly *observes* per-line cost spikes as
  the *used* byte count (`pagerhist_as_bytes()`) passes the thresholds
  `1,048,586 / 2,097,186 / 3,145,730 B` — i.e. each time usage crosses a ~1 MiB mark — and the §8.4
  total-footprint delta (grown capacity necessarily ≥ serialized content) is consistent with a
  capacity that steps ahead of usage in ≥1 MiB blocks.
- **Circular-slot indexing and the `start_of_data` advance.** That live lines occupy slots
  `(start_of_data + count) % ynum` and that eviction advances `start_of_data` (`history.c:277–282`)
  is *inferred*: `start_of_data` is not exposed. *Corroboration:* the observed strict oldest-first
  eviction order (Scenario E retention) is exactly what an advancing ring start produces.
- **The eviction gate `count == ynum`.** The specific condition in `historybuf_push`
  (`history.c:279`) is *inferred* from code. *Corroboration:* pager bytes begin to accrue at precisely
  the observed saturation point and not before (Scenario B).
- **The per-record byte *composition* in the pager.** The `17 B` record *size* is observed
  (Scenario B); its composition — SGR reset `"\x1b[m"` (3) + cell bytes (12) + `\r` + `\n`-unless-the
  line-wrapped (`history.c:266–270`) — is *inferred* from code (the size, not the byte layout, is what
  the runtime exposed).
- **The segment *pointer-array* `realloc`.** `add_segment` first `realloc`s the array of segment
  pointers (`history.c:20`) before `calloc`-ing the new block. The `realloc` is *inferred* — it is a
  few tens of bytes and stays below Massif's 1% threshold; only the per-segment `calloc`
  (`history.c:25`) is large enough to be observed.
- **`DECAWM` on by default.** That auto-wrap is enabled by default (`screen.c:33`) is *inferred* from
  code; the *effect* (a 200-column line wrapping to 3 rows) is observed (Scenario E).
- **The exact anchoring source line.** The formula
  `scrolled_by = MIN(scrolled_by + history_line_added_count, count)` (`screen.c:2716`) is a code
  citation; its *behavior* is observed to match exactly (Scenario D), so the formula is
  observation-corroborated but the specific line attribution is from source.

## 11. Citation appendix

Every `file:line` below was read in the checked-out tree at commit
`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`. The specific behavioral lines cited inline are exact; the
accompanying function and block ranges bracket the named construct.

**`kitty/history.c` — segmented store + pager ring**
- `SEGMENT_SIZE` (2048) — `history.c:15`
- `add_segment` (pointer-array `realloc` L20, per-segment `calloc` L25) — `history.c:17–29`
- `segment_for` (lazy-allocate condition L39) — `history.c:36–42`
- `cpu_lineptr` — `history.c:52`
- initial pager ring size `MIN(1 MiB, sz)` — `history.c:66–67`
- `alloc_pagerhist` (returns NULL when size 0, L72; `ringbuf_new` L76; `maximum_size` L78) — `history.c:69–80`
- `pagerhist_extend` (`≥ maximum_size` guard L92; grow L93–94; copy L97) — `history.c:89–101`
- `create_historybuf` (construction `add_segment` L127; `alloc_pagerhist` L130) — `history.c:116–133`
- `init_line` — `history.c:164`
- `pagerhist_push` (SGR reset `"\x1b[m"` L266; `\r` L269; `\n` unless wrapped L270) — `history.c:258–273`
- `historybuf_push` (slot index L277; `pagerhist_push` L280; `start_of_data` advance L281; `count++` L282) — `history.c:275–284`
- `historybuf_add_line` — `history.c:286–291`
- `pagerhist_as_bytes` / `pagerhist_as_text` — `history.c:460–483` / `485–494`
- `xnum` / `ynum` / `count` exposed READONLY to Python — `history.c:554–559`
- `alloc_historybuf` — `history.c:577–579`

**`kitty/data-types.h` — structs and cell sizes**
- `GPUCell` (20 B; `static_assert` L221) — `data-types.h:215–221`
- `CPUCell` (12 B; `static_assert` L228) — `data-types.h:223–228`
- `PromptKind` — `data-types.h:230`
- `LineAttrs` **union** (`sizeof == 4`; **no** `static_assert`) — `data-types.h:231–239`
- `HistoryBufSegment` — `data-types.h:262–266`
- `PagerHistoryBuf` — `data-types.h:268–272`
- `HistoryBuf` — `data-types.h:282–290`

**`kitty/screen.c` — canonical caller, ingest, scroll anchoring**
- `empty_modes.mDECAWM = true` (auto-wrap default) — `screen.c:33`
- `alloc_historybuf` at `new_screen_object` — `screen.c:130`
- `continue_to_next_line` (wrap continuation) — `screen.c:521–527`
- `screen_index` / `INDEX_UP` (add line L1558, counter++ L1559; gate L1574, INDEX_UP L1575) — `screen.c:1552–1575`
- `screen_linefeed` — `screen.c:1645`
- `screen_reset_dirty` (resets `history_line_added_count`, L2600) — `screen.c:2598–2600`
- `screen_update_only_line_graphics_data` (capture L2714; anchor `scrolled_by = MIN(scrolled_by + history_line_added_count, count)` L2716; reset L2717) — `screen.c:2713–2717`
- `screen_update_cell_data` (anchor L2761) — `screen.c:2756–2761`
- `screen_history_scroll` (`new_scroll = MIN(scrolled_by+amt, count)` L4111) — `screen.c:4091–4111`
- `test_parse_written_data` (used by `parse_bytes`) — `screen.c:4776`
- `scrolled_by` READONLY L4903; `history_line_added_count` writable L4908

**`kitty/line.c` / `kitty/line-buf.c` — wrapping**
- `last_char_has_wrapped_flag` (checks continuation, L429) — `line.c:426–431`
- `linebuf_set_last_char_as_continuation` (L196) / `is_continued` (L145) — `line-buf.c:194–197`, `145`

**`3rdparty/ringbuf/ringbuf.c` — vendored byte ring**
- `ringbuf_new` (mallocs `capacity+1`, no power-of-two rounding) — `ringbuf.c:50`
- `ringbuf_memcpy_into` (overflow branch L216; tail advance L233; full assert L234) — `ringbuf.c:211–238`

**`kitty/vt-parser.c` — ingest path (appears in Massif stacks)**
- `consume_normal` L236, `consume_input` L1377, `run_worker` L1432, `parse_worker` L1496, `alloc_vt_parser` L1565

**Configuration**
- `scrollback_lines` default `2000` — `kitty/options/definition.py:372–373`
- `scrollback_pager_history_size` default `0` (disabled) — `kitty/options/definition.py:406–407`
- MB→bytes conversion for the pager option — `kitty/options/utils.py:564–566`

**Observation harness (canonical entry point)**
- `parse_bytes` — `kitty_tests/__init__.py:30–36`
- `filled_history_buf` — `kitty_tests/__init__.py:184–189`
- `set_options` — `kitty_tests/__init__.py:223–231`
- `create_screen` — `kitty_tests/__init__.py:237–241`
- existing pager/scrollback test `test_pagerhist` (mirrored for raw-byte feeding) — `kitty_tests/screen.py:695–741`

**Build / toolchain**
- `dev.sh` entry — `dev.sh:9`
- build / launcher / `--debug` / `--sanitize` — `docs/build.rst:19, 22, 54, 58`
- `get_python_include_paths` L325; `get_python_flags` if/else L337+; `env_cflags` L515; `-march=native` L585; `--debug` (optimize L479/482, no-LTO L523, `-DKITTY_DEBUG_BUILD` L529, `-fno-omit-frame-pointer` L537); `-Werror` L491 — `setup.py`
- Go toolchain `1.22` — `go.mod:3`
- Python floor `>=3.8` — `pyproject.toml:2`
- CI Python matrix `3.8` (L26), `3.9` (L34), `3.10` (L30), highest `"3.11"` (L85) — `.github/workflows/ci.yml`

## 12. Cleanup and read-only verification

This investigation is read-only on the source repository. Every temporary artifact lived **outside**
the repository tree, in a private workspace, and the release build was restored bit-for-bit after the
one profiling detour that required a debug build.

**Private workspace (addresses predictable-name concerns).** The workspace `$WORK` was created with
`mktemp -d` + `chmod 700` (unpredictable name, owner-only `rwx`) **before** any script ran, and the
release artifacts were backed up into it — both commands, with their real output, appear in §2.6. All
scenario scripts (`scenarioA.py` … `scenarioF.py`, `units.py`, `mallocinfo.py`, `massif_target.py`),
the `sizeof_probe.c` helper (§3.1), and all captured outputs (`out_*.txt`, `massif.out.*`,
`msprint.*`) were written under `$WORK`, never inside the repository.

**Release build preserved across the profiling detour.** Before building the debug variant for Massif
(§8.4), the release artifacts were backed up (§2.6); afterward they were restored and verified
bit-for-bit identical to that backup, and the launcher still reports the same version:

```text
$ cmp kitty/fast_data_types.so "$WORK/release_backup/fast_data_types.so"  && echo IDENTICAL
IDENTICAL
$ cmp kitty/launcher/kitty      "$WORK/release_backup/launcher/kitty"     && echo IDENTICAL
IDENTICAL
$ cmp kitty/launcher/kitten     "$WORK/release_backup/launcher/kitten"    && echo IDENTICAL
IDENTICAL
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

**Commit provenance (acceptance checkout).** The observations characterize the required source commit
`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`. The acceptance checkout's `HEAD` is a **descendant** of it,
and the product tree is **byte-identical** between the two (the only difference is this deliverable
under `blitzy/`), so every cited `file:line` and observed value applies without change:

```text
$ git merge-base --is-ancestor 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD && echo "REQ is an ancestor of HEAD"
REQ is an ancestor of HEAD
$ git rev-parse HEAD
e6486adb6975d2c063659f6a17d6068bc55a157d
$ git diff --name-only 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD -- . ':(exclude)blitzy/**'
$ # (no output above — product tree byte-identical outside blitzy/)
```

**Final cleanup and clean-tree check.** After authoring, the entire private workspace is removed and
the repository is confirmed to contain exactly one change — this document:

```text
$ rm -rf "$WORK"                       # removes all temp scripts + profiler output
$ git status --porcelain
 M blitzy/documentation/kitty_815df1e210e0.md
```

No product source, header, test, configuration, build file, or vendored dependency is modified; no
observation script is committed; the only tracked change is the deliverable itself. The runtime facts
in this document reflect the default release build of commit
`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` on branch `kitty_815df1e210e0`.
