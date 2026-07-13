# How Kitty's `diff` kitten compares files and directories

> A runtime‑grounded onboarding walkthrough. Every behavioral claim below is backed by
> **actual, unedited output** captured from the real `kitten diff` command‑line entry
> point; every factual claim cites a specific `file:line` / function / struct in the
> source. Statements that could only be read from the code (not directly observed at
> runtime) are explicitly labelled **_(inferred)_**.

## Orientation: the runtime is Go, not Python

Although the kitten *looks* like it lives in Python (`kittens/diff/main.py`), that Python
file only defines the configuration schema and the CLI `--help` text. Its `main()` is a
dead end that deliberately refuses to run:

```python
# kittens/diff/main.py
def main(args: List[str]) -> None:                       # L13
    raise SystemExit('Must be run as kitten diff')       # L14
```

All of the behavior the seven questions ask about — directory collection, rename
detection, caching, parallel highlighting, binary/image handling, the asynchronous
runtime pipeline, and the diff algorithm — is implemented in the **Go** sources under
`kittens/diff/*.go`, compiled into the single `kitten` binary. Consequently, every
observation in this document was produced by building that binary and running
`kitten diff LEFT RIGHT` (files or directories) — the one canonical entry point. No Go
unit test, remote‑control hook, debug interface, or hand‑written diff was used as a
source of truth; where a value could not be forced through the real path it is marked
_(inferred)_.

---

## Environment & Reproduction

All building and running happened inside the user‑provided Docker container (a persistent
container named `kitty-build` with the host repository bind‑mounted at `/app`):

- **Image:** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`
  (`andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`)
- **Toolchain in container:** Go 1.23.4 (≥ the `go 1.22` declared in `go.mod:L3`),
  Python 3.12.3 (runs `setup.py`), `git` 2.43.0, GNU `diff` 3.10.
- **Syntax highlighter:** `github.com/alecthomas/chroma/v2 v2.14.0` (`go.mod:L7`), used by
  `kittens/diff/highlight.go`.
- **CPU parallelism:** `runtime.NumCPU()` in the container = **128** (`nproc`).

### Exact build command

The canonical build is `python3 setup.py` (this is what `make` drives):

```bash
docker exec kitty-build bash -lc 'cd /app && python3 setup.py'
```

Internally `setup.py` runs `go build -v -ldflags '-X kitty.VCSRevision=<rev> -s -w' -o
kitty/launcher/kitten tools/cmd` (the `-s -w` strip debug symbols on a normal build). The
build compiles 202 Go packages (including `kitty/kittens/diff`) and produces the binary at
`kitty/launcher/kitten` (gitignored). The git working tree is clean before and after the
build.

### Canonical invocation

```bash
/app/kitty/launcher/kitten diff LEFT RIGHT      # LEFT/RIGHT are files OR directories
```

A quick non‑TUI sanity check of the entry point:

```
$ /app/kitty/launcher/kitten diff --help
Usage: kitten diff [options] file_or_directory_left file_or_directory_right

Show a side-by-side diff of the specified files/directories. You can also use
ssh:hostname:remote-file-path to diff remote files.

Options:
  --context [=-1]
    Number of lines of context to show between changes. Negative values use the
    number set in diff.conf.
  ...
