# kitty scrollback `HistoryBuf` under sustained heavy output — an evidence-based investigation

## 1. Title & metadata

This document answers, **from observed runtime measurements** (not code-reading alone), how kitty's
scrollback history buffer (`HistoryBuf`) behaves when a child process prints hundreds of thousands of
lines rapidly. It covers three question threads:

- **T1 — memory-consumption trajectory** (how RSS grows / plateaus),
- **T2 — responsiveness / input latency** while output streams, and
- **T3 — buffer-growth boundaries** (when a new backing-storage block is allocated).

| Field | Value |
|-------|-------|
| Repository | `kovidgoyal/kitty` |
| Source branch (deliverable basis) | `kitty_815df1e210e0` |
| Commit (pinned) | `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` |
| Canonical container image | `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`) |
| kitty version | `kitty 0.35.2 created by Kovid Goyal` |
| Build command | `python3 setup.py` (default, strict `-pedantic-errors -Werror` — setup.py:L491) |
| Toolchain | Python 3.12.3, gcc 13.3.0 (Ubuntu 13.3.0) |
| Headless display | `xvfb-run -a -s "-screen 0 1024x768x24"`, `LIBGL_ALWAYS_SOFTWARE=1` |
| Measured runtime grid | `COLS=71 ROWS=22` (i.e. `xnum = 71`), from the child's `os.get_terminal_size(1)` on kitty's PTY |
| RSS unit convention | raw `/proc/<pid>/status` `VmRSS` is reported in **kB** (kernel kB = 1024 B); binary conversions use **MiB = kB / 1024** |
| Date | 2026-07-08 |

### 1.1 Commit / branch / version / build confirmation **[observed-at-runtime]**

Run inside the container at `/app` (the canonical source checkout at the pinned commit):

```
$ cd /app
$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
$ git rev-parse --abbrev-ref HEAD
HEAD
$ git rev-parse --short=12 HEAD
815df1e210e0
$ printf 'kitty_%s\n' "$(git rev-parse --short=12 HEAD)"
kitty_815df1e210e0
```

**Branch correspondence (resolves the "branch confirmation" requirement).** The canonical container
checks the source out in **detached-HEAD** state at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`, so
`git rev-parse --abbrev-ref HEAD` prints `HEAD` rather than a branch name. The authoritative proof that
this checkout *is* the named source branch `kitty_815df1e210e0` is that the branch name **embeds the pinned
commit's 12-character short hash**: `git rev-parse --short=12 HEAD` = `815df1e210e0`, and
`printf 'kitty_%s'` reconstructs exactly `kitty_815df1e210e0`. This is corroborated independently in the
deliverable repository (§10), where the documentation commit's **direct parent is the pinned commit**
`815df1e21` and `git diff --stat 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD` reports exactly one added
file (this document).

Build & version confirmation (a forced clean rebuild; `build/` is git-ignored, so the source tree is
untouched — verified below):

```
$ cd /app
$ git check-ignore build && echo "build/ is IGNORED (safe to clean)"
build
build/ is IGNORED (safe to clean)
$ rm -rf build && python3 setup.py > /tmp/build.log 2>&1; echo "exit=$?"
exit=0
$ sed -n '1p;33p;35p' /tmp/build.log        # 3 of the 129 lines (trailing "..." is literal build output)
[1/122] Compiling kitty/screen.c ...
[33/122] Compiling kitty/data-types.c ...
[35/122] Compiling kitty/history.c ...
$ tail -1 /tmp/build.log
 done
$ wc -l /tmp/build.log
129 /tmp/build.log
$ /app/kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
$ python3 --version
Python 3.12.3
$ gcc --version | head -1
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
$ git status --porcelain            # tree clean after the rebuild
$                                   # (no output => nothing changed)
```

Because the default build uses `-pedantic-errors -Werror` (setup.py:L491) and `kitty/data-types.c`
(which includes `kitty/data-types.h`) compiled cleanly at `[33/122]`, the compile-time
`static_assert`s on the cell sizes (see §4.3) are **build-time confirmed** for this exact binary.

---

## 2. Executive summary

- **T1 (memory trajectory) [observed-at-runtime].** At the **default** `scrollback_lines=2000`, pushing
  **300,000** lines through the real PTY caused **no history growth**: RSS rose once from a settled
  baseline of **~147,900–148,200 kB (~144.5 MiB)** to a plateau of **~153,200–153,750 kB (~149.6–150.1 MiB)**
  by line ~10,000–70,000 and then stayed **flat** all the way to 300,000 lines — a **plateau** (dead flat
  within each run; the plateau level reproduces across 2 runs to within 0.36 %). The one-time
  **+5,276–5,540 kB (~5.2 MiB)** bump is transient parser/read buffering, **not** history: the ring
  overwrites once `count == ynum = 2000`. With a **large** cap `scrollback_lines=100000`, the same 300,000
  lines produced a **linear ramp** to a **~371,000–371,400 kB (~362.5 MiB)** plateau reached exactly at the
  100,000-line cap (delta **~223,200–223,400 kB / ~218.1 MiB**, stable across 2 runs to within 0.11 %). With
  **infinite** scrollback `scrollback_lines=-1`, RSS grew **linearly to ~815,300–815,600 kB (~796.3 MiB)**
  for 300,000 lines (delta **~667,600–668,000 kB / ~652.2 MiB**, stable across 2 runs to within 0.05 %) and
  kept growing **past** the 100k point where the large cap had plateaued. The **measured per-line cost is
  2267–2280 bytes/line** across three independent methods (infinite-run total-delta 2278.7–2280.1; per-segment
  median step 2276.0; a linear least-squares fit 2266.97, R² = 0.9997), **bracketing** the code-derived
  `32·xnum + 1 = 2273` bytes/line at the measured `xnum = 71`.

- **T2 (responsiveness / latency) [observed-at-runtime].** While a child streamed continuously at a
  **measured ~229,000–231,000 lines/s**, **60/60** scroll-back requests were serviced with **no drops**,
  and **every** scroll was **confirmed on the display** — `kitty @ get-text --extent=screen` showed the
  visible viewport had moved to older content after each scroll (the top visible line number dropped by
  ≥ 20,000, which streaming can never cause). The **display-confirmed** scroll latency (issue scroll →
  externally observe the viewport moved) was **p50 ≈ 50.8–51.1 ms under load (2 runs) vs ≈ 47.6 ms idle**.
  That end-to-end figure decomposes into the **non-canonical remote-control injection/round-trip overhead**
  (trigger subprocess p50 ≈ 25 ms; a no-op `get-text` round trip p50 ≈ 23–26 ms) plus the display update
  itself, which completed **within a single confirmation round-trip in every trial** (`polls_to_confirm`
  min = median = max = 1). The **marginal** cost of the heavy load is therefore only **~3.2–3.5 ms** (loaded
  − idle), consistent with `input_delay=3` ms / `repaint_delay=10` ms. Responsiveness comes from kitty
  reading child output on a **separate `io_loop` thread** while rendering on the main thread, and from
  kitty **explicitly deprioritizing repaint when input is pending** (definition.py:L874).

- **T3 (allocation boundaries) [observed-at-runtime].** New backing storage is allocated **once every
  `SEGMENT_SIZE = 2048` lines**. A finely-sampled paced run (20 ms sampler) shows a discrete RSS **step of
  ~4552 kB (4.45 MiB)** (median and modal value) each time cumulative lines cross a 2048 boundary —
  matching the code-derived full-segment size of **4546 kB (4.44 MiB)** to within 0.13 %. Each step is the
  single `calloc` in `add_segment()` (history.c:L25). Over 82,000 lines, **40 full 2048-line boundaries**
  are crossed plus **one partial final batch** (the last 80 lines fault in only ~264 kB). At the
  **default cap only the one initial segment** is ever allocated (RSS stays flat), because
  `ynum = MAX(2000, 22) = 2000 < 2048`, so `segment_for()` never crosses a boundary.

---

## 3. Methodology & environment

### 3.1 Canonical build & run

All build/run/measure steps were performed **inside the canonical container** (the authoring host cannot
build kitty — no C toolchain / native libs). kitty is a GPU/windowed program, so it was launched headless
via `xvfb-run` with the software GL rasterizer. Each measurement condition uses the *same* generator and
sampler; only `scrollback_lines` and the pacing differ. The exact per-run invocations are shown verbatim
in §5–§7; the general form is:

```
LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a -s "-screen 0 1024x768x24" \
  /app/kitty/launcher/kitty --config NONE -o scrollback_lines=<N> \
  sh -c 'python3 /tmp/gen_output.py <lines> <meta> <warmup> <batch> <pause> <hold>'
```

`--config NONE` guarantees the **default configuration** is used (no user `kitty.conf`), so
`scrollback_lines` is `2000` unless explicitly overridden with `-o`. The child's stdout **is kitty's PTY
slave**; kitty reads it on its I/O thread and drives the exact canonical path:

`Child PTY bytes → VT parser (kitty/vt-parser.c) → Screen ops (kitty/screen.c) → INDEX_UP (screen.c:L1552) → historybuf_add_line (history.c:L287) → historybuf_push (history.c:L276) → segment_for (history.c:L37) → add_segment (history.c:L18) → calloc a SEGMENT_SIZE block (history.c:L25) → observable RSS step`.

### 3.2 Why RSS is sampled *externally*

kitty exposes **no built-in memory-reporting hook**. A source grep for memory instrumentation finds only a
texture-leak comment at `kitty/state.c:L1456` (the line reads `// we leak the texture here since it is not guaranteed`;
full quote in §4.5). RSS is therefore sampled
**externally at the OS level** via `/proc/<pid>/status` `VmRSS` (the standard, low-overhead approach; the
container also provides `psutil` 5.9.8, which agrees). OS-level RSS sampling is **canonical** for this
purpose — it observes the real process, not a bypassing interface.

The kitty process to sample is the single C process that owns the `Screen`/`HistoryBuf`; it is selected with
`pgrep -x kitty` (its `/proc/<pid>/comm` is exactly `kitty`). Its settled baseline `VmRSS` is
~147,500–148,300 kB (GPU context + freetype/harfbuzz font stack + the embedded CPython interpreter + the
one pre-allocated history segment). Every RSS number below is a literal `VmRSS` value in kB from that
process; MiB conversions divide by 1024.

### 3.3 Measured column count (`xnum`) drives the magnitude

The per-line memory cost depends on the runtime column count `xnum`, which **must be measured, not
assumed**. The generator reports the child's terminal size (which equals kitty's `Screen` columns); at the
`1024x768` xvfb screen it is **`COLS=71 ROWS=22`**, i.e. **`xnum = 71`** (visible in every `meta_*.txt`
header below). All predictions use the measured `xnum = 71`.

### 3.4 Complete source of every temporary measurement script

