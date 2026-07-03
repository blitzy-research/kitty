# How the Kitty `diff` kitten works — a runtime‑investigated Q&A

## Orientation

The **`diff` kitten** is Kitty's side‑by‑side file/directory comparison tool. When you run `kitten diff <left> <right>` it walks the two inputs, pairs their files, computes per‑file diffs, syntax‑highlights the text, renders binary/image files specially, and paints a scrollable side‑by‑side view in your terminal. This document explains — **from observed runtime behavior**, not from reading source alone — how it decides "what belongs together," how it recognizes renames, how its caches stay efficient, how it processes many files in parallel (and how highlighting stays isolated), what changes for binary/image files, the full end‑to‑end runtime trace, and how the matching‑region algorithm finds anchors while the cache keeps everything fast.

> **Critical framing — the runtime path is Go, not Python.** The `diff` kitten is implemented in **Go** and compiled into the `kitten` binary (its `main` package lives under `tools/cmd/`). The file `kittens/diff/main.py` is a **legacy configuration‑definition stub — it is NOT the runtime path**. The live entry point is the Go function `main()` in `kittens/diff/main.go` [kittens/diff/main.go:102]. Throughout this document, `main.py` is cited only as the *source of the config‑option defaults* (which the build code‑generates into Go), never as executed code.

Every behavioral claim below is paired with the exact command that produced it and the **verbatim** output line demonstrating *that* claim, followed by the cause→effect explanation with exact `file:line` citations. Statements that were read from source but not directly executed are explicitly labeled **[Inferred]**.

---

## How this was investigated