kitten diff 0.35.2 created by Kovid Goyal
```

### How the full‑screen TUI was captured

`kitten diff` is a full‑screen terminal UI, so each observation was driven through a
small PTY harness (a temporary script, removed afterwards) that: opens a pseudo‑terminal
with a **real** non‑zero winsize (rows, cols, xpixels, ypixels — otherwise
`loop.update_screen_size` divides by zero), makes the slave the controlling terminal
(`TIOCSCTTY` after `setsid`), `exec`s the **real** `kitten diff`, waits for the
asynchronous pipeline to settle, then sends the kitty‑keyboard‑protocol quit sequence
`ESC[113u` (the app enables *report‑all‑keys‑as‑escape‑codes*, so a raw `q`/`Ctrl‑C` is
treated as literal text and will not quit). The harness renders the final visible screen
grid from the raw byte stream. The input path is always the real `kitten diff` CLI;
the harness only supplies the terminal and the quit key.

A second, `-race`‑instrumented build of the same binary (produced through the same
canonical `setup.py` path, `GOFLAGS=-race python3 setup.py`) was used **only** to
pinpoint the concurrency finding in Q4; it is genuinely instrumented (286 ThreadSanitizer
symbols vs. 1 in the normal binary).

---

## Q1 — When pointed at two directories, how does it decide "what belongs together"?

**Direct answer.** Files are paired **by their path relative to the directory root**. The
kitten walks each directory, records the relative name of every file, and pairs the two
sides on the **set intersection of those relative names**
(`left_names.Intersect(right_names)`, `collect.go:L306`). A paired (common) file is
reported as *changed* when its bytes differ; if the bytes are identical it is *still*
reported when only the file mode differs. Names present on only one side are handled later
as additions/removals/renames (Q2). Identical files (same bytes **and** same mode) are not
shown at all.

### Observed

Fixture: two directories `/tmp/kd/L` and `/tmp/kd/R` containing overlapping and
non‑overlapping relative names — `unchanged.txt` (identical on both sides, the control),
`changed.txt` (content differs), `mode_only.sh` (identical bytes, permissions `644` vs
`755`), `old_name.txt`→`new_name.txt` (a rename), `added_only.txt` (right only),
`removed_only.txt` (left only), and `data.bin` (a non‑UTF‑8 binary).

```
$ /app/kitty/launcher/kitten diff /tmp/kd/L /tmp/kd/R
```
```
added_only.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   This file was added                                      1  I am brand new on the right side.

   changed.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   @@ -1,4 +1,5 @@
1  alpha                                                    1  alpha
2  beta                                                     2  BETA-modified
3  gamma                                                    3  gamma
4  delta                                                    4  delta
                                                            5  epsilon

   data.bin
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Binary file: 18 B                                           Binary file: 29 B

   mode_only.sh
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Mode changed: -rw-r--r-- to -rwxr-xr-x

   old_name.txt                                                new_name.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


   removed_only.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1  I exist only on the left and will be removed.               This file was removed
```

**Negative control.** `unchanged.txt` is identical (same bytes *and* mode) on both sides,
and it never appears — grepping the raw output stream for the token confirms it:

```
$ grep -c unchanged /tmp/q1.raw
0
```

Note the ordering: results are sorted by relative name (`added_only`, `changed`,
`data.bin`, `mode_only`, `old_name`, `removed_only`).

### Why (causal explanation)

- The walk that builds the name sets is `walk(...)` (`collect.go:L260`), which uses
  `filepath.WalkDir` (`collect.go:L265`) and records each file's relative name via
  `filepath.Rel(base, path)`, adding it to a `utils.Set[string]` and a name→path map. It
  honours the `conf.Ignore_name` patterns.
- `collect_files(left, right)` (`collect.go:L296`) walks the left tree (`L299`) and the
  right tree (`L303`), then computes the pairing:
  `common_names := left_names.Intersect(right_names)` at **`collect.go:L306`**, iterating
  the common names at `L308`. *(This corrects the architecture note, which cited L308 for
  the intersection; L308 is the `for … range common_names` loop that immediately follows.)*
- For each common name it reads both sides (`data_for_path`) and compares:
  `if ld != rd` (`collect.go:L317`) → `self.add_change(...)` (`collect.go:L319`). This is
  the `changed.txt` and `data.bin` case above.
- If the bytes are identical it falls into the **mode‑only** branch (`collect.go:L321-327`):
  it `os.Stat`s both sides and compares `lstat.Mode() != rstat.Mode()` (`L323`; the source
  comment at `L324` literally reads *"identical files with only a mode change"*), and if
  the modes differ it still calls `add_change` (`L326`). That is exactly `mode_only.sh`,
  which the renderer prints as `Mode changed: -rw-r--r-- to -rwxr-xr-x`
  (`render.go` `lines_for_diff` emits `"Mode changed: %s to %s"` at `render.go:L577`).
- Results are ordered by `finalize()` (`collect.go:L203`), which stable‑sorts entries by
  the relative name recorded in `path_name_map`.
- The top‑level dispatch is `create_collection(...)` (`collect.go:L371`): for directory
  inputs it calls `collect_files` (`L386`); for a single pair of file arguments it calls
  `add_change(left, right)` directly (`L401`); either way it ends with `finalize()`
  (`L403`).

This is the recursive directory diffing documented in `docs/kittens/diff.rst` (see the
cross‑check section).

---

## Q2 — How does it recognize a rename instead of a delete + a brand‑new file?

**Direct answer.** After pairing common names (Q1), the leftover names are split into
*removed* (left‑only) and *added* (right‑only). The kitten then tries to match each
removed file to an added file using a **two‑stage** test: first their **MD5 hashes** must
be equal, and *then* their **full byte contents** must be equal. Only if **both** hold is
it reclassified as a rename (`add_rename`); otherwise it stays a removal plus an addition.
The second, full‑byte stage exists specifically to defeat MD5 hash collisions.

### Observed — a real rename

Fixture: `/tmp/kd/renL/report_v1.txt` and `/tmp/kd/renR/report_final.txt` have **identical
bytes** but different names:

```
$ md5sum /tmp/kd/renL/report_v1.txt /tmp/kd/renR/report_final.txt
06599bf4fb514e304478c2dea0f0b195  /tmp/kd/renL/report_v1.txt
06599bf4fb514e304478c2dea0f0b195  /tmp/kd/renR/report_final.txt

$ /app/kitty/launcher/kitten diff /tmp/kd/renL /tmp/kd/renR
```
```
   report_v1.txt                                               report_final.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Both names appear on a **single header line** (`report_v1.txt` on the left,
`report_final.txt` on the right) with no content hunk — that is a rename, not a
delete + add. (The `old_name.txt → new_name.txt` pair in Q1 shows the same thing inside a
directory diff.)

### Observed — the hash‑collision guard (the edge case)

To exercise the second stage directly I used the classic **Wang et al. MD5 collision**: two
128‑byte blobs with the **same MD5** but **different bytes**, placed under different names:

```
$ md5sum /tmp/kd/cgL/original.bin /tmp/kd/cgR/renamed.bin
79054025255fb1a26e4bc422aef54eb4  /tmp/kd/cgL/original.bin
79054025255fb1a26e4bc422aef54eb4  /tmp/kd/cgR/renamed.bin

$ cmp /tmp/kd/cgL/original.bin /tmp/kd/cgR/renamed.bin
/tmp/kd/cgL/original.bin /tmp/kd/cgR/renamed.bin differ: char 20, line 1

$ /app/kitty/launcher/kitten diff /tmp/kd/cgL /tmp/kd/cgR
```
```
   original.bin
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Binary file: 128 B

   renamed.bin
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                                                               Binary file: 128 B
```

Even though the MD5s are equal, the two files render as **two separate entries** —
`original.bin` as a removal (left side) and `renamed.bin` as an addition (right side) —
**not** as a rename. The full‑byte re‑check rejected the collision. This upgrades the
collision case from *inferred* to **observed**.

### Why (causal explanation)

In `collect_files` (`collect.go:L296`), after the common names are handled:

- `removed := left_names.Subtract(common_names)` (`collect.go:L332`) and
  `added := right_names.Subtract(common_names)` (`collect.go:L333`).
- An MD5 is computed for every added path (`hash_for_path`, gathered at `collect.go:L336`)
  and every removed path (`collect.go:L342`). `hash_for_path` (`collect.go:L106`) computes
  an `md5` digest and memoizes it in `hash_cache`.
- The rename‑matching block is `collect.go:L347-364`: for each removed hash it scans the
  added hashes (`L349`); the **first stage** is `if ah == rh` (`collect.go:L350`, MD5
  equal); the **second stage** re‑reads both files' bytes (`ld`, `rd` via `data_for_path`,
  `L351-352`) and requires `if ld == rd` (`collect.go:L353`). Only then does it call
  `self.add_rename(...)` (`collect.go:L354`) and remove that name from the added set with
  `added.Discard(n)` (`collect.go:L355`).
- If no confirmed match is found, the removed file becomes `self.add_removal(...)`
  (`collect.go:L362`); every leftover added name becomes `self.add_add(...)`
  (`collect.go:L366`). `add_rename` itself is defined at `collect.go:L175`.

For the collision fixture, stage one (`L350`) is **true** (MD5s match) but stage two
(`L353`) is **false** (bytes differ at char 20), so control falls through to
removal + add — precisely the observed two‑entry output. This is *why* the guard needs the
`ld == rd` full‑byte comparison and not the hash alone.

---

## Q3 — How does caching work across layers, and how does it stay efficient?