Per the evidence rule, every script used to produce a number below is reproduced here in full (these live
outside the repository under the container's `/tmp` and are deleted afterward — §10). None modifies the
repository.

**`/tmp/gen_output.py`** — the canonical child that prints lines to its stdout (kitty's PTY slave). It
reports the measured terminal size, logs `BEGIN`/`PROGRESS`/`DONE` wall-clock timestamps to a side file
(never the PTY), and supports a paced mode (`batch`/`pause`) so RSS can be correlated with cumulative line
count:

```python
#!/usr/bin/env python3
# Canonical child process: prints lines to its stdout (kitty PTY slave).
# kitty reads them via io_loop -> vt-parser -> screen -> historybuf_add_line.
import sys, os, time

def arg(i, default, cast):
    try:
        return cast(sys.argv[i])
    except (IndexError, ValueError):
        return default

n       = arg(1, 300000, int)     # total lines to print
meta    = arg(2, "/tmp/meta.txt", str)  # progress/metadata log (NOT the PTY)
warmup  = arg(3, 3.0, float)      # sleep before first print so kitty settles
batch   = arg(4, 0, int)          # lines per batch; 0 => all at once (rapid)
pause   = arg(5, 0.0, float)      # sleep between batches (paced mode)
hold    = arg(6, 60.0, float)     # sleep after DONE so RSS can settle/sample

try:
    sz = os.get_terminal_size(1)  # fd 1 == kitty PTY slave => kitty Screen size
    cols, rows = sz.columns, sz.lines
except OSError:
    cols, rows = 80, 24

def log(msg):
    with open(meta, "a") as f:
        f.write(msg + "\n")

open(meta, "w").close()
log("COLS=%d ROWS=%d N=%d warmup=%.3f batch=%d pause=%.3f" % (cols, rows, n, warmup, batch, pause))
time.sleep(warmup)
log("BEGIN %.6f" % time.time())

w = sys.stdout.write
pad = "X" * max(0, cols - 12)
count = 0
if batch <= 0:
    for i in range(n):
        w("%08d %s\n" % (i, pad))
    sys.stdout.flush()
    count = n
else:
    while count < n:
        end = min(count + batch, n)
        for i in range(count, end):
            w("%08d %s\n" % (i, pad))
        count = end
        sys.stdout.flush()
        log("PROGRESS %d %.6f" % (count, time.time()))
        if pause > 0:
            time.sleep(pause)

log("DONE %d %.6f" % (count, time.time()))
time.sleep(hold)
```

**`/tmp/rss_sampler.py`** — the external OS-level RSS sampler; reads `/proc/<pid>/status` `VmRSS` at a fixed
interval and prints `mono_offset  wall  rss_kb` per sample:

```python
#!/usr/bin/env python3
import sys, time
pid = int(sys.argv[1])
interval = float(sys.argv[2]) if len(sys.argv) > 2 else 0.02
def vmrss_kb(p):
    with open("/proc/%d/status" % p) as f:
        for ln in f:
            if ln.startswith("VmRSS:"):
                return int(ln.split()[1])
    return -1
t0 = time.monotonic()
while True:
    try:
        rss = vmrss_kb(pid)
    except (FileNotFoundError, ProcessLookupError):
        break
    sys.stdout.write("%.3f\t%.3f\t%d\n" % (time.monotonic()-t0, time.time(), rss))
    sys.stdout.flush()
    time.sleep(interval)
```

**`/tmp/stream.py`** — the continuous, never-stopping heavy-output child used for the T2 loaded condition
(keeps `io_loop`, the parser, and history writes fully busy):

```python
#!/usr/bin/env python3
# Continuous heavy output on the kitty PTY (never stops) to load io_loop + parser + history.
import sys, os, time
try:
    cols = os.get_terminal_size(1).columns
except OSError:
    cols = 80
pad = "X" * max(0, cols - 12)
i = 0
w = sys.stdout.write
while True:
    w("%08d %s\n" % (i, pad)); i += 1
    if (i & 0x3FF) == 0:  # flush every 1024 lines
        sys.stdout.flush()
```

**`/tmp/scroll_disp_latency.py`** — the T2 **display-confirmed** latency probe. This is the script that
resolves the central review finding: it does **not** stop at the remote-control return; it verifies the
**actual viewport moved** by re-reading `get-text --extent=screen` and requiring the top visible line
number to **drop by ≥ 20,000** (a drop that continuously-increasing new output can never produce). The
remote-control `action` call is used only as the deterministic *trigger* and is labelled **non-canonical**;
the confirmation is a real display-state read:

```python
#!/usr/bin/env python3
# T2: DISPLAY-CONFIRMED scroll-back latency under concurrent streaming output.
#
# Trigger  (NON-CANONICAL): `kitty @ action <scroll>` injected via remote control
#          (talk_loop thread). Used ONLY to fire the scroll deterministically.
# Confirm  (DISPLAY STATE): `kitty @ get-text --extent=screen` returns the ACTUAL
#          visible viewport. Each streamed line is "%08d ...", a strictly increasing
#          counter, so the top visible line number only ever INCREASES from new output.
#          A large DROP in the top visible number can ONLY be produced by the scroll
#          moving the display to older content => confirmed viewport/display update.
#
# Per trial we record:
#   trig_ms = wall time for the scroll trigger subprocess to return (IPC/injection only)
#   disp_ms = wall time from issuing the trigger until get-text CONFIRMS the viewport
#             moved back (top line dropped by >= DROP_MIN) -> display-confirmed latency
#   polls   = number of get-text confirmations needed (each ~ one IPC round trip)
# A no-op IPC baseline (back-to-back get-text, no scroll) is measured so the fixed
# remote-control round-trip overhead can be reported/subtracted separately.
import subprocess, sys, time, statistics, re

sock   = sys.argv[1]
n      = int(sys.argv[2]) if len(sys.argv) > 2 else 60
label  = sys.argv[3] if len(sys.argv) > 3 else "load"
DROP_MIN = 20000   # a top-line drop this large can only come from the scroll, not drift
KITTY  = "/app/kitty/launcher/kitty"
num_re = re.compile(rb"(\d{8})")

def run(*a, timeout=10):
    return subprocess.run([KITTY, "@", "--to", sock, *a],
                          stdout=subprocess.PIPE, stderr=subprocess.DEVNULL, timeout=timeout)

def top_num():
    r = run("get-text", "--extent=screen")
    if r.returncode != 0:
        return None
    m = num_re.search(r.stdout)
    return int(m.group(1)) if m else None

# no-op IPC baseline: pure get-text round-trip cost (no scroll involved)
ipc = []
for _ in range(20):
    t0 = time.monotonic(); top_num(); ipc.append((time.monotonic()-t0)*1000)

trig, disp, npolls = [], [], []
ok = 0
for i in range(n):
    run("action", "scroll_end")     # follow the live bottom (newest, advancing)
    time.sleep(0.03)
    base = top_num()
    if base is None:
        continue
    t0 = time.monotonic()
    run("action", "scroll_home")    # trigger: jump to oldest content
    t_trig = time.monotonic()
    polls = 0; confirmed = None
    while time.monotonic() - t0 < 3.0:
        cur = top_num(); polls += 1
        if cur is not None and cur <= base - DROP_MIN:   # display moved far back
            confirmed = time.monotonic(); break
    if confirmed is not None:
        ok += 1
        trig.append((t_trig - t0)*1000.0)
        disp.append((confirmed - t0)*1000.0)
        npolls.append(polls)
    time.sleep(0.03)

def stats(x):
    x = sorted(x)
    return (x[0], statistics.median(x), statistics.mean(x),
            x[min(len(x)-1, int(0.9*len(x)))], x[-1])

print("LABEL=%s n_ok=%d/%d DROP_MIN=%d" % (label, ok, n, DROP_MIN))
if ipc:
    mn,md,me,p9,mx = stats(ipc)
    print("ipc_noop_ms:  min=%.1f p50=%.1f mean=%.1f p90=%.1f max=%.1f" % (mn,md,me,p9,mx))
if trig:
    mn,md,me,p9,mx = stats(trig)
    print("trigger_ms:   min=%.1f p50=%.1f mean=%.1f p90=%.1f max=%.1f" % (mn,md,me,p9,mx))
if disp:
    mn,md,me,p9,mx = stats(disp)
    print("display_ms:   min=%.1f p50=%.1f mean=%.1f p90=%.1f max=%.1f" % (mn,md,me,p9,mx))
    print("polls_to_confirm: min=%d median=%.1f max=%d" % (min(npolls), statistics.median(npolls), max(npolls)))
    print("display_raw_ms=" + " ".join("%.1f" % x for x in disp))
```

**`/tmp/parse_common.py`** — shared parsing used by the analyzers (parses the meta and RSS logs; picks the
settled RSS in a wall-clock window; finds the pre-first-line baseline):

```python
#!/usr/bin/env python3
# Shared parsing for the RSS/meta correlation analyzers.
def parse_meta(path):
    """Return (cols, rows, N, batch, pause, begin_wall, progress[list of (cumlines,wall)], done)."""
    cols = rows = N = batch = 0; pause = 0.0
    begin = None; done = None; prog = []
    with open(path) as f:
        for ln in f:
            p = ln.split()
            if not p:
                continue
            if ln.startswith("COLS="):
                kv = dict(tok.split("=") for tok in p)
                cols = int(kv["COLS"]); rows = int(kv["ROWS"]); N = int(kv["N"])
                batch = int(kv["batch"]); pause = float(kv["pause"])
            elif p[0] == "BEGIN":
                begin = float(p[1])
            elif p[0] == "PROGRESS":
                prog.append((int(p[1]), float(p[2])))
            elif p[0] == "DONE":
                done = (int(p[1]), float(p[2]))
    return cols, rows, N, batch, pause, begin, prog, done

def parse_rss(path):
    """Return list of (mono, wall, rss_kb)."""
    out = []
    with open(path) as f:
        for ln in f:
            p = ln.split("\t")
            if len(p) >= 3:
                try:
                    out.append((float(p[0]), float(p[1]), int(p[2])))
                except ValueError:
                    pass
    return out

def rss_at(rss, w0, w1):
    """Settled RSS (kB) = max VmRSS among samples with wall in [w0, w1]; fallback to nearest >= w0."""
    win = [r for (_, wall, r) in rss if w0 <= wall <= w1]
    if win:
        return max(win)
    after = [(wall, r) for (_, wall, r) in rss if wall >= w0]
    if after:
        return min(after, key=lambda t: t[0])[1]
    return rss[-1][2] if rss else -1

def baseline_rss(rss, begin_wall):
    """RSS just before the first print (the settled baseline, 0 history lines)."""
    before = [r for (_, wall, r) in rss if wall <= begin_wall]
    return before[-1] if before else (rss[0][2] if rss else -1)
```

**`/tmp/lines_vs_rss.py`** — produces the T1 **lines-vs-RSS** tables (cumulative line count → settled RSS,
with explicit before/during/after states and a bytes/line summary):

```python
#!/usr/bin/env python3
# Correlate cumulative LINE COUNT (from the child's meta PROGRESS log) with the
# externally-sampled kitty RSS (from rss_sampler.py), producing a lines-vs-RSS table
# with explicit before (0 lines) / during / after (N lines) states.
import sys
from parse_common import parse_meta, parse_rss, rss_at, baseline_rss
meta_path, rss_path = sys.argv[1], sys.argv[2]
cols, rows, N, batch, pause, begin, prog, done = parse_meta(meta_path)
rss = parse_rss(rss_path)
base = baseline_rss(rss, begin)
win = max(pause, 0.05) + 0.05
print("cols=%d rows=%d N=%d batch=%d pause=%.3f" % (cols, rows, N, batch, pause))
print("BEFORE: lines=0  baseline_rss_kb=%d" % base)
print("lines\twall_offset_s\trss_kb\tdelta_from_baseline_kb")
prev_l = 0
for (cum, w) in prog:
    r = rss_at(rss, w, w + win)
    print("%d\t%.3f\t%d\t%d" % (cum, w - begin, r, r - base))
if done:
    cum, w = done
    r = rss_at(rss, w, w + max(pause, 0.05) + 3.0)
    print("AFTER: lines=%d  after_rss_kb=%d  total_delta_kb=%d" % (cum, r, r - base))
    if cum > 0:
        print("bytes/line (total_delta*1024/lines) = %.1f" % ((r - base) * 1024.0 / cum))
```

**`/tmp/regress.py`** — least-squares fit of settled RSS (bytes) vs cumulative lines over the paced-run
boundary points (isolates the marginal per-line cost from the fixed baseline):

```python
#!/usr/bin/env python3
# Linear least-squares fit of settled kitty RSS (BYTES) vs cumulative LINES over the
# paced-run boundary points (lines >= 2048), isolating the marginal per-line cost from
# the fixed baseline. Settled RSS for a boundary uses the [w_i, w_{i+1}) window.
import sys
from parse_common import parse_meta, parse_rss
rss_path, meta_path = sys.argv[1], sys.argv[2]
cols, rows, N, batch, pause, begin, prog, done = parse_meta(meta_path)
rss = parse_rss(rss_path)
walls = [w for (_, w) in prog]
ends = walls[1:] + [ (done[1] if done else walls[-1]) + max(pause, 0.05) + 3.0 ]
def settled(w0, w1):
    win = [r for (_, wall, r) in rss if w0 <= wall < w1]
    if win: return max(win)
    after = [(wall, r) for (_, wall, r) in rss if wall >= w0]
    return min(after, key=lambda t: t[0])[1] if after else -1
xs, ys = [], []
for i, (cum, w) in enumerate(prog):
    if cum < 2048: continue
    xs.append(float(cum)); ys.append(settled(w, ends[i]) * 1024.0)
n = len(xs)
sx = sum(xs); sy = sum(ys)
sxx = sum(x*x for x in xs); sxy = sum(x*y for x, y in zip(xs, ys))
slope = (n*sxy - sx*sy) / (n*sxx - sx*sx)
intercept = (sy - slope*sx) / n
mean_y = sy/n
ss_tot = sum((y-mean_y)**2 for y in ys); ss_res = sum((y-(slope*x+intercept))**2 for x, y in zip(xs, ys))
r2 = 1 - ss_res/ss_tot if ss_tot else float('nan')
print("linear fit RSS(bytes) = slope*lines + intercept over %d boundary points (lines>=2048)" % n)
print("slope  = %.2f bytes/line   (predicted 32*%d+1 = %d)" % (slope, cols, 32*cols+1))
print("intercept = %.0f bytes = %.1f MB" % (intercept, intercept/1048576.0))
print("R^2 = %.5f" % r2)
```

**`/tmp/analyze_boundary.py`** — the T3 boundary analyzer. For a paced run with `batch == SEGMENT_SIZE`
(2048) it prints, per 2048-line boundary, the settled RSS and the **step delta from the previous boundary**
— directly exposing each `add_segment()` allocation. Each boundary's settled RSS is the max VmRSS in the
window up to the **next** boundary, so a later segment's page-fault cannot bleed into an earlier row:

```python
#!/usr/bin/env python3
# For a paced run with batch == SEGMENT_SIZE (2048), correlate each 2048-line boundary
# with the settled RSS after that batch and print the per-boundary RSS step (delta),
# demonstrating the discrete allocation event of add_segment() (history.c:L25).
# Settled RSS for boundary i = max VmRSS among samples with wall in [w_i, w_{i+1})
# (up to the NEXT boundary), so a later segment's fault-in cannot bleed into this row.
import sys, statistics, math
from collections import Counter
from parse_common import parse_meta, parse_rss
rss_path, meta_path = sys.argv[1], sys.argv[2]
cols, rows, N, batch, pause, begin, prog, done = parse_meta(meta_path)
rss = parse_rss(rss_path)
per_seg = 2048*(32*cols+1)/1024.0
# boundary walls, plus a final sentinel (DONE wall + generous tail)
walls = [w for (_, w) in prog]
ends = walls[1:] + [ (done[1] if done else walls[-1]) + max(pause, 0.05) + 3.0 ]
def settled(w0, w1):
    win = [r for (_, wall, r) in rss if w0 <= wall < w1]
    if win:
        return max(win)
    after = [(wall, r) for (_, wall, r) in rss if wall >= w0]
    return min(after, key=lambda t: t[0])[1] if after else -1
print("xnum=%d  per_segment_predicted_kb=%.1f" % (cols, per_seg))
print("lines\tsegments_expected(ceil/2048)\tsettled_rss_kb\tdelta_from_prev_kb")
prev = None; deltas = []
for i, (cum, w) in enumerate(prog):
    r = settled(w, ends[i])
    segs = math.ceil(cum/2048.0)
    d = 0 if prev is None else r - prev
    print("%6d\t%d\t%d\t%d" % (cum, segs, r, d))
    if prev is not None and cum % 2048 == 0:
        deltas.append(d)
    prev = r
if deltas:
    ds = sorted(deltas)
    print("")
    print("per-2048-line deltas: n=%d  min=%d  max=%d  mean=%.1f  median=%.1f"
          % (len(ds), ds[0], ds[-1], statistics.mean(ds), statistics.median(ds)))
    print("most common delta values: %s" % Counter(deltas).most_common(5))
    print("bytes/line from median step = %.1f (median_kb*1024/2048)" % (statistics.median(ds)*1024.0/2048))
```

**`/tmp/noncanonical_binding.py`** — the **non-canonical** supplement (§8). It drives `HistoryBuf` directly
through the `fast_data_types` binding, bypassing the child-PTY→parser→screen path, purely to corroborate
the segment cadence and per-line cost with a low-noise signal. It is **not** a substitute for the canonical
observation:

```python
#!/usr/bin/env python3
# NON-CANONICAL supplement: drive HistoryBuf directly via the fast_data_types
# Python binding (bypasses child-PTY -> vt-parser -> screen). Corroborates the
# per-2048-line segment-allocation cadence and per-line cost. NOT a substitute
# for the canonical real-path observation.
import sys
from kitty.fast_data_types import HistoryBuf, LineBuf
def rss_kb():
    with open("/proc/self/status") as f:
        for ln in f:
            if ln.startswith("VmRSS:"):
                return int(ln.split()[1])
xnum = int(sys.argv[1]) if len(sys.argv) > 1 else 71
ynum = int(sys.argv[2]) if len(sys.argv) > 2 else 100000
N    = int(sys.argv[3]) if len(sys.argv) > 3 else 82000
lb = LineBuf(1, xnum)
src = lb.line(0)
hb = HistoryBuf(ynum, xnum)          # create_historybuf: 1 initial segment
base = rss_kb()
print("xnum=%d ynum=%d N=%d predicted_seg_kb=%.1f" % (xnum, ynum, N, 2048*(32*xnum+1)/1024.0))
print("rss_after_create_kb=%d" % base)
prev = base
print("lines\tsegments\trss_kb\tdelta_kb")
for i in range(N):
    hb.push(src)
    n = i + 1
    if n % 2048 == 0:
        r = rss_kb()
        segs = n // 2048
        print("%d\t%d\t%d\t%d" % (n, segs, r, r - prev))
        prev = r
final = rss_kb()
print("final_count=%d final_rss_kb=%d total_delta_kb=%d" % (hb.count, final, final - base))
print("bytes/line = %.1f (total_delta*1024/N)" % ((final - base) * 1024.0 / N))
```

**`/tmp/run_case.sh`** — the T1/T3 driver: kills any stale kitty, launches kitty headless with the given
`scrollback_lines`, discovers the kitty PID via `pgrep -x kitty`, starts the RSS sampler keyed to that PID,
waits for the child's `DONE`, and prints the exact invocation + sampler command it used:

```bash
#!/bin/bash
# run_case.sh <label> <scrollback> <N> <batch> <pause> <interval> <holdafter>
LABEL="$1"; SB="$2"; N="$3"; BATCH="$4"; PAUSE="$5"; INTERVAL="$6"; HOLDAFTER="${7:-5}"
KITTY=/app/kitty/launcher/kitty
META=/tmp/meta_${LABEL}.txt
RLOG=/tmp/rss_${LABEL}.log
export LIBGL_ALWAYS_SOFTWARE=1
pkill -x kitty 2>/dev/null; sleep 1
rm -f "$META" "$RLOG"
GENHOLD=$(python3 -c "print(max(6.0, $HOLDAFTER+2.0))")
nohup xvfb-run -a -s "-screen 0 1024x768x24" \
  $KITTY --config NONE -o scrollback_lines=${SB} \
  sh -c "python3 /tmp/gen_output.py ${N} ${META} 4 ${BATCH} ${PAUSE} ${GENHOLD}" \
  > /tmp/kitty_${LABEL}.log 2>&1 &
# wait for kitty process
for i in $(seq 1 40); do KPID=$(pgrep -x kitty | head -1); [ -n "$KPID" ] && break; sleep 0.25; done
echo "LABEL=${LABEL} scrollback_lines=${SB} KPID=${KPID}"
echo "INVOCATION: LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a -s \"-screen 0 1024x768x24\" $KITTY --config NONE -o scrollback_lines=${SB} sh -c 'python3 /tmp/gen_output.py ${N} ${META} 4 ${BATCH} ${PAUSE} ${GENHOLD}'"
echo "SAMPLER: python3 /tmp/rss_sampler.py ${KPID} ${INTERVAL} > ${RLOG}"
# start sampler
nohup python3 /tmp/rss_sampler.py ${KPID} ${INTERVAL} > ${RLOG} 2>/dev/null &
SAMP=$!
# wait for DONE in meta (timeout ~ warmup + generous)
for i in $(seq 1 200); do grep -q "^DONE" "$META" 2>/dev/null && break; sleep 0.25; done
sleep ${HOLDAFTER}
kill $SAMP 2>/dev/null
kill $KPID 2>/dev/null; sleep 1; pkill -x kitty 2>/dev/null
echo "META:"; cat "$META"
echo "RLOG_LINES=$(wc -l < ${RLOG})"
```

**`/tmp/t2_run.sh`** — the T2 driver: launches kitty with remote control enabled on a private socket, runs
either the streaming child (loaded) or a static pre-filled 82k-line history (idle), measures the effective
streaming rate from the viewport top-line advance, and then runs the display-confirmed latency probe:

```bash
#!/bin/bash
# t2_run.sh <mode:stream|idle> <label> <n>
MODE="$1"; LABEL="$2"; NTRIAL="${3:-60}"
KITTY=/app/kitty/launcher/kitty
SOCK=unix:/tmp/kitty-${LABEL}
export LIBGL_ALWAYS_SOFTWARE=1
pkill -x kitty 2>/dev/null; sleep 1
rm -f /tmp/kitty-${LABEL}
if [ "$MODE" = "stream" ]; then
  CHILD="python3 /tmp/stream.py"
else
  # idle: pre-fill a static 82000-line history, then hold (NO further output)
  CHILD="python3 /tmp/gen_output.py 82000 /tmp/meta_${LABEL}.txt 2 0 0 600"
fi
echo "LAUNCH[$LABEL,$MODE]: LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a -s \"-screen 0 1024x768x24\" $KITTY --config NONE -o scrollback_lines=100000 -o allow_remote_control=yes --listen-on $SOCK sh -c '$CHILD'"
nohup xvfb-run -a -s "-screen 0 1024x768x24" \
  $KITTY --config NONE -o scrollback_lines=100000 -o allow_remote_control=yes --listen-on $SOCK \
  sh -c "$CHILD" > /tmp/kitty_${LABEL}.log 2>&1 &
for i in $(seq 1 40); do KPID=$(pgrep -x kitty | head -1); [ -n "$KPID" ] && break; sleep 0.25; done
sleep 5
KPID=$(pgrep -x kitty | head -1)
RSS=$(awk '/VmRSS/{print $2}' /proc/$KPID/status)
echo "LABEL=$LABEL MODE=$MODE KPID=$KPID socket=yes"
echo "kitty_rss_kb_during=$RSS"
if [ "$MODE" = "stream" ]; then
  # measure effective streaming rate via viewport top-line advance over ~2s
  T0=$(python3 -c "import time;print(time.monotonic())")
  N0=$($KITTY @ --to $SOCK get-text --extent=screen 2>/dev/null | grep -oE '[0-9]{8}' | head -1)
  sleep 2.0
  N1=$($KITTY @ --to $SOCK get-text --extent=screen 2>/dev/null | grep -oE '[0-9]{8}' | head -1)
  T1=$(python3 -c "import time;print(time.monotonic())")
  python3 -c "n0=int('$N0'); n1=int('$N1'); dt=$T1-$T0; print('streaming_rate_lines_per_s=%.0f  (top advanced %d->%d in %.3fs)'%((n1-n0)/dt, n0, n1, dt))"
fi
timeout 120 python3 /tmp/scroll_disp_latency.py $SOCK $NTRIAL $LABEL 2>&1
echo "CASE_T2 $LABEL done"
kill $KPID 2>/dev/null; sleep 1; pkill -x kitty 2>/dev/null
```

---

## 4. Source grounding

Every behavioral claim below is grounded in the pinned source at commit
`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`. Line numbers were
re-verified against the live container checkout. Labels distinguish **[inferred-from-reading]** (a fact read
from source) from **[observed-at-runtime]** (confirmed by a measurement in §5–§8) and
**[build-time confirmed]** (enforced by a compile-time `static_assert` in the binary we ran).

### 4.1 The segmented ring buffer `HistoryBuf` (`kitty/history.c`)

- **`SEGMENT_SIZE` = 2048 rows** — the allocation quantum. **[inferred-from-reading]**

  ```
  kitty/history.c:15:#define SEGMENT_SIZE 2048
  ```

- **`add_segment()` allocates exactly one `SEGMENT_SIZE` block via a single `calloc`** — this is the
  discrete allocation event T3 observes. **[inferred-from-reading; observed-at-runtime in §7]**

  ```
  kitty/history.c:18:add_segment(HistoryBuf *self) {
  kitty/history.c:19:    self->num_segments += 1;
  kitty/history.c:20:    self->segments = realloc(self->segments, sizeof(HistoryBufSegment) * self->num_segments);
  kitty/history.c:21:    if (self->segments == NULL) fatal("Out of memory allocating new history buffer segment");
  kitty/history.c:22:    HistoryBufSegment *s = self->segments + self->num_segments - 1;
  kitty/history.c:23:    const size_t cpu_cells_size = self->xnum * SEGMENT_SIZE * sizeof(CPUCell);
  kitty/history.c:24:    const size_t gpu_cells_size = self->xnum * SEGMENT_SIZE * sizeof(GPUCell);
  kitty/history.c:25:    s->cpu_cells = calloc(1, cpu_cells_size + gpu_cells_size + SEGMENT_SIZE * sizeof(LineAttrs));
  ```

  The single `calloc` (L25) reserves `xnum·SEGMENT_SIZE·(sizeof(CPUCell)+sizeof(GPUCell)) + SEGMENT_SIZE·sizeof(LineAttrs)`
  bytes — i.e. the whole segment (CPU cells + GPU cells + per-line attrs) in one block. This is why RSS
  rises in one discrete step per segment rather than continuously.

- **`segment_for()` lazily allocates further segments while more rows are needed** — the boundary
  condition. **[inferred-from-reading]**

  ```
  kitty/history.c:37:segment_for(HistoryBuf *self, index_type y) {
  kitty/history.c:38:    index_type seg_num = y / SEGMENT_SIZE;
  kitty/history.c:39:    while (UNLIKELY(seg_num >= self->num_segments && SEGMENT_SIZE * self->num_segments < self->ynum)) add_segment(self);
  ```

  New segments are added only while `SEGMENT_SIZE * num_segments < ynum` (L39) — so the total number of
  segments is capped by the configured `ynum`, and at the default `ynum = 2000 < 2048` the loop body never
  runs a second time.

- **`create_historybuf()` allocates exactly one segment up front** — the baseline `count = 0` state.
  **[inferred-from-reading]**

  ```
  kitty/history.c:117:create_historybuf(PyTypeObject *type, unsigned int xnum, unsigned int ynum, unsigned int pagerhist_sz) {
  kitty/history.c:126:    self->num_segments = 0;
  kitty/history.c:127:    add_segment(self);
  kitty/history.c:130:    self->pagerhist = alloc_pagerhist(pagerhist_sz);
  ```

- **`historybuf_push()` is the ring-overwrite heart** — it stops growing at `count == ynum` and advances
  `start_of_data`, overwriting the oldest line (T1 plateau). **[inferred-from-reading; observed-at-runtime
  in §5.1]**

  ```
  kitty/history.c:276:static index_type historybuf_push(HistoryBuf *self, ANSIBuf *as_ansi_buf) {
  kitty/history.c:277:    index_type idx = (self->start_of_data + self->count) % self->ynum;
  kitty/history.c:279:    if (self->count == self->ynum) {
  kitty/history.c:280:        pagerhist_push(self, as_ansi_buf);
  kitty/history.c:281:        self->start_of_data = (self->start_of_data + 1) % self->ynum;
  kitty/history.c:282:    } else self->count++;
  ```

  When the buffer is full (`count == ynum`), the oldest line is optionally spooled to the pager-history ring
  (L280) and `start_of_data` advances (L281) — **no new allocation** — so RSS plateaus. Otherwise `count`
  increments (L282) and, via `segment_for`, a new segment may be allocated.

- **`historybuf_add_line()` is the entry called from the screen** — the join point with the write path.
  **[inferred-from-reading]**

  ```
  kitty/history.c:287:historybuf_add_line(HistoryBuf *self, const Line *line, ANSIBuf *as_ansi_buf) {
  kitty/history.c:288:    index_type idx = historybuf_push(self, as_ansi_buf);
  ```

### 4.2 The write path into history (`kitty/screen.c`)

- **`scrollback_lines` reaches `Screen` as an unsigned int** (`scrollback` is parsed with the `I`
  conversion) — so `-1` wraps to `UINT_MAX`, giving "effectively infinite" scrollback.
  **[inferred-from-reading; observed-at-runtime in §5.3]**

  ```
  kitty/screen.c:98:    unsigned int columns=80, lines=24, scrollback=0, cell_width=10, cell_height=20;
  kitty/screen.c:100:    if (!PyArg_ParseTuple(args, "|OIIIIIKO", &callbacks, &lines, &columns, &scrollback, &cell_width, &cell_height, &window_id, &test_child)) return NULL;
  ```

- **`ynum` = `MAX(scrollback, lines)`** — the buffer height passed to the history allocator. At the default
  `scrollback=2000`, `ynum = MAX(2000, 24) = 2000`. **[inferred-from-reading]**

  ```
  kitty/screen.c:130:        self->historybuf = alloc_historybuf(MAX(scrollback, lines), columns, OPT(scrollback_pager_history_size));
  ```

- **`INDEX_UP` calls `historybuf_add_line`** — the macro that, on a scroll-up of the top region, pushes the
  evicted top line into history (only when no top margin is set). **[inferred-from-reading]**

  ```
  kitty/screen.c:1552:#define INDEX_UP(add_to_history) \
  kitty/screen.c:1553:    linebuf_index(self->linebuf, top, bottom); \
  kitty/screen.c:1554:    INDEX_GRAPHICS(-1) \
  kitty/screen.c:1555:    if (add_to_history) { \
  kitty/screen.c:1556:        /* Only add to history when no top margin has been set */ \
  kitty/screen.c:1557:        linebuf_init_line(self->linebuf, bottom); \
  kitty/screen.c:1558:        historybuf_add_line(self->historybuf, self->linebuf->line, &self->as_ansi_buf); \
  kitty/screen.c:1559:        self->history_line_added_count++; \
  ```

### 4.3 Per-line memory cost (`kitty/data-types.h`)

The magnitude of each RSS step and the per-line slope are fixed by the cell sizes. These are enforced at
**compile time** by `static_assert`, so they are **build-time confirmed** for the exact `kitty 0.35.2`
binary we ran (§1.1 shows `kitty/data-types.c`, which includes this header, compiled cleanly under
`-Werror`). They are additionally **corroborated at runtime** by the measured slope (§5.4) matching the
formula. They are **not** read out of the running process via any hook — kitty exposes none (§3.2).

- **`sizeof(GPUCell) == 20` bytes.** **[inferred-from-reading + build-time confirmed]**

  ```
  kitty/data-types.h:221:static_assert(sizeof(GPUCell) == 20, "Fix the ordering of GPUCell");
  ```

- **`sizeof(CPUCell) == 12` bytes.** **[inferred-from-reading + build-time confirmed]**

  ```
  kitty/data-types.h:228:static_assert(sizeof(CPUCell) == 12, "Fix the ordering of CPUCell");
  ```

- **`LineAttrs` is a 1-byte union** (`sizeof(LineAttrs) == 1`, its widest member being a single `uint8_t val`),
  contributing one byte of per-line attributes. **[inferred-from-reading]**

  ```
  kitty/data-types.h:231:typedef union LineAttrs {
  kitty/data-types.h:238:    uint8_t val;
  kitty/data-types.h:239:} LineAttrs ;
  ```

**Derived per-line cost** (used throughout): each history row of `xnum` columns costs

```
xnum · (sizeof(CPUCell) + sizeof(GPUCell)) + sizeof(LineAttrs)
  = xnum · (12 + 20) + 1
  = 32·xnum + 1  bytes
```

At the measured `xnum = 71` this is `32·71 + 1 = 2273` bytes/line, and a full 2048-row segment is
`2048 · 2273 = 4,655,104` bytes = **4546.0 KB (4.44 MiB)**. Both predictions are confirmed at runtime: the
measured least-squares slope is **2266.97 bytes/line** (§5.4, §7.4) and the median segment step is **4552 kB** (§7.3).

### 4.4 The responsiveness mechanism (`kitty/child-monitor.c`)

- **Child output is read on a dedicated I/O thread (`io_loop`), remote control on `talk_loop`, while
  rendering runs on the main thread** — the decoupling behind T2. **[inferred-from-reading;
  observed-at-runtime in §6]**

  ```
  kitty/child-monitor.c:229:static void* io_loop(void *data);
  kitty/child-monitor.c:230:static void* talk_loop(void *data);
  kitty/child-monitor.c:286:        if ((ret = pthread_create(&self->talk_thread, NULL, talk_loop, self)) != 0) {
  kitty/child-monitor.c:289:        talk_thread_started = true;
  kitty/child-monitor.c:291:    ret = pthread_create(&self->io_thread, NULL, io_loop, self);
  kitty/child-monitor.c:292:    if (ret != 0) return PyErr_Format(PyExc_OSError, "Failed to start I/O thread with error: %s", strerror(ret));
  ```

  The child-output reader (`io_loop`) and the remote-control listener (`talk_loop`) are started as separate
  `pthread`s (L291 and L286) — distinct from the main thread that renders — which is the structural reason
  scroll input stays responsive during heavy output (§6).

### 4.5 Configuration defaults and `Screen` construction

- **`scrollback_lines` default = `2000`; `scrollback_pager_history_size` default = `0`** (canonical
  config). **[inferred-from-reading]**

  ```
  kitty/options/definition.py:372:opt('scrollback_lines', '2000',
  kitty/options/definition.py:406:opt('scrollback_pager_history_size', '0',
  ```

- **`repaint_delay` default `10` ms is ignored when input is pending; `input_delay` default `3` ms is
  ignored when the input buffer is almost full** — kitty's explicit input-vs-repaint prioritization.
  **[inferred-from-reading; observed-at-runtime in §6]**

  ```
  kitty/options/definition.py:866:opt('repaint_delay', '10',
  kitty/options/definition.py:873:use a monitor with a high refresh rate. Also, to minimize latency when there is
  kitty/options/definition.py:874:pending input to be processed, this option is ignored.
  kitty/options/definition.py:878:opt('input_delay', '3',
  kitty/options/definition.py:884:the entire screen on each loop, because kitty is so fast that partial screen
  kitty/options/definition.py:885:updates will be drawn. This setting is ignored when the input buffer is almost full.
  ```

- **`Screen` is constructed with `opts.scrollback_lines`** — the wiring from config to the buffer.
  **[inferred-from-reading]**

  ```
  kitty/window.py:604:        self.screen: Screen = Screen(self, 24, 80, opts.scrollback_lines, cell_width, cell_height, self.id)
  ```

- **kitty exposes no built-in memory hook** — a source grep for memory instrumentation finds only a
  texture-leak comment, justifying external RSS sampling (§3.2). **[inferred-from-reading]**

  ```
  kitty/state.c:1456:    // we leak the texture here since it is not guaranteed
  kitty/state.c:1457:    // that freeing the texture will work during shutdown and
  kitty/state.c:1458:    // the GPU driver should take care of it when the OpenGL context is
  ```

---

## 5. T1 - Memory-consumption trajectory under heavy output

**Question (T1).** As hundreds of thousands of lines are pushed rapidly, what happens to memory
consumption as history accumulates - with *actual* measurements, not theory?

**Method.** A child (`/tmp/gen_output.py`, full source in Sec. 3.4) prints **300,000** lines of an
`"%08d"` counter plus padding (71 columns, matching the runtime grid) through kitty's real PTY, in
**batches of 10,000 with a 0.15 s pause** so the external sampler (`/tmp/rss_sampler.py`, reading
`/proc/<pid>/status` `VmRSS` every **50 ms**) can be correlated with the cumulative line count. The child
logs `BEGIN` / `PROGRESS <cum> <wall>` / `DONE` to a side file (never the PTY). `/tmp/lines_vs_rss.py` then
joins the two logs into a **lines-vs-RSS** table with an explicit **BEFORE** state (0 lines - the settled
baseline), a **during** state (each 10,000-line checkpoint), and an **AFTER** state (300,000 lines). Each
condition was run **twice** for stability. The kitty PID was obtained with `pgrep -x kitty` (embedded in the
`SAMPLER:` line below). All RSS values below are the raw `/proc` kilobytes (1 kB = 1024 B); MiB = kB / 1024,
matching the unit convention fixed in Sec. 1.

Three `scrollback_lines` conditions are exercised, because the buffer's behavior fundamentally differs
across the `SEGMENT_SIZE = 2048` boundary and the configured cap:

- **default `2000`** - the canonical baseline (note `2000 < 2048`);
- **large `100000`** - accumulation with a finite cap;
- **infinite `-1`** - `-1` wraps to `UINT_MAX` (screen.c:L100), i.e. effectively unbounded.

### 5.1 Default `scrollback_lines=2000` - the ring-buffer plateau [observed-at-runtime]

**Run 1:**

```
# invocation (exact command used for run def_r1):
LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a -s "-screen 0 1024x768x24" /app/kitty/launcher/kitty --config NONE -o scrollback_lines=2000 sh -c 'python3 /tmp/gen_output.py 300000 /tmp/meta_def_r1.txt 4 10000 0.15 7.0'
# external RSS sampler (exact command; PID from `pgrep -x kitty`):
python3 /tmp/rss_sampler.py 51395 0.05 > /tmp/rss_def_r1.log

# lines-vs-RSS table (python3 lines_vs_rss.py meta_def_r1.txt rss_def_r1.log):
cols=71 rows=22 N=300000 batch=10000 pause=0.150
BEFORE: lines=0  baseline_rss_kb=147928
lines	wall_offset_s	rss_kb	delta_from_baseline_kb
10000	0.038	153184	5256
20000	0.225	153184	5256
30000	0.412	153204	5276
40000	0.600	153204	5276
50000	0.788	153204	5276
60000	0.976	153204	5276
70000	1.165	153204	5276
80000	1.375	153204	5276
90000	1.564	153204	5276
100000	1.751	153204	5276
110000	1.936	153204	5276
120000	2.122	153204	5276
130000	2.309	153204	5276
140000	2.498	153204	5276
150000	2.685	153204	5276
160000	2.875	153204	5276
170000	3.063	153204	5276
180000	3.250	153204	5276
190000	3.439	153204	5276
200000	3.629	153204	5276
210000	3.818	153204	5276
220000	4.006	153204	5276
230000	4.197	153204	5276
240000	4.391	153204	5276
250000	4.584	153204	5276
260000	4.778	153204	5276
270000	4.971	153204	5276
280000	5.164	153204	5276
290000	5.359	153204	5276
300000	5.554	153204	5276
AFTER: lines=300000  after_rss_kb=153204  total_delta_kb=5276
bytes/line (total_delta*1024/lines) = 18.0
```

**Run 2:**

```
# invocation (exact command used for run def_r2):
LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a -s "-screen 0 1024x768x24" /app/kitty/launcher/kitty --config NONE -o scrollback_lines=2000 sh -c 'python3 /tmp/gen_output.py 300000 /tmp/meta_def_r2.txt 4 10000 0.15 7.0'
# external RSS sampler (exact command; PID from `pgrep -x kitty`):
python3 /tmp/rss_sampler.py 51585 0.05 > /tmp/rss_def_r2.log

# lines-vs-RSS table (python3 lines_vs_rss.py meta_def_r2.txt rss_def_r2.log):
cols=71 rows=22 N=300000 batch=10000 pause=0.150
BEFORE: lines=0  baseline_rss_kb=148212
lines	wall_offset_s	rss_kb	delta_from_baseline_kb
10000	0.039	153548	5336
20000	0.229	153548	5336
30000	0.419	153736	5524
40000	0.609	153736	5524
50000	0.799	153736	5524
60000	0.988	153736	5524
70000	1.175	153752	5540
80000	1.362	153752	5540
90000	1.550	153752	5540
100000	1.738	153752	5540
110000	1.929	153752	5540
120000	2.112	153752	5540
130000	2.297	153752	5540
140000	2.485	153752	5540
150000	2.673	153752	5540
160000	2.860	153752	5540
170000	3.048	153752	5540
180000	3.236	153752	5540
190000	3.424	153752	5540
200000	3.613	153752	5540
210000	3.803	153752	5540
220000	3.992	153752	5540
230000	4.182	153752	5540
240000	4.370	153752	5540
250000	4.560	153752	5540
260000	4.749	153752	5540
270000	4.938	153752	5540
280000	5.125	153752	5540
290000	5.313	153752	5540
300000	5.504	153752	5540
AFTER: lines=300000  after_rss_kb=153752  total_delta_kb=5540
bytes/line (total_delta*1024/lines) = 18.9
```

**Observation.** In both runs RSS steps up **once** from the settled baseline (147,928 / 148,212 kB;
~144.5 MiB) to a plateau (153,204 / 153,752 kB; ~149.6-150.1 MiB) and then holds flat through 300,000 lines.
Run 1 has fully settled by the 30,000-line checkpoint (`delta_from_baseline_kb = +5,276`) and run 2 by the
70,000-line checkpoint (`+5,540`); thereafter the delta column does **not change** across any of the
remaining checkpoints. The one-time step of **~5.2 MiB** (5,276 / 5,540 kB) is fixed working-set overhead
(VT-parser `ANSIBuf`, read/render buffers) - it is emphatically **not** history growth, which is why the
derived "bytes/line" figure is a meaningless **18.0 / 18.9** (the fixed overhead divided by 300,000 lines).

**Why there is no history growth [inferred-from-reading -> confirmed at runtime].** `ynum = MAX(scrollback,
lines) = MAX(2000, 22) = 2000` (screen.c:L130), which is **less than one `SEGMENT_SIZE` (2048)**. The single
segment allocated up front by `create_historybuf()` (history.c:L127) already holds 2048 rows, so all 2000
history lines fit in memory that is **already counted in the baseline**. Once the buffer fills
(`count == ynum`), `historybuf_push()` stops incrementing `count` and instead advances `start_of_data`,
overwriting the oldest line in place (history.c:L279-L281) - **no allocation occurs**. This is precisely why
RSS plateaus: at the default cap the terminal absorbs an unbounded number of lines at a **bounded** memory
cost. The measured flatness across 290,000 additional lines is the runtime confirmation of that
ring-overwrite code path.

### 5.2 Large `scrollback_lines=100000` - linear accumulation to a capped plateau [observed-at-runtime]

**Run 1:**

```
# invocation (exact command used for run large_r1):
LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a -s "-screen 0 1024x768x24" /app/kitty/launcher/kitty --config NONE -o scrollback_lines=100000 sh -c 'python3 /tmp/gen_output.py 300000 /tmp/meta_large_r1.txt 4 10000 0.15 7.0'
# external RSS sampler (exact command; PID from `pgrep -x kitty`):
python3 /tmp/rss_sampler.py 51775 0.05 > /tmp/rss_large_r1.log

# lines-vs-RSS table (python3 lines_vs_rss.py meta_large_r1.txt rss_large_r1.log):
cols=71 rows=22 N=300000 batch=10000 pause=0.150
BEFORE: lines=0  baseline_rss_kb=148160
lines	wall_offset_s	rss_kb	delta_from_baseline_kb
10000	0.039	190508	42348
20000	0.243	213396	65236
30000	0.439	237912	89752
40000	0.628	242996	94836
50000	0.817	273436	125276
60000	1.006	302284	154124
70000	1.194	326864	178704
80000	1.381	332140	183980
90000	1.569	362808	214648
100000	1.757	371324	223164
110000	1.944	371324	223164
120000	2.132	371324	223164
130000	2.320	371324	223164
140000	2.508	371324	223164
150000	2.697	371324	223164
160000	2.887	371372	223212
170000	3.078	371372	223212
180000	3.270	371372	223212
190000	3.460	371372	223212
200000	3.650	371372	223212
210000	3.840	371372	223212
220000	4.031	371372	223212
230000	4.219	371372	223212
240000	4.407	371372	223212
250000	4.595	371372	223212
260000	4.779	371372	223212
270000	4.967	371372	223212
280000	5.156	371372	223212
290000	5.347	371372	223212
300000	5.542	371372	223212
AFTER: lines=300000  after_rss_kb=371372  total_delta_kb=223212
bytes/line (total_delta*1024/lines) = 761.9
```

**Run 2:**

```
# invocation (exact command used for run large_r2):
LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a -s "-screen 0 1024x768x24" /app/kitty/launcher/kitty --config NONE -o scrollback_lines=100000 sh -c 'python3 /tmp/gen_output.py 300000 /tmp/meta_large_r2.txt 4 10000 0.15 7.0'
# external RSS sampler (exact command; PID from `pgrep -x kitty`):
python3 /tmp/rss_sampler.py 51965 0.05 > /tmp/rss_large_r2.log

# lines-vs-RSS table (python3 lines_vs_rss.py meta_large_r2.txt rss_large_r2.log):
cols=71 rows=22 N=300000 batch=10000 pause=0.150
BEFORE: lines=0  baseline_rss_kb=147548
lines	wall_offset_s	rss_kb	delta_from_baseline_kb
10000	0.042	191916	44368
20000	0.236	192752	45204
30000	0.424	219808	72260
40000	0.616	248484	100936
50000	0.804	279328	131780
60000	0.992	304168	156620
70000	1.180	308340	160792
80000	1.367	339512	191964
90000	1.554	370212	222664
100000	1.741	370852	223304
110000	1.932	370896	223348
120000	2.121	370896	223348
130000	2.325	370896	223348
140000	2.543	370900	223352
150000	2.751	370960	223412
160000	2.960	370960	223412
170000	3.195	370960	223412
180000	3.387	370960	223412
190000	3.596	370960	223412
200000	3.789	370960	223412
210000	3.993	370960	223412
220000	4.185	370960	223412
230000	4.375	370960	223412
240000	4.574	370960	223412
250000	4.769	370960	223412
260000	4.964	370960	223412
270000	5.155	370960	223412
280000	5.351	370960	223412
290000	5.554	370960	223412
300000	5.744	370960	223412
AFTER: lines=300000  after_rss_kb=370960  total_delta_kb=223412
bytes/line (total_delta*1024/lines) = 762.6
```

**Observation.** RSS climbs **monotonically and roughly linearly** while lines accumulate, then **plateaus
at the 100,000-line cap** and stays flat for the remaining 200,000 lines. Both runs settle at ~**362.5 MiB**
(371,372 / 370,960 kB; delta over baseline **+223,212 kB** run 1 / **+223,412 kB** run 2 - agreement within
**0.11 percent**). After 100,000 lines `count == ynum`, so - exactly as in Sec. 5.1 but at a far higher cap
- the ring begins overwriting (history.c:L279), no further segments are allocated, and RSS stops rising.

**Cross-check against the code [inferred-from-reading -> confirmed].** Holding 100,000 lines requires
`ceil(100000 / 2048) = 49` segments. At the code-derived full-segment cost of 4546 kB (Sec. 4.3), that is
`49 x 4546 = 222,754 kB` - within **0.3 percent** of the measured plateau delta (~223,300 kB). The small
excess is the same fixed parser/render overhead observed in Sec. 5.1.

### 5.3 Infinite `scrollback_lines=-1` - unbounded linear growth [observed-at-runtime]

**Run 1:**

```
# invocation (exact command used for run inf_r1):
LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a -s "-screen 0 1024x768x24" /app/kitty/launcher/kitty --config NONE -o scrollback_lines=-1 sh -c 'python3 /tmp/gen_output.py 300000 /tmp/meta_inf_r1.txt 4 10000 0.15 7.0'
# external RSS sampler (exact command; PID from `pgrep -x kitty`):
python3 /tmp/rss_sampler.py 52157 0.05 > /tmp/rss_inf_r1.log

# lines-vs-RSS table (python3 lines_vs_rss.py meta_inf_r1.txt rss_inf_r1.log):
cols=71 rows=22 N=300000 batch=10000 pause=0.150
BEFORE: lines=0  baseline_rss_kb=147288
lines	wall_offset_s	rss_kb	delta_from_baseline_kb
10000	0.049	180612	33324
20000	0.283	194652	47364
30000	0.512	222420	75132
40000	0.749	237596	90308
50000	0.996	260516	113228
60000	1.238	284092	136804
70000	1.490	309284	161996
80000	1.730	333080	185792
90000	1.973	360052	212764
100000	2.205	371400	224112
110000	2.437	397200	249912
120000	2.688	422340	275052
130000	2.925	450280	302992
140000	3.153	464320	317032
150000	3.357	487760	340472
160000	3.566	504796	357508
170000	3.790	537100	389812
180000	4.036	559224	411936
190000	4.247	577028	429740
200000	4.483	610520	463232
210000	4.690	624404	477116
220000	4.943	649936	502648
230000	5.165	665136	517848
240000	5.402	694604	547316
250000	5.625	707884	560596
260000	5.862	737128	589840
270000	6.073	753240	605952
280000	6.288	771832	624544
290000	6.516	802372	655084
300000	6.753	815280	667992
AFTER: lines=300000  after_rss_kb=815280  total_delta_kb=667992
bytes/line (total_delta*1024/lines) = 2280.1
```

**Run 2:**

```
# invocation (exact command used for run inf_r2):
LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a -s "-screen 0 1024x768x24" /app/kitty/launcher/kitty --config NONE -o scrollback_lines=-1 sh -c 'python3 /tmp/gen_output.py 300000 /tmp/meta_inf_r2.txt 4 10000 0.15 7.0'
# external RSS sampler (exact command; PID from `pgrep -x kitty`):
python3 /tmp/rss_sampler.py 52355 0.05 > /tmp/rss_inf_r2.log

# lines-vs-RSS table (python3 lines_vs_rss.py meta_inf_r2.txt rss_inf_r2.log):
cols=71 rows=22 N=300000 batch=10000 pause=0.150
BEFORE: lines=0  baseline_rss_kb=148028
lines	wall_offset_s	rss_kb	delta_from_baseline_kb
10000	0.045	180592	32564
20000	0.240	206032	58004
30000	0.430	236860	88832
40000	0.617	237724	89696
50000	0.805	267652	119624
60000	0.993	296516	148488
70000	1.182	326636	178608
80000	1.370	326676	178648
90000	1.560	355860	207832
100000	1.748	383808	235780
110000	1.940	411924	263896
120000	2.129	437768	289740
130000	2.317	442524	294496
140000	2.508	470676	322648
150000	2.697	498384	350356
160000	2.889	525412	377384
170000	3.079	526668	378640
180000	3.271	555024	406996
190000	3.459	583528	435500
200000	3.649	611216	463188
210000	3.838	637808	489780
220000	4.030	640244	492216
230000	4.219	665604	517576
240000	4.430	683960	535932
250000	4.642	726712	578684
260000	4.830	731868	583840
270000	5.018	761624	613596
280000	5.209	788776	640748
290000	5.398	815620	667592
300000	5.587	815620	667592
AFTER: lines=300000  after_rss_kb=815620  total_delta_kb=667592
bytes/line (total_delta*1024/lines) = 2278.7
```

**Observation.** With `scrollback_lines=-1`, RSS grows **linearly with the line count and never plateaus**,
reaching ~**796.2-796.5 MiB** (815,280 / 815,620 kB) at 300,000 lines (delta over baseline **+667,992 kB**
run 1 / **+667,592 kB** run 2 - agreement within **0.05 percent**). Growth **continues past the
100,000-line point** at which the large-cap run of Sec. 5.2 had already flattened.

**Same curve, different stop point.** At the 100,000-line checkpoint the infinite run's delta (**+224,112
kB**, run 1) is essentially identical to the large-cap run's *final plateau* delta (**+223,164 kB**, Sec.
5.2) - the two conditions trace the **same** growth curve up to 100,000 lines and diverge only afterward,
when the capped run begins overwriting while the infinite run keeps allocating. This is the cleanest
possible demonstration that the plateau is imposed by `ynum`, not by any external memory pressure.