**Environment.** All build and runtime observation was performed inside the user‑provided container toolchain (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`, image `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`), on branch `blitzy-5900e980-186b-42ac-b10f-47549fbb3c92`.

**Build/investigation revision (the identity behind every observation below).** The working tree was at commit `e3ae9a081` — full `e3ae9a081ed071a46d494bfe60ae264ce4477337`. This is the value `git rev-parse HEAD` returns and the exact value the canonical build stamps into the binary as `kitty.VCSRevision` (shown verbatim in the build section below). Note the deliberate distinction from the deliverable's *filename*: the filename derives from the **source branch** `kitty_815df1e210e0`, whose base commit is `815df1e21` (the commit immediately before this Q&A was added). The documentation commits that carry this file sit on top of that base, so the actual revision that built the binary and produced every observation here is `e3ae9a081`, **not** the source‑branch base `815df1e21`.

**Canonical build.** The kitten was built exactly as a normal user would, with the two‑stage canonical build:

```text
$ CI=true GOTOOLCHAIN=local ASAN_OPTIONS=detect_leaks=0 python3 setup.py build --verbose
# -> produces kitty/launcher/kitten, kitty/launcher/kitty, and *_generated.bin blobs
```

The build chain has two distinct pieces that are easy to conflate. The top‑level `build` **action** [setup.py:2115‑2121] first runs `build(args)` [setup.py:2116] and then, on non‑macOS, `build_launcher(...)` [setup.py:2120] and `build_static_kittens(...)` [setup.py:2121]. The `build()` **function** itself [setup.py:1084‑1095] compiles only the C pieces — the C extension, GLFW, and the C kittens (`compile_c_extension` [setup.py:1090], `compile_glfw` [setup.py:1094], `compile_kittens` [setup.py:1095]); it does **not** generate any Go source. The Go generation happens inside `build_static_kittens` [setup.py:1130], which calls `update_go_generated_files` [setup.py:1144] → `subprocess.run([kitty_exe, '+launch', os.path.join(src_base, 'gen/go_code.py')], …)` [setup.py:1112] (function defined at [setup.py:1102]) to generate the Go sources and the embedded `data_generated.bin` blobs **before** it assembles the Go build command `cmd = [go, 'build', '-v']` [setup.py:1148].

**Why the build must be two‑stage (observed).** A naive Go build *before* generation fails. Reproduced in a pristine tree obtained with `git archive HEAD | tar -x` into a throw‑away directory **outside** the repository (so no generated files are present):

```text
$ go build ./tools/cmd            # in a pristine, pre-generation tree
kittens/ssh/main.go:15:2: package kitty is not in std (/usr/local/go/src/kitty)
tools/tui/shell_integration/data.go:19:12: pattern data_generated.bin: no matching files found
tools/unicode_names/query.go:20:12: pattern data_generated.bin: no matching files found
```

Cause→effect: the `//go:embed data_generated.bin` directives at `tools/tui/shell_integration/data.go:19` and `tools/unicode_names/query.go:20` require blobs that do not exist until the generation stage runs; and the root package `kitty` (imported bare at `kittens/ssh/main.go:15`) is itself generated — on this branch the only root‑level `.go` file is the untracked, generated `constants_generated.go`. After the canonical `python setup.py build` completes, the same command succeeds:

```text
$ go build -mod=readonly ./tools/cmd
$ echo $?
0
```

The diff kitten's own `Config` and `Options` Go types are themselves **generated** into `kittens/diff/`: `type Config struct` lives at `kittens/diff/conf_generated.go:12` and `type Options struct` at `kittens/diff/cli_generated.go:39`. Both generated Go files are **untracked and git‑ignored** — absent from the tracked tree until the generation stage runs (verified with `git ls-files`/`git check-ignore`). This is separate from `.gitattributes`, which marks the *Python* option‑definition sources `kittens/diff/options/types.py` [.gitattributes:15] and `kittens/diff/options/parse.py` [.gitattributes:16] as `linguist-generated`; those Python files are the option *definitions* the generator reads, not the Go runtime types (and no `kittens/diff/options/` directory is present in the working tree).

**Version banner (verbatim, canonical build).** The exact binary invoked in all runs below is `kitty/launcher/kitten`:

```text
$ ./kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
```

**Build‑stamped revision (verbatim, canonical build).** The two‑stage build's final Go link step stamps the working‑tree revision into the binary. The verbatim `--verbose` build‑log line that performs this is:

```text
$ CI=true GOTOOLCHAIN=local ASAN_OPTIONS=detect_leaks=0 python3 setup.py build --verbose
# … C build, then "Updating Go generated files…", then:
/usr/local/bin/go build -v -ldflags '-X kitty.VCSRevision=e3ae9a081ed071a46d494bfe60ae264ce4477337 -s -w' -o kitty/launcher/kitten <repo>/tools/cmd
```

The stamped value `e3ae9a081ed071a46d494bfe60ae264ce4477337` is exactly the `git rev-parse HEAD` output (short `e3ae9a081`). The `-X kitty.VCSRevision=…` flag is assembled at [setup.py:1151] from `get_vcs_rev()` [setup.py:674], which runs `git rev-parse HEAD`. Confirmed by reading it back out of the binary: `strings kitty/launcher/kitten | grep VCSRevision` → `kitty.VCSRevision=e3ae9a081ed071a46d494bfe60ae264ce4477337`. Every observation in this document was produced by this exact build at revision `e3ae9a081`.

**Real entry point.** Every behavioral observation uses the real entry point `kitten diff <left> <right>` (equivalently `kitty +kitten diff <left> <right>`). Because the kitten is a full‑screen TUI that requires a real PTY with a **non‑zero** window size (otherwise it panics with a divide‑by‑zero at loop start), interactive captures were driven through a Python `pty.fork()` harness that sets `TIOCSWINSZ` to a non‑zero size (e.g. 50×200), reads the rendered output, then sends `q` to quit. Terminal escape sequences were stripped for readability; the raw bytes were retained. Error‑path claims (wrong argument count, directory‑vs‑file) print to stderr **before** the TUI loop starts and were captured directly. Two observations that cannot be produced through the TUI — `runtime.NumCPU()` and a Go **race‑detector** confirmation — were obtained from scratch programs and are **explicitly labeled** where they appear.

**Fixture matrix (all created OUTSIDE the repository, removed afterward).** A `left/`+`right/` directory pair was built to trigger each behavior on purpose:

| Fixture | What it triggers | Question |
|---|---|---|
| `changed.txt` (same path, different bytes) | a content change | Q1, Q8 |
| `original_name.txt` → `renamed_name.txt` (identical bytes, different path) | a rename | Q2 |
| `added.txt` (right only), `removed.txt` (left only) | an addition / a removal | Q1, Q2 |
| `mode.sh` (identical bytes, `chmod 600` vs `755`) | a mode‑only change | Q1 |
| `blob.bin` (256 random bytes) | a binary file | Q6 |
| `pic.png` (real PNG) | an image file | Q6 |
| `invalid.dat` (bytes `ff fe 00 01 …`) | a non‑UTF‑8 file | Q6 |
| `scale250/` and `scale1000/` — 250 and 1000 changed `.go` files/side | the parallel worker pool | Q4, Q5, Q7 |

**Stability (and an explicit caveat about the scale fixtures).** Values that depend on scale/timing state the scale used, and this document keeps them in **two clearly separated categories** rather than implying uniform reproducibility:

- **(a) Reliably reproducible (every run):** the worker‑count ceiling `runtime.NumCPU()` (scratch program, stable ×2), the single‑`.py` highlighted digest (**5/5 runs, 0 crashes**), and the deterministic Go race‑detector corroboration (×3).
- **(b) Rare surviving samples (selected non‑crashing runs):** *every* timing, digest, and frame value taken from the 250‑ and 1000‑file scale fixtures. Those fixtures **crash on the large majority of attempts** with `fatal error: concurrent map writes` — measured at the current build as **250‑file 20/20 and 39/40**, **1000‑file built‑in 20/20**, and **1000‑file default 50/50** (see Q5/Q7). Each scale value shown is therefore explicitly a **selected surviving (non‑crashing) run, presented next to its crash denominator**, and is **not** a claim that repeated runs succeed. Where surviving runs recurred (the 250‑file digest), the two survivors captured produced the *identical* value; the 1000‑file frame timing could **not** be reproduced at all (0/50 survivors) and is not reported.

The repository was verified unchanged (`git status --porcelain` empty) before and after (see Appendix).

---

## Q1 — Directory pairing: how it decides "what belongs together"

**Observed.** Pointing the kitten at the two directories produces one titled block per file. Files that exist on **both** sides under the **same relative path** are paired; the changed one shows a diff, the mode‑only one shows a mode message:

Command (driven through the PTY harness; output below is the rendered screen with terminal escape sequences stripped):

```text
$ ./kitty/launcher/kitten diff /tmp/kitty_obs/left /tmp/kitty_obs/right
```

```text
   changed.txt
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   @@ -1,4 +1,5 @@
1  line one
1  line one
2  line two
2  line two CHANGED
3  line three
3  line three
4  line four
4  line four

5  line five

   mode.sh
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Mode changed: -rw------- to -rwxr-xr-x
```

The left `changed.txt` (4 lines) and the right `changed.txt` (5 lines, line 2 edited) are placed **in the same titled block** and diffed against each other because they share the relative path `changed.txt`; `mode.sh` (byte‑identical, `chmod 600` on the left vs `755` on the right) is likewise paired and reports a mode‑only change.

**Cause→effect.** Pairing is decided by **relative path equality**. `collect_files` [kittens/diff/collect.go:296] walks the left tree [collect.go:299] and the right tree [collect.go:303] into two name‑sets. The `walk` helper [collect.go:260] keys every file by its path **relative to its base** via `filepath.Rel(base, path)` [collect.go:283]. It then intersects the two sets:

```text
common_names := left_names.Intersect(right_names)   // collect.go:306
```

So **"what belongs together" = the same relative path on both sides.** For each common name it reads both sides through `data_for_path` [collect.go:309, collect.go:312] and:

- if the bytes differ, `if ld != rd` [collect.go:317], it records a change via `self.add_change(...)` [collect.go:319];
- if the bytes are identical it compares `os.Stat().Mode()`, and when `lstat.Mode() != rstat.Mode()` [collect.go:323] — the in‑code comment reads `// identical files with only a mode change` [collect.go:324] — it still records a change [collect.go:326]. This is exactly the observed `Mode changed: -rw------- to -rwxr-xr-x` line (our fixture used `chmod 600` vs `755`).

**Corroboration by the existing test (run, read‑only).** The ignore‑pattern behavior of `walk` is confirmed by the repository's own test:

```text
$ GOTOOLCHAIN=local go test -run TestDiffCollectWalk -v ./kittens/diff/
=== RUN   TestDiffCollectWalk
--- PASS: TestDiffCollectWalk (0.00s)
PASS
ok  	kitty/kittens/diff	0.021s
```

`TestDiffCollectWalk` [kittens/diff/collect_test.go:19] calls `walk(tdir, []string{"*~", "#*#", "b"}, names, pmap, map[string]string{})` [collect_test.go:42] and asserts with `cmp.Diff` [collect_test.go:45, collect_test.go:51] (using `github.com/google/go-cmp/cmp` [collect_test.go:14]) that the ignore patterns `*~`, `#*#`, and `b` are filtered out of the walk.

**Sibling variants (enumerated).**
- The ignore patterns come from `conf.Ignore_name`, passed to both `walk` calls [collect.go:299, collect.go:303]. Its config option is `+ignore_name` with an **empty** base default `opt('+ignore_name', '', …)` [kittens/diff/main.py:56] and `add_to_default=False` [main.py:57] — so **no ignore patterns ship by default**. The patterns `.git` [main.py:63], `*~` [main.py:64], and `*.pyc` [main.py:65] are only **examples** in the option's help text, listed under `For example::` [main.py:61]; they are illustrations of how a user *could* configure `ignore_name`, not shipped defaults. Matching is by basename via `filepath.Match` in `allowed` [collect.go:230].
- The number of context lines around a change defaults to `num_context_lines` = `'3'` [kittens/diff/main.py:37].
- Directories themselves are skipped as entries (only files are keyed); an ignored directory is pruned by returning `fs.SkipDir` inside `walk`'s `filepath.WalkDir` callback — when `is_allowed := allowed(path, patterns...)` [collect.go:269] is false [collect.go:270] and the entry is a directory `d.IsDir()` [collect.go:271], it returns `fs.SkipDir` [collect.go:272].

---

## Q2 — Rename recognition: rename vs. delete + add

**Observed.** In a single run over the fixture directories (the same command as Q1), the identical‑content file that lives at `original_name.txt` on the left and `renamed_name.txt` on the right is rendered with **both names stacked in one titled block** — a rename — while the add‑only and remove‑only files each get their **own** block with a `This file was added` / `This file was removed` line:

```text
$ ./kitty/launcher/kitten diff /tmp/kitty_obs/left /tmp/kitty_obs/right
```

The rename shows **both** relative paths under one rule, with no add/remove text and no content diff (the bytes are identical):

```text
   original_name.txt
   renamed_name.txt
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

The file present only on the right renders as an addition:

```text
   added.txt
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   This file was added
1  brand new file only on the right side ADDED
```

The file present only on the left renders as a removal:

```text
   removed.txt
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1  this file exists only on the left side REMOVED
   This file was removed
```

**Cause→effect.** After the common names are handled, the leftovers are split: `removed := left_names.Subtract(common_names)` [kittens/diff/collect.go:332] and `added := right_names.Subtract(common_names)` [collect.go:333]. Every added path is MD5‑hashed [collect.go:335‑340] and every removed path is MD5‑hashed [collect.go:341‑346] through `hash_for_path` [collect.go:106] (which computes `md5.Sum` over the file's bytes — `crypto/md5` is imported at [collect.go:6] — via `hash_cache.GetOrCreate` → `data_for_path`). For each removed hash it looks for an added hash that matches, `if ah == rh` [collect.go:350], and then **confirms full byte‑equality** `if ld == rd` [collect.go:353] before recording the rename with `self.add_rename(...)` [collect.go:354], discarding the matched addition with `added.Discard(n)` [collect.go:355] and `break` [collect.go:357]. Removed paths with no match become removals via `self.add_removal(...)` [collect.go:362]; still‑unmatched additions become adds via `self.add_add(...)` [collect.go:366].

So **a rename is identical content at a different relative path** — a hash match *and* a byte‑equality confirmation. That two‑step test is exactly why the kitten reports our fixture as a rename (`original_name.txt renamed_name.txt` on one row) instead of a deletion plus a new file.

**Sibling variants (enumerated).**
- **Add** (`add_add` [collect.go:181]) — present only on the right; observed as `This file was added`.
- **Remove** (`add_removal` [collect.go:192]) — present only on the left; observed as `This file was removed`.
- **Rename** (`add_rename` [collect.go:175]) — hash + byte match at a different path; observed as the paired header.
- The MD5 hash is only a fast pre‑filter; the byte‑equality check at [collect.go:353] is what prevents a hash collision from being mis‑reported as a rename. **[Inferred]** — no hash collision was engineered at runtime; this is read from the `ah == rh` → `ld == rd` sequence.

---

## Q3 — Cache efficiency: from raw contents to highlighted output

**Observed (read‑once).** The directly‑observable cache effect is that a file's bytes are read from disk **exactly once** even though several consumers need them (change‑detection, the diff itself, and highlighting). Running the builtin engine over a one‑file pair under `strace` shows a single successful `openat` for each side's file — captured by following the child with `strace -f`:

```text
$ strace -f -e trace=openat -o strace.log \
      python3 harness.py -- ./kitty/launcher/kitten diff -o diff_cmd=builtin /tmp/kitty_obs/q8l /tmp/kitty_obs/q8r
$ grep openat strace.log | grep changed.txt | grep -v ENOENT
155670 openat(AT_FDCWD, "/tmp/kitty_obs/q8l/changed.txt", O_RDONLY|O_CLOEXEC) = 10
155670 openat(AT_FDCWD, "/tmp/kitty_obs/q8r/changed.txt", O_RDONLY|O_CLOEXEC) = 10
```

Counting those opens confirms exactly one per side, and the count is stable across two runs:

```text
$ grep -c 'q8l/changed.txt' strace.log        # left side, run 1 and run 2
1
$ grep -c 'q8r/changed.txt' strace.log        # right side, run 1 and run 2
1
```

That single read is what `data_cache` provides: the first consumer materializes the bytes and every later consumer in the same run reuses the cached value. **[Inferred from source]** — the following claims are read from the code, not separately instrumented: that the seven caches are all created together at startup (`init_caches()`), that each `kitten diff` invocation builds a **fresh** per‑process set of caches, and the specific derivation chain (`data` → `lines` → `highlighted`) described below.

**Cause→effect.** All seven caches are constructed in `init_caches()` [kittens/diff/collect.go:26‑37], each with capacity `const sz = 4096` [collect.go:29]. Enumerated **by name and type**:

1. `mimetypes_cache` — `*utils.LRUCache[string, string]` [collect.go:20]
2. `data_cache` — `*utils.LRUCache[string, string]` [collect.go:20]
3. `hash_cache` — `*utils.LRUCache[string, string]` [collect.go:20]
4. `size_cache` — `*utils.LRUCache[string, int64]` [collect.go:21]
5. `lines_cache` — `*utils.LRUCache[string, []string]` [collect.go:22]
6. `highlighted_lines_cache` — `*utils.LRUCache[string, []string]` [collect.go:23]
7. `is_text_cache` — `*utils.LRUCache[string, bool]` [collect.go:24]

**Derivation chain (compute‑once).** `data_for_path` [collect.go:65] reads the file's bytes exactly once with `os.ReadFile` [collect.go:67] and wraps them via `utils.UnsafeBytesToString` to avoid a copy [collect.go:68]. From that single cached read: `lines_for_path` [collect.go:138] sanitizes and splits into lines (caching the slice), and `highlighted_lines_for_path` [collect.go:148] returns the highlighted lines **only when** a cached highlighted entry exists whose line count matches the plain lines, `if ans, found := highlighted_lines_cache.Get(path); found && len(ans) == len(plain_lines)` [collect.go:153], otherwise it returns the plain lines. The same `data_cache` also backs hashing — `hash_for_path` [collect.go:106] computes its `md5.Sum` over the bytes returned by `data_for_path` [collect.go:108] — and the builtin diff reads through it as well (see Q8), so a path's **content** is read from disk once and reused by diffing, hashing, line‑splitting, and highlighting. File **size** is a *separate* cache and code path: `size_for_path` [collect.go:72] does **not** touch `data_cache`; it uses its own `size_cache.GetOrCreate` [collect.go:73] backed by `os.Stat(path)` [collect.go:74] returning `s.Size()` [collect.go:78] — so sizes come from `stat`, never from reading the file's contents.

**How the LRU stays bounded — `GetOrCreate`.** The generic cache is `utils.LRUCache[K,V]` [tools/utils/cache.go:13], built by `NewLRUCache` [cache.go:20]. On a miss, `GetOrCreate` [cache.go:39‑58] first does an `RLock` read‑check [cache.go:40‑42], computes the value [cache.go:46], then takes a write `Lock()` [cache.go:48], stores it, `PushFront`s the **newly‑created** key onto the list [cache.go:50], and when the list exceeds capacity it removes the entry at the **back** of that list, `if self.max_size > 0 && self.lru.Len() > self.max_size { … Remove(self.lru.Back()) … }` [cache.go:51‑53]. Precisely because the key is pushed **only on a miss/creation** — a *found* entry is returned without any `PushFront` [cache.go:43‑44], and `Get` is a plain `RLock` lookup that never promotes [cache.go:25‑30] — the list tracks **insertion (miss) order**, not access recency. So `Back()` is the **oldest‑inserted** still‑resident key, which is *not* the same as a true least‑recently‑*used* (access‑recency) entry; nothing in this implementation re‑orders a key when it is read.

**REPORT‑VERBATIM #1 — `LRUCache.Set` takes a READ lock and does not track eviction (reported exactly as observed, not "corrected").** `func (self *LRUCache[K, V]) Set` [tools/utils/cache.go:32] takes `self.lock.RLock()` — a **read** lock — [cache.go:33] and simply assigns `self.data[key] = val` [cache.go:34]; it does **not** call `lru.PushFront` and does **not** evict. Consequently, entries stored via `Set` — for example `highlighted_lines_cache.Set(path, text_to_lines(raw))` in `highlight_all` [kittens/diff/highlight.go:224] — are **not** tracked for LRU eviction the way `GetOrCreate` entries are. `MustGetOrCreate` [cache.go:60‑72] likewise never `PushFront`s or evicts (it does take a write `Lock()` at [cache.go:68]); it backs `mimetype_for_path` [collect.go:51] and `is_path_text` [collect.go:87]. A real, observable second‑order consequence of the read‑lock `Set` is documented under **Q5**.

**Cache‑synergy note (ties to Q8).** Because both the diff and the rename‑hash read through the single `data_cache` [collect.go:65], a path's bytes are materialized once and then reused by diffing, hashing, line‑splitting, and highlighting.

---

## Q4 — Concurrency: what happens when many files are processed at once

**Observed (scale + timing — surviving, non‑crashing runs only).** A 250‑changed‑`.go`‑file‑per‑side fixture drives the shared worker pool hard. **At this scale the run crashes on the large majority of attempts** with `fatal error: concurrent map writes` — measured in **Q5** below as **20/20** crashed in one fresh batch and **39/40** in another. The timings here are therefore taken from the **rare surviving (non‑crashing) runs**, explicitly *selected* because they did not crash — **not** a claim that every attempt succeeds. On such a surviving run the built‑in‑diff view reaches a settled render in ≈1.2–1.4 s; two independent surviving runs gave:

```text
# SETTLE time of runs that did NOT crash; the large majority of attempts crashed first (see Q5)
$ ./kitty/launcher/kitten diff -o diff_cmd=builtin /tmp/.../scale250/left /tmp/.../scale250/right
surviving run A: SETTLE_SECONDS=1.354
surviving run B: SETTLE_SECONDS=1.211
```

**Worker count (labeled — obtained via a scratch Go program, NOT the TUI).** The TUI emits no worker‑count line, so `runtime.NumCPU()` and the pool‑sizing arithmetic were observed with a standalone Go program (outside the repo) that replicates the sizing logic; stable across 2 runs:

```text
runtime.NumCPU()=128
GOMAXPROCS=128
Parallel pool for count=1   -> 1   workers
Parallel pool for count=5   -> 5   workers
Parallel pool for count=200 -> 128 workers
```

(`nproc` reports 4 here because of the cgroup CPU quota; `runtime.NumCPU()` reads the machine's 128 logical CPUs, matching `/proc/cpuinfo`.)

**Cause→effect.** Both per‑file diffing and highlighting fan out through the **shared** worker pool `func (self *Context) Parallel(start, stop int, fn func(<-chan int))` [tools/utils/images/utils.go:27]. Worker count is `procs := self.NumberOfThreads()` [utils.go:33] (defined at [utils.go:22]); when unset, `if procs <= 0 { procs = runtime.NumCPU() }` [utils.go:34‑36], then it is capped at the item count, `if procs > count { procs = count }` [utils.go:37‑39]. All indices are pushed into a buffered channel `c := make(chan int, count)` [utils.go:41] (fill loop [utils.go:42‑44], `close(c)` [utils.go:45]); `procs` goroutines are launched and joined on a `sync.WaitGroup` [utils.go:47], with `wg.Wait()` [utils.go:55]. This is exactly why the observed pool is `min(runtime.NumCPU()=128, item_count)` — 128 workers for 200+ items, but only 5 for 5 items.

**Per‑file diffing uses this pool.** `func diff(jobs []diff_job, context_count int)` [kittens/diff/patch.go:352] builds `results := make(chan result, len(jobs))` [patch.go:360], runs `ctx.Parallel(0, len(jobs), …)` [patch.go:361] where each goroutine takes `job := jobs[i]` [patch.go:363], calls `do_diff(...)` [patch.go:365], sends `results <- r` [patch.go:366], and the results are collected into a map keyed by the left file, `ans[r.file1] = r.patch` [patch.go:374].

**Sibling variant.** `NumberOfThreads()` can be set via `SetNumberOfThreads` [tools/utils/images/utils.go:18]; the diff kitten leaves it at the default (0), so `runtime.NumCPU()` governs the ceiling. **[Inferred]** — no non‑default thread count was exercised; read from `Parallel`'s branch at [utils.go:34‑36].

---

## Q5 — Parallel highlighting isolation ("without stepping on itself")

**Observed (value‑level determinism — separating the reliably‑reproducible case from the rare surviving‑at‑scale case).** The syntax‑highlighted diff body was captured through the **real** `kitten diff` TUI (driven in a PTY with a non‑zero winsize), the cursor/erase control sequences were stripped while the **SGR colour sequences** (`ESC[…m`) were kept, and the visible region from the first `@@` hunk onward was MD5‑hashed (`SGR_COUNT` counts the `ESC[…m` sequences). Two fixtures were used and they behave **very differently** at runtime: **(1)** a single highlighted `.py` file, which has effectively no highlight parallelism and therefore **never crashes** and is byte‑for‑byte identical on every run; and **(2)** a 250‑changed‑`.go`‑file‑per‑side tree, which forces heavy parallel highlighting and therefore **crashes on the large majority of attempts** (`fatal error: concurrent map writes`; see the crash‑frequency block below), so its digest can only be read from the **rare surviving (non‑crashing) runs**, explicitly *selected* because they did not crash:

```text
# (1) RELIABLY REPRODUCIBLE: single .py fixture (one highlighted file), builtin engine
#     5 runs, 0 crashes — one file => no highlight parallelism => no shared-map race
$ ./kitty/launcher/kitten diff -o diff_cmd=builtin /tmp/.../hl/left /tmp/.../hl/right
run1: MD5_BODY: 4abde92ef0e39366dd91f3597f94db1d SGR_COUNT: 108
run2: MD5_BODY: 4abde92ef0e39366dd91f3597f94db1d SGR_COUNT: 108
run3: MD5_BODY: 4abde92ef0e39366dd91f3597f94db1d SGR_COUNT: 108
run4: MD5_BODY: 4abde92ef0e39366dd91f3597f94db1d SGR_COUNT: 108
run5: MD5_BODY: 4abde92ef0e39366dd91f3597f94db1d SGR_COUNT: 108

# (2) RARE SURVIVING SAMPLE: 250-changed-.go-file-per-side tree (heavy parallel highlight), builtin
#     crash frequency at this scale: 39/40 and 20/20 attempts crashed first (see crash-frequency block below);
#     the two lines below are the RARE survivors, explicitly selected because they did NOT crash
$ ./kitty/launcher/kitten diff -o diff_cmd=builtin /tmp/.../scale250/left /tmp/.../scale250/right
surviving run A: MD5_BODY: 88256ed956844aeeacb5aa089525adeb SGR_COUNT: 416
surviving run B: MD5_BODY: 88256ed956844aeeacb5aa089525adeb SGR_COUNT: 416
```

The two cases must not be conflated. Fixture **(1)** is reproducible on *every* run (5/5, 0 crashes): with one file there is no highlight parallelism, so the highlighted **values** are trivially stable — the `CHANGED` token is coloured the same way, with no interleaving. Fixture **(2)**'s identical digest, by contrast, is a **selected surviving sample** — the *majority* of attempts crash before any digest is produced, and on the infrequent run that survives, the two survivors captured happened to produce the identical digest. The lesson is that value‑level isolation (distinct index → distinct path → distinct key) protects the *computed values*, while the shared‑map `Set` (Q3, REPORT‑VERBATIM #1) is **not** memory‑safe — so at scale a full highlighted render is usually **not** produced at all. Both facts are shown next.

**Cause→effect (the isolation design).** `func highlight_all(paths []string)` [kittens/diff/highlight.go:217] runs `ctx.Parallel(0, len(paths), …)` [highlight.go:219]; each goroutine consumes a **distinct index** `i` → a **distinct path** `path := paths[i]` [highlight.go:221] → a **distinct cache key**. `highlight_file` builds its output into a **local** `w := strings.Builder{}` [highlight.go:210] pre‑grown with `w.Grow(len(text) * 2)` [highlight.go:211] via `chroma.FormatterFunc(ansi_formatter)` [highlight.go:209], then performs a **single** `highlighted_lines_cache.Set(path, text_to_lines(raw))` per path [highlight.go:224]. Because no two goroutines share a key or a builder, the *values* they compute cannot corrupt each other.

**REPORT‑VERBATIM #1's real consequence (observed on the real entry point — reported exactly, not "corrected").** Value‑level isolation is **not** the same as memory‑safety on the shared map. Because `highlighted_lines_cache.Set` guards the map with only a **read** lock (`RLock`, [tools/utils/cache.go:33]) and then writes `self.data[key] = val` [cache.go:34], concurrent `Set` calls from many highlight goroutines write the same Go map at once. At scale this trips Go's built‑in concurrent‑map‑write detector. Running the **real** `kitten diff` on the 1000‑changed‑`.go`‑file tree in built‑in mode produced a genuine crash, captured verbatim through the PTY (ANSI stripped). The `[...]` printed after `LRUCache` is Go's own rendering of the generic type parameters in a fatal‑error trace — it is **not** an elision; every frame below is exact:

```text
$ ./kitty/launcher/kitten diff -o diff_cmd=builtin /tmp/.../scale1000/left /tmp/.../scale1000/right
fatal error: concurrent map writes

goroutine 336 [running]:
kitty/tools/utils.(*LRUCache[...]).Set(0x35bc, {0xc00044db30?, 0x0?}, {0xc00962d908?, 0xce, 0x200})
	/tmp/blitzy/kitty/blitzy-5900e980-186b-42ac-b10f-47549fbb3c92_09725a/tools/utils/cache.go:34 +0x85
kitty/kittens/diff.highlight_all.func1(0xc0012ee000)
	/tmp/blitzy/kitty/blitzy-5900e980-186b-42ac-b10f-47549fbb3c92_09725a/kittens/diff/highlight.go:224 +0xae
kitty/tools/utils/images.(*Context).Parallel.func1()
	/tmp/blitzy/kitty/blitzy-5900e980-186b-42ac-b10f-47549fbb3c92_09725a/tools/utils/images/utils.go:52 +0x4e
created by kitty/tools/utils/images.(*Context).Parallel in goroutine 164
	/tmp/blitzy/kitty/blitzy-5900e980-186b-42ac-b10f-47549fbb3c92_09725a/tools/utils/images/utils.go:50 +0xe5
```

This is **not** a rare one‑off — at scale it is the *common* outcome. Counting a run as a crash when the child self‑exits with `concurrent map writes`, fresh batches at the current build gave a very high, repeatable crash frequency across independent batches (each denominator is the total number of attempts in that batch):

```text
s250_builtin  batch 1 (20 attempts): 20/20 crashed with 'concurrent map writes'   (0 survived)
s250_builtin  batch 2 (40 attempts): 39/40 crashed with 'concurrent map writes'   (1 survived)
s1000_builtin batch   (20 attempts): 20/20 crashed with 'concurrent map writes'   (0 survived)
s1000_default batch   (50 attempts): 50/50 crashed with 'concurrent map writes'   (0 survived)
```

The stack lands **exactly** on the read‑lock `Set` (`tools/utils/cache.go:34`), reached from `highlight_all`'s goroutine body (`kittens/diff/highlight.go:224`) inside the shared pool (`tools/utils/images/utils.go:52`) — precisely the `Set` path described under REPORT‑VERBATIM #1. That the crash dominates but a small minority of runs still survive (1 in 40 at 250 files; 0 in 50 at 1000 files) is the hallmark of a data race on the map `Set` guards with only `RLock`: whether the fatal collision occurs depends on goroutine scheduling, so it is intermittent — heavily biased toward crashing at this scale. **This is exactly why every scale timing/digest/frame value in this document is labeled a surviving (non‑crashing) sample: the numbers above are the denominators those samples were selected from.**

**Deterministic corroboration (labeled NON‑CANONICAL — Go race detector on the REAL cache + REAL pool via a scratch module; NOT the `kitten diff` entry point).** The crash above is intermittent by nature, so to confirm the mechanism *deterministically* a scratch Go module outside the repo (`module kittyrace`, `replace kitty => <repo>`) linked the **actual** `kitty/tools/utils.NewLRUCache[...].Set` and the **actual** `kitty/tools/utils/images.Context.Parallel`, and drove them exactly as `highlight_all` does (distinct index → distinct path → a single `Set` per path, mirroring [kittens/diff/highlight.go:219‑225]). Under `go run -race` it flagged the write‑write race on **3/3** runs; the report points at the same `cache.go:34` and `utils.go:52`/`utils.go:50` frames as the real crash (the `main.go:*` frames are the scratch driver standing in for `highlight_all.func1`). Full verbatim output:

```text
==================
WARNING: DATA RACE
Write at 0x00c000527410 by goroutine 15:
  runtime.mapassign_faststr()
      /usr/local/go/src/runtime/map_faststr.go:223 +0x0
  kitty/tools/utils.(*LRUCache[go.shape.string,go.shape.[]string]).Set()
      /tmp/blitzy/kitty/blitzy-5900e980-186b-42ac-b10f-47549fbb3c92_09725a/tools/utils/cache.go:34 +0xa4
  main.main.func1()
      /tmp/kitty_race/main.go:26 +0x75
  kitty/tools/utils/images.(*Context).Parallel.func1()
      /tmp/blitzy/kitty/blitzy-5900e980-186b-42ac-b10f-47549fbb3c92_09725a/tools/utils/images/utils.go:52 +0x8d

Previous write at 0x00c000527410 by goroutine 9:
  runtime.mapassign_faststr()
      /usr/local/go/src/runtime/map_faststr.go:223 +0x0
  kitty/tools/utils.(*LRUCache[go.shape.string,go.shape.[]string]).Set()
      /tmp/blitzy/kitty/blitzy-5900e980-186b-42ac-b10f-47549fbb3c92_09725a/tools/utils/cache.go:34 +0xa4
  main.main.func1()
      /tmp/kitty_race/main.go:26 +0x75
  kitty/tools/utils/images.(*Context).Parallel.func1()
      /tmp/blitzy/kitty/blitzy-5900e980-186b-42ac-b10f-47549fbb3c92_09725a/tools/utils/images/utils.go:52 +0x8d

Goroutine 15 (running) created at:
  kitty/tools/utils/images.(*Context).Parallel()
      /tmp/blitzy/kitty/blitzy-5900e980-186b-42ac-b10f-47549fbb3c92_09725a/tools/utils/images/utils.go:50 +0x126
  main.main()
      /tmp/kitty_race/main.go:23 +0x374

Goroutine 9 (running) created at:
  kitty/tools/utils/images.(*Context).Parallel()
      /tmp/blitzy/kitty/blitzy-5900e980-186b-42ac-b10f-47549fbb3c92_09725a/tools/utils/images/utils.go:50 +0x126
  main.main()
      /tmp/kitty_race/main.go:23 +0x374
==================
```

(For completeness: `go test -race ./kittens/diff/` **passes** — `ok kitty/kittens/diff` — but that package suite only exercises `walk()` in `collect_test.go`; it does **not** drive `highlight_all`, so it neither confirms nor refutes this race. The race is demonstrated by the real crash above and by this scratch `-race` run, not by the package test.)

So the honest, precise answer to "how does highlighting run in parallel without stepping on itself?" is: **each goroutine works on a distinct index → path → cache key with its own local `strings.Builder`, so the computed highlight values never overwrite one another** [highlight.go:210, highlight.go:221, highlight.go:224]; **however**, because the shared `highlighted_lines_cache` is written through `Set` (which takes only an `RLock`, [cache.go:33‑34]), the concurrent map writes are not themselves serialized, and at high concurrency this is observable as `fatal error: concurrent map writes`.

**Chroma sibling detail (lexer selection, by name and file:line).** Highlighting uses `github.com/alecthomas/chroma/v2 v2.14.0` [go.mod:7]. Lexer selection happens in `highlight_file` [kittens/diff/highlight.go:161]: the base filename is taken with `filepath.Base` [highlight.go:162], its lower‑cased extension is looked up in the configurable `conf.Syntax_aliases[ext]` map [highlight.go:166] (so a configured alias can rewrite the filename used for detection, e.g. mapping an extension to another language), the lexer is then chosen **by filename/aliases** via `lexers.Match(filename_for_detection)` [highlight.go:175], and **only if that returns `nil`** does it fall back to **content analysis** via `lexers.Analyse(text)` [highlight.go:178]. The per‑token formatter `ansi_formatter` [highlight.go:84] then maps each token to a terminal SGR escape using `SGR_PREFIX = "\033["` [highlight.go:85] and `SGR_SUFFIX = "m"` [highlight.go:86], writing into the local `strings.Builder` [highlight.go:210] through `chroma.FormatterFunc(ansi_formatter)` [highlight.go:209]. The default style is memoized once with `sync.OnceValue` in `DefaultStyle` [highlight.go:26]. The backing regex engine is `github.com/dlclark/regexp2 v1.11.0` [go.mod:9].


---

## Q6 — Binary and image files alongside plain text

**Observed.** In one directory run — driven through the PTY, ANSI‑stripped, then grep‑filtered to the file markers of the first rendered frame (stable across 3 runs) — the plain‑text file is line‑diffed (a `@@` unified hunk), the binary and the non‑UTF‑8 file are each shown as `Binary file: <size>`, and the image is shown with its dimensions and an async `Loading image...` placeholder that is later replaced by an actual in‑terminal image. Each file name is followed by its classification, with binary/image metadata shown once per side (left and right):

```text
$ python3 harness.py 2.5 1 out.raw -- ./kitty/launcher/kitten diff /tmp/.../left /tmp/.../right \
    | sed -n '/TEXT_VIEW_START/,/TEXT_VIEW_END/p' \
    | grep -E 'blob.bin|invalid.dat|pic.png|changed.txt|Binary file:|Dimensions:|Loading image|@@ '
   blob.bin
   Binary file: 256 B
   Binary file: 256 B
   changed.txt
   @@ -1,4 +1,5 @@
   invalid.dat
   Binary file: 6 B
   Binary file: 6 B
   pic.png
   Dimensions: 4x4 Size: 73 B
   Dimensions: 4x4 Size: 77 B
   Loading image...
   Loading image...
```

The non‑UTF‑8 fixture really is invalid UTF‑8 (so it is classified as binary rather than text):

```text
$ python3 -c "d=open('/tmp/.../left/invalid.dat','rb').read()
try:
    d.decode('utf-8'); print('utf8_valid=True')
except UnicodeDecodeError as e:
    print(f'utf8_valid=False ({e})')"
utf8_valid=False ('utf-8' codec can't decode byte 0xff in position 0: invalid start byte)
```

And the image path really emits Kitty graphics‑protocol packets. Counting the graphics Application‑Programming‑Command escapes (`ESC _ G`) in the raw PTY byte stream gives a stable count of **15** across two runs:

```text
$ for r in 1 2; do python3 harness.py 2.5 1 out_$r.raw -- \
      ./kitty/launcher/kitten diff /tmp/.../left /tmp/.../right \
      | grep -oE 'GRAPHICS_APC_COUNT=[0-9]+'; done
GRAPHICS_APC_COUNT=15
GRAPHICS_APC_COUNT=15
```

In the same run there are exactly **two** `Loading image...` placeholders (one per side), drawn before the images finish loading and then replaced by the in‑terminal images on a later re‑render:

```text
$ python3 harness.py 2.5 1 out.raw -- ./kitty/launcher/kitten diff /tmp/.../left /tmp/.../right \
    | sed -n '/TEXT_VIEW_START/,/TEXT_VIEW_END/p' | grep -c 'Loading image'
2
```

**Cause→effect.**
- **MIME detection.** `mimetype_for_path` [kittens/diff/collect.go:50] calls `utils.GuessMimeTypeWithFileSystemAccess` [tools/utils/mimetypes.go:100] (which returns `inode/directory` for directories [mimetypes.go:110]; the pure‑name variant is `GuessMimeType` [mimetypes.go:70]). It defaults to `application/octet-stream` when unknown [collect.go:54], and remaps `utils.KnownTextualMimes` entries to `text/…` [collect.go:56‑60].
- **Image test.** `is_image` [collect.go:82] is `strings.HasPrefix(mimetype_for_path(path), "image/")` [collect.go:83].
- **Text gate.** `is_path_text` [collect.go:86] returns **false for images** [collect.go:88‑90], **false for `/dev/null`** (via `os.Stat` + `os.SameFile`) [collect.go:91‑96], and otherwise returns `utf8.ValidString(d)` [collect.go:102]. Our `blob.bin` and `invalid.dat` both fail this, hence `Binary file:` instead of a line diff.
- **Effect on the pipeline.** A per‑file diff job is only enqueued when **both** sides are text: `generate_diff` guards with `if is_path_text(path) && is_path_text(changed_path)` [kittens/diff/ui.go:147], so non‑text/non‑image files **skip diffing**. They also **skip highlighting**: `Handler.highlight_all` [kittens/diff/ui.go:179] first filters the highlight set with `text_files := utils.Filter(self.collection.paths_to_highlight.AsSlice(), is_path_text)` [ui.go:180] and hands only the text files to the parallel `highlight_all(text_files)` [ui.go:183] — so a binary or image path is never highlighted. Image files are instead routed to the separate asynchronous image path `load_all_images` [kittens/diff/ui.go:190]: it checks `is_image` [ui.go:192], calls `image_collection.AddPaths(...)` for each side [ui.go:193, ui.go:197], and finally `image_collection.LoadAll()` [ui.go:206], which loads/resizes and triggers a re‑render — exactly the observed `Loading image...` → graphics‑protocol sequence.

**Sibling variants (enumerated).**
- **Plain text** → line diff (both sides pass `is_path_text`).
- **Binary** (`blob.bin`, unknown MIME → `application/octet-stream`) → `Binary file: 256 B`, no line diff.
- **Non‑UTF‑8** (`invalid.dat`, fails `utf8.ValidString`) → `Binary file: 6 B`, no line diff.
- **Image** (`pic.png`, MIME `image/*`) → dimensions + async load via `load_all_images` [ui.go:190‑206].
- **`/dev/null`** → treated as non‑text by the explicit `os.SameFile` check [collect.go:91‑96]. **[Inferred]** — not exercised at runtime; read from the `is_path_text` body.

---

## Q7 — End‑to‑end runtime trace: dirs → changes/renames/adds/removals fully understood

**Observed (the entry‑and‑validation edges).** The two pre‑loop validations print verbatim to stderr from the real binary:

```text
$ ./kitty/launcher/kitten diff /tmp/.../left/changed.txt
Error: You must specify exactly two files/directories to compare

$ ./kitty/launcher/kitten diff /tmp/.../left /tmp/.../right/changed.txt
Error: The items to be diffed should both be either directories or files. Comparing a directory to a file is not valid.'
```

And the async, progressive rendering is directly visible in a **frame‑by‑frame** capture (a PTY harness with a non‑zero winsize timestamps each read from process start and names which render landed in that frame). **A caveat stated up front:** at 1000 changed `.go` files the run *never completed* in the current environment — **50/50** default‑engine attempts (and 20/20 built‑in) crashed with `fatal error: concurrent map writes` **before** the unified‑diff frame was ever reached — so a 1000‑file frame timing is **not reproducible here and is not reported**. The two‑phase ordering is instead shown on a **250‑file tree**, whose diff is slow enough to land as a distinct frame; that fixture *also* crashes on the large majority of attempts (**39/40** in a fresh batch, see Q5), so the capture below is again a **rare surviving (non‑crashing) run**, explicitly selected because it did not crash. On that surviving run the collection banner is painted almost immediately (≈21 ms) and the unified‑diff content lands ≈0.6 s later, only after the parallel per‑file diffing completes:

```text
# SURVIVING (non-crashing) run; in a fresh batch 39/40 attempts crashed before this frame (see Q5)
$ python3 harness.py frames -- ./kitty/launcher/kitten diff -o diff_cmd=builtin /tmp/.../scale250/left /tmp/.../scale250/right
elapsed_ms  marker
     21.0    Calculating-diff-please-wait   (collection banner — painted immediately)
    615.6    unified-hunk(@@)               (diff content — after parallel per-file diffing)
```

The banner‑first / diff‑later **ordering** — not the absolute milliseconds — is the point: the collection banner is drawn before diffing finishes, and the diff frame arrives only once the async worker pool has produced the per‑file patches.

The image branch is the **same** async mechanism one step later: the `Loading image...` placeholder is drawn on an early frame and the actual in‑terminal image (Kitty graphics‑protocol data) on a subsequent re‑render — the two placeholders and the 15 graphics‑protocol escapes were captured verbatim in **Q6** above. (On the tiny mixed fixture the image loads within the same ≈20 ms burst, so diff/placeholder/graphics coalesce into a single logged frame; the 250‑file tree above separates the collection and diff frames cleanly on the runs that survive.)

**Cause→effect (the pipeline).**
1. **`main`** [kittens/diff/main.go:102]: `load_config` [main.go:104] → require exactly two args, else `return 1, fmt.Errorf("You must specify exactly two files/directories to compare")` [main.go:108‑109] → `set_diff_command(conf.Diff_cmd)` [main.go:111] → `init_caches()` [main.go:114] → `create_formatters()` [kittens/diff/render.go:164], called at [main.go:115] → optional remote `ssh:` fetch via `get_remote_file` [main.go:121, main.go:125].

2. **Reject directory‑vs‑file** when `isdir(left) != isdir(right)` [main.go:129]. **REPORT‑VERBATIM #2 — the error string ends with a stray `.'` (period + apostrophe), reported exactly as observed:** the message at [main.go:130] is literally

   ```text
   The items to be diffed should both be either directories or files. Comparing a directory to a file is not valid.'
   ```

   Note the trailing `.'`. This was captured verbatim from the real `kitten diff <dir> <file>` invocation above and is **not** silently cleaned up.

3. **`Handler.initialize`** [kittens/diff/ui.go:114]: sets `self.async_results = make(chan AsyncResult, 32)` [ui.go:132], launches a **goroutine** [ui.go:133] that runs `create_collection(self.left, self.right)` [ui.go:135] and then `self.lp.WakeupMainThread()` [ui.go:137]; the main thread meanwhile does an initial `self.draw_screen()` [ui.go:139] (the `Calculating diff, please wait...` banner). `create_collection` [collect.go:371] runs `collect_files` (Q1/Q2) for directories.

4. **`handle_async_result`** [kittens/diff/ui.go:245] dispatches by result type:
   - `case COLLECTION` [ui.go:247] → fires `self.generate_diff()` [ui.go:249] **+** `self.highlight_all()` [ui.go:250] **+** `self.load_all_images()` [ui.go:251], concurrently.
   - `case DIFF` [ui.go:252] → `self.render_diff()` [ui.go:256] then `self.draw_screen()` [ui.go:268].
   - `case IMAGE_RESIZE` [ui.go:269] → `self.rerender_diff()` [ui.go:271].
   - `case IMAGE_LOAD, HIGHLIGHT` [ui.go:272] → `self.rerender_diff()` [ui.go:273].
   - `on_wakeup` drains the channel [ui.go:161‑177]; the loop finally runs at `err = lp.Run()` [main.go:163].

**Causal point.** The screen renders first from the **DIFF** result and then **re‑renders** as **HIGHLIGHT** and **IMAGE** results land later — precisely the observed `Calculating diff…` → diff → `Loading image…` → image sequence. Once `collect_files` has classified every path (change / rename / add / removal / mode‑only), the changes, renames, additions, and removals are "fully understood," and the later async results only refine how they are *drawn* (syntax colors, images), not *what* they are.

---

## Q8 — Matching regions + cache synergy: the "fast" part

**Observed.** Each `diff_cmd` engine was run through the real `kitten diff` (driven in a PTY, ANSI‑stripped, then filtered to the changed‑file hunk region). The change fixture edits `line two` and appends `line five`; all three engines produce the identical unified‑diff chunk header `@@ -1,4 +1,5 @@` and the same side‑by‑side line pairing (each side's line number precedes its text).

Built‑in engine (`diff_cmd=builtin`):

```text
$ ./kitty/launcher/kitten diff -o diff_cmd=builtin /tmp/.../q8l /tmp/.../q8r
   changed.txt
   @@ -1,4 +1,5 @@
1  line one
1  line one
2  line two
2  line two CHANGED
3  line three
3  line three
```

External `diff` engine (`diff_cmd=diff`):

```text
$ ./kitty/launcher/kitten diff -o diff_cmd=diff /tmp/.../q8l /tmp/.../q8r
   changed.txt
   @@ -1,4 +1,5 @@
1  line one
1  line one
2  line two
2  line two CHANGED
3  line three
3  line three
```

External `git` engine (`diff_cmd=git`):

```text
$ ./kitty/launcher/kitten diff -o diff_cmd=git /tmp/.../q8l /tmp/.../q8r
   changed.txt
   @@ -1,4 +1,5 @@
1  line one
1  line one
2  line two
2  line two CHANGED
3  line three
3  line three
```

The default (`auto`, run with no `-o` override) reproduces the same hunk verbatim — on this machine `auto` selects `git` (both `/usr/bin/git` and `/usr/bin/diff` are present, and `find_differ` tries git first).

**Cause→effect (the anchored / "patience" diff).** The **built‑in** engine — dispatched when `diff_cmd` is `builtin` or empty [kittens/diff/patch.go:48‑49] (the built‑in `Diff(...)` is called when `len(diff_cmd) == 0` [patch.go:294, patch.go:303]) — is an **anchored diff** copied from the Go standard library. (The **default config is `auto`, not built‑in**; `auto` selects an external git/diff engine via `find_differ()` [kittens/diff/main.py:41, patch.go:44‑47] — see *Dispatch* below, where on this machine `auto` resolves to git.) The built‑in engine's header reads `// Copied from the Go stdlib, with modifications.` [kittens/diff/diff.go:1] with the upstream URL [diff.go:2]. Its doc comment [diff.go:21‑48] explains the algorithm: rather than minimizing all inserted/removed lines (worst‑case quadratic), it minimizes the number of **"unique" lines** — lines that appear exactly once in *both* old and new — and uses those unique lines to **anchor** the matching regions [diff.go:32‑35]. It "guarantees to run in O(n log n) time" [diff.go:39] "instead of the standard O(n²) time" [diff.go:40], and the comment notes some systems call this a "patience diff" [diff.go:42] (a name it explicitly avoids). This matches the upstream Go `internal/diff` package the file is copied from (validated against upstream documentation).

- `func Diff(oldName, old, newName, new string, num_of_context_lines int) []byte` [diff.go:49] returns `nil` when the inputs are identical, `if old == new { return nil }` [diff.go:50‑51].
- `func lines(x string)` [diff.go:172] splits with `strings.SplitAfter(x, "\n")` [diff.go:173], and appends the BSD/GNU marker `"\n\\ No newline at end of file\n"` when the input lacks a trailing newline [diff.go:179].
- `func tgs(x, y []string)` [diff.go:192] selects the longest common subsequence of lines unique in both sides — **Szymanski's Algorithm A**, referenced at [diff.go:189‑191] (Princeton TR #170, `https://research.swtch.com/tgs170.pdf`); the comment names "Algorithm A" at [diff.go:229].
- Output is emitted as unified‑diff chunk headers `@@ -%d,%d +%d,%d @@` via `fmt.Fprintf(&out, "@@ -%d,%d +%d,%d @@\n", …)` [diff.go:142] — exactly the observed `@@ -1,4 +1,5 @@`.

**Dispatch and the external engines (all variants enumerated).** `run_diff` [kittens/diff/patch.go:282] first resolves symlinks (because `git diff` does not follow them) with `filepath.EvalSymlinks` [patch.go:286, patch.go:290], then dispatches: when `len(diff_cmd) == 0` [patch.go:294] it calls the built‑in `Diff(...)` [patch.go:303]; otherwise it shells out to the external command. The command is chosen by `set_diff_command(q)` [kittens/diff/patch.go:44]:

| `diff_cmd` value | Branch | Resulting command |
|---|---|---|
| `auto` | `find_differ()` [patch.go:47] | git if available, else diff, else built‑in [patch.go:34‑42] |
| `builtin` or `""` | empty `diff_cmd` [patch.go:48‑49] | **built‑in** anchored diff |
| `diff` | `shlex.Split(DIFF_DIFF)` [patch.go:50‑51] | `diff -p -U _CONTEXT_ --` [patch.go:22] |
| `git` | `shlex.Split(GIT_DIFF)` [patch.go:52‑53] | `git diff --no-color --no-ext-diff --exit-code -U_CONTEXT_ --no-index --` [patch.go:21] |
| anything else | `shlex.Split(q)` [patch.go:54‑59] | custom command |

Both external templates are defined as constants: `GIT_DIFF` [patch.go:21] and `DIFF_DIFF` [patch.go:22]. The config default is `diff_cmd` = `'auto'` [kittens/diff/main.py:41]; on this machine `auto` selects **git** (git 2.51.0 present; `find_differ` tries git before diff [patch.go:34‑38]), which is why the default run's hunks come from git — and all three engines produced the identical `@@ -1,4 +1,5 @@`.

**Cache‑synergy point (the "quietly keeps everything fast" link).** The diff engine and the rename‑hash both read file bytes through the single `data_cache` populated by `data_for_path` [collect.go:65], so the algorithm's data reads are served once per path from cache while it anchors on unique lines — the caches (Q3) and the anchored algorithm together are what make repeated, multi‑file comparisons feel fast.


---

## Coverage pass

Each row lists the mechanism, its exact literal + `file:line`, and the section holding the observed evidence.

- [x] **Q1 — pairing.** `collect_files` [collect.go:296]; walk left/right [collect.go:299, collect.go:303]; `filepath.Rel(base, path)` [collect.go:283]; `Intersect` [collect.go:306]; change `if ld != rd` [collect.go:317] → `add_change` [collect.go:319]; mode‑only comment [collect.go:324] + `lstat.Mode() != rstat.Mode()` [collect.go:323] → [collect.go:326]; ignore‑walk test `TestDiffCollectWalk` [collect_test.go:19, collect_test.go:42, collect_test.go:45, collect_test.go:51]. Evidence: **Q1** (`Calculating diff…`, `@@`, `Mode changed:`, `--- PASS: TestDiffCollectWalk`).
- [x] **Q2 — rename.** `removed`/`added` [collect.go:332, collect.go:333]; hash loops [collect.go:335‑340, collect.go:341‑346]; `hash_for_path` [collect.go:106] (`crypto/md5` [collect.go:6]); `ah == rh` [collect.go:350]; `ld == rd` [collect.go:353]; `add_rename` [collect.go:354]; `added.Discard(n)` [collect.go:355]; `add_removal` [collect.go:362]; `add_add` [collect.go:366]. Evidence: **Q2** (`original_name.txt renamed_name.txt` paired vs. solo `added.txt`/`removed.txt`).
- [x] **Q3 — caches.** All SEVEN named with types [collect.go:20‑24]; `const sz = 4096` [collect.go:29]; `init_caches` [collect.go:26‑37]; chain `data_for_path` [collect.go:65] → `lines_for_path` [collect.go:138] → `highlighted_lines_for_path` [collect.go:148, collect.go:153]; `GetOrCreate` [cache.go:39‑58]; **REPORT‑VERBATIM #1** `Set` RLock/no‑evict [cache.go:32‑37], `MustGetOrCreate` [cache.go:60‑72], `Get` [cache.go:25‑30]. Evidence: **Q3** + **Q5** (real consequence).
- [x] **Q4 — concurrency.** `Parallel` [images/utils.go:27]; `NumberOfThreads` [utils.go:22]; `runtime.NumCPU()` [utils.go:34‑36]; cap [utils.go:37‑39]; buffered chan + fill + close [utils.go:41‑45]; WaitGroup [utils.go:47, utils.go:55]; `diff()` pool [patch.go:352‑374]. **Observed worker ceiling `runtime.NumCPU()`=128 (scratch program, stable ×2); 250‑file scale SETTLE≈1.21–1.35 s taken from SURVIVING (non‑crashing) runs only — 20/20 and 39/40 attempts crashed first (see Q5).**
- [x] **Q5 — highlight isolation.** `highlight_all` [highlight.go:217]; `ctx.Parallel` [highlight.go:219]; distinct `path := paths[i]` [highlight.go:221]; local `strings.Builder` [highlight.go:210, highlight.go:211]; single `Set` [highlight.go:224]; chroma `v2.14.0` [go.mod:7], regexp2 `v1.11.0` [go.mod:9], `ansi_formatter` [highlight.go:84‑86]. **Observed:** single‑`.py` digest reliably reproducible (5/5 runs, 0 crashes); 250‑file digest identical only across the RARE surviving runs (`88256…`, SGR 416) while 20/20 & 39/40 attempts crashed; real crash `fatal error: concurrent map writes` at [cache.go:34] via [highlight.go:224] + [utils.go:50‑52]; deterministic `-race` corroboration ×3.
- [x] **Q6 — binary/image.** `mimetype_for_path` [collect.go:50]; `GuessMimeTypeWithFileSystemAccess` [mimetypes.go:100] (`inode/directory` [mimetypes.go:110]; `GuessMimeType` [mimetypes.go:70]); default `application/octet-stream` [collect.go:54]; `is_image` [collect.go:82‑83]; `is_path_text` [collect.go:86‑102] (image [collect.go:88‑90], `/dev/null` [collect.go:91‑96], `utf8.ValidString` [collect.go:102]); diff gate [ui.go:147]; image path [ui.go:190‑206]. Evidence: **Q6** (`Binary file: 256 B`, `Binary file: 6 B`, `Dimensions: 4x4`, `Loading image...`, graphics APCs).
- [x] **Q7 — trace.** `main` [main.go:102]; 2‑arg error [main.go:108‑109]; dir‑vs‑file **REPORT‑VERBATIM #2** [main.go:130]; `initialize` [ui.go:114, ui.go:132‑137]; `handle_async_result` [ui.go:245]; COLLECTION [ui.go:247‑251]; DIFF [ui.go:252, ui.go:256, ui.go:268]; IMAGE_RESIZE [ui.go:269‑271]; IMAGE_LOAD/HIGHLIGHT [ui.go:272‑273]; `lp.Run()` [main.go:163]. Evidence: **Q7** (both error strings verbatim; banner→diff ordering on a SURVIVING 250‑file run — banner ≈21 ms, `@@` ≈616 ms — since the 1000‑file frame timing was NOT reproducible, 50/50 default & 20/20 built‑in attempts crashed first; `Calculating…`→diff→`Loading image…`).
- [x] **Q8 — diff + cache.** Header [diff.go:1‑2]; doc comment [diff.go:21‑48] (anchored [diff.go:35], O(n log n) [diff.go:39], O(n²) [diff.go:40], "patience" [diff.go:42]); `Diff` [diff.go:49‑51]; `lines` [diff.go:172‑179]; `tgs` [diff.go:192] (Szymanski [diff.go:189‑191]); `@@` [diff.go:142]; `run_diff` [patch.go:282] (EvalSymlinks [patch.go:286, patch.go:290], builtin dispatch [patch.go:294, patch.go:303]); `GIT_DIFF` [patch.go:21]; `DIFF_DIFF` [patch.go:22]; `set_diff_command` all branches [patch.go:44‑59]; `diff_cmd` default `auto` [main.py:41]; `data_cache` synergy [collect.go:65]. Evidence: **Q8** (`@@ -1,4 +1,5 @@` for builtin/diff/git).
- [x] **Both REPORT‑VERBATIM nuances present and NOT "corrected":** #1 `Set` RLock/no‑evict [cache.go:32‑37] (with its observed concurrent‑map‑write consequence in Q5); #2 dir‑vs‑file error ending `valid.'` [main.go:130], captured verbatim from the real binary.
- [x] **Build banner + exact commands recorded** (Investigation + Appendix); **`git status --porcelain` empty before & after** (Appendix); **all temp artifacts removed** (Appendix); **`main.py` noted as a legacy stub, not the runtime path** (Orientation).

---

## Appendix — build/version banners, repository integrity, cleanup

**Exact binary invoked:** `kitty/launcher/kitten` (canonical `python setup.py build` output; 15,962,372 bytes).

**Version banner (verbatim):**

```text
$ ./kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
```

**Repository baseline — before investigation:**

```text
$ git rev-parse --abbrev-ref HEAD; git rev-parse --short HEAD; git rev-parse HEAD
blitzy-5900e980-186b-42ac-b10f-47549fbb3c92
e3ae9a081
e3ae9a081ed071a46d494bfe60ae264ce4477337
$ git status --porcelain
```

(The `git status --porcelain` output above is empty — a clean tree. The working tree is at commit `e3ae9a081`, full `e3ae9a081ed071a46d494bfe60ae264ce4477337` — the same revision the canonical build stamps as `kitty.VCSRevision` (see the build section), and therefore the build/investigation revision for every observation in this document. That revision sits two documentation commits above `815df1e21`, which is the **base commit of the source branch** `kitty_815df1e210e0` after which this file is named. In other words, `815df1e21` is the source‑branch base the filename encodes, **not** the revision that produced these observations; the revision that did is `e3ae9a081`.)

**Repository integrity — after investigation:**

```text
$ git status --porcelain
```

(Empty again — clean.) All temporary observation artifacts lived **outside** the repository tree under `/tmp` — the fixture directories (`/tmp/kitty_obs/left`+`/right`, the focused single‑file pair `/tmp/kitty_obs/q8l`+`/q8r`, the highlightable pair `/tmp/.../hl`, and the scale sets `/tmp/.../scale250` [250 files/side] and `/tmp/.../scale1000` [1000 files/side] — the highlight and scale fixtures were freshly regenerated under `/tmp` to re‑verify the Q4/Q5/Q7 timing, digest, and frame evidence at the current build), the PTY harness scripts (`/tmp/.../harness.py`, `/tmp/.../rawcap.py`), the raw captures (`/tmp/.../caps/`), the pristine pre‑generation tree (`/tmp/kitty_pregen`), the `NumCPU` scratch program (`/tmp/ncpu`), and the Go race‑detector scratch module (`/tmp/kitty_race`, `module kittyrace` with `replace kitty => <repo>`) — and were all removed with `rm -rf` after the observations were captured. (The Go race‑detector confirmation shown in **Q5** was produced by that `/tmp/kitty_race` scratch module under `go run -race`, as labeled non‑canonical there; the diff package's own `go test -race ./kittens/diff/` passes but does not drive `highlight_all`. Either way, `-race` writes only to Go's build cache, never the repository.) The generated build products (`data_generated.bin`, the root `constants_generated.go`, and the generated Go files `kittens/diff/conf_generated.go` and `kittens/diff/cli_generated.go`) are git‑ignored/untracked, so they never appear in `git status --porcelain`.

**Read‑only compliance.** No existing repository file was modified. The only file added by this task is this document, `blitzy/documentation/kitty_815df1e210e0.md`. Temporary scripts and fixtures were confined to `/tmp` and cleaned up, leaving the repository unchanged.

**External reference.** The only non‑repository reference consulted was the upstream Go `internal/diff` package documentation, used solely to corroborate the "anchored / patience diff" terminology and the O(n log n) guarantee that `kittens/diff/diff.go` is copied from; the primary evidence remains the in‑file comment [diff.go:21‑48] and the observed `@@` output.