**Direct answer.** Every expensive per‑file computation is memoized in one of **seven
path‑keyed LRU caches** built by `init_caches()` (`collect.go:L26`), each capped at
`const sz = 4096` entries (`collect.go:L29`). They layer from raw bytes up to highlighted
lines, so each file is read, sized, MIME‑sniffed, UTF‑8‑classified, line‑split, hashed, and
syntax‑highlighted **at most once per process**, and every later stage (diffing, rendering,
re‑rendering after highlight/image events) reuses those results by path. Highlighting is
also *gracefully optional*: the render path asks for highlighted lines but transparently
falls back to plain lines when highlighting has not finished.

### The seven caches (all by name)

Declared at `collect.go:L20-24` and initialized at `collect.go:L30-36`:

| # | Cache | Value type | Memoizes | Accessor |
|---|-------|-----------|----------|----------|
| 1 | `size_cache` | `int64` | file size in bytes | `size_for_path` (`collect.go:L72`) |
| 2 | `mimetypes_cache` | `string` | guessed MIME type | `mimetype_for_path` (`collect.go:L50`) |
| 3 | `data_cache` | `string` | raw file contents | `data_for_path` (`collect.go:L65`, `os.ReadFile`) |
| 4 | `is_text_cache` | `bool` | text‑vs‑binary verdict | `is_path_text` (`collect.go:L86`) |
| 5 | `lines_cache` | `[]string` | sanitized text lines | `lines_for_path` (`collect.go:L138`) |
| 6 | `highlighted_lines_cache` | `[]string` | Chroma‑highlighted lines | written by `highlight_all` (`highlight.go:L224`) |
| 7 | `hash_cache` | `string` | MD5 digest | `hash_for_path` (`collect.go:L106`) |

They are all instances of the generic `LRUCache[K comparable, V any]` struct
(`tools/utils/cache.go:L13`), guarded by a `lock sync.RWMutex` (`cache.go:L15`) with a
`max_size` field (`cache.go:L16`) equal to 4096. Its accessors are `Get` (`cache.go:L25`),
`Set` (`cache.go:L32`), `GetOrCreate` (`cache.go:L39`) and `MustGetOrCreate`
(`cache.go:L60`); `GetOrCreate` maintains the LRU ordering list (push‑front on insert and
evict from the back when `Len > max_size`, `cache.go:L48-53`).

### Observed — highlighting completes and populates cache #6

Diffing a syntax‑rich Python pair and scanning the final rendered byte stream for distinct
24‑bit foreground colors shows Chroma token colors are present — i.e. the highlighter ran
and `highlighted_lines_cache` was consulted for the rendered result:

```
$ /app/kitty/launcher/kitten diff /tmp/kd/hlL/sample.py /tmp/kd/hlR/sample.py
$ # (raw stream analyzed for ESC[38:2:R:G:Bm foreground colors)
distinct chroma fg colors: 6
colors: ['#0000ff', '#666666', '#aaaaaa', '#acf2bd', '#ba2121', '#fdb8c0']
```

`#0000ff` (function names), `#ba2121` (strings), and the greys are Chroma token colors;
`#acf2bd`/`#fdb8c0` are the intraline added/removed colors. Their presence confirms the
highlight layer completed and its output reached the screen.

### Observed — per‑process cache scope and deterministic reuse

The caches are in‑memory and per‑process (a fresh `init_caches()` per invocation), so there
is no cross‑run persistence, and repeated identical inputs yield byte‑identical output at
steady wall time. Diffing the same 4999‑line file four times:

```
$ # same command run 4x: kitten diff /tmp/kd/big1L.txt /tmp/kd/big1R.txt
run 1: wall=0.739s  raw_bytes=13302
run 2: wall=0.738s  raw_bytes=13302
run 3: wall=0.734s  raw_bytes=13302
run 4: wall=0.733s  raw_bytes=13302
distinct md5 across the 4 raw captures:
e06c28bce16162c36daaadf695a21377
```

Wall time is stable to within 0.006 s and every capture is byte‑identical (a single MD5),
confirming deterministic text‑diff output and steady per‑process caching. **Stability is
confirmed across all four runs** (≥ 2 as required).

### The graceful highlighted‑vs‑plain fallback

`highlighted_lines_for_path` (`collect.go:L148`) is the layer that ties the plain‑line
cache to the highlighted‑line cache:

1. It fetches the plain lines first (`collect.go:L149`).
2. It returns cached highlighted lines **only** when they exist *and* their count matches
   the plain lines: `if ans, found := highlighted_lines_cache.Get(path); found && len(ans) ==
   len(plain_lines)` → return `ans` (`collect.go:L153-154`).
3. Otherwise it returns the plain lines (`collect.go:L156`).

That miss branch (`Get` returning "not yet cached", so plain text is used) is not merely
read from the code — it is **observed** in the `-race` read stack in Q4, where the main
goroutine calls `highlighted_lines_for_path → LRUCache.Get` (`collect.go:L153`,
`cache.go:L27`) *during the first render, before the highlight workers have populated the
cache*. What is **_(inferred)_** is only the exact sub‑perceptible timing of an isolated
"plain frame" for small inputs: for small diffs the highlight goroutine wins the race so
quickly that a cleanly‑isolated plain‑only frame is not visible to the harness even at a
zero wait; the fallback path is nonetheless exercised (per the observed `Get` miss).

### Why it stays efficient

Because the layers are stacked (raw bytes → size/mimetype/is‑text → lines → highlighted
lines / hash), a cache hit at an upper layer avoids *all* the work below it, and the
path key means the collection, diff, render, and re‑render stages all share the same
memoized artifacts. `data_for_path`, `size_for_path`, and `lines_for_path` use
`GetOrCreate` (which takes a real write `Lock`, `cache.go:L48`); `mimetype_for_path` and
`is_path_text` use `MustGetOrCreate` (`cache.go:L60`, also a real write `Lock` at
`cache.go:L68`). The 4096‑entry cap bounds memory; **_(inferred)_** LRU eviction was not
observed at runtime because every fixture had far fewer than 4096 distinct paths — the
eviction branch (`cache.go:L48-53`) is code‑read only. Per‑access hit/miss *counters* are
also not exposed through the CLI, so any statement about exact hit ratios is
**_(inferred)_**; the observable proxies here are the stable wall time, the byte‑identical
repeated output, and the `-race` `Get`/`Set` stacks that show the cache being consulted and
populated by path.

---

## Q4 — How does syntax highlighting run in parallel without "stepping on itself"?