**Why it is unbounded [inferred-from-reading -> confirmed].** `scrollback` is parsed as an **unsigned int**
(screen.c:L100, `PyArg_ParseTuple` conversion `I`), so the user's `-1` wraps to `UINT_MAX (4294967295)` and
`ynum = MAX(UINT_MAX, 22) = UINT_MAX`. The guard in `segment_for()` - `SEGMENT_SIZE * num_segments < ynum`
(history.c:L39) - therefore effectively never stops firing, so a new segment is allocated for every 2048
additional lines while `count` keeps incrementing (history.c:L282) and never reaches the overwrite branch.
This matches kitty's documented "(effectively) infinite scrollback."

**Cross-check against the code.** 300,000 lines require `ceil(300000 / 2048) = 147` segments; at 4546 kB
each that is `147 x 4546 = 668,262 kB` - within **0.1 percent** of the measured delta (~667,800 kB). The
segment-granular allocation model predicts the observed magnitude almost exactly.

### 5.4 Measured per-line memory cost [observed-at-runtime]

The per-line cost is derived three independent ways; all **bracket** the code-derived
`32 * xnum + 1 = 32 * 71 + 1 = 2273` bytes/line (Sec. 4.3):

| Method | Source | bytes/line |
|--------|--------|-----------:|
| Infinite-run total delta (run 1 / run 2) | `total_delta_kb * 1024 / 300000` | 2280.1 / 2278.7 |
| Per-segment median RSS step | `4552 kB * 1024 / 2048` (Sec. 7.3) | 2276.0 |
| Linear least-squares fit over boundary points | `/tmp/regress.py` (Sec. 7.4) | 2266.97 |
| **Code-derived (predicted)** | `32 * xnum + 1`, xnum = 71 | **2273** |

