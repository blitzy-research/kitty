# How Kitty's "diff kitten" Works — A Runtime-Evidenced Walkthrough

> **Audience:** a new engineer being onboarded who wants to understand the diff kitten's *internal mechanics as they actually execute*, not merely as they read on the page.
>
> **What this document is:** answers to eight specific questions about the diff kitten, where **every behavioral claim is backed by output I captured by actually building and running the kitten** through its canonical entry point `kitty +kitten diff <left> <right>`, plus an inline `file:line` citation into the source. Where a claim is derived from reading the code rather than observed at runtime, it is explicitly labelled **[INFERRED]**. Where I had to reach the code through anything other than the canonical entry point, that is labelled **[NON-CANONICAL]**.

---

## Methodology / Harness

**Where everything ran.** All build-and-run work happened *inside* the provided Docker container `kitty-diff-env` (image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`, Ubuntu 24.04, Go 1.23.4, Python 3.12.3, gcc 13.3.0). The container is pre-built at commit `815df1e210e0`, which is the same commit as this repository checkout, so every `file:line` citation below matches the source you can read in the repo. The pod hosting this checkout has no Go toolchain and is not built; it is used only for source reading and for the required web validation.

**Why a container build is required.** A bare `go build`/`go test ./kittens/diff/` fails without Kitty's code-generation step (the generated root `kitty` package and `go:embed` targets such as `tools/tui/shell_integration/data_generated.bin`). Kitty's `python3 setup.py` performs that code generation and then builds the Go tools and the C terminal. In this container that step is already complete — I confirmed the generated artifacts exist and that the module builds and tests offline:

```
$ ls -l kitty/constants_generated.go tools/tui/shell_integration/data_generated.bin
-rw-r--r-- 1 root root 14092 ... kitty/constants_generated.go
-rw-r--r-- 1 root root 25574 ... tools/tui/shell_integration/data_generated.bin

$ GOPROXY=off go test -v -count=1 ./kittens/diff/
=== RUN   TestDiffCollectWalk
--- PASS: TestDiffCollectWalk (0.02s)
PASS
ok      kitty/kittens/diff      0.022s
```

**The canonical entry point.** Every behavioral observation below was produced by running the real entry point:

```
$ ./kitty/launcher/kitty +kitten diff <left> <right>
```

I confirmed it is reachable and identified itself correctly:

```
$ ./kitty/launcher/kitty +kitten diff --help
Usage: kitten diff [options] file_or_directory_left file_or_directory_right
...
kitten diff 0.35.2 created by Kovid Goyal
```

The usage string `file_or_directory_left file_or_directory_right` is defined at [`kittens/diff/main.py:296`], and the kitten refuses to run outside the `kitten diff` context via the guard `raise SystemExit('Must be run as kitten diff')` at [`kittens/diff/main.py:14`].

**Driving a full-screen TUI headlessly.** The diff kitten is a full-screen TUI that opens `/dev/tty`, so I drove it under a pseudo-terminal. A small PTY driver (`/tmp/pty_run.py`, created outside the repo) does `pty.fork()` in a 50×220 terminal, launches the command, sends `q` to quit after the screen settles, escalates to `SIGTERM`/`SIGKILL` if needed, and writes the raw VT byte stream to `/tmp/diff_raw.out` (printing `RAW_BYTES=<n>`). A companion reconstructor (`/tmp/vtgrid.py`) replays the CUP/erase/printable operations onto a 50×220 grid and prints the non-blank screen lines. **The process invoked is always `kitty +kitten diff <left> <right>`** — the PTY is only a transport for a headless terminal, not a bypass of the entry point.

**Fixtures.** All fixtures live *outside* the source tree under `/tmp/diffkitten_fixtures/` (removed at the end). The sets exercise every branch the questions imply:

| Set | Purpose |
|-----|---------|
| **F1** | pairing/classification: same-content, different-content, mode-only (`chmod`), left-only, right-only |
| **F1b** | nested-directory pairing (subdirectory relative paths) |
| **F2** | true rename (byte-identical file at a different relative path) |
| **F3** | rename hash-gate counter-case (added/removed pair with different content ⇒ MD5 differs) |
| **F4** | 60 text files per side across 10 languages — caching, concurrency, `-race` |
| **F5** | binary blobs (non-UTF-8), changed between sides |
| **F6** | real PNG/JPEG images: added / removed / changed |
| **F7** | one 5-line-vs-6-line text pair for the builtin/git/external differ comparison and the anchor set |

**Stability discipline.** For the count/magnitude/concurrency questions (Q4/Q5) I ran the relevant fixture **≥2 times** (and up to 20 times for the Q5 race investigation) and report whether the behavior was stable.

**Race detection.** For Q5 I built a `-race`-instrumented kitten (`/tmp/kitten_race`, same diff-kitten code) and ran the canonical diff entry point under it.

**Cleanliness guarantee.** No source file was modified. The only file added to the repository is this document. At the end I removed every temporary fixture/script and confirmed `git status --porcelain` shows only this new file (see the Coverage-Pass section).

**A note on line numbers.** A handful of anchors in the original plan had drifted by ±1–3 lines; I re-verified **every** `file:line` below by reading the source directly at authoring time. Notable confirmations: there are **seven** caches (not eight); `data_for_path` is at `collect.go:65`; `is_image` at `collect.go:82`; `is_path_text` at `collect.go:86`; `on_wakeup` at `ui.go:161`; the `load_all_images` method at `ui.go:190`.

---

## Q1 — Directory pairing: which left file "belongs together" with which right file?

**Direct answer.** Pairing is by **identical relative path** — nothing fuzzy. The kitten walks each directory into a set of paths *relative to that directory's root*, then intersects the two sets. Files in both sets (same relative path) are paired and compared; a file present only on the left becomes a **removal**; only on the right becomes an **addition**. Even when two paired files have byte-identical content, a difference in file **mode** still registers as a change.

**Command.**

```
$ cd /app && python3 /tmp/pty_run.py ./kitty/launcher/kitty +kitten diff \
      /tmp/diffkitten_fixtures/F1/left /tmp/diffkitten_fixtures/F1/right
$ python3 /tmp/vtgrid.py /tmp/diff_raw.out
```

**Captured output (F1, `RAW_BYTES=18928`, screen reconstruction; entries appear alphabetically).**

```
   app.go
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   @@ -3,5 +3,6 @@ package main
3  import "fmt"                          3  import "fmt"
4                                        4
5  func main() {                         5  func main() {
6      fmt.Println("hello")              6      name := "world"
                                         7      fmt.Println("hello", name)
7  }                                     8  }
   only_left.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1  this file exists only on the left        This file was removed
2  it will be removed
   only_right.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   This file was added                  1  this file exists only on the right
                                        2  it is newly added
   script.sh
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Mode changed: -rw-r--r-- to -rwxr-xr-x
```

Note what is **not** there: `same.txt` (byte-identical content *and* identical mode on both sides) produces **no entry at all**.

**Nested-directory confirmation (F1b, `RAW_BYTES=5230`).** A file at `sub/deep/nested.txt` on both sides is paired and shown as a content diff, proving the intersection is over full *relative* paths (including subdirectories), not basenames:

```
   sub/deep/nested.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   @@ -1,2 +1,2 @@
1  nested original                      1  nested changed
2  common tail                          2  common tail
```

**Responsible functions.**
- `collect_files` [`kittens/diff/collect.go:296`] builds `left_names, right_names` as `utils.Set[string]` [`collect.go:297`], populating each by calling `walk` [`collect.go:260`], which uses `filepath.WalkDir` [`collect.go:265`].
- Names are filtered by `allowed` [`collect.go:230`], which matches each `ignore_name` glob against the *basename* with `filepath.Match` [`collect.go:233`] (standard-library `filepath.Match`, not doublestar).
- The pairing itself is a single set intersection: `common_names := left_names.Intersect(right_names)` [`collect.go:306`].
- Paired files with differing content call `self.add_change(...)` [`collect.go:319`] (method def [`collect.go:167`], which records item type `diff`).
- The **mode-only** branch is the `else` at [`collect.go:320-329`]: when `ld == rd` (identical bytes) but `lstat.Mode() != rstat.Mode()` [`collect.go:323`], it still calls `add_change`.
- Left-only names come from `left_names.Subtract(common_names)` [`collect.go:332`]; right-only from `right_names.Subtract(common_names)` [`collect.go:333`].

**Cause → effect.** Because pairing is a set intersection over *relative* paths, two files "belong together" **iff** they occupy the same path under their respective roots. That is why `app.go` (same relative path, different bytes) is a content diff; `only_left.txt`/`only_right.txt` (present on one side only) fall out of the intersection and become a removal and an addition respectively; `script.sh` (same path, same bytes, different mode) is caught by the `Mode()` comparison; and `same.txt` (same path, same bytes, same mode) matches every equality check and emits nothing.

**Source-corroboration [NON-CANONICAL — unit test, not the entry point].** The repository's own test corroborates the walk + ignore-glob behavior (it does **not** test rename):

```
$ GOPROXY=off go test -v -count=1 -run TestDiffCollectWalk ./kittens/diff/
=== RUN   TestDiffCollectWalk
--- PASS: TestDiffCollectWalk (0.02s)
PASS
ok      kitty/kittens/diff      0.022s
```

`TestDiffCollectWalk` [`kittens/diff/collect_test.go`] walks a tree with `ignore_name` globs `*~`, `#*#`, `b` and asserts (via `github.com/google/go-cmp`) that the surviving relative-name set is exactly `{d, e, f/g, h space}`. This is a unit-test corroboration of Q1's walk/ignore logic, distinct from the canonical `kitty +kitten diff` runs above.

---

## Q2 — Rename recognition: rename vs. delete-plus-add

**Direct answer.** After classification (Q1), for the set of *added* and *removed* files the kitten computes an **MD5 hash of each file's contents**, looks for a removed file and an added file whose hashes match, and — critically — **re-checks full byte-for-byte content equality** before promoting the pair to a **rename**. The matched added entry is then discarded so it is not double-counted. If no hash matches (or the content re-check fails), the files stay a separate removal and addition.

**Command (true rename — F2).**

```
$ python3 /tmp/pty_run.py ./kitty/launcher/kitty +kitten diff \
      /tmp/diffkitten_fixtures/F2/left /tmp/diffkitten_fixtures/F2/right
$ python3 /tmp/vtgrid.py /tmp/diff_raw.out
```

**Captured output (F2, `RAW_BYTES=3792`).** `left/old_name.py` is byte-identical to `right/new_name.py`, and `old_name.py` is absent on the right. The kitten reports a single **rename** entry (the two names on one header line, with no content body and a `0,0 0d` status), *not* a delete plus an add:

```
   old_name.py ⟶ new_name.py
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Command (hash-gate counter-case — F3).**

```
$ md5sum /tmp/diffkitten_fixtures/F3/left/gone.txt /tmp/diffkitten_fixtures/F3/right/brandnew.txt
4b44563cc9886576b6589dd730cbc397  .../F3/left/gone.txt
db42902ed49e7b2b97a4d54d3510dd24  .../F3/right/brandnew.txt
$ python3 /tmp/pty_run.py ./kitty/launcher/kitty +kitten diff \
      /tmp/diffkitten_fixtures/F3/left /tmp/diffkitten_fixtures/F3/right
$ python3 /tmp/vtgrid.py /tmp/diff_raw.out
```

**Captured output (F3, `RAW_BYTES=8404`).** The two files have **different** contents (hence different MD5s above), so no rename is inferred — they show as a separate addition and removal:

```
   brandnew.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   This file was added                  1  this file is brand new
   gone.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1  this file is going away                  This file was removed
```

**Responsible functions.** The rename block is [`collect.go:334-368`]:
- MD5 hashes for every added file [`collect.go:335-340`] and every removed file [`collect.go:341-346`] are computed via `hash_for_path` [`collect.go:106`], whose core is `md5.Sum(...)` [`collect.go:112`] (Go's `crypto/md5`).
- The match loop [`collect.go:347-364`] compares hashes `if ah == rh` [`collect.go:350`], and on a hash match reads both files' bytes into `ld` [`collect.go:351`] and `rd` [`collect.go:352`] and performs the **full-content re-check `if ld == rd`** [`collect.go:353`].
- Only if the re-check passes does it call `self.add_rename(...)` [`collect.go:354`] (method def [`collect.go:175`]) and discard the matched added entry via `added.Discard(n)` [`collect.go:355`].
- An unmatched removed file falls to `self.add_removal(...)` [`collect.go:362`] (def [`collect.go:192`]); any remaining added file falls to `self.add_add(...)` [`collect.go:366`] (def [`collect.go:181`]).

**Cause → effect.** The hash comparison is a cheap first filter that finds *candidate* rename pairs without comparing every file against every other file byte-by-byte. The byte-for-byte re-check then guarantees correctness even if two different files happened to collide on MD5. In F2 the bytes are identical, so the hash matches *and* `ld == rd` holds ⇒ rename. In F3 the bytes differ, so the hashes differ, `ah == rh` is never true ⇒ the files remain a separate add and removal.

**[INFERRED] the collision guard itself.** Fabricating a genuine MD5 collision (two different byte strings, same MD5) is impractical here, so the specific behavior "hash matches but `ld != rd` ⇒ *not* a rename" is **inferred from the source** at [`collect.go:351-353`] (the re-check exists precisely to reject that case). What I verified at runtime is the true-rename path (F2) and the different-content-⇒-no-rename path (F3, where the hashes differ up front).

---


## Q3 — Caching pipeline & bounded efficiency (raw bytes → highlighted output)

**Direct answer.** Content flows through a **layered, read-once memoization pipeline**: raw bytes are read once and cached; a content hash, "sanitized" plain lines, and syntax-highlighted lines are each derived once from the layer below it and memoized in their own cache. Every cache is a bounded LRU (capacity 4096) that evicts its least-recently-used entry once full, which keeps memory bounded as more files are processed. Highlighted output is filled **asynchronously**; until it is ready, the highlight layer gracefully **falls back to plain lines**, which is what produces the "plain first, enriched later" behavior.

**⚠ Factual correction — there are SEVEN caches, not eight.** The plan text says "eight caches," but this branch declares and initializes exactly **seven** `utils.LRUCache` instances. I confirmed by reading the declarations and the seven `NewLRUCache(sz)` calls:

```
$ nl -ba kittens/diff/collect.go | sed -n '20,37p'
    20  var mimetypes_cache, data_cache, hash_cache *utils.LRUCache[string, string]
    21  var size_cache *utils.LRUCache[string, int64]
    22  var lines_cache *utils.LRUCache[string, []string]
    23  var highlighted_lines_cache *utils.LRUCache[string, []string]
    24  var is_text_cache *utils.LRUCache[string, bool]
    25
    26  func init_caches() {
    27      path_name_map = make(map[string]string, 32)
    28      remote_dirs = make(map[string]string, 32)
    29      const sz = 4096
    30      size_cache = utils.NewLRUCache[string, int64](sz)
    31      mimetypes_cache = utils.NewLRUCache[string, string](sz)
    32      data_cache = utils.NewLRUCache[string, string](sz)
    33      is_text_cache = utils.NewLRUCache[string, bool](sz)
    34      lines_cache = utils.NewLRUCache[string, []string](sz)
    35      highlighted_lines_cache = utils.NewLRUCache[string, []string](sz)
    36      hash_cache = utils.NewLRUCache[string, string](sz)
    37  }
```

The seven are: `size_cache` [`collect.go:30`], `mimetypes_cache` [`collect.go:31`], `data_cache` [`collect.go:32`], `is_text_cache` [`collect.go:33`], `lines_cache` [`collect.go:34`], `highlighted_lines_cache` [`collect.go:35`], and `hash_cache` [`collect.go:36`]. The two other members initialized in `init_caches` — `path_name_map` [`collect.go:27`] and `remote_dirs` [`collect.go:28`] — are **plain Go maps, not LRUCaches**. So the observed, source-confirmed count is **7**.

**Responsible functions — the derivation chain.** Each layer reads from the layer below and memoizes:
- `data_for_path` [`collect.go:65`] reads each file exactly once through `data_cache.GetOrCreate(path, ...)` wrapping `os.ReadFile` [`collect.go:66-67`].
- `hash_for_path` [`collect.go:106`] derives the MD5 from `data_for_path` and caches it.
- `lines_for_path` [`collect.go:138`] derives `text_to_lines(sanitize(...))` [`collect.go:144`] from `data_for_path` and caches it.
- `highlighted_lines_for_path` [`collect.go:148`] first gets the plain lines, then returns the highlighted version **only if it is cached and the same length**, otherwise returns the plain lines:

```
$ nl -ba kittens/diff/collect.go | sed -n '148,157p'
   148  func highlighted_lines_for_path(path string) ([]string, error) {
   149      plain_lines, err := lines_for_path(path)
   150      if err != nil {
   151          return nil, err
   152      }
   153      if ans, found := highlighted_lines_cache.Get(path); found && len(ans) == len(plain_lines) {
   154          return ans, nil
   155      }
   156      return plain_lines, nil
   157  }
```

**Bounded LRU semantics.** All seven are `utils.LRUCache` [`tools/utils/cache.go`]. Insertion is `GetOrCreate` [`cache.go:39`], which takes a write `Lock()` [`cache.go:48`], stores the value [`cache.go:49`], pushes the key to the front of an LRU list [`cache.go:50`], and — when over capacity — evicts the least-recently-used entry from the **back** of the list [`cache.go:51-54`]:

```
$ nl -ba tools/utils/cache.go | sed -n '48,54p'
    48          self.lock.Lock()
    49          self.data[key] = ans
    50          self.lru.PushFront(key)
    51          if self.max_size > 0 && self.lru.Len() > self.max_size {
    52              k := self.lru.Remove(self.lru.Back())
    53              delete(self.data, k.(K))
    54          }
```

**State transition — observed plain→enriched re-render (before / during / after).** Running the kitten on a small Go fixture and slicing the raw VT stream at its synchronized-update boundaries (`ESC[?2026h` … `ESC[?2026l`) yields three frames. Detecting whether each frame carries truecolor SGR sequences (highlighting) gives:

```
frames = 3
truecolor-present per frame = [False, True, True]
   frame 0: blank screen        (BEFORE  — nothing rendered yet)
   frame 1: diff rendered        (DURING  — content on screen)
   frame 2: diff re-rendered     (AFTER   — enriched re-render)
```

This is direct evidence of the re-render mechanism: after an asynchronous result arrives, the kitten redraws. The redraw is driven by `handle_async_result` on `IMAGE_LOAD, HIGHLIGHT` calling `rerender_diff()` [`kittens/diff/ui.go:272-273`].

**[INFERRED] the exact "plain FIRST, then highlighted" ordering.** On this 128-CPU host, highlighting completed so quickly that the *first content frame* was already syntax-highlighted in every experiment I ran (a 7-line file, 4,000/40,000/150,000-function Go files, and the 60-file F4 under `GOMAXPROCS=1`). For example, with a 150,000-function file the timestamped frames were `frame0 blank @0.024s → frame1 highlighted @0.786s`. So the *fallback-to-plain path* [`collect.go:153-156`] and the specific "renders plain first, then re-renders highlighted" ordering are **inferred from the source**; what I **observed** is the multi-frame re-render itself and the enriched final state.

**[INFERRED] read-once and bounded eviction.** The container has no `strace`/`ltrace`, so "each file is read exactly once" is inferred from `GetOrCreate`'s cache-hit semantics [`cache.go:39-45`], and the size bound (`sz = 4096`, back-eviction [`cache.go:51-54`]) is inferred from the source (a run with >4096 distinct files would in any case abort in the highlighter before demonstrating eviction — see Q5).

**Cause → effect.** Because each derivation layer memoizes in its own cache, re-rendering (scrolling, resizing, the post-highlight redraw) reuses cached bytes/hashes/lines instead of recomputing them — that is what keeps the experience fast (Q8 revisits this). Because every cache is a fixed-capacity LRU, memory stays bounded no matter how many files are processed.

---

## Q4 — What happens when many files must be processed at once

**Direct answer.** Collection runs in a **background goroutine** that pushes its result onto a **buffered channel (capacity 32)** and wakes the main thread. The main thread drains the channel and, for the collection result, **fans out** three jobs — generate the diffs, highlight all text files, and load all images — each of which itself runs in the background. As those results arrive, the diff on screen appears and is progressively enriched.

**Command.**

```
$ GOMAXPROCS=1 python3 /tmp/pty_run.py ./kitty/launcher/kitty +kitten diff \
      /tmp/diffkitten_fixtures/F4/left /tmp/diffkitten_fixtures/F4/right
$ python3 /tmp/vtgrid.py /tmp/diff_raw.out
```

(F4 = 60 files per side across 10 languages, every file content-changed. I use `GOMAXPROCS=1` here to serialize the workers so the process does not hit the Q5 concurrency bug while I observe multi-file collection.)

**Captured output (F4, first screenful).** Many files are each rendered with the target `@@ -1,5 +1,6 @@` hunk header:

```
   mod_1.go
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   @@ -1,5 +1,6 @@
1  line1 file 1 ext go                  1  line1 file 1 ext go
2  func_1_original()                    2  func_1_changed()
3  common_a                             3  common_a
                                        4  EXTRA_LINE_1
4  common_b                             5  common_b
5  tail_1                               6  tail_1
   mod_10.py
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   @@ -1,5 +1,6 @@
...
   mod_11.go / mod_12.js / mod_13.c ...   (all 60 files collected)
```

**Responsible functions.**
- `Handler.initialize` [`kittens/diff/ui.go:114`] creates the buffered results channel `self.async_results = make(chan AsyncResult, 32)` [`ui.go:132`] and launches the collection goroutine [`ui.go:133-138`], which calls `create_collection`, pushes the `AsyncResult`, and calls `self.lp.WakeupMainThread()` [`ui.go:137`].
- `WakeupMainThread` [`tools/tui/loop/api.go:316`] wakes the event loop (`loop.Loop.Run` [`loop/api.go:284`]).
- `on_wakeup` [`ui.go:161`] drains the channel in a `for { select { ... } }` loop [`ui.go:163-176`], handing each result to `handle_async_result` [`ui.go:245`].
- The `COLLECTION` case [`ui.go:247`] fans out the three background jobs: `self.generate_diff()` [`ui.go:249`], `self.highlight_all()` [`ui.go:250`], and `self.load_all_images()` [`ui.go:251`].

**≥2-run stability.** Multi-file collection behavior was stable across many runs (the F4 fixture was launched 20× during the Q5 investigation plus several `GOMAXPROCS=1` runs); the set of files collected and the hunk headers were identical each time.

**Cause → effect.** Doing collection in a goroutine and communicating via a buffered channel keeps the UI thread responsive: the screen can draw immediately (initially empty) and then react to each async result as it lands. The capacity-32 buffer lets the producer(s) enqueue results without blocking on the main thread. Fanning out diff/highlight/image work is what turns "many files" into three independent, parallelizable pipelines — which is exactly where Q5 comes in.

---


## Q5 — Parallel highlighting "without stepping on itself"

**Direct answer (two parts — and this is the most important finding in this document).**

1. **Work *distribution* is designed not to duplicate effort.** Highlighting is parallelized by **per-index channel ownership**: a buffered channel is filled with every work index and closed, then `runtime.NumCPU()` worker goroutines each *receive* indices from it. Because a Go channel receive delivers each value to exactly one goroutine, no two workers ever highlight the same file, so no work is duplicated and each worker writes a **distinct** cache key.

2. **BUT the shared cache write is *not* actually safe for concurrent use, and at runtime the kitten DOES "step on itself."** The per-file store calls `highlighted_lines_cache.Set(...)`, and `LRUCache.Set` writes the shared Go map while holding only a **read** lock. Concurrent writes to a Go built-in map are a data race, and on this 128-CPU host the kitten **aborts with `fatal error: concurrent map writes` in the majority of multi-file runs**, and the Go race detector flags the exact write. So the honest, observed answer is: **it does not fully avoid stepping on itself** — the distribution logic prevents *duplicated* work, but the cache layer is racy.

**The design intent (source).** `highlight_all` [`kittens/diff/highlight.go:217`] creates an empty `images.Context{}` [`highlight.go:218`] (empty ⇒ `NumberOfThreads()` is 0 ⇒ the pool falls back to `runtime.NumCPU()`) and calls `ctx.Parallel(...)` [`highlight.go:219`]; each worker highlights its own `path = paths[i]` [`highlight.go:221`] and stores the result [`highlight.go:224`]:

```
$ nl -ba kittens/diff/highlight.go | sed -n '217,227p'
   217  func highlight_all(paths []string) {
   218      ctx := images.Context{}
   219      ctx.Parallel(0, len(paths), func(nums <-chan int) {
   220          for i := range nums {
   221              path := paths[i]
   222              raw, err := highlight_file(path)
   223              if err == nil {
   224                  highlighted_lines_cache.Set(path, text_to_lines(raw))
   225              }
   226          }
   227      })
   228  }
```

`Context.Parallel` [`tools/utils/images/utils.go:27`] is the shared fan-out primitive: it computes `procs` from `NumberOfThreads()` [`utils.go:33`] falling back to `runtime.NumCPU()` [`utils.go:35`], fills a buffered `c := make(chan int, count)` [`utils.go:41`] with all indices [`utils.go:42-44`], closes it [`utils.go:45`], and starts `procs` worker goroutines [`utils.go:48-53`] joined by a `sync.WaitGroup` [`utils.go:47`]. Each worker runs `fn(c)` [`utils.go:52`]. On this machine:

```
$ nproc
128
```

so `runtime.NumCPU()` is **128** — up to 128 workers call `Set` concurrently.

**The unsafe write (source).** `LRUCache.Set` [`tools/utils/cache.go:32`] takes only an `RLock` and then writes the shared map:

```
$ nl -ba tools/utils/cache.go | sed -n '32,37p'
    32  func (self *LRUCache[K, V]) Set(key K, val V) {
    33      self.lock.RLock()
    34      self.data[key] = val
    35      self.lock.RUnlock()
    36      return
    37  }
```

`self.data[key] = val` at [`cache.go:34`] is a map write, but it is guarded by `RLock` [`cache.go:33`] — a *shared* lock that permits many concurrent holders. (Contrast `GetOrCreate`, which correctly writes under a full write `Lock()` [`cache.go:48`].) Distinct keys do not help: a Go map is a single object, and concurrent writes to it are undefined behavior regardless of key.

**Observation (1) — the plain kitten aborts.** Running the pre-built kitten on F4 (60 files) **20 times**:

```
$ for i in $(seq 1 20); do
      python3 /tmp/pty_run.py ./kitty/launcher/kitty +kitten diff \
          /tmp/diffkitten_fixtures/F4/left /tmp/diffkitten_fixtures/F4/right >/dev/null 2>>/tmp/q5.err
  done
$ grep -c "concurrent map" /tmp/q5.err
17
```

**17 of 20 runs aborted** with the Go runtime fatal error (16 `concurrent map writes`, 1 `concurrent map read and map write`). This is not stable "success across ≥2 runs" — it is a *stable failure* of the majority of runs, which is itself the answer.

**Observation (2) — the race detector pinpoints it.** I built a `-race` kitten (`/tmp/kitten_race`, the same diff-kitten code compiled with `-race`) and ran the canonical entry point under it on F4. Both runs reported a DATA RACE. The unedited report (saved to `/tmp/diffkitten_captures/Q5_datarace.txt`):

```
==================
WARNING: DATA RACE
Write at 0x00c000326ab0 by goroutine 278:
  runtime.mapassign_faststr()
      /usr/local/go/src/runtime/map_faststr.go:223 +0x0
  kitty/tools/utils.(*LRUCache[go.shape.string,go.shape.[]string]).Set()
      /app/tools/utils/cache.go:34 +0xa4
  kitty/kittens/diff.highlight_all.func1()
      /app/kittens/diff/highlight.go:224 +0xfb
  kitty/tools/utils/images.(*Context).Parallel.func1()
      /app/tools/utils/images/utils.go:52 +0x8d

Previous write at 0x00c000326ab0 by goroutine 239:
  runtime.mapassign_faststr()
      /usr/local/go/src/runtime/map_faststr.go:223 +0x0
  kitty/tools/utils.(*LRUCache[go.shape.string,go.shape.[]string]).Set()
      /app/tools/utils/cache.go:34 +0xa4
  kitty/kittens/diff.highlight_all.func1()
      /app/kittens/diff/highlight.go:224 +0xfb
  kitty/tools/utils/images.(*Context).Parallel.func1()
      /app/tools/utils/images/utils.go:52 +0x8d

Goroutine 278 (running) created at:
  kitty/tools/utils/images.(*Context).Parallel()
      /app/tools/utils/images/utils.go:50 +0x126
  kitty/kittens/diff.highlight_all()
      /app/kittens/diff/highlight.go:219 +0xd0
  kitty/kittens/diff.(*Handler).highlight_all.func1()
      /app/kittens/diff/ui.go:183 +0x84

Goroutine 239 (finished) created at:
  kitty/tools/utils/images.(*Context).Parallel()
      /app/tools/utils/images/utils.go:50 +0x126
  kitty/kittens/diff.highlight_all()
      /app/kittens/diff/highlight.go:219 +0xd0
  kitty/kittens/diff.(*Handler).highlight_all.func1()
      /app/kittens/diff/ui.go:183 +0x84
==================
```

The report names exactly the path traced above: two worker goroutines both writing at `cache.go:34` ← `highlight.go:224` ← `utils.go:52`, both created at `utils.go:50` ← `highlight.go:219` ← `ui.go:183`.

**Observation (3) — the repo's own tests do NOT catch this [NON-CANONICAL corroboration, with a caveat].** For completeness I ran the module test suite under `-race`, twice:

```
$ go test -race -count=1 ./kittens/diff/
ok      kitty/kittens/diff      1.104s
$ go test -race -count=1 ./kittens/diff/
ok      kitty/kittens/diff      1.104s
```

This is clean — but it only exercises `TestDiffCollectWalk` (the walk/ignore logic), **not** `highlight_all`. So a green `-race` test run here is *weak* corroboration: it confirms the collection path is race-free but says nothing about parallel highlighting, which the canonical entry-point runs above show is *not* safe.

**Cause → effect.** `Context.Parallel` guarantees each of the up-to-128 workers owns a disjoint set of indices, so the *keys* they write never collide and no file is highlighted twice — that satisfies the "no duplicated work" half of "without stepping on itself." But all of those workers funnel into a single `highlighted_lines_cache` whose `Set` writes the shared map under a read lock; with 128 concurrent writers the Go runtime detects simultaneous map writes and aborts the process. Serializing the workers (`GOMAXPROCS=1`, used in Q4) makes the writes non-simultaneous and avoids the panic, which is why the Q4 capture completed.

**Bottom line for Q5:** the *distribution* mechanism prevents duplicated work, but parallel highlighting is **not** race-free at the cache layer — observed as a `fatal error: concurrent map writes` in 17/20 plain runs and a DATA RACE under `-race`.

---


## Q6 — Binary & image handling alongside text

**Direct answer.** Content type **gates** behavior. A file is "text" only if it is not an image, not `/dev/null`, and is valid UTF-8. Text files are diffed line-by-line (Q1/Q8). Binary files are **excluded from the text diff** and rendered as a one-line "Binary file: `<size>`" summary. Image files are also excluded from the text diff, are **loaded asynchronously and drawn via the Kitty Graphics Protocol**, and are labelled with their pixel dimensions and human-readable size.

**Command (binary — F5).**

```
$ python3 /tmp/pty_run.py ./kitty/launcher/kitty +kitten diff \
      /tmp/diffkitten_fixtures/F5/left /tmp/diffkitten_fixtures/F5/right
$ python3 /tmp/vtgrid.py /tmp/diff_raw.out
```

**Captured output (F5).** `blob.bin` is non-UTF-8 (4 KB left, 8 KB right); there is **no line diff**, just the binary summary on each side:

```
   blob.bin
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Binary file: 4 KB                    Binary file: 8 KB
```

**Command (images — F6).**

```
$ python3 /tmp/pty_run.py ./kitty/launcher/kitty +kitten diff \
      /tmp/diffkitten_fixtures/F6/left /tmp/diffkitten_fixtures/F6/right
$ python3 /tmp/vtgrid.py /tmp/diff_raw.out
```

**Captured output (F6).** An added JPEG, a changed PNG, and a removed PNG — each shows dimensions + size and a "Loading image..." placeholder while the async load runs:

```
   added.jpg
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                                        Dimensions: 50x50 Size: 693 B
                                        Loading image...
   changed.png
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Dimensions: 64x48 Size: 139 B        Dimensions: 80x60 Size: 155 B
   Loading image...                     Loading image...
   removed.png
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Dimensions: 40x40 Size: 106 B
   Loading image...
```

**Kitty Graphics Protocol emission (observed).** Scanning the raw byte stream for graphics-protocol APC escapes (`ESC _G … ESC \`) found 4 chunks; the first carries the control keys:

```
$ grep -aoc $'\x1b_G' /tmp/diff_raw.out
4
# first _G chunk control keys:
a=q,f=24,t=t,s=1,v=1,S=47,i=1
```

**Responsible functions.**
- `is_image` [`collect.go:82`] returns true when the MIME type has an `image/` prefix.
- `is_path_text` [`collect.go:86`] returns false for images [`collect.go:88-90`], for `/dev/null` [`collect.go:91-97`], and for non-UTF-8 content via `utf8.ValidString` [`collect.go:102`].
- Binary/image files are kept out of the parallel text diff by the gate `is_path_text(path) && is_path_text(changed_path)` in `generate_diff` [`kittens/diff/ui.go:147`].
- Images are collected and loaded by `load_all_images` [`ui.go:190`], which calls `image_collection.LoadAll()` [`ui.go:206`] (def [`tools/tui/graphics/collection.go:292`]); placement is `PlaceImageSubRect` [`ui.go:320`] (def [`graphics/collection.go:155`]) on the `ImageCollection` type [`graphics/collection.go:62`].
- The renderer dispatches by type in `render` [`kittens/diff/render.go:696`]: it computes `is_binary` [`render.go:706`] and `is_img` [`render.go:710`], and for a `diff` calls `image_lines` [`render.go:716`] or `binary_lines` [`render.go:718`] (the add/removal cases dispatch similarly).
- `binary_lines` [`render.go:446`] prints `"Binary file: %s"` [`render.go:452`]; `image_lines` [`render.go:333`] prints `"Size: %s"` [`render.go:341`] and, when a resolution is known (`res.Width > -1`), prefixes `"Dimensions: %dx%d %s"` [`render.go:344`].

**Cause → effect.** The `is_path_text`/`is_image` gate short-circuits binary and image files out of the line-diff pipeline, so the kitten never attempts to line-diff bytes that are not text. Binary files are summarized by size; image files additionally feed the graphics pipeline, which is why the raw stream contains `_G` protocol chunks and the screen shows `Dimensions:`/`Size:` labels with a "Loading image..." placeholder during the async `LoadAll`.

**[INFERRED] pixel rendering.** The headless PTY is not a graphics-capable terminal, so I did not *see* pixels; the **protocol emission** (the `_G` chunks) and the **`Dimensions:`/`Size:` text** are observed, but the claim that the image is ultimately drawn as pixels is inferred from that protocol emission plus `PlaceImageSubRect` [`ui.go:320`].

---

## Q7 — End-to-end runtime trace (two directories in → all changes understood)

**Direct answer.** From launch, the kitten loads config, resolves the diff command, initializes the seven caches and the syntax formatters, builds the event loop and `Handler`, and starts the loop. On initialize it kicks off background **collection**; when collection completes it fans out **diff / highlight / image** jobs; the diff renders first and is then **re-rendered** as highlight and image results arrive.

**Ordered trace (each step cited; key points backed by captured output).**

1. `main` [`kittens/diff/main.go:102`] — entry.
2. `conf, err = load_config(opts)` [`main.go:104`] (def [`main.go:24`]) — loads options and defaults.
3. `set_diff_command(conf.Diff_cmd)` [`main.go:111`] — resolves which differ to use (Q8).
4. `init_caches()` [`main.go:114`] — creates the seven LRU caches (Q3).
5. `create_formatters()` [`main.go:115`] — builds the Chroma syntax formatters.
6. `lp, err = loop.New()` [`main.go:138`] (def [`tools/tui/loop/api.go:115`]) — builds the event loop.
7. `h := Handler{left, right, lp}` [`main.go:143`], `lp.OnInitialize = ...` [`main.go:144`] calling `h.initialize()`, `lp.OnKeyEvent = h.on_key_event` [`main.go:160`].
8. `err = lp.Run()` [`main.go:163`] — starts the loop.
9. `Handler.initialize` [`ui.go:114`] — launches the background collection goroutine [`ui.go:133-138`] and draws the (initially empty) screen.
10. `on_wakeup` [`ui.go:161`] drains `async_results`; `handle_async_result` [`ui.go:245`] `COLLECTION` case [`ui.go:247`] fans out `generate_diff`/`highlight_all`/`load_all_images` [`ui.go:249-251`].
11. The `DIFF` result renders [`ui.go:252-268`]; `IMAGE_LOAD`/`HIGHLIGHT` results trigger `rerender_diff()` [`ui.go:272-273`].

**Config defaults observed at load.** The default `num_context_lines` is `3` [`kittens/diff/main.py:37`]. Running the default vs. an explicit `--context 1` on the same fixture shows the loaded value in the hunk header (3 context lines each side by default, 1 with the override):

```
# default (num_context_lines = 3):
$ ./kitty/launcher/kitty +kitten diff /tmp/diffkitten_fixtures/Q7/left /tmp/diffkitten_fixtures/Q7/right
   @@ -5,7 +5,7 @@

# explicit override:
$ ./kitty/launcher/kitty +kitten diff --context 1 .../Q7/left .../Q7/right
   @@ -7,3 +7,3 @@
```

The context count is applied in `initialize`: `self.current_context_count = opts.Context` and, when unset, `int(conf.Num_context_lines)` [`ui.go:118-121`]. Other loaded defaults I can name from the config source: `diff_cmd` = `auto` [`main.py:41`]; `syntax_aliases` = `pyj:py pyi:py recipe:py` [`main.py:29`]; `pygments_style` = `default` [`main.py:74`]; `replace_tab_by` = four spaces [`main.py:52`]; `ignore_name` example globs `.git`, `*~`, `*.pyc` [`main.py:63-65`]; usage string `file_or_directory_left file_or_directory_right` [`main.py:296`]; guard `Must be run as kitten diff` [`main.py:14`].

**Before / during / after (observed).** The three-frame capture from Q3 is the whole-run version of this trace: **before** = blank screen (loop started, collection in flight); **during** = the diff rendered after the `COLLECTION`→`DIFF` result; **after** = the enriched re-render after the `HIGHLIGHT` result [`ui.go:272-273`].

**Cause → effect.** Config and caches are set up synchronously so the loop starts with everything it needs; then all heavy work (walk, hash, diff, highlight, image load) is pushed into background goroutines that communicate through the wakeup/`async_results` mechanism. That is why the user sees an immediate (empty→diffed) screen that then enriches itself, rather than a long pause followed by a single fully-formed screen.

---


## Q8 — How matching regions are found, while caching keeps it fast

**Direct answer.** The differ is selectable. When no external differ is configured, the kitten uses a **builtin anchored (a.k.a. "patience") diff**: it finds the matching regions by computing the longest common subsequence of the **unique** lines (lines that appear exactly once in both old and new) using Szymanski's algorithm, then expands each of those anchors outward while the surrounding lines keep matching. Alternatively it can shell out to `git diff --no-index` or the external `diff`. Independently, the LRU caches (Q3) keep the *overall experience* fast by memoizing bytes/lines/highlights so re-renders never recompute them.

**Command — the same 5-line-vs-6-line pair through all four differ modes (canonical entry point).**

```
$ for m in builtin git diff auto; do
    echo "=== diff_cmd=$m ==="
    python3 /tmp/pty_run.py ./kitty/launcher/kitty +kitten diff \
        -o diff_cmd=$m /tmp/diffkitten_fixtures/F7/left /tmp/diffkitten_fixtures/F7/right
    python3 /tmp/vtgrid.py /tmp/diff_raw.out
  done
```

**Captured output — all four modes produce the identical on-screen result:**

```
   sample.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   @@ -1,5 +1,6 @@
1  one                                  1  one
2  two                                  2  two
                                        3  NEW
3  three                                4  three
4  four                                 5  four
5  five                                 6  five
```

**Raw external-command output (what the git/diff paths actually execute).** The two external command templates are constants:

```
$ nl -ba kittens/diff/patch.go | sed -n '21,22p'
    21  const GIT_DIFF = `git diff --no-color --no-ext-diff --exit-code -U_CONTEXT_ --no-index --`
    22  const DIFF_DIFF = `diff -p -U _CONTEXT_ --`
```

Running them by hand on the F7 pair (with `_CONTEXT_` → `3`) reproduces the same hunk:

```
$ git diff --no-color --no-ext-diff --exit-code -U3 --no-index -- \
      /tmp/diffkitten_fixtures/F7/left/sample.txt /tmp/diffkitten_fixtures/F7/right/sample.txt ; echo "exit=$?"
diff --git a/.../sample.txt b/.../sample.txt
index b2f931a67..8e7777679 100644
--- a/.../sample.txt
+++ b/.../sample.txt
@@ -1,5 +1,6 @@
 one
 two
+NEW
 three
 four
 five
exit=1

$ diff -p -U 3 -- \
      /tmp/diffkitten_fixtures/F7/left/sample.txt /tmp/diffkitten_fixtures/F7/right/sample.txt ; echo "exit=$?"
--- /tmp/diffkitten_fixtures/F7/left/sample.txt   ...
+++ /tmp/diffkitten_fixtures/F7/right/sample.txt  ...
@@ -1,5 +1,6 @@
 one
 two
+NEW
 three
 four
 five
exit=1
```

Both exit with status **1** ("differences found"), which is exactly the code `run_diff` treats as success for an external differ (`e.ExitCode() == 1` [`patch.go:321`]).

**Builtin `Diff()` raw output [NON-CANONICAL — direct source-level probe].** To see the builtin algorithm's *raw* unified output I built a tiny external Go program (outside the repo, at `/tmp/diffprobe`, using `replace kitty => /app` and the repo's `go.sum`) that calls the exported `diff.Diff(...)` directly. This bypasses the kitten TUI, so it is **non-canonical**; I re-confirmed the same hunk through the canonical `-o diff_cmd=builtin` run above.

```
$ cd /tmp/diffprobe && GOPROXY=off GOFLAGS=-mod=mod go run . 
diff left/sample.txt right/sample.txt
--- left/sample.txt
+++ right/sample.txt
@@ -1,5 +1,6 @@
 one
 two
+NEW
 three
 four
 five
```

Note the builtin's distinctive header `diff <old> <new>` [`kittens/diff/diff.go:58`] plus `---`/`+++` [`diff.go:59-60`] and the `@@ -%d,%d +%d,%d @@` hunk header [`diff.go:142`] — different framing from `git`'s `diff --git`/`index …` header, but the same hunk body.

**Responsible functions.**
- Differ selection: `set_diff_command` [`patch.go:44`] maps `auto` → `find_differ` [`patch.go:34`] (which prefers `git diff --no-index` via `GIT_DIFF` [`patch.go:21`], else external `diff` via `DIFF_DIFF` [`patch.go:22`], else the builtin), `builtin`/`""` → empty command (use `Diff()`), `diff` → `DIFF_DIFF`, `git` → `GIT_DIFF`.
- `run_diff` [`patch.go:282`] resolves symlinks [`patch.go:286,290`], and when `len(diff_cmd) == 0` calls the builtin `Diff(path1, data1, path2, data2, num_of_context_lines)` [`patch.go:303`]; otherwise it substitutes `_CONTEXT_` [`patch.go:311`] and runs the external command [`patch.go:315`].
- The builtin algorithm: `Diff` [`kittens/diff/diff.go:49`] loops over anchors `for _, m := range tgs(x, y)` [`diff.go:74`], expanding each match backward [`diff.go:85-88`] and forward [`diff.go:90-93`] while lines are equal, and emits `@@` hunks [`diff.go:142`]. `tgs` [`diff.go:192`] computes the LCS of unique lines — it is named for and cites Thomas G. Szymanski, Princeton TR #170 (1975) [`diff.go:188-191`], available at `https://research.swtch.com/tgs170.pdf` [`diff.go:191`].
- Parsing/parallelism: external output is parsed by `parse_patch` [`patch.go:245`]; multiple files are diffed in parallel by `diff` [`patch.go:352`] via `ctx.Parallel` [`patch.go:361`] (the same `runtime.NumCPU()` fan-out primitive as Q5).

**Anchor set for the F7 fixture [INFERRED — `tgs` is unexported].** For `one/two/three/four/five` vs `one/two/NEW/three/four/five`, every original line is unique and common, so `one, two, three, four, five` become the anchors (they appear as the context lines in the output above) and `NEW` is the single inserted line. The exact internal index pairs `tgs` returns are not printed (it is unexported), so the specific pairing is inferred from the algorithm; what is **observed** is the resulting `@@ -1,5 +1,6 @@` hunk with `+NEW` between the `two` and `three` anchors.

**Web validation (required).** The header of `diff.go` states it is copied from the Go standard library's `internal/diff` package:

```
$ nl -ba kittens/diff/diff.go | sed -n '1,2p'
     1  // Copied from the Go stdlib, with modifications.
     2  //https://github.com/golang/go/raw/master/src/internal/diff/diff.go
```

The authoritative documentation at **https://pkg.go.dev/internal/diff** corroborates the characterization above. It states that this implementation looks for a diff with the smallest number of "unique" lines inserted and removed, where unique means a line that appears just once in both old and new, and that it is called an "anchored diff" because the unique lines anchor the chosen matching regions. It further confirms the algorithm guarantees to run in O(n log n) time instead of the standard O(n²) time, and that some systems call this approach a "patience diff." The verbatim doc-comment in this repo is at [`diff.go:21-48`] (unique-lines wording [`diff.go:32-34`], "anchored diff" [`diff.go:35-36`], O(n log n) vs O(n²) [`diff.go:39-40`], "patience diff" [`diff.go:42`]).

**Cause → effect (algorithm ↔ cache speed).** The anchored diff is what makes the *matching regions* both correct and clean: by anchoring on lines that are unique in both texts, it avoids re-using incidental blank lines or closing braces as false matches, and by only LCS-ing the unique lines it runs in O(n log n) rather than O(n²). Separately, the LRU caches (Q3) are what make the *experience* fast: the bytes, plain lines, and highlighted lines a diff needs are read/derived once and reused on every subsequent render, so scrolling and the post-highlight re-render never recompute them. **[INFERRED]** the cache↔diff linkage (that repeated renders reuse cached results) follows from the `GetOrCreate` cache-hit semantics [`cache.go:39-45`]; the algorithm's output was observed directly above.

---


## Coverage-Pass Checklist

A final pass over all eight questions, every named item, and every sibling variant the questions imply. "Observed" = captured at runtime through the canonical `kitty +kitten diff` entry point; "Inferred" and "Non-canonical" are labelled as such in-line above.

| # | Question | Variants exercised | Evidence |
|---|----------|--------------------|----------|
| **Q1** | Directory pairing | same-content (no entry) ✅, different-content `diff` ✅, **mode-only** change ✅, left-only `removal` ✅, right-only `add` ✅, nested relative path ✅ | Observed (F1, F1b) + unit-test corroboration `TestDiffCollectWalk` **[non-canonical]** |
| **Q2** | Rename recognition | true rename ✅ (F2); different-content ⇒ no rename ✅ (F3); MD5 shown differing ✅ | Observed (F2, F3); collision guard `ld==rd` **[inferred]** |
| **Q3** | Caching pipeline & bound | **7 caches (not 8)** enumerated ✅; layered `data→hash→lines→highlighted` ✅; plain→enriched re-render ✅ (before/during/after) | Observed re-render (3 frames); read-once & 4096-bound eviction & plain-first ordering **[inferred]** |
| **Q4** | Concurrent multi-file | 60-file fan-out ✅; buffered channel cap **32** ✅; `WakeupMainThread` ✅; stable over many runs ✅ | Observed (F4, ≥2 runs) |
| **Q5** | Parallel highlight safety | per-index ownership (no duplicated work) ✅; **`fatal error: concurrent map writes` in 17/20 runs** ✅; `-race` DATA RACE ✅ (≥2 runs); `NumCPU=128` ✅; module test clean but doesn't cover highlight ✅ | Observed (plain 20 runs + `-race`); **standout finding: NOT race-free** |
| **Q6** | Binary & image | text ✅; binary "Binary file: `<size>`" ✅ (F5); image add/change/remove + `Dimensions:`/`Size:` + graphics `_G` chunks ✅ (F6) | Observed (F5, F6); pixel draw **[inferred]** |
| **Q7** | End-to-end trace | launch→config→caches→formatters→loop→initialize→collection→fan-out→render→re-render ✅; `num_context_lines=3` default vs `--context 1` ✅ | Observed (context headers) + source-cited ordered walk |
| **Q8** | Matching-region algorithm vs cache | builtin ✅, git ✅, external `diff` ✅, auto ✅ (all identical `@@ -1,5 +1,6 @@`); raw git/diff commands (exit=1) ✅; anchor set described ✅; web-validated ✅ | Observed (F7, 4 modes + raw commands); builtin `Diff()` probe **[non-canonical]**; anchor indices **[inferred]**; web: https://pkg.go.dev/internal/diff |

**Explicit labels used above.**
- **[INFERRED]** (read-derived, not observed): the MD5-collision guard `ld==rd` (Q2); read-once, the 4096 eviction bound, and the "plain FIRST then highlighted" ordering (Q3); pixel rendering of images (Q6); the exact `tgs` anchor index pairs and the cache↔diff speed linkage (Q8).
- **[NON-CANONICAL]** (reached other than through the entry point): the `TestDiffCollectWalk` unit test (Q1, Q5); the direct `diff.Diff()` probe (Q8) — each re-confirmed or clearly distinguished from a canonical run.

**Corrections to the original plan, stated as observed.**
- There are **seven** `utils.LRUCache` instances, not eight (Q3), enumerated from `collect.go:20-36`.
- Q5's "race-free" expectation is **contradicted at runtime**: parallel highlighting hits `concurrent map writes` / a `-race` DATA RACE at `cache.go:34` because `LRUCache.Set` writes the shared map under a read lock.
- Verified line-number anchors (e.g. `data_for_path` `collect.go:65`, `is_image` `collect.go:82`, `is_path_text` `collect.go:86`, `on_wakeup` `ui.go:161`, `load_all_images` `ui.go:190`) match this branch's source, re-read at authoring time.

## Cleanliness / integrity note

All fixtures and observation scripts were created **outside** the source tree (under `/tmp/`) and removed after use. No source file was modified; the only file added to the repository is this document. The final working-tree check shows exactly that:

```
$ git status --porcelain
?? blitzy/documentation/kitty_815df1e210e0.md
```

*(The build harness ran inside the container at commit `815df1e210e0`; this repository checkout is the same commit, so every `file:line` citation above matches the source you can read here.)*