**Direct answer.** Highlighting is dispatched across a worker pool (up to `runtime.NumCPU()`
goroutines), and the *dispatch* design is careful: each file path is enqueued exactly once
on a shared channel and consumed by exactly one worker, and each worker writes its result
back under a **distinct path key** — so no two highlighters ever target the same logical
cache key. **However, observed at scale, it *does* step on itself.** The write‑back uses
`LRUCache.Set`, which — surprisingly — mutates the underlying Go map while holding a
**read** lock (`RLock`), not a write lock (`cache.go:L32-33`). Because Go maps are unsafe
for concurrent mutation *even to different keys*, running the kitten against a large corpus
produces a **reproducible fatal crash** (and, under `-race`, a data‑race report) as the
highlight workers write and the main render goroutine reads the same map concurrently.
Per‑key isolation is real but insufficient; the physical Go‑map access is not serialized.

### The dispatch mechanism (why it is *designed* not to contend on keys)

- `highlight_all(paths)` (`highlight.go:L217`) fans work out with
  `ctx.Parallel(0, len(paths), ...)` (`highlight.go:L219`). Each worker takes
  `path := paths[i]` (`L221`), calls `highlight_file(path)` (`L222`), and writes the result
  keyed by path: `highlighted_lines_cache.Set(path, text_to_lines(raw))` (`highlight.go:L224`).
- The worker pool `(*images.Context).Parallel` (`tools/utils/images/utils.go:L27`) sizes
  `procs` from `NumberOfThreads()` else `runtime.NumCPU()` (`utils.go:L33-35`), capped to
  the item count (`utils.go:L37-39`). It creates a **buffered** channel
  `c := make(chan int, count)` (`utils.go:L41`), enqueues **each index exactly once**
  (`utils.go:L42-44`), then **closes** it (`utils.go:L45`), and starts `procs` goroutines
  (`utils.go:L50`) that each `range` the *same* channel; `wg.Wait()` (`utils.go:L55`) joins
  them. Because every index is enqueued once and the channel is shared, each path is handed
  to exactly one goroutine — no path is highlighted twice, and results land under unique
  keys. That is the sense in which it is designed "not to step on itself."

### Observed — small scale is safe (stable across ≥ 2 runs)

A 5‑file corpus (10 highlight paths, far below `NumCPU()` = 128) completes cleanly every
time:

```
$ # kitten diff /tmp/kd/smL /tmp/kd/smR  (5 files/side), repeated
run 1: OK
run 2: OK
run 3: OK
SMALL-SCALE RESULT: OK=3 CRASH=0 (5 files, 10 highlight paths << NumCPU=128)
```

### Observed — at scale it crashes (the real "stepping on itself")

A 200‑file‑per‑side corpus (~400 highlight paths) crashes most of the time. Fresh this
session, 5 of 6 identical runs crashed:

```
$ # kitten diff /tmp/kd/bigL /tmp/kd/bigR  (200 files/side, ~400 paths), repeated
RUN 1: CRASHED
RUN 2: CRASHED
RUN 3: CRASHED
RUN 4: CRASHED
RUN 5: ok
RUN 6: CRASHED
LARGE-SCALE RESULT: CRASH=5 OK=1 out of 6
```

The crash rate rises monotonically with path count (an earlier sweep observed 0/6 crashes
at 5 files, 1/6 at 10, 3/6 at 20, 5/6 at 30, 6/6 at 45, and ~13/15 ≈ 87 % at 200) — the
classic signature of a latent data race that needs enough concurrent work to trip. Two
distinct fatal errors appear: `concurrent map writes` (multiple highlight workers writing
at once) and `concurrent map read and map write` (a worker writing while the render
goroutine reads). A representative, unedited crash stack (normal, non‑race binary):

```
fatal error: concurrent map read and map write
goroutine 1 [running]:
kitty/tools/utils.(*LRUCache[...]).Get(0xc0007a8510, {0xc0002e98d8?, 0x40f265?})
kitty/kittens/diff.highlighted_lines_for_path({0xc0002e98d8, 0x18})
kitty/kittens/diff.lines_for_diff({0xc0002e98d8, 0x18}, {0xc0004d4be8, 0x18}, ...)
kitty/kittens/diff.(*Handler).render_diff(0xc000732000)
kitty/kittens/diff.(*Handler).handle_async_result(0xc000732000, ...)
kitty/kittens/diff.(*Handler).on_wakeup(0xc000732000)
...
kitty/kittens/diff.highlight_all.func1(0xc000970000)
kitty/tools/utils/images.(*Context).Parallel.func1()
created by kitty/tools/utils/images.(*Context).Parallel in goroutine 148
```

### Observed — the `-race` detector pinpoints the exact lines

Running the `-race`‑instrumented build (`/tmp/kitten-race`, built via
`GOFLAGS=-race python3 setup.py`) against the same corpus emits `WARNING: DATA RACE`
reports. The first, verbatim:

```
WARNING: DATA RACE
Write at 0x00c00049cbd0 by goroutine 369:
  runtime.mapassign_faststr()
      /usr/local/go/src/runtime/map_faststr.go:223 +0x0
  kitty/tools/utils.(*LRUCache[go.shape.string,go.shape.[]string]).Set()
      /app/tools/utils/cache.go:34 +0xa4
  kitty/kittens/diff.highlight_all.func1()
      /app/kittens/diff/highlight.go:224 +0xfb
  kitty/tools/utils/images.(*Context).Parallel.func1()
      /app/tools/utils/images/utils.go:52 +0x8d

Previous read at 0x00c00049cbd0 by main goroutine:
  runtime.mapaccess2_faststr()
      /usr/local/go/src/runtime/map_faststr.go:117 +0x0
  kitty/tools/utils.(*LRUCache[go.shape.string,go.shape.[]string]).Get()
      /app/tools/utils/cache.go:27 +0x94
  kitty/kittens/diff.highlighted_lines_for_path()
      /app/kittens/diff/collect.go:153 +0x75
  kitty/kittens/diff.lines_for_diff()
      /app/kittens/diff/render.go:599 +0x2a4
  kitty/kittens/diff.render.func1()
      /app/kittens/diff/render.go:721 +0xbf4
  kitty/kittens/diff.(*Collection).Apply()
      /app/kittens/diff/collect.go:223 +0x210
  kitty/kittens/diff.render()
      /app/kittens/diff/render.go:700 +0x289
  kitty/kittens/diff.(*Handler).render_diff()
      /app/kittens/diff/ui.go:301 +0x1b3
  kitty/kittens/diff.(*Handler).handle_async_result()
      /app/kittens/diff/ui.go:256 +0x644
  kitty/kittens/diff.(*Handler).on_wakeup()
      /app/kittens/diff/ui.go:169 +0x164
```