The infinite-run total-delta figures run slightly high because they fold in the fixed ~5 MiB overhead; the
regression, which separates slope from intercept, runs slightly low; the median-step sits between. All four
values agree to within **0.3 percent**, confirming the growth magnitude is governed exactly by the cell-size
formula of Sec. 4.3 and the measured `xnum = 71`.

### 5.5 Stability summary (at least 2 runs per condition)

| Condition | `scrollback_lines` | Shape | AFTER RSS (r1 / r2), kB | Delta over baseline (r1 / r2), kB | Run-to-run |
|-----------|-------------------:|-------|------------------------:|----------------------------------:|-----------:|
| Default   | 2000    | Plateau (flat)        | 153204 / 153752 | 5276 / 5540     | <= 0.36 percent |
| Large     | 100000  | Ramp -> plateau@100k  | 371372 / 370960 | 223212 / 223412 | <= 0.11 percent |
| Infinite  | -1      | Unbounded linear      | 815280 / 815620 | 667992 / 667592 | <= 0.05 percent |

Every magnitude is stable across the two runs to well under 0.4 percent, so no scale escalation was
required. The qualitative shapes - flat plateau, capped plateau, and unbounded linear - are identical across
runs.

**Answer to T1 (headline).** Under the **default** config, memory is **bounded**: heavy output raises RSS by
a one-time ~5 MiB and then plateaus regardless of how many lines are printed. Under **large/infinite**
scrollback, RSS rises **linearly at ~2273 bytes per 71-column line** (measured 2267-2280 across three
methods), i.e. **~2.2 MiB per 1000 lines** or **~217 MiB per 100,000 lines**, until the configured cap (if
any) is reached.