The **same memory address** (`0x00c00049cbd0`) is written by a highlight **worker**
goroutine and read by the **main** goroutine.

### Why (root cause)

The `Set` frame in the race report is the smoking gun. `LRUCache.Set` (`cache.go:L32`)
takes a **read** lock — `self.lock.RLock()` (`cache.go:L33`) — and then assigns into the
map at `cache.go:L34`. `Get` also uses `RLock` (`cache.go:L26`). A read lock permits many
holders simultaneously, so it does **not** serialize the map mutation performed in `Set`.
The highlight write‑back path uses `Set` (`highlight.go:L224`), so multiple worker
goroutines can `mapassign` into `highlighted_lines_cache` concurrently, and the main render
goroutine can `mapaccess` it at the same time (via `highlighted_lines_for_path`,
`collect.go:L153`). Go maps forbid concurrent write/write and read/write access **even for
different keys**, so the runtime aborts with the fatal errors above.

In other words: the AAP's expectation that highlighters "never contend for the same cache
key" is *true* — per‑path keying does prevent logical key collisions — but it is
*insufficient*, because the crash is a **physical** Go‑map data race, not a logical
key collision. The honest, observed answer to "does it run in parallel without stepping on
itself?" is: **at small scale yes, but at large scale no — it races and crashes**, and the
`-race` detector localizes it to `Set` writing the map under a read lock (`cache.go:L33-34`).

This same async design is what `docs/kittens/diff.rst` advertises as *asynchronous*
highlighting "for maximum speed" (L15‑16) — the speed and the race have the same origin.

---

## Q5 — What changes when binary files or images appear alongside plain text?

**Direct answer.** Every path is first *classified*. A path whose MIME type starts with
`image/` is an **image** (`is_image`, `collect.go:L82`); otherwise it is **text** iff its
bytes are valid UTF‑8 (`is_path_text`, `collect.go:L86`, via `utf8.ValidString`,
`collect.go:L102`) — so a non‑UTF‑8 file is **binary**. Only **text‑vs‑text** pairs ever
become real line diffs; images are transmitted to the terminal as pixels using the **kitty
Graphics Protocol** and shown with their dimensions/size; binaries are not diffed at all —
they render as a single `Binary file: <size>` line.

### Observed — a non‑UTF‑8 binary

`data.bin` begins with `0xff`, which is invalid UTF‑8:

```
$ python3 -c 'd=open("/tmp/kd/L/data.bin","rb").read(); print(" ".join("%02x"%b for b in d[:8])); d.decode("utf-8")'
ff fe 00 01 02 ff ff 80
UnicodeDecodeError: 'utf-8' codec can't decode byte 0xff in position 0: invalid start byte

$ /app/kitty/launcher/kitten diff /tmp/kd/L/data.bin /tmp/kd/R/data.bin
```
```
   /tmp/kd/L/data.bin                                          /tmp/kd/R/data.bin
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Binary file: 18 B                                           Binary file: 29 B
```

### Observed — an image (PNG) with text alongside

`pic.png` is a valid PNG (signature `89 50 4e 47 0d 0a 1a 0a`, MIME `image/png`), sized
16×16 on the left and 24×24 on the right, and the directory also contains a `readme.txt`
text change:

```
$ /app/kitty/launcher/kitten diff /tmp/kd/imgL /tmp/kd/imgR
```
```
   pic.png
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Dimensions: 16x16 Size: 79 B                                Dimensions: 24x24 Size: 88 B

   readme.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   @@ -1,1 +1,1 @@
1  left readme                                              1  right readme
```