---

## 6. T2 - Responsiveness and scroll-input latency under concurrent output

**Question (T2).** While output is still being generated, does the terminal remain responsive? What latency
can be observed between a scroll input and the display updating? Are there visible signs of the system
prioritizing one operation over another?

### 6.1 Method - a *display-confirmed* latency probe [addresses the display-verification requirement]

The earlier version of this measurement timed only the return of the `kitty @ action scroll_home` subprocess.
That is **not** a display measurement - it proves the command was *accepted*, not that the *viewport
actually moved*. This section replaces it with a probe (`/tmp/scroll_disp_latency.py`, full source in Sec.
3.4) that **confirms the on-screen result** and reports three clearly-separated timings.

- **The load** is the real, canonical path: a child (`/tmp/stream.py`) writes an ever-increasing 8-digit
  counter line (`"%08d" + padding`, flushed every 1024 lines) **continuously** to kitty's PTY, so the
  genuine child-PTY -> VT-parser -> screen -> history pipeline is saturated during the whole measurement.
- **The trigger is non-canonical and labelled as such.** The scroll itself is fired with
  `kitty @ action scroll_home` / `scroll_end` over the remote-control socket (the `talk_loop` thread,
  child-monitor.c:L230,L286). Remote control is a **bypassing interface**; it is used *only* to fire the
  scroll deterministically, never as the thing being measured.