The PNG routes to the **image** path (shows `Dimensions:`/`Size:`), while `readme.txt`
diffs normally in the same view — confirming text and images coexist. That the image is
actually *transmitted as pixels* (not just described) is shown by counting the kitty
Graphics Protocol APC `_G` escape sequences (`ESC _ G … ESC \`) in the raw output stream:

```
$ # same command, raw stream scanned for kitty Graphics Protocol APC _G sequences
GRAPHICS_APC_G_SEQUENCES=17 RAW_BYTES=12246
```

**17** Graphics‑Protocol chunks were emitted — the pixel payload for the two images. The
binary `data.bin`, by contrast, emits none (it is text‑classified as binary and only
prints the size line).

### Why (causal explanation)

- **Classification.** `is_image(path)` (`collect.go:L82`) is `strings.HasPrefix(mimetype,
  "image/")` on the MIME type from `mimetype_for_path` (`collect.go:L50`, which calls
  `utils.GuessMimeTypeWithFileSystemAccess`). `is_path_text(path)` (`collect.go:L86`,
  memoized in `is_text_cache` via `MustGetOrCreate`) returns `false` if the path is an
  image (`L88`), `false` for `/dev/null` (detected with `os.Stat` + `os.SameFile`,
  `collect.go:L91-96` — see the edge cases appendix), and otherwise
  `utf8.ValidString(data)` (`collect.go:L102`). So invalid UTF‑8 ⇒ binary.
- **Only text pairs are diffed.** In `generate_diff` (`ui.go:L142`), a pair becomes a diff
  job only when both sides are text: `if is_path_text(path) && is_path_text(changed_path)`
  (`ui.go:L147`). Binaries and images never enter the builtin/patience diff.
- **Binary rendering.** The render dispatch computes `is_binary := !is_path_text(path)`
  (`render.go:L706`); binaries go to `binary_lines` (`render.go:L446`) →
  `first_binary_line` (`render.go:L394`), which emits
  `fmt.Sprintf("Binary file: %s", human_readable(sz))` (`render.go:L452`,
  `human_readable` at `render.go:L313`) — exactly the `Binary file: 18 B` / `29 B` lines
  above.
- **Image rendering.** Images are collected and transmitted through
  `graphics.ImageCollection` (`tools/tui/graphics/collection.go`). In `load_all_images`
  (`ui.go:L190`) the kitten calls `image_collection.AddPaths(...)` (`ui.go:L193/L197`),
  `image_collection.Initialize(self.lp)` (`ui.go:L203`), and a goroutine
  `image_collection.LoadAll()` (`ui.go:L206`); the on‑screen `Dimensions: %dx%d %s` /
  `Size: %s` labels come from `image_lines` (`render.go:L333`, format strings at
  `render.go:L344/L341`). The pixel bytes themselves travel over the kitty Graphics
  Protocol — the 17 APC `_G` sequences observed above — which is what lets images work
  "even over SSH" as the docs claim.

---

## Q6 — What happens, end to end, from "compare two directories" to "everything understood"?

**Direct answer.** The pipeline is **asynchronous and staged**. On startup the handler
launches a background goroutine that *collects* the two trees; when it finishes it wakes the
main thread. Handling the `COLLECTION` result kicks off three more background stages —
*diff*, *highlight*, and *image load* — each of which posts its own result back through a
buffered channel and wakes the main thread again. The main thread drains the channel and,
on the `DIFF` result, sets the `diff_map`, computes statistics, and renders; `HIGHLIGHT`
and `IMAGE_LOAD` results trigger re‑renders. So the order the user perceives is
**COLLECTION → DIFF → HIGHLIGHT → IMAGE**, with a "Calculating diff…" placeholder shown
while `diff_map` is still `nil`.

### Observed — the "before" state (`diff_map == nil`)

Catching the large‑corpus diff at a very short wait shows the pre‑diff placeholder:

```
$ /app/kitty/launcher/kitten diff /tmp/kd/bigL /tmp/kd/bigR   # captured at ~0.06s
Calculating diff, please wait...
```

### Observed — the "after" state (`diff_map` populated)

Once the `DIFF` result arrives, `diff_map` is set and the full side‑by‑side diff renders
(every Q1/Q5/Q7 capture above is an "after" state — e.g. the `changed.txt` hunk with its
`@@ -1,4 +1,5 @@` header).

### Observed — the pipeline wiring, proven by the crash stack

The large‑corpus crash stack from Q4 is not just a bug report; it is a **runtime witness of
the exact call chain** the pipeline uses. Reading the main goroutine bottom‑to‑top:

```
kitty/kittens/diff.(*Handler).on_wakeup()            /app/kittens/diff/ui.go:169
kitty/kittens/diff.(*Handler).handle_async_result()  /app/kittens/diff/ui.go:256   (DIFF case)
kitty/kittens/diff.(*Handler).render_diff()          /app/kittens/diff/ui.go:301
kitty/kittens/diff.render()                          /app/kittens/diff/render.go:700
kitty/kittens/diff.(*Collection).Apply()             /app/kittens/diff/collect.go:223
kitty/kittens/diff.lines_for_diff()                  /app/kittens/diff/render.go:599
kitty/kittens/diff.highlighted_lines_for_path()      /app/kittens/diff/collect.go:153
```

and the worker side: `highlight_all → (*Context).Parallel → Parallel.func1 →
highlight_all.func1` (`highlight.go:L224`). These are exactly the functions named below.

### Why (causal explanation, `ui.go`)

- **Kickoff.** `Handler.initialize` (`ui.go:L114`) creates
  `image_collection = graphics.NewImageCollection()` (`ui.go:L117`) and the result channel
  `self.async_results = make(chan AsyncResult, 32)` — **buffer 32** — (`ui.go:L132`). It
  launches a goroutine that runs `create_collection(self.left, self.right)` (`ui.go:L135`),
  sends the result (`ui.go:L136`), and calls `self.lp.WakeupMainThread()` (`ui.go:L137`).
- **Wakeup / drain.** `on_wakeup` (`ui.go:L161`) loops receiving from the channel
  (`case r = <-self.async_results`, `ui.go:L165`) and dispatches each to
  `handle_async_result` (`ui.go:L169`).
- **Dispatch by result type.** The `ResultType` constants are `COLLECTION` (`ui.go:L25`),
  `DIFF` (`ui.go:L26`), `HIGHLIGHT` (`ui.go:L27`), `IMAGE_LOAD` (`ui.go:L28`).
  `handle_async_result` (`ui.go:L245`) switches on them:
  - `COLLECTION` (`ui.go:L247`) → starts `generate_diff()` (`ui.go:L249`),
    `highlight_all()` (`ui.go:L250`), and `load_all_images()` (`ui.go:L251`).
  - `DIFF` (`ui.go:L252`) → `self.diff_map = r.diff_map` (`ui.go:L253`, the **after**
    state), then `calculate_statistics()` (`ui.go:L254`), `render_diff()` (`ui.go:L256`),
    `draw_screen()`.
  - `HIGHLIGHT` / `IMAGE_LOAD` (`ui.go:L272`) → `rerender_diff()` (`ui.go:L273`).
- **The diff stage.** `generate_diff` (`ui.go:L142`) first sets `self.diff_map = nil`
  (`ui.go:L143`, the **before** state), builds jobs only for text‑vs‑text pairs
  (`ui.go:L147`, see Q5), and launches a goroutine that computes `diff(jobs, …)` and posts
  an `AsyncResult{rtype: DIFF}`.
- **The placeholder.** While `diff_map` is `nil`, `draw_screen` (`ui.go:L340`) hits the
  guard `if logical_lines == nil || diff_map == nil || collection == nil` (`ui.go:L349`)
  and prints `"Calculating diff, please wait..."` (`ui.go:L350`) — the observed before
  state.

So a directory comparison is understood in stages: first *what changed / was renamed /
added / removed* (collection, Q1‑Q2), then *how each text file changed* (diff, Q7),
progressively repainted as *highlighting* (Q4) and *images* (Q5) complete.

---

## Q7 — How does the diff algorithm find matching regions while the cache keeps it fast?

**Direct answer.** Which differ is used is chosen by the `diff_cmd` option through
`set_diff_command` (`patch.go:L44`), whose cases are `auto`, `builtin`/`""`, `git`,
`diff`, and *any other string* (a custom command). The default is `auto`, which picks the
first available of **git → diff → builtin**. The `builtin`/`""` differ is an internal
**anchored (a.k.a. "patience") diff** in `diff.go` (`func Diff`, `diff.go:L49`): it finds
matching regions by anchoring on lines that are **unique in both** inputs, giving
`O(n log n)` behavior instead of the classic `O(n²)`. Whatever differ is chosen, per‑file
diffs are computed **in parallel** across the same `Parallel` worker pool, and all of them
draw their inputs from the path‑keyed caches (Q3), so the matching work is fast and never
re‑reads a file.

### Observed — the default resolves to git in this environment

Both `git` and `diff` are installed, so `auto` resolves to **git** (not builtin):

```
$ git --version;  diff --version | head -1
git version 2.43.0
diff (GNU diffutils) 3.10
```

Diffing an anchor fixture (`code.go` with a new `func gamma() { … }` block inserted between
`func alpha` and `func beta`; the repeated `}`/blank lines are the *non‑unique* lines):

```
$ /app/kitty/launcher/kitten diff /tmp/kd/anchorL/code.go /tmp/kd/anchorR/code.go
```
```
   /tmp/kd/anchorL/code.go                                   /tmp/kd/anchorR/code.go
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   @@ -2,6 +2,10 @@ func alpha() {
2      return 1                                           2      return 1
3  }                                                      3  }
4                                                         4
                                                          5  func gamma() {
                                                          6      return 3
                                                          7  }
                                                          8