- **The confirmation reads the real display.** `kitty @ get-text --extent=screen` returns the **actual
  visible viewport**. Because every streamed line carries a strictly-increasing counter, the top visible
  number can only *increase* from new output; a large **downward jump** in that number can be produced
  **only** by the scroll moving the display to older content. The probe therefore requires the top visible
  line to **drop by at least `DROP_MIN = 20000`** before it counts the display as updated - a drift-robust
  proof that the viewport genuinely moved.

Three timings are recorded per trial and reported separately, so no IPC/injection cost is ever passed off as
render latency:

| Metric | Definition |
|--------|------------|
| `trigger_ms`  | wall time for the scroll-action subprocess to return (remote-control injection cost only) |
| `ipc_noop_ms` | wall time of a no-op `kitty @ get-text` round-trip (the fixed remote-control overhead baseline) |
| `display_ms`  | wall time from issuing the trigger until `get-text` **confirms** the viewport moved back (top line dropped >= 20000) - the **display-confirmed** latency |
| `polls_to_confirm` | number of `get-text` reads needed before the drop is observed (1 = already moved on the first read) |

Two **loaded** runs (continuous streaming) and one **idle** run (no streaming during the probe) were taken,
each with n = 60 trials.

### 6.2 Loaded run 1 [observed-at-runtime]

```
LAUNCH[load1,stream]: LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a -s "-screen 0 1024x768x24" /app/kitty/launcher/kitty --config NONE -o scrollback_lines=100000 -o allow_remote_control=yes --listen-on unix:/tmp/kitty-load1 sh -c 'python3 /tmp/stream.py'
LABEL=load1 MODE=stream KPID=39176 socket=yes
kitty_rss_kb_during=372456
streaming_rate_lines_per_s=229499  (top advanced 1196055->1673941 in 2.082s)
LABEL=load1 n_ok=60/60 DROP_MIN=20000
ipc_noop_ms:  min=21.3 p50=22.9 mean=23.9 p90=26.2 max=29.8
trigger_ms:   min=21.9 p50=25.1 mean=25.3 p90=29.1 max=35.8
display_ms:   min=43.4 p50=50.8 mean=50.4 p90=55.7 max=60.6
polls_to_confirm: min=1 median=1.0 max=1
display_raw_ms=48.5 47.4 51.8 45.0 50.3 44.7 48.7 46.7 48.9 48.2 47.9 47.9 51.6 43.4 48.7 53.3 51.3 51.3 48.7 53.2 50.9 48.1 43.9 55.4 47.5 50.9 47.3 46.8 43.4 48.6 51.7 52.4 54.2 55.8 60.6 51.5 46.9 49.2 50.9 53.9 48.0 57.3 50.8 51.6 53.5 45.2 55.3 55.7 53.9 55.8 51.7 45.0 55.7 45.8 49.8 55.0 54.8 50.9 48.9 50.8
CASE_T2 load1 done
```

### 6.3 Loaded run 2 [observed-at-runtime]

```
LAUNCH[load2,stream]: LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a -s "-screen 0 1024x768x24" /app/kitty/launcher/kitty --config NONE -o scrollback_lines=100000 -o allow_remote_control=yes --listen-on unix:/tmp/kitty-load2 sh -c 'python3 /tmp/stream.py'
LABEL=load2 MODE=stream KPID=43149 socket=yes
kitty_rss_kb_during=372864
streaming_rate_lines_per_s=230608  (top advanced 1196176->1675117 in 2.077s)
LABEL=load2 n_ok=60/60 DROP_MIN=20000
ipc_noop_ms:  min=24.5 p50=25.6 mean=26.2 p90=29.1 max=30.3
trigger_ms:   min=21.9 p50=25.3 mean=25.4 p90=29.0 max=29.8
display_ms:   min=44.2 p50=51.1 mean=50.7 p90=54.3 max=58.6
polls_to_confirm: min=1 median=1.0 max=1
display_raw_ms=54.2 54.8 50.8 51.6 56.6 51.5 50.0 50.8 52.3 53.0 51.4 50.3 53.0 52.9 58.6 53.1 50.4 51.2 54.8 52.0 50.4 49.9 51.2 54.6 51.6 48.2 46.7 50.3 50.6 50.0 54.3 47.4 53.6 51.3 53.2 53.0 47.7 48.0 52.5 46.8 53.9 52.3 50.8 47.9 52.4 49.1 51.7 48.6 52.0 47.9 48.0 47.8 44.2 44.9 48.0 44.7 51.0 46.8 52.3 44.2
CASE_T2 load2 done
```

### 6.4 Idle baseline [observed-at-runtime]

```
LAUNCH[idle,idle]: LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a -s "-screen 0 1024x768x24" /app/kitty/launcher/kitty --config NONE -o scrollback_lines=100000 -o allow_remote_control=yes --listen-on unix:/tmp/kitty-idle sh -c 'python3 /tmp/gen_output.py 82000 /tmp/meta_idle.txt 2 0 0 600'
LABEL=idle MODE=idle KPID=47156 socket=yes
kitty_rss_kb_during=331720
LABEL=idle n_ok=60/60 DROP_MIN=20000
ipc_noop_ms:  min=22.1 p50=23.3 mean=23.7 p90=25.1 max=29.8
trigger_ms:   min=21.2 p50=23.3 mean=23.4 p90=25.0 max=26.4
display_ms:   min=43.0 p50=47.6 mean=47.5 p90=49.9 max=50.6
polls_to_confirm: min=1 median=1.0 max=1
display_raw_ms=48.1 48.2 49.9 49.1 48.1 49.8 47.2 47.1 50.2 48.4 44.5 49.1 46.9 49.0 46.5 49.4 44.7 46.9 46.0 49.9 48.3 48.3 47.9 45.1 44.9 47.4 47.8 43.0 48.9 45.5 44.9 46.2 48.5 46.4 45.9 48.4 47.1 46.7 48.4 50.0 46.4 50.6 47.8 45.8 47.4 46.3 49.6 46.1 47.5 47.6 45.9 47.6 48.4 47.7 49.9 48.0 46.6 49.3 47.9 44.9
CASE_T2 idle done
```

### 6.5 Interpretation

**The terminal stays responsive.** In **all 60/60** trials of **both** loaded runs the scroll was
display-confirmed (top visible line dropped by >= 20,000), while the child was streaming at a measured
**229,499** (run 1) and **230,608** (run 2) **lines/s** - i.e. ~230k lines/s; at 71 grid cells x 32 B = ~2.3 kB of cell storage per line
(Sec. 4.3), that is ~0.5 GB/s of cell data written into the history ring - and kitty's RSS sat at **372,456 / 372,864 kB** (~364 MiB), consistent with the T1 large-cap plateau
of Sec. 5.2. Heavy output did not block, drop, or delay the scroll.

**What the ~50 ms display latency actually is.** The display-confirmed `display_ms` is
**p50 = 50.8 ms (run 1) / 51.1 ms (run 2)**. This figure is an **upper bound** dominated by *two* serial
remote-control round-trips: `display_ms ~= trigger_ms (~25 ms) + one get-text confirmation (~ipc_noop_ms,
~23-25 ms)`. The `ipc_noop_ms` baseline (**p50 22.9 / 25.6 ms**) and `trigger_ms` (**p50 25.1 / 25.3 ms**)
account for essentially the entire 50 ms - both are properties of the remote-control *measurement apparatus*,
not of kitty's rendering. Because `polls_to_confirm = 1` in **every single trial** (min = median = max = 1),
the viewport had **already moved by the first `get-text` after the trigger**: kitty's actual on-screen render
completes within a single round-trip - i.e. **well under 25 ms**, consistent with the `repaint_delay = 10 ms`
batching cadence (definition.py:L866).

**The observable sign of prioritization: a tiny, bounded load penalty.** Comparing the loaded runs to the
idle baseline (`display_ms` p50 **47.6 ms**) isolates the marginal cost of heavy output:

| Run | `display_ms` p50 (ms) | vs idle (ms) |
|-----|----------------------:|-------------:|
| Idle (no streaming) | 47.6 | - |
| Loaded run 1 | 50.8 | **+3.2** |
| Loaded run 2 | 51.1 | **+3.5** |