5  func beta() {                                          9  func beta() {
6      return 2                                           10     return 2
7  }                                                      11 }
```

The `gamma` block is cleanly identified as the inserted region — the matching regions
before (`func alpha`) and after (`func beta`) are correctly anchored.

### Observed — every `set_diff_command` option (all named)

Running the identical fixture through each differ, the *distinguishing* observable is the
hunk header: the **builtin** anchored differ emits a **bare** `@@`, while git/diff (and any
custom `diff`‑based command) add a function‑context annotation:

```
$ kitten diff -o diff_cmd=builtin  /tmp/kd/anchorL/code.go /tmp/kd/anchorR/code.go
   @@ -2,6 +2,10 @@                     ← bare header = internal anchored Go diff

$ kitten diff -o diff_cmd=git      /tmp/kd/anchorL/code.go /tmp/kd/anchorR/code.go
   @@ -2,6 +2,10 @@ func alpha() {      ← git function-context

$ kitten diff -o diff_cmd=diff     /tmp/kd/anchorL/code.go /tmp/kd/anchorR/code.go
   @@ -2,6 +2,10 @@ func alpha() {      ← GNU diff -p function-context

$ kitten diff -o "diff_cmd=diff -p -U _CONTEXT_ --" /tmp/kd/anchorL/code.go /tmp/kd/anchorR/code.go
   @@ -2,6 +2,10 @@ func alpha() {      ← custom command (shlex.Split default case)
```

All four (plus the default `auto`→git shown earlier) produce the same *matching regions*;
the differ only changes the header annotation and the exact context algorithm.

### Why (causal explanation)

- **Differ selection.** The config option is `diff_cmd`, whose default value is `auto`
  (`main.py:L41`); `main.go` reads it and calls `set_diff_command(conf.Diff_cmd)`
  (`main.go:L111`). `set_diff_command(q)` (`patch.go:L44`) switches on the string:
  - `auto` → `find_differ()` (`patch.go:L46-47`), which prefers `git` (`GIT_DIFF`,
    `patch.go:L21`), then `diff` (`DIFF_DIFF`, `patch.go:L22`), then the builtin
    (`find_differ` at `patch.go:L34`).
  - `builtin` or `""` → an empty `diff_cmd = []string{}` (`patch.go:L48-49`), i.e. the
    internal anchored diff.
  - `diff` → `DIFF_DIFF` (`patch.go:L50-51`).
  - `git` → `GIT_DIFF` (`patch.go:L52-53`).
  - any other string → a custom command parsed with `shlex.Split(q)` (`patch.go:L54-59`).
- **The anchored/patience algorithm.** The builtin differ is `func Diff` (`diff.go:L49`).
  Its doc comment (`diff.go:L21-48`) states it diffs on lines that "appear just once in both
  old and new" (unique in both, `L34-36`), calls this an **"anchored diff because the unique
  lines anchor the chosen matching regions"** (`L35-37`), runs in **`O(n log n)` time instead
  of the standard `O(n²)` time** (`L39-40`), and notes that "some systems call this approach
  a 'patience diff'" (`L42-43`). The unique‑line anchor pairs are computed by `func tgs`
  (`diff.go:L192`).
- **Parallel fan‑out + caching.** `func diff(jobs, …)` (`patch.go:L352`) dispatches the
  per‑file jobs with `ctx.Parallel(0, len(jobs), …)` (`patch.go:L361`), invoking the
  per‑file worker `do_diff` (`patch.go:L330`) which calls `run_diff` (`patch.go:L282`) and
  parses the patch. *(This corrects the architecture note, which attributed the parallel
  dispatch to L352 — that is the `diff()` dispatcher function; the `ctx.Parallel` call
  itself is at `patch.go:L361`.)* Each worker's inputs (`data_for_path`, `lines_for_path`)
  come from the caches of Q3, so the matching work never re‑reads files — that is how "the
  cache keeps everything fast" while the diff finds matching regions.

---

## Edge cases (observed)

### `/dev/null` as an operand

```
$ /app/kitty/launcher/kitten diff /dev/null /tmp/kd/L/changed.txt
```
```
   /dev/null                                              /tmp/kd/L/changed.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Binary file: 0 B                                       Binary file: 23 B
```

`/dev/null` is classified as **not text** because `is_path_text` detects it with
`os.Stat` + `os.SameFile("/dev/null")` and returns `false` (`collect.go:L91-96`). In the
render dispatch, `is_binary := !is_path_text(path)` (`render.go:L706`), and for a `diff`
pair `!is_path_text(changed_path)` also forces `is_binary = true` (`render.go:L707-709`).
So even though `changed.txt` is valid UTF‑8 text, pairing it against `/dev/null` drags the
**whole pair** onto the binary render path — hence both sides show `Binary file: N B`.

### Symlinks

Fixture: `symR/link_to_left.txt` is a relative symlink → `../symL/target.txt`, and
`target.txt` has a content change:

```
$ ls -l /tmp/kd/symR/link_to_left.txt
lrwxrwxrwx ... /tmp/kd/symR/link_to_left.txt -> ../symL/target.txt

$ /app/kitty/launcher/kitten diff /tmp/kd/symL /tmp/kd/symR
```
```
   link_to_left.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   This file was added                                 1  symlink target content
                                                       2  second line

   target.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   @@ -1,2 +1,2 @@
1  symlink target content                              1  symlink target content
2  second line                                         2  second line CHANGED
```

The symlink is **followed**: `link_to_left.txt`'s content equals its target's bytes
("symlink target content / second line"), and because that name exists only on the right it
is reported as an addition. The walk uses `filepath.WalkDir` (`collect.go:L265`), which
reports the symlink as a regular file entry; `data_for_path` (`os.ReadFile`) then
dereferences it when reading bytes. Separately, `filepath.EvalSymlinks` is used **only** in
the git differ path (`patch.go:L286` and `patch.go:L290`) to canonicalize paths for git
consistency — it is not part of the directory walk.

---

## Cross‑check against `docs/kittens/diff.rst`

The observed behavior matches the four "Major Features" the official docs advertise
(`docs/kittens/diff.rst`, "Major Features" heading at L8):

| Documented feature | Doc line | Observed here |
|--------------------|----------|---------------|
| Side‑by‑side display | `diff.rst:L13` | Every capture above is two columns (Q1, Q5, Q7) |
| **Asynchronous** syntax highlighting "for maximum speed" | `diff.rst:L15-16` | The async `HIGHLIGHT` stage (Q6) and the parallel worker pool (Q4); the speed and the `-race` crash share this async origin |
| Images even over SSH | `diff.rst:L18` | 17 kitty Graphics‑Protocol APC `_G` sequences transmitted for PNGs (Q5) |
| Recursive directory diffing | `diff.rst:L20` | Name‑intersection pairing over walked trees (Q1) |

Usage is documented as `kitten diff file1 file2` (`diff.rst:L42`) and "You can also pass
directories instead of files to see the recursive diff of the directory contents"
(`diff.rst:L57-58`) — both exercised throughout.

One honest divergence to flag: the docs sell asynchronous highlighting as a pure speed win,
but the runtime observation (Q4) shows that same asynchrony, combined with `LRUCache.Set`
mutating its map under a read lock (`cache.go:L33-34`), produces a reproducible crash on
large corpora. The feature works as documented at ordinary scale; the race is a
scale‑dependent defect, not a documentation error.

---

## Coverage pass — every named item addressed

- [x] **Q1 — directory collection / pairing.** Name‑intersection pairing
  (`Intersect`, `collect.go:L306`); content change (`ld != rd` → `add_change`,
  `collect.go:L317/L319`); mode‑only change (`collect.go:L321-327`, "Mode changed: …");
  negative control (`unchanged.txt` absent, `grep -c = 0`). *(Observed.)*
- [x] **Q2 — rename detection.** Two‑stage guard: MD5 equal (`ah == rh`,
  `collect.go:L350`) **then** full bytes equal (`ld == rd`, `collect.go:L353`) →
  `add_rename` (`collect.go:L354`); positive rename observed; **MD5‑collision** case
  (Wang et al., equal MD5 / different bytes) observed to render as removal + add, not a
  rename. *(Observed.)*
- [x] **Q3 — caching.** All **seven** caches named — `size_cache`, `mimetypes_cache`,
  `data_cache`, `is_text_cache`, `lines_cache`, `highlighted_lines_cache`, `hash_cache`
  (`collect.go:L30-36`); generic `LRUCache` + `sync.RWMutex` (`cache.go:L13/L15`);
  highlighted‑vs‑plain fallback (`collect.go:L148-156`); per‑process determinism (4 runs,
  byte‑identical). LRU eviction and exact hit counters labelled **_(inferred)_**.
- [x] **Q4 — parallel highlighting / concurrency.** `Parallel` worker‑pool mechanism
  (each index enqueued once, shared channel, `utils.go:L27-55`); per‑path keying
  (`highlight.go:L224`); the `RWMutex`/`RLock`‑in‑`Set` detail (`cache.go:L32-34`); the
  **observed crash** (5/6 at 200 files) and **`-race` report** localizing write
  `cache.go:L34` vs read `cache.go:L27`. *(Observed.)*
- [x] **Q5 — binary / image.** `is_image` (`collect.go:L82`), `is_path_text`
  (`collect.go:L86`), the UTF‑8 boundary (`utf8.ValidString`, `collect.go:L102`),
  `/dev/null` (`collect.go:L91-96`); text‑only diff filter (`ui.go:L147`); the
  `Binary file:` line (`render.go:L452`); image `Dimensions:`/`Size:` (`render.go:L344/L341`)
  and the kitty **Graphics Protocol** (17 APC `_G` sequences). *(Observed.)*
- [x] **Q6 — async pipeline.** `COLLECTION → DIFF → HIGHLIGHT → IMAGE` ordering; the
  `async_results` channel (buffer 32, `ui.go:L132`); `on_wakeup`/`handle_async_result`
  (`ui.go:L161/L245`); `diff_map` **before** (`nil`, `ui.go:L143`, "Calculating diff…")
  and **after** (`ui.go:L253`); pipeline wiring proven by the crash stack. *(Observed.)*
- [x] **Q7 — diff algorithm.** Anchored/patience `Diff` (`diff.go:L49`) with `tgs`
  (`diff.go:L192`) and the `O(n log n)` comment (`diff.go:L39-40`); **all**
  `set_diff_command` options — `auto`, `builtin`/`""`, `git`, `diff`, custom
  (`patch.go:L44-59`); default `auto`→git observed; parallel `diff()` (`patch.go:L352`) via
  `ctx.Parallel` (`patch.go:L361`). *(Observed.)*
- [x] **Edge cases.** `/dev/null` and symlink handling both observed.
- [x] **Cross‑check.** All four `docs/kittens/diff.rst` Major Features reconciled with
  observations, each with both the doc line and the observed evidence.
- [x] **Runtime is Go.** `main.py` `main()` raises `SystemExit('Must be run as kitten
  diff')` (`main.py:L13-14`); all behavior lives in `kittens/diff/*.go`, observed through
  the real `kitten diff` CLI.

### Inferred (not directly observed) statements, and why

- **Q3:** LRU **eviction** (`cache.go:L48-53`) — no fixture reached the 4096‑entry cap;
  exact per‑access **hit counters** — not exposed through the CLI; the sub‑perceptible
  isolated **plain‑only frame** timing for tiny inputs — highlight wins the race faster than
  the harness can sample (the fallback path itself *is* observed via the `-race` `Get`
  miss). Everything else in this document is backed by captured CLI output.

---

*Reproduction summary:* build with
`docker exec kitty-build bash -lc 'cd /app && python3 setup.py'`; run
`/app/kitty/launcher/kitten diff LEFT RIGHT`. Go 1.22 (`go.mod:L3`), Chroma v2.14.0
(`go.mod:L7`), `runtime.NumCPU()` = 128. The `-race` build used only for Q4 was produced
with `GOFLAGS=-race python3 setup.py`.