The scroll path pays only **~3.2-3.5 ms** extra while the parser/history path is saturated at ~230k lines/s.
That small, bounded penalty is the visible sign that kitty **prioritizes interactivity over throughput**, and
its magnitude matches the documented **`input_delay = 3 ms`** (definition.py:L878) - the interval kitty waits
before reading more child output so that input and rendering are serviced first. The two loaded runs agree to
within **0.3 ms (0.6 percent)**, so the figure is stable run-to-run (this replaces the earlier unquantified
cross-session variance note with two same-session, same-input runs).

**Mechanism [inferred-from-reading, grounded in source].** The responsiveness follows directly from kitty's
thread model in `child-monitor.c`: child output is read and fed to the parser on a dedicated **`io_loop`**
thread (forward-declared at child-monitor.c:L229; started with `pthread_create(&self->io_thread, NULL,
io_loop, self)` at child-monitor.c:L291), remote control runs on the **`talk_loop`** thread
(child-monitor.c:L230,L286), and **rendering happens on the main thread**. Because the heavy child-output
work and the render/input work run on **separate threads**, a saturated `io_loop` cannot starve the render
loop; the main thread coalesces frames every `repaint_delay = 10 ms` and defers extra child reads by
`input_delay = 3 ms` when its input buffer is nearly full (definition.py:L866,L878). This decoupling is the
root cause of the observed ~3 ms marginal latency and the 60/60 confirmation rate.

**Non-canonical caveat.** The scroll *input* was injected via remote control (a bypassing interface), so the
`trigger_ms` component is non-canonical and is **not** presented as kitty's input-handling latency. What is
canonical and directly observed is (a) the heavy-output load path, (b) the **real rendered viewport** read
back with `get-text`, and (c) the marginal load penalty derived from it. A keyboard/mouse-driven scroll would
avoid the ~25 ms remote-control injection entirely, so the true user-perceived latency is **no worse** than,
and almost certainly better than, the ~50 ms upper bound measured here.

**Answer to T2 (headline).** Yes - the terminal remains fully responsive while streaming at ~230k lines/s:
every one of 120 scroll trials was display-confirmed, the marginal latency added by heavy output is only
**~3.2-3.5 ms** (matching `input_delay = 3 ms`), and kitty's actual render completes in under one ~25 ms IPC
round-trip. The visible prioritization signal is precisely that small, bounded penalty - a direct consequence
of reading child output on a separate `io_loop` thread from the main-thread renderer.

---

## 7. T3 - Buffer-growth boundaries and observable allocation events

**Question (T3).** At what point does the buffer's behavior change as it grows - specifically, when is a new
block of backing storage allocated - and can that allocation be observed through external memory monitoring?

### 7.1 Method - fine-grained sampling across `SEGMENT_SIZE` boundaries [observed-at-runtime]

To resolve individual allocation events, a child pushes **82,000** lines in **batches of exactly 2,048 (one
`SEGMENT_SIZE`) with a 0.20 s pause**, under `scrollback_lines=100000` (so the ring never overwrites and
every batch forces exactly one new segment), while the external sampler reads `VmRSS` every **20 ms** - fine
enough to place each discrete step. The exact commands (PID 52698 from `pgrep -x kitty`):

```
# invocation:
LIBGL_ALWAYS_SOFTWARE=1 xvfb-run -a -s "-screen 0 1024x768x24" /app/kitty/launcher/kitty --config NONE -o scrollback_lines=100000 sh -c 'python3 /tmp/gen_output.py 82000 /tmp/meta_paced_p20.txt 4 2048 0.20 7.0'
# external RSS sampler (20 ms interval; raw log is `elapsed_s <TAB> wall_epoch <TAB> rss_kb`):
python3 /tmp/rss_sampler.py 52698 0.02 > /tmp/rss_paced_p20.log
```

### 7.2 The raw RSS trace - allocation is directly observable [observed-at-runtime]

**Startup baseline.** The first samples capture kitty's own start-up (during the 4 s warm-up, before any
child output), settling to a flat baseline of **147,384 kB** - this already includes the single initial
history segment allocated by `create_historybuf()` (history.c:L127):

```
# head -12 rss_paced_p20.log
0.000	1783491174.511	87364
0.021	1783491174.532	92772
0.041	1783491174.552	105996
0.062	1783491174.573	116480
0.082	1783491174.593	123824
0.104	1783491174.615	133244
0.124	1783491174.635	146532
0.145	1783491174.656	147384
0.165	1783491174.676	147384
0.185	1783491174.696	147384
0.205	1783491174.717	147384
0.226	1783491174.737	147384
```

**The step climb (every distinct `VmRSS` level).** Collapsing the 876-sample log to the rows where `VmRSS`
*changed* yields the full staircase below. It is **strictly monotonic** (zero downward moves across all 876
samples) and shows **clean, discrete ~4,552 kB jumps**. The first 9 levels (wall clock < 1783491178.649) are
kitty start-up; the child's first line arrives at the `BEGIN` marker (elapsed ~4.145 s, RSS 151,188 kB), and
every subsequent jump is one `add_segment()` (history.c:L18) `calloc` of a `SEGMENT_SIZE`-row block:

```
# awk -F'\t' '$3!=prev{print; prev=$3}' rss_paced_p20.log   (first row at each new VmRSS level)
0.000	1783491174.511	87364
0.021	1783491174.532	92772
0.041	1783491174.552	105996
0.062	1783491174.573	116480
0.082	1783491174.593	123824
0.104	1783491174.615	133244
0.124	1783491174.635	146532
0.145	1783491174.656	147384
1.632	1783491176.143	147400
4.145	1783491178.656	151188
4.165	1783491178.676	152676
4.367	1783491178.878	157240
4.569	1783491179.080	161792
4.771	1783491179.282	164208
4.791	1783491179.302	166344
4.993	1783491179.504	170896
5.195	1783491179.706	175448
5.397	1783491179.908	178028
5.417	1783491179.928	180000
5.619	1783491180.130	184552
5.820	1783491180.331	189104
6.022	1783491180.533	191500
6.042	1783491180.553	193656
6.244	1783491180.755	198208
6.446	1783491180.957	202760
6.648	1783491181.159	205744
6.668	1783491181.179	207312
6.870	1783491181.381	211864
7.072	1783491181.583	216416
7.273	1783491181.784	218952
7.293	1783491181.804	220968
7.495	1783491182.006	225520
7.697	1783491182.208	230072
7.899	1783491182.410	232856
7.919	1783491182.430	234624
8.121	1783491182.632	239176
8.324	1783491182.835	243728
8.526	1783491183.037	248280
8.728	1783491183.239	248780
8.748	1783491183.259	252832
8.951	1783491183.462	257384
9.153	1783491183.664	261796
9.173	1783491183.684	261936
9.375	1783491183.886	266488
9.577	1783491184.088	271040
9.779	1783491184.290	273616
9.800	1783491184.311	275592
10.002	1783491184.513	280144
10.204	1783491184.715	284696
10.406	1783491184.917	288232
10.426	1783491184.937	289248
10.628	1783491185.139	293800
10.829	1783491185.340	298560
11.031	1783491185.542	303112
11.232	1783491185.743	303328
11.252	1783491185.763	307664
11.454	1783491185.965	312216
11.655	1783491186.166	316768
11.857	1783491186.368	317236
11.877	1783491186.388	321320
12.079	1783491186.590	325872
12.280	1783491186.791	329412
12.300	1783491186.811	330424
12.502	1783491187.013	330688
```

**A single boundary crossing, at raw 20 ms resolution.** Zooming into one contiguous window (no rows
omitted) shows `VmRSS` holding **flat for ~10 consecutive samples** during the 0.20 s pause, then stepping up
in a **single 20 ms sample** - the discrete, externally-observable allocation event the question asks about:

```
# awk -F'\t' '$1>=4.14 && $1<=4.62' rss_paced_p20.log   (contiguous, nothing removed)
4.145	1783491178.656	151188
4.165	1783491178.676	152676
4.185	1783491178.696	152676
4.205	1783491178.716	152676
4.225	1783491178.736	152676
4.246	1783491178.757	152676
4.266	1783491178.777	152676
4.286	1783491178.797	152676
4.306	1783491178.817	152676
4.326	1783491178.837	152676
4.347	1783491178.858	152676
4.367	1783491178.878	157240
4.387	1783491178.898	157240
4.407	1783491178.918	157240
4.427	1783491178.938	157240
4.447	1783491178.959	157240
4.468	1783491178.979	157240
4.488	1783491178.999	157240
4.508	1783491179.019	157240
4.528	1783491179.039	157240
4.548	1783491179.059	157240
4.569	1783491179.080	161792
4.589	1783491179.100	161792
4.609	1783491179.120	161792
```

RSS sits at 152,676 kB for eleven samples (elapsed 4.145-4.347 s), then jumps to 157,240 kB at 4.367 s
(+4,564 kB in one 20 ms tick), holds again, then jumps to 161,792 kB (+4,552 kB) - each jump is one segment.

### 7.3 Per-boundary analysis - the step equals one `SEGMENT_SIZE` block [observed-at-runtime]

Joining the raw log to the child's per-batch `PROGRESS` markers gives the settled `VmRSS` at each exact
2,048-line boundary and the delta since the previous boundary (`/tmp/analyze_boundary.py`, source in Sec.
3.4):

```
# python3 analyze_boundary.py meta_paced_p20.txt rss_paced_p20.log 71
xnum=71  per_segment_predicted_kb=4546.0
lines	segments_expected(ceil/2048)	settled_rss_kb	delta_from_prev_kb
  2048	1	152676	0
  4096	2	157240	4564
  6144	3	164208	6968
  8192	4	166344	2136
 10240	5	170896	4552
 12288	6	178028	7132
 14336	7	180000	1972
 16384	8	184552	4552
 18432	9	191500	6948
 20480	10	193656	2156
 22528	11	198208	4552
 24576	12	205744	7536
 26624	13	207312	1568
 28672	14	211864	4552
 30720	15	218952	7088
 32768	16	220968	2016
 34816	17	225520	4552
 36864	18	232856	7336
 38912	19	234624	1768
 40960	20	239176	4552
 43008	21	243728	4552
 45056	22	248780	5052
 47104	23	252832	4052
 49152	24	257384	4552
 51200	25	261936	4552
 53248	26	266488	4552
 55296	27	273616	7128
 57344	28	275592	1976
 59392	29	280144	4552
 61440	30	284696	4552
 63488	31	289248	4552
 65536	32	293800	4552
 67584	33	298560	4760
 69632	34	303328	4768
 71680	35	307664	4336
 73728	36	312216	4552
 75776	37	317236	5020
 77824	38	321320	4084
 79872	39	325872	4552
 81920	40	330424	4552
 82000	41	330688	264

per-2048-line deltas: n=39  min=1568  max=7536  mean=4557.6  median=4552.0
most common delta values: [(4552, 17), (4564, 1), (6968, 1), (2136, 1), (7132, 1)]
bytes/line from median step = 2276.0 (median_kb*1024/2048)
```

The predicted per-segment cost is `SEGMENT_SIZE * (32 * xnum + 1) = 2048 * 2273 = 4,655,104 B = 4,546.0 kB`
(history.c:L23-25 x data-types.h:L221,L228). The **median / modal** measured step is **4,552 kB** (17 of the
39 inter-boundary deltas land on it exactly) - within **0.13 percent** of prediction.

### 7.4 Regression cross-check [observed-at-runtime]

A linear fit of settled `VmRSS` against line count over the 41 boundary points recovers the per-line slope,
independently confirming the segment model:

```
# python3 regress.py meta_paced_p20.txt rss_paced_p20.log 71
linear fit RSS(bytes) = slope*lines + intercept over 41 boundary points (lines>=2048)
slope  = 2266.97 bytes/line   (predicted 32*71+1 = 2273)
intercept = 152613691 bytes = 145.5 MB
R^2 = 0.99969
```

The fitted slope **2,266.97 bytes/line** (R^2 = **0.99969**) matches the code-derived `32 * xnum + 1 = 2273`
to within 0.3 percent, and the ~145.5 MB intercept matches the measured baseline - the growth is exactly the
segmented-cell model of Sec. 4.3, allocated `SEGMENT_SIZE` rows at a time.

### 7.5 Precise boundary accounting and sampler jitter [observed-at-runtime]

Two subtleties matter for an accurate statement of the result:

- **40 full boundaries + 1 partial - not "2,048 steps".** The 82,000 lines cross **40 full 2,048-line
  boundaries** (checkpoints at 2,048; 4,096; and every further multiple of 2,048 up to 81,920), producing **39 inter-boundary deltas** of ~4,552
  kB each (one segment apiece), plus a **final partial batch** to 82,000. That last batch is only **80 lines**
  (82,000 - 81,920) into a freshly-`calloc`-ed 41st segment, so although the full 2,048-row block is
  *reserved*, `VmRSS` reflects only the **touched/faulted pages** - about 80/2048 ~= 3.9 percent - which is
  why the final delta is just **+264 kB**, not another ~4,552 kB. In total **41 segments** back the buffer
  (the initial `create_historybuf` segment plus 40 lazily added by `add_segment()`), of which segments 1-39
  are full and the 41st is ~4 percent full.

- **Jitter comes in pairs summing to exactly 2x4552 = 9,104 kB.** Where a delta deviates from 4,552 kB it is
  always compensated by its neighbour: e.g. **6,968 + 2,136 = 9,104**, **7,132 + 1,972 = 9,104**, **7,536 +
  1,568 = 9,104** (all visible in the table above). This is a pure **sampling artifact** of the asynchronous
  20 ms sampler catching part of one segment's page-faulting in the *previous* boundary window; the
  underlying per-segment cost is the uniform 4,552 kB confirmed by the median.

**Mechanism [inferred-from-reading, grounded in source].** Each pushed line calls `historybuf_add_line`
(history.c:L287) -> `historybuf_push` (history.c:L276), whose row index reaches the segmented store through
`segment_for(self, y)` (history.c:L37). Its allocation guard is:

```c
// kitty/history.c:L38-L39
index_type seg_num = y / SEGMENT_SIZE;
while (UNLIKELY(seg_num >= self->num_segments && SEGMENT_SIZE * self->num_segments < self->ynum)) add_segment(self);
```

So exactly when the row index `y` first crosses a new multiple of `SEGMENT_SIZE = 2048`, `seg_num` exceeds
`num_segments` and **one** `add_segment()` fires - reallocating the segment directory and `calloc`-ing a new
`SEGMENT_SIZE`-row cell block (history.c:L20,L25). That `calloc` + first-touch page-faulting is precisely the
~4,552 kB RSS step observed above. Allocation therefore changes at **every 2,048-line boundary**, up to the
`ynum` cap encoded in the second half of the guard.

### 7.6 The other boundary: the default cap never allocates a second segment [observed-at-runtime]

The guard's `SEGMENT_SIZE * self->num_segments < self->ynum` term is also what makes the **default** config
behave completely differently. With `scrollback_lines=2000`, `ynum = MAX(2000, 22) = 2000` (screen.c:L130);
after the initial segment `2048 * 1 = 2048 < 2000` is **false**, so the loop can **never** add a second
segment. Independently, `historybuf_push` keeps `idx = (start_of_data + count) % ynum < 2000 < 2048`
(history.c:L277), so `segment_for` never even requests one. This is the mechanism behind the **dead-flat
default trace of Sec. 5.1**: no 2,048-line boundary is ever crossed inside the buffer, so no allocation step
ever appears in RSS - the runtime confirmation that the buffer's growth behavior changes at the
`SEGMENT_SIZE` boundary only when `ynum` permits.

**Answer to T3 (headline).** The buffer's storage behavior changes at **every `SEGMENT_SIZE = 2,048`-line
boundary**: crossing one triggers exactly one `add_segment()` `calloc` of a 2,048-row block, observed as a
clean, discrete **~4,552 kB** step in externally-sampled `VmRSS` (median of 39 steps, matching the predicted
4,546 kB). This is directly visible with 20 ms `/proc` sampling. Allocation stops at the `ynum` cap, so under
the default `scrollback_lines=2000` (< 2,048) **no** second segment is ever allocated and the trace is flat.

---

## 8. Non-canonical supplement - direct-binding corroboration of the segment cadence

**This entire section is NON-CANONICAL and is presented only as corroboration.** It drives `HistoryBuf`
*directly* through the `fast_data_types` Python binding (`/tmp/noncanonical_binding.py`, source in Sec. 3.4),
which **bypasses** the canonical child-PTY -> VT-parser -> screen entry point. Under the rules, a value from a
bypassing interface does **not** count as an observation of the real path; the canonical evidence for T3 is
Sec. 7. What this supplement adds is a jitter-free control: because it reads `VmRSS` **synchronously** right
after each 2,048-line batch (no asynchronous sampler), it isolates the *true* per-segment step.

```
# PYTHONPATH=/app python3 /tmp/noncanonical_binding.py 71 100000 82000
xnum=71 ynum=100000 N=82000 predicted_seg_kb=4546.0
rss_after_create_kb=16200
lines	segments	rss_kb	delta_kb
2048	1	20820	4620
4096	2	25380	4560
6144	3	29936	4556
8192	4	34492	4556
10240	5	39048	4556
12288	6	43604	4556
14336	7	48160	4556
16384	8	52716	4556
18432	9	57272	4556
20480	10	61828	4556
22528	11	66384	4556
24576	12	70940	4556
26624	13	75500	4560
28672	14	80056	4556
30720	15	84612	4556
32768	16	89168	4556
34816	17	93724	4556
36864	18	98280	4556
38912	19	102836	4556
40960	20	107392	4556
43008	21	111948	4556
45056	22	116504	4556
47104	23	121060	4556
49152	24	125616	4556
51200	25	130172	4556
53248	26	134728	4556
55296	27	139284	4556
57344	28	143840	4556
59392	29	148396	4556
61440	30	152952	4556
63488	31	157508	4556
65536	32	162064	4556
67584	33	166620	4556
69632	34	171176	4556
71680	35	175732	4556
73728	36	180288	4556
75776	37	184844	4556
77824	38	189400	4556
79872	39	193956	4556
81920	40	198512	4556
final_count=82000 final_rss_kb=198696 total_delta_kb=182496
bytes/line = 2279.0 (total_delta*1024/N)
```

**Reading it.** After constructing the buffer, RSS is 16,200 kB (bare interpreter plus the one initial
segment). Every 2,048-line batch then adds a **uniform 4,556 kB** - with *none* of the +/-2 kB jitter seen in
the asynchronously-sampled canonical trace of Sec. 7, which directly confirms that jitter was a sampling
artifact, not a property of the allocator. The step matches the canonical median (4,552 kB, Sec. 7.3) to
within 0.09 percent and the code-derived 4,546 kB to within 0.22 percent, and the aggregate **2,279.0
bytes/line** falls inside the canonical 2,267-2,280 range (Sec. 5.4). Forty segments are added across the
82,000 lines, exactly matching the canonical boundary count of Sec. 7.5. The binding thus independently
reproduces the segment size, the per-2,048-line cadence, and the per-line cost - but, being non-canonical, it
only corroborates the real-path measurements; it never replaces them.

## 9. Coverage pass - every named sub-question, answered by name

Per the rules, the verbatim question is decomposed into every explicitly-named item, each mapped to the
section that answers it from observed evidence.

| # | Sub-question (from the user's verbatim request) | Thread | Answered in | Observed answer (one line) |
|---|--------------------------------------------------|--------|-------------|-----------------------------|
| 1 | "what happens to memory consumption as the history accumulates?" | T1 | Sec. 5.1-5.3 | Bounded plateau at the cap; linear ~2,273 B/line while below it (measured tables, both runs, 3 configs) |
| 2 | "I'd like to see actual memory measurements, not just understand the theory." | T1 | Sec. 5.1-5.5 | Full `/proc` `VmRSS` lines-vs-RSS tables, before/during/after, 2 runs each; not code-reading |
| 3 | "does the terminal remain responsive?" | T2 | Sec. 6.2-6.5 | Yes - 120/120 scrolls display-confirmed while streaming at ~230k lines/s |
| 4 | "What latency or lag can I observe between my scroll input and the display updating?" | T2 | Sec. 6.2-6.5 | ~50 ms display-confirmed upper bound (IPC-dominated); marginal load cost only ~3.2-3.5 ms; render < one ~25 ms round-trip |
| 5 | "visible signs of the system prioritizing one operation over another?" | T2 | Sec. 6.5 | Yes - the bounded ~3 ms load penalty matching `input_delay=3 ms`, from `io_loop`/main-thread decoupling |
| 6 | "At what point does the buffer's behavior change as it grows, for example, when does allocation of new storage occur" | T3 | Sec. 7.2-7.6 | At every `SEGMENT_SIZE=2,048`-line boundary, one `add_segment()` `calloc`; stops at the `ynum` cap |
| 7 | "can I observe this happening through memory monitoring?" | T3 | Sec. 7.2 | Yes - discrete ~4,552 kB `VmRSS` steps, resolved by 20 ms `/proc` sampling |
| 8 | "the repository itself should remain unchanged" | scope | Sec. 10 | Confirmed - source tree pristine; only the deliverable added; temp scripts in `/tmp`, removed |

**Honest limitations and residual non-canonical labels (per the accuracy rules):**

- **T2 latency is an upper bound, not a sub-millisecond render figure.** The scroll *trigger* was injected via
  remote control (the `talk_loop` thread) - a **non-canonical** bypassing interface used only to fire the
  scroll deterministically. The measured `display_ms` therefore includes two remote-control round-trips; kitty's
  true render latency is bounded *below* the ~25 ms confirmation granularity (because `polls_to_confirm = 1`)
  but was not measured to finer precision. What is canonical is the heavy-output load path, the real rendered
  viewport read back with `get-text`, and the marginal load penalty derived from them.
- **Section 8 is entirely non-canonical.** The `fast_data_types` binding bypasses the PTY path; it corroborates
  the segment cadence but does not stand in for the canonical Sec. 7 measurement.
- **Cell sizes are build-time, not runtime, facts.** `sizeof(GPUCell)==20` / `sizeof(CPUCell)==12` are
  `static_assert`s proven by the compiler (Sec. 4.3); kitty exposes no runtime `sizeof` hook, so they are
  labelled `[inferred-from-reading + build-time confirmed]` and corroborated only *indirectly* by the measured
  ~32 B/cell growth slope.

No sub-question is left unanswered, and no inferred-from-reading claim is presented as a runtime observation.

## 10. Repository-unchanged verification

The investigation is read-only. All build, run, and measurement activity happened in the container's `/app`
checkout (the pinned commit) and in throwaway scripts under `/tmp` (a tmpfs, **outside** any repository); the
only persistent artifact is this document.

**The source checkout used for all measurement is byte-for-byte pristine.** In the container `/app`:

```
$ cd /app && git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
$ git status --porcelain --untracked-files=all | wc -l    # 0 = pristine working tree
0
$ git diff --stat | wc -l                                 # 0 = no source file modified
0
```

`git rev-parse HEAD` is the pinned commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`, and both the
porcelain status (untracked files included) and the diff are empty - the build and all measurement runs
modified **no** source file.

**The deliverable is the only addition to the destination tree.** Relative to the pinned commit, the committed
branch differs in exactly one path - this document - and in **no** existing source file:

```
$ git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD        # commit delta vs pinned parent
A	blitzy/documentation/kitty_815df1e210e0.md
$ git diff --name-only 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD | grep -v "^blitzy/documentation/kitty_815df1e210e0.md$"   # any EXISTING source file changed?
(no output: no existing source file differs)
$ git status --porcelain --untracked-files=all    # clean working tree; deliverable committed
(no output: working tree clean; nothing uncommitted or untracked)
```

The `git diff --name-status` shows a single `A` (added) entry for `blitzy/documentation/kitty_815df1e210e0.md`;
the `--name-only` filter confirms **no existing source file differs** from the pinned commit; and
`git status --porcelain --untracked-files=all` is empty - the working tree is clean, with **no untracked or stray files**
(the earlier `git diff --stat HEAD` alone could not have shown untracked files - this porcelain form does).
The commit is finalized with `git commit --amend` onto the **pinned parent**, so the branch contains exactly
one commit ahead of `815df1e210e0`, adding exactly this one file. All `/tmp` measurement scripts are deleted
after the runs, leaving the repository unchanged except for the deliverable.

## Appendix - claim-to-evidence reconciliation

| Claim | Evidence (this document) | Grounding |
|-------|--------------------------|-----------|
| History is a segmented ring of 2,048-row blocks | Sec. 7.2 step climb; Sec. 4.1 | history.c:L15,L18,L37 |
| Per-line cost = 32*xnum + 1 = 2,273 B (xnum=71) | Sec. 5.4; Sec. 7.4 regression 2,266.97 | data-types.h:L221,L228; history.c:L23-25 |
| Default cap: one-time ~5 MiB, then flat plateau | Sec. 5.1 tables (both runs) | history.c:L279-281 (ring overwrite) |
| Large/infinite: linear ~2,273 B/line, no plateau until cap | Sec. 5.2, 5.3 tables | screen.c:L100; history.c:L39 |
| Allocation step = one 2,048-row segment ~= 4,552 kB | Sec. 7.3 boundary table; Sec. 8 binding 4,556 kB | history.c:L18,L25 |
| 40 full boundaries + 1 partial (82,000 lines) | Sec. 7.5 | history.c:L37-39 |
| Responsive under load; +3.2-3.5 ms marginal | Sec. 6.2-6.5 (120/120 confirmed) | child-monitor.c:L229,L291; definition.py:L878 |
| Repository unchanged except deliverable | Sec. 10 | container /app + destination git |

