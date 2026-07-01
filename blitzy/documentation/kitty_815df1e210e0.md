# How the kitty `diff` kitten works at runtime

> Repository `kovidgoyal/kitty` · branch `kitty_815df1e210e0` · HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`

This document answers, from **built-and-run observation**, how the kitty `diff` kitten behaves when it
compares files and directories: how it decides *what belongs together* when comparing directories; how it
sometimes recognizes a **rename** instead of a delete + a new file (the part that feels "almost magical");
how caching works from raw file contents all the way to highlighted output and how it stays efficient; what
happens when multiple files are processed at once; how syntax highlighting runs in parallel *without stepping
on itself*; what changes when binary files or images show up alongside plain text; and — end to end — what
actually happens at runtime from the moment two directories are compared through to the point where changes,
renames, additions, and removals are fully understood, including how the diff algorithm finds matching regions
and how the cache "quietly keeps everything fast."

Every behavioral claim below is paired with a **verbatim observed output line** from a real run and an exact
`file:line` citation into the source. Where a value is deterministic (a constant, a config default, a diff
command string), it is quoted exactly. Where a value depends on the specific test fixture (for example, an MD5
hash of a file's bytes), that is called out explicitly so the *mechanism* is never confused with the
*fixture-specific literal*.

---

## Methodology — how the evidence was gathered

The governing rule for this task is *investigate by running first, then write*. All evidence here was captured
by **building and running the real code paths**, not by reading alone.

**Harness strategy.** The `diff` kitten is a Go package at `kittens/diff/` that imports the shared kitty Go
libraries under `kitty/tools/...`. To exercise the real code without a full interactive TUI, two scratch Go
modules were created **outside the repository** under `/tmp`:

- `/tmp/obs` — module `obs`, containing a **byte-identical copy of the entire `kittens/diff` Go package** and a
  `go.mod` that points back at the real repository so that the shared libraries (`kitty/tools/utils`,
  `kitty/tools/utils/images`, the real MIME tables, etc.) are compiled from the actual source:

  ```
  module obs
  go 1.22
  require kitty v0.0.0
  replace kitty => /tmp/blitzy/kitty/blitzy-e22ef2ce-f046-451f-b141-d3c834ec0465_cb4028
  ```

  Because of the `replace` directive, everything except the copied `kittens/diff/*.go` files is the **real,
  unmodified** shared code. The copies were verified byte-identical with `diff -q` against the originals:

  ```
  IDENTICAL  collect.go
  IDENTICAL  diff.go
  IDENTICAL  patch.go
  IDENTICAL  highlight.go
  IDENTICAL  ui.go
  IDENTICAL  render.go
  IDENTICAL  main.go
  IDENTICAL  mouse.go
  IDENTICAL  search.go
  IDENTICAL  collect_test.go
  IDENTICAL  cli_generated.go
  IDENTICAL  conf_generated.go
  ```

- `/tmp/racetest` — module `racetest`, containing a copy of `tools/utils/cache.go` that is **identical except
  its `package` clause** (needed to run the concurrency race characterization in isolation, because that test
  intentionally crashes the process). Byte-identity was verified as `IDENTICAL_EXCEPT_PACKAGE`, and the map
  write remained at line 34:

  ```
  34	self.data[key] = val
  ```

**Fidelity check.** To prove the harness faithfully exercises the real code, the upstream Go test
`TestDiffCollectWalk` (`kittens/diff/collect_test.go`) was run **byte-identical** and passed:

```
=== RUN   TestDiffCollectWalk
--- PASS: TestDiffCollectWalk (0.00s)
PASS
ok  	obs/diff	0.019s
```

**Environment facts** (this machine, relevant to `diff_cmd auto` and worker counts):

- `go version go1.22.12 linux/amd64` (`go.mod` requires `go 1.22`).
- `runtime.NumCPU()` returns **`128`** (observed directly from a run — see O5). Note: the shell's `nproc`
  reported `4`, which reflects a cgroup CPU quota, not the logical-CPU count that Go's `runtime.NumCPU()`
  actually returns; the reported worker magnitudes below use the genuine `128` observed from Go.
- `git version 2.51.0` and `diff (GNU diffutils) 3.10`, both present at `/usr/bin` (so `diff_cmd auto` finds
  `git`).

**Read-only guarantee.** The source tree was never modified. Every harness lived under `/tmp`; after authoring,
all scratch artifacts are removed and `git status --porcelain` lists only this document.

**A note on fixture-dependent values.** One value in the pre-supplied investigation notes — an MD5 hash of the
renamed file — depends entirely on the fixture's byte content. The runs reproduced here used a fixture whose
moved file contained the bytes `moved content\n`, which hashes to `4aa504e85be1af675a7157d6bdfafb56`. That
exact literal is reported below as observed. What matters for the *rename mechanism* is not any particular hash
string but that the removed file and the added file hash to the **same** value; that equality is what triggers
rename detection, and it is shown verbatim in O2.

**The fixture.** Most observations use one `LEFT`/`RIGHT` directory pair built by the harness, containing an
identical file (`same.txt`), a content-changed file (`changed.txt`: `version one\n` → `version two\n`), a
mode-only change (`modeonly.sh`, `0644` → `0755`), a moved-but-identical file (`old_name.txt` →
`brand_new_name.txt`, both `moved content\n`), an added file (`added.txt`), a removed file (`removed.txt`), a
non-UTF-8 binary (`blob.bin`), an image (`pic.png`), a nested text file (`sub/nested.txt`), and two files that
match `ignore_name` globs (`editor.bak~`, `.git/config`).

---

## O1 — Directory pairing: "what belongs together"

**Answer.** When two directories are compared, each side is traversed by `walk()` and every file is keyed by
its path **relative to the walked base** (`filepath.Rel(base, path)`), then added into a `utils.Set[string]`.
The two relative-name sets are combined with **set intersection** — `common_names := left_names.Intersect(right_names)`
— and that intersection defines the candidate "same file" pairs. Files whose filename matches an `ignore_name`
glob are dropped during the walk, before pairing.

**Observed output** (`go test ./diff/ -run TestObsWalkPairing -v`):

```
O1 walk(LEFT) relative-name keys = [blob.bin changed.txt modeonly.sh old_name.txt pic.png removed.txt same.txt sub/nested.txt]
O1 conf.Ignore_name=[.git *~ *.pyc] ; 'editor.bak~' kept? false ; '.git/config' kept? false
O1 pmap['sub/nested.txt'] = /tmp/TestObsWalkPairing2762104520/001/LEFT/sub/nested.txt
```

**Citations.**

- `walk()` traverses one tree — `kittens/diff/collect.go:260-294`; the relative key is computed at
  `collect.go:283` (`filepath.Rel(base, path)`) and inserted at `collect.go:289` (`names.Add(...)`).
- `collect_files()` builds and pairs both sides — `kittens/diff/collect.go:296`; the intersection is
  `collect.go:306`:

  ```go
  common_names := left_names.Intersect(right_names)
  ```

- The ignore filter is `allowed()` using `filepath.Match` on the filename only — `kittens/diff/collect.go:230-238`.
- The harness reuses the upstream test template `kittens/diff/collect_test.go:19-54`.

**Rationale.** Keys are **relative**, not absolute: note that the nested file retains its relative sub-path
`sub/nested.txt` in the observed key list. That is exactly what lets a file at `LEFT/sub/nested.txt` pair with
`RIGHT/sub/nested.txt` regardless of the absolute directory prefix — the intersection of relative names is the
definition of "belongs together." The second observed line shows the ignore globs (`.git`, `*~`, `*.pyc`) doing
their job *before* pairing: `editor.bak~` (matches `*~`) and `.git` (matches `.git`) both report `kept? false`,
so they never enter the name sets. The third line confirms `walk` also records each relative name's absolute
path in a side map, which every later cache uses as its key (see O3/O4).

**Independent confirmation.** The byte-identical upstream `TestDiffCollectWalk` (shown in *Methodology*) passes,
and it pins exactly this behavior: relative-name keying plus `ignore_name`-style skipping. Its assertions live
at `kittens/diff/collect_test.go:19-54`.

> **Note on the `ignore_name` default.** The *compiled* default for `conf.Ignore_name` is empty
> (`conf_generated.go`); the values `.git`, `*~`, and `*.pyc` are the **documented examples** at
> `kittens/diff/main.py:63-65`. The harness set `conf.Ignore_name = [".git", "*~", "*.pyc"]` explicitly to
> demonstrate the ignore mechanism, which is why those three appear in the observed output.

---

## O2 — Rename detection: recognizing a rename instead of a delete + a new file

*(This is the user's central "almost magical" example, answered directly and by name.)*

**Answer.** After pairing (O1), the names present **only on the left** are `removed := left_names.Subtract(common_names)`
and the names present **only on the right** are `added := right_names.Subtract(common_names)`. The kitten then
MD5-hashes every member of both leftover sets (via `hash_for_path` → `md5.Sum` over the read-once file bytes)
and **cross-matches** them: for each removed file it searches the added files for one with an **equal hash**
(`if ah == rh`), and — crucially — only then re-reads and compares the **full contents** (`if ld == rd`) before
reclassifying the pair as a single `rename` (`self.add_rename(...)`) and discarding that added entry
(`added.Discard(n)`). A removed file with no content-equal partner falls through to `add_removal`; any added
file left over becomes `add_add`. There is **no path/name heuristic** involved — the match is purely by content.

**Observed output** — classification (`go test ./diff/ -run TestObsCreateCollectionClassify -v`):

```
  rename   old_name.txt -> brand_new_name.txt
O2 RENAME detected: "old_name.txt" -> "brand_new_name.txt" (reclassified from removal+add)
```

**Observed output** — the content-hash cross-match (`go test ./diff/ -run TestObsRenameHashCrossMatch -v`):

```
O2 md5(old_name.txt)=4aa504e85be1af675a7157d6bdfafb56
O2 md5(brand_new_name.txt)=4aa504e85be1af675a7157d6bdfafb56
O2 hash_for_path equal? true -> triggers add_rename after full-content equality check (contents equal? true)
```

**Citations.**

- Leftover-set construction — `removed := left_names.Subtract(common_names)` at `kittens/diff/collect.go:332`
  and `added := right_names.Subtract(common_names)` at `collect.go:333`.
- The MD5 cross-match loop — `kittens/diff/collect.go:347-364`:

  ```go
  for name, rh := range rhash {          // L347
      found := false
      for n, ah := range ahash {
          if ah == rh {                  // L350  hashes equal?
              ld, _ := data_for_path(left_path_map[name])
              rd, _ := data_for_path(right_path_map[n])
              if ld == rd {              // L353  full-content equality guard
                  self.add_rename(left_path_map[name], right_path_map[n])  // L354
                  added.Discard(n)       // L355  consume the added entry
                  found = true
                  break
              }
          }
      }
      if !found {
          self.add_removal(left_path_map[name])   // L362  no partner -> removal
      }
  }
  ```

- Hashing itself — `hash_for_path` at `kittens/diff/collect.go:106-116` (`md5.Sum(...)` at `collect.go:112`).
- Reclassification bookkeeping — `add_rename` sets `type_map[left] = "rename"` at `kittens/diff/collect.go:175-179`.

**Rationale — dispelling the "magic."** The effect that feels magical is nothing more than **content-hash
matching across the removed/added sets, verified by a full byte-for-byte equality check**. In the run, the
moved file `old_name.txt` and its new location `brand_new_name.txt` both hash to the **same** value
(`4aa504e85be1af675a7157d6bdfafb56`), so the hashes are `equal? true`; the code then re-reads both files and
confirms `contents equal? true`; only then does `add_rename` fire, reclassifying what would otherwise have been
a separate `removal` + `add` into one `rename`. The full-content check at `collect.go:353` is deliberate
defense against an MD5 collision — hash equality alone is not trusted.

> **Fixture-dependent literal, reported honestly.** The specific hash `4aa504e85be1af675a7157d6bdfafb56` is a
> function of the fixture's file bytes (`moved content\n`). A different moved file would produce a different
> hash string. The invariant that actually drives rename detection — *the removed file and the added file share
> the same content hash* — is what the middle observed line demonstrates (`equal? true`), and that is
> independent of any particular fixture.

---

## O3 — Caching layers: "from raw file contents to highlighted output"

**Answer.** `init_caches()` creates **seven** `utils.LRUCache` stores, each with fixed capacity `const sz = 4096`,
all keyed by absolute path. In source order they are: `size_cache`, `mimetypes_cache`, `data_cache`,
`is_text_cache`, `lines_cache`, `highlighted_lines_cache`, and `hash_cache`.

**Citations.** `init_caches()` — `kittens/diff/collect.go:26-37`, capacity `const sz = 4096` at `collect.go:29`.
The seven declarations, by exact identifier and line:

```go
const sz = 4096                                                   // collect.go:29
size_cache = utils.NewLRUCache[string, int64](sz)                 // collect.go:30
mimetypes_cache = utils.NewLRUCache[string, string](sz)           // collect.go:31
data_cache = utils.NewLRUCache[string, string](sz)                // collect.go:32
is_text_cache = utils.NewLRUCache[string, bool](sz)               // collect.go:33
lines_cache = utils.NewLRUCache[string, []string](sz)             // collect.go:34
highlighted_lines_cache = utils.NewLRUCache[string, []string](sz) // collect.go:35
hash_cache = utils.NewLRUCache[string, string](sz)                // collect.go:36
```

**Observed output** — which caches are populated after one `create_collection`, probed per file classification
(`go test ./diff/ -run TestObsCachesByType -v`):

```
O3 [changed.txt(diff)] data=true hash=false mimetypes=false is_text=false lines=false size=false highlighted=false
O3 [removed.txt(removal)] data=true hash=true mimetypes=true is_text=true lines=true size=false highlighted=false
O3 [added.txt(add)] data=true hash=true mimetypes=true is_text=true lines=true size=false highlighted=false
```

**Rationale.** The seven caches form a pipeline from raw bytes to rendered lines: `data_cache` holds the raw
file text; from it derive `hash_cache` (the MD5 used for rename detection), `mimetypes_cache` and
`is_text_cache` (classification), `lines_cache` (the split lines), and finally `highlighted_lines_cache`
(Chroma-highlighted output). The observed reality is nuanced and reported exactly as seen: a plain **changed**
(`diff`) file only populates `data_cache` during collection (its hash is not needed — it is not a rename
candidate), whereas **removed** and **added** files, which *are* run through the rename cross-match, populate
five caches (`data`, `hash`, `mimetypes`, `is_text`, `lines`). Two caches — `size_cache` and
`highlighted_lines_cache` — remain **lazy** after collection; they are filled on demand later by the render and
highlight paths (see O4 for `size_cache` and O10 for `highlighted_lines_cache`).

---

## O4 — Cache efficiency: "stay efficient"

**Answer.** Three mechanisms keep the caches efficient: **(a)** fixed-capacity LRU eviction (each store is
bounded at `4096` entries), **(b)** read-once file loads (`data_for_path` reads each file from disk exactly
once), and **(c)** derived-value memoization (hash, is-text, lines, size, highlight all reuse the cached
bytes). `data_for_path` wraps `data_cache.GetOrCreate(path, …os.ReadFile…)`, and `GetOrCreate` takes an
**exclusive** `Lock()` to write, pushes the key to an LRU list, and evicts the least-recently-used entry once
the list exceeds `max_size`.

**Citations.**

- `data_for_path` reads each file once — `kittens/diff/collect.go:65-70`:

  ```go
  func data_for_path(path string) (string, error) {
      return data_cache.GetOrCreate(path, func(path string) (string, error) {
          ans, err := os.ReadFile(path)
          return utils.UnsafeBytesToString(ans), err
      })
  }
  ```

- `LRUCache.GetOrCreate` — `tools/utils/cache.go:39-58`; exclusive `self.lock.Lock()` at `cache.go:48`;
  eviction at `cache.go:51-53`:

  ```go
  self.lock.Lock()                                           // cache.go:48
  self.data[key] = ans
  self.lru.PushFront(key)
  if self.max_size > 0 && self.lru.Len() > self.max_size {   // cache.go:51
      k := self.lru.Remove(self.lru.Back())
      delete(self.data, k.(K))
  }
  self.lock.Unlock()
  ```

- Capacity `4096` — `kittens/diff/collect.go:29`.

**Observed output** (read-once + eviction, `go test ./diff/ -run TestObsLRUReadOnceAndEviction -v`;
cached-after-delete and lazy `size_cache`, `-run TestObsCaches`):

```
O4 create() invoked 1 time(s) for 3 GetOrCreate calls on same key (read-once)
O4 capacity=3 after inserting keys 0..4 -> key0 present=false key1 present=false key4 present=true (LRU evicts oldest)
O4/O10 after deleting changed.txt from disk, data_for_path still returns cached "version one\n" (err=<nil>)
O4 size_cache(added.txt) found before size_for_path=false, after=true
```

**Rationale.** The first line proves **read-once**: three `GetOrCreate` calls on the same key invoked the
loader exactly `1 time(s)`. The second proves **bounded memory**: with capacity `3`, inserting keys `0..4`
evicts the oldest (`key0`/`key1` gone, `key4` resident) — the `4096` bound in the real caches caps memory the
same way while keeping hot files resident. The third is the decisive efficiency proof: after the file was
**deleted from disk**, `data_for_path` still returned its bytes `"version one\n"` with `err=<nil>` — the disk
read happened once, and every downstream computation is served from that in-memory copy. The fourth shows the
**lazy** `size_cache`: not present before `size_for_path` is called, present after — derived values are
memoized on first use, not eagerly.

---

## O5 — Concurrency: "when multiple files are being processed at once"

**Answer.** Two compute-heavy phases each spin up a worker pool through the shared helper
`images.Context{}.Parallel`: **diffing** (`diff()` in `patch.go`) and **highlighting** (`highlight_all()` in
`highlight.go`). `Parallel` uses `procs = runtime.NumCPU()` goroutines (when the context has no explicit thread
count), **capped at the number of items**, and each goroutine drains a single shared buffered channel of
indices.

**Citations.**

- `Context.Parallel` — `tools/utils/images/utils.go:27-56`; `procs = runtime.NumCPU()` at `utils.go:35`; the
  cap `if procs > count { procs = count }` at `utils.go:37-39`; the buffered channel + worker spawn at
  `utils.go:41-54`; `wg.Wait()` at `utils.go:55`:

  ```go
  procs := self.NumberOfThreads()
  if procs <= 0 {
      procs = runtime.NumCPU()      // utils.go:35
  }
  if procs > count {
      procs = count                 // utils.go:38
  }
  c := make(chan int, count)        // utils.go:41
  for i := start; i < stop; i++ { c <- i }
  close(c)
  var wg sync.WaitGroup
  for i := 0; i < procs; i++ {
      wg.Add(1)
      go func() { defer wg.Done(); fn(c) }()   // utils.go:50-53
  }
  wg.Wait()
  ```

- The **diff** pool — `kittens/diff/patch.go:352-377`: `ctx := images.Context{}` at `patch.go:354`,
  `ctx.Parallel(0, len(jobs), …)` at `patch.go:361`.
- The **highlight** pool — `kittens/diff/highlight.go:217-228`: `ctx.Parallel(0, len(paths), …)` at
  `highlight.go:219`.

**Observed output** — at real scale (`go test ./diff/ -run TestObsParallel -v`):

```
O5 runtime.NumCPU()=128
O5 items=2000 -> worker goroutines spawned=128 ; peak concurrent workers=128
O5 items=2 -> worker goroutines spawned=2 ; peak concurrent workers=2
```

**Rationale.** With 2000 items the pool used the full `runtime.NumCPU()=128` goroutines and all 128 ran
concurrently (peak `128`) — this is the genuine magnitude on this 128-CPU machine, observed by running at
sufficient scale as the rule requires. With only 2 items the pool capped at 2 workers: the code never spawns
more goroutines than there is work (`if procs > count { procs = count }`). So "multiple files at once" means up
to `runtime.NumCPU()` files being diffed (and, separately, highlighted) in parallel, bounded below by the item
count.

---

## O6 — Parallel-highlight safety: "without stepping on itself"

**Answer.** Two mechanisms prevent workers from interfering. **(1) Work partitioning:** `Parallel` pre-fills
**one** buffered channel with every index and each worker `range`s over that same channel, so Go's channel
semantics deliver each index to **exactly one** worker — no two workers ever process the same file/path. **(2)
Result storage:** each worker highlights a **distinct** path and writes its result keyed by that path via
`highlighted_lines_cache.Set(path, …)`.

**Observed reality — reported exactly, not "fixed."** `LRUCache.Set` guards the underlying map write with a
**read** lock (`RLock`), which is different from `GetOrCreate`'s exclusive `Lock`. Under `go test -race`,
concurrent `Set` calls produce a **data race** and Go's runtime aborts with **`fatal error: concurrent map
writes`**. In the real kitten each highlight worker writes a *distinct* key, but this locking characteristic of
the current code is documented here as observed — it is not modified.

**Citations.**

- Channel index distribution (exactly-once delivery) — `tools/utils/images/utils.go:41-53`.
- Worker result storage — `highlighted_lines_cache.Set(path, text_to_lines(raw))` at `kittens/diff/highlight.go:224`.
- `LRUCache.Set` uses `RLock` while writing the map — `tools/utils/cache.go:32-37` (the map write
  `self.data[key] = val` is `cache.go:34`, held under `self.lock.RLock()` at `cache.go:33`):

  ```go
  func (self *LRUCache[K, V]) Set(key K, val V) {
      self.lock.RLock()          // cache.go:33  -- a READ lock ...
      self.data[key] = val       // cache.go:34  -- ... while WRITING the map
      self.lock.RUnlock()
      return
  }
  ```

- Contrast with `GetOrCreate`'s exclusive `Lock` — `tools/utils/cache.go:48-55`.

**Observed output** — exactly-once index distribution (`go test ./diff/ -run TestObsParallel -v`):

```
O6 indices processed=2000 duplicates=0 missing=0 (each index handled exactly once)
O6 indices processed=2 duplicates=0 missing=0 (each index handled exactly once)
O6 concurrent GetOrCreate (exclusive Lock) completed without race
```

**Observed output** — the `Set` RLock-during-write nuance under `go test -race`
(`go test -race -run TestObserveSetRLockDuringWriteRACE -v`, which exited non-zero, `RACE_EXIT=1`):

```
WARNING: DATA RACE
Write at 0x00c00029c480 by goroutine 13:
  runtime.mapassign_fast64()
      /usr/local/go/src/runtime/map_fast64.go:93 +0x0
  racetest.(*LRUCache[go.shape.int,go.shape.int]).Set()
      /tmp/racetest/cache.go:34 +0x89
```

The same run terminated with Go's runtime fatal error (observed **4×** in the output):

```
fatal error: concurrent map writes
```

`/tmp/racetest/cache.go:34` is the **byte-identical** copy of `tools/utils/cache.go:34`
(`self.data[key] = val`), so the race points at the real code's map write.

**Rationale.** In normal operation the kitten does not "step on itself" for two reasons that the evidence
demonstrates directly. First, the channel-based partitioning delivered every index **exactly once**
(`duplicates=0 missing=0`) at both 2000 and 2 items — so each worker owns a disjoint set of paths, and
`highlight_all` gives each worker a **distinct** map key. Second, the exclusive-lock path (`GetOrCreate`) ran
64 concurrent goroutines with no race. However — reported exactly as observed and **not** softened — because
`Set` guards the map with `RLock` rather than an exclusive `Lock`, concurrent writes to the Go map are not
memory-safe: the race detector flags a `WARNING: DATA RACE` at `cache.go:34`, and the process aborts with
`fatal error: concurrent map writes`. This is a characteristic of the current code; the safety in practice
comes from workers writing distinct keys, not from `Set`'s locking.

---

## O7 — Binary files & images: "what changes when binary files or images show up alongside plain text"

**Answer.** Text detection is `is_path_text(path)`: a path is treated as **non-text** if `is_image(path)` is
true, if it is `/dev/null`, or if its bytes are **not valid UTF-8** (`utf8.ValidString`). Content that is
non-text **and** non-image renders a placeholder line `"Binary file: <human-readable size>"`. **Images** take a
different path entirely: their mimetype resolves to `image/*`, `is_image` is true, and they are routed to the
kitty graphics-protocol image renderer rather than a text diff.

**Citations.**

- `is_path_text` — `kittens/diff/collect.go:86-104`; the UTF-8 test `utf8.ValidString(d)` is at `collect.go:102`;
  `is_image` at `kittens/diff/collect.go:82-84`.
- Binary placeholder — `fmt.Sprintf("Binary file: %s", human_readable(sz))` at `kittens/diff/render.go:452`
  (inside `binary_lines`, `render.go:446-459`); `human_readable` at `render.go:313-330`.
- Render dispatch — `kittens/diff/render.go:704-719`: `is_binary := !is_path_text(path)` at `render.go:706`,
  the image branch `is_img` at `render.go:710`, and the `image_lines` vs `binary_lines` choice at
  `render.go:714-719`:

  ```go
  is_binary := !is_path_text(path)                 // render.go:706
  ...
  if is_binary {
      if is_img {
          ans, err = image_lines(...)              // render.go:716  (graphics protocol)
      } else {
          ans, err = binary_lines(...)             // render.go:718  ("Binary file: <size>")
      }
  }
  ```

**Observed output** (`go test ./diff/ -run TestObsBinaryImage -v`):

```
O7 is_path_text(changed.txt)=true is_path_text(blob.bin)=false is_path_text(pic.png)=false
O7 is_image(pic.png)=true mimetype_for_path(pic.png)="image/png" mimetype_for_path(blob.bin)="application/octet-stream"
O7 binary placeholder (render.go binary_lines L452) = "Binary file: 7 B"
```

**Rationale.** The plain UTF-8 file `changed.txt` is `is_path_text=true`, so it flows into the normal text
diff. The non-UTF-8 `blob.bin` is `is_path_text=false` and is **not** an image (mimetype
`"application/octet-stream"`), so it renders the placeholder `"Binary file: 7 B"` (its 7 bytes formatted by
`human_readable`). The `.png` is `is_path_text=false` **and** `is_image=true` with mimetype `"image/png"`, so
it is routed to `image_lines` (graphics protocol), not a byte diff. In short: plain text diffs normally; a
non-UTF-8 binary collapses to a one-line size placeholder; an image is displayed as an image. As O8 shows,
binary and image pairs are also **excluded** from the text-diff job list, so no attempt is made to line-diff
their bytes.

---

## O8 — End-to-end runtime trace: from "two directories compared" to "changes, renames, additions, removals fully understood"

**Answer.** `main()` loads the config, calls `set_diff_command(conf.Diff_cmd)`, `init_caches()`, resolves any
`ssh:` remote files, and validates that the two arguments are **both** directories or **both** files (rejecting
a directory-vs-file comparison), then starts the terminal loop. `OnInitialize` runs `initialize()`, which
launches a **background goroutine** to build the `Collection` via `create_collection` and posts a `COLLECTION`
async result. The `COLLECTION` case of `handle_async_result` fans out into **three** background activities —
`generate_diff()`, `highlight_all()`, and `load_all_images()` — and each subsequent async result (`DIFF`,
`HIGHLIGHT`, `IMAGE_LOAD`, `IMAGE_RESIZE`) triggers an **incremental re-render**.

**Citations.**

- `main()` — `kittens/diff/main.go:102-163`: `set_diff_command(conf.Diff_cmd)` at `main.go:111`, `init_caches()`
  at `main.go:114`, the dir/file validation `if isdir(left) != isdir(right)` at `main.go:129`, and `lp.Run` at
  `main.go:163`. The validation error is (exact literal, `main.go:130`):

  ```
  The items to be diffed should both be either directories or files. Comparing a directory to a file is not valid.'
  ```

- SSH remote-file fetch — `get_remote_file` invoked at `main.go:121`/`main.go:125`.
- Async pipeline — `initialize` at `kittens/diff/ui.go:114-140` (background `create_collection` at
  `ui.go:133-138`); `generate_diff` at `ui.go:142-159` (text-only job filter `typ == "diff"` at `ui.go:146` and
  `is_path_text` on both sides at `ui.go:147`); `highlight_all` method at `ui.go:179-188`; `load_all_images` at
  `ui.go:190-211`; `handle_async_result` at `ui.go:245-276` (COLLECTION fan-out at `ui.go:247-251`; the
  incremental re-render `case IMAGE_LOAD, HIGHLIGHT: return self.rerender_diff()` at `ui.go:272-273`).

**Observed output** — STAGE 1, the fully-understood classification (`go test ./diff/ -run TestObsCreateCollectionClassify -v`):

```
O8 STAGE 1 (create_collection) done; all_paths order:
   added.txt [add]
   blob.bin [diff]
   changed.txt [diff]
   modeonly.sh [diff]
   old_name.txt [rename]
   pic.png [diff]
   removed.txt [removal]
   sub/nested.txt [diff]
```

**Observed output** — STAGE 2, the diff jobs and the builtin unified patch
(`go test ./diff/ -run TestObsEndToEndPipeline -v`):

```
O8 STAGE 2 (generate_diff) built 3 text-diff jobs (binary/image pairs excluded)
O8 builtin patch for changed.txt:
diff changed.txt changed.txt
--- changed.txt
+++ changed.txt
@@ -1,1 +1,1 @@
-version one
+version two
```

**Rationale.** STAGE 1 is precisely the moment where "changes, renames, additions, and removals are fully
understood": the classified `all_paths` list shows `added.txt [add]`, `removed.txt [removal]`,
`old_name.txt [rename]`, and the content-changed files as `[diff]`. STAGE 2 shows that only text `diff` pairs
become diff jobs — `3` of them — while the binary `blob.bin` and image `pic.png` pairs are **excluded** (they
are handled by O7's binary/image paths, not line-diffing), and the builtin engine emits the unified patch for
`changed.txt`. The `HIGHLIGHT`/`IMAGE_LOAD` results that arrive afterward each call `rerender_diff`
(`ui.go:272-273`), which is the incremental-upgrade behavior explained in O10.

---

## O9 — Matching regions / the diff algorithm, and the `diff_cmd` variants

**Answer.** The builtin diff is an **anchored diff**. It finds the longest common subsequence of **unique**
lines (lines that appear exactly once in *both* sides) using `tgs()` — an implementation of Szymanski's
algorithm that runs in `O(n log n)` — and uses those unique lines as **anchors**, then expands each anchor
backward and forward while lines continue to match. It deliberately does not reuse unrelated blank lines or
closing braces. Identical inputs return a nil slice (no output). A missing final newline is annotated. **Which
engine runs** is chosen by `set_diff_command`: `auto` (pick `git`, else `diff`, else builtin), `builtin` or
`""` (the anchored builtin `Diff`, no external process), `git`, `diff`, or an arbitrary custom command.

**Citations.**

- The rationale comment for the algorithm — `kittens/diff/diff.go:21-48`. Verbatim excerpts (note the source
  uses typographic quotes and a superscript `²`):

  ```
  // In contrast, this implementation looks for a diff with the
  // smallest number of “unique” lines inserted and removed,
  // where unique means a line that appears just once in both old and new.
  // We call this an “anchored diff” because the unique lines anchor
  // the chosen matching regions. An anchored diff is usually clearer
  // than a standard diff, because the algorithm does not try to
  // reuse unrelated blank lines or closing braces.
  // The algorithm also guarantees to run in O(n log n) time
  // instead of the standard O(n²) time.
  ```

  It also names the "patience diff" lineage at `diff.go:42-43`.
- `Diff()` — `kittens/diff/diff.go:49-167`; identical-input short-circuit `if old == new { return nil }` at
  `diff.go:50-52`; anchor expansion at `diff.go:85-93`.
- `lines()` missing-newline handling — `kittens/diff/diff.go:172-182`; the annotation
  `"\n\\ No newline at end of file\n"` is appended at `diff.go:179`.
- `tgs()` (Szymanski) — `kittens/diff/diff.go:184-263`.
- Engine selection consts and `set_diff_command` — `kittens/diff/patch.go:21-62`; `GIT_DIFF` at `patch.go:21`,
  `DIFF_DIFF` at `patch.go:22`.

**Observed output** — matching regions on crafted input (`go test ./diff/ -run TestObsAnchoredDiff -v`, context 3):

```
@@ -1,5 +1,6 @@
 func A() {
 	x := 1
+	x++
 }
 
 func B() {
O9 tgs() anchor pairs (x,y line indexes incl sentinels): [{0 0} {0 0} {1 1} {3 4} {4 5} {5 6} {7 8}]
O9 number of x lines=7 y lines=8
```

**Observed output** — missing-final-newline annotation (`go test ./diff/ -run TestObsNoNewline -v`):

```
@@ -1,2 +1,2 @@
 hello
-world
\ No newline at end of file
+world
O9 lines("hello\nworld") => ["hello\n" "world\n\\ No newline at end of file\n"]
```

**Observed output** — identical inputs return nil (`go test ./diff/ -run TestObsIdentical -v`):

```
O9 identical inputs Diff() returns nil? true (len=0)
```

**Observed output** — `diff_cmd` selection (`go test ./diff/ -run TestObsDiffCmdVariants -v`):

```
O9 set_diff_command("auto"       ) -> diff_cmd=[git diff --no-color --no-ext-diff --exit-code -U_CONTEXT_ --no-index --]  [EXTERNAL]
O9 set_diff_command("builtin"    ) -> diff_cmd=[]  [BUILTIN (anchored Diff, no external process)]
O9 set_diff_command(""           ) -> diff_cmd=[]  [BUILTIN (anchored Diff, no external process)]
O9 set_diff_command("git"        ) -> diff_cmd=[git diff --no-color --no-ext-diff --exit-code -U_CONTEXT_ --no-index --]  [EXTERNAL]
O9 set_diff_command("diff"       ) -> diff_cmd=[diff -p -U _CONTEXT_ --]  [EXTERNAL]
O9 set_diff_command("mydiff --foo") -> diff_cmd=[mydiff --foo]  [EXTERNAL]
O9 GIT_DIFF const = "git diff --no-color --no-ext-diff --exit-code -U_CONTEXT_ --no-index --"
O9 DIFF_DIFF const = "diff -p -U _CONTEXT_ --"
```

**Rationale.** The `tgs()` anchor pairs `[{0 0} {0 0} {1 1} {3 4} {4 5} {5 6} {7 8}]` are the matched unique
lines (with sentinels `{0 0}` at the start and `{7 8}` at the end). The single inserted line `+	x++` shows the
anchors — the unique lines `func A() {` and `func B() {` — framing the matching region; the algorithm does not
try to reuse the shared `}` or the blank line as a match, which is exactly the "clearer than a standard diff"
property the comment describes. The no-newline run shows the literal annotation `\ No newline at end of file`
inserted for the side lacking a trailing newline, and `lines()` embedding it into the last line. Identical
inputs short-circuit to `nil`. For engine selection, `set_diff_command("builtin")` and the empty string both
yield an empty `diff_cmd` (`[]`), meaning the in-process anchored `Diff` is used with no external process;
`git`/`diff`/`auto` yield the exact external command templates; and `auto` selected **git** in this environment
because `git` is present (see *Environment facts*). The two constant strings are quoted exactly above from
`patch.go:21` and `patch.go:22`.

---

## O10 — Perceived speed: "how does the cache quietly keep everything fast"

**Answer.** Two cooperating mechanisms. **(1) Read-once `data_cache`:** every derived computation reuses the
single in-memory copy of each file's bytes, so nothing re-reads disk. **(2) Asynchronous highlighting with
graceful fallback:** the renderer reads highlighted lines via `highlighted_lines_for_path`, which **falls back
to the plain `lines_for_path`** whenever highlighting is absent or the line counts disagree — so plain text
renders immediately and is later **upgraded** to highlighted text when the `HIGHLIGHT` async result arrives and
triggers `rerender_diff`.

**Citations.**

- Read-once `data_for_path` — `kittens/diff/collect.go:65-70`.
- Plain fallback in `highlighted_lines_for_path` — `kittens/diff/collect.go:148-157` (the length-agreement check
  and the `return` of plain lines at `collect.go:156`).
- Async upgrade — `case IMAGE_LOAD, HIGHLIGHT: return self.rerender_diff()` at `kittens/diff/ui.go:272-273`.
- Documented intent — `docs/kittens/diff.rst:15-16`, which states highlighting is done *"asynchronously, for
  maximum speed."*

**Observed output** (`go test ./diff/ -run TestObsCaches -v`):

```
O4/O10 after deleting changed.txt from disk, data_for_path still returns cached "version one\n" (err=<nil>)
O10 highlighted_lines_for_path falls back to plain (equal to lines_for_path)? true (lines=1)
```

**Rationale.** The first line is the "quietly fast" proof: the file was **deleted from disk**, yet
`data_for_path` still returned its bytes — the read happened exactly once and everything downstream (hash,
is-text, lines, size, highlight) is served from that cached copy without touching disk again. The second line
shows that **before** highlighting completes, `highlighted_lines_for_path` returns exactly the plain lines
(`equal to lines_for_path? true`), so the UI never blocks waiting on Chroma; the switch to colored output later
is just an incremental `rerender_diff`. That combination — pay for each file's bytes once, render plain text
immediately, upgrade to highlighted text asynchronously — is how the cache "quietly keeps everything fast."

---

## Configuration & dependencies (as observed / declared)

**Config defaults and shortcuts** (from the config/CLI DSL `kittens/diff/main.py`):

- `syntax_aliases` default `pyj:py pyi:py recipe:py` — `kittens/diff/main.py:29`.
- `num_context_lines` default `3` — `kittens/diff/main.py:37`.
- `diff_cmd` default `auto` — `kittens/diff/main.py:41`.
- `replace_tab_by` default four spaces (`\x20\x20\x20\x20`) — `kittens/diff/main.py:52`.
- `ignore_name` — documented example patterns `.git`, `*~`, `*.pyc` — `kittens/diff/main.py:63-65` (the
  *compiled* default is empty; see the O1 note).
- Keyboard shortcuts: next change `n` — `kittens/diff/main.py:217`; previous change `p` — `kittens/diff/main.py:221`;
  increase context `+` (`change_context 5`) — `kittens/diff/main.py:233`; decrease context `-`
  (`change_context -5`) — `kittens/diff/main.py:237`.
- `syntax_aliases` is parsed into `k:v` pairs by the helper `syntax_aliases()` — `kittens/diff/__init__.py:4-9`.

**Highlighting engine.** The current implementation highlights with the Go **Chroma** library (not the legacy
Python `pygments` path referenced by old issues). Dependency versions are quoted exactly from `go.mod`:

- `go 1.22` — `go.mod:3`.
- `github.com/alecthomas/chroma/v2 v2.14.0` — `go.mod:7`.
- `github.com/bmatcuk/doublestar/v4 v4.6.1` — `go.mod:8`.
- `github.com/edwvee/exiffix v0.0.0-20240229113213-0dbb146775be` — `go.mod:10`.
- `github.com/kovidgoyal/imaging v1.6.3` — `go.mod:13`.
- `github.com/zeebo/xxh3 v1.0.2` — `go.mod:16`.
- `github.com/disintegration/imaging v1.6.2` (indirect) — `go.mod:24`.

**Published behavior** (`docs/kittens/diff.rst`), each a short exact quote:

- Side-by-side diffs — `docs/kittens/diff.rst:13`: *"Displays diffs side-by-side in the kitty terminal"*.
- Asynchronous highlighting — `docs/kittens/diff.rst:15-16`: *"asynchronously, for maximum speed"*.
- Images, even over SSH — `docs/kittens/diff.rst:18`: *"Displays images as well as text diffs, even over SSH"*.
- Recursive directory diffing — `docs/kittens/diff.rst:20`: *"Does recursive directory diffing"*.

**External documentation corroboration.** The official kitty `kitten-diff` documentation corroborates the
observed runtime behavior: `diff_cmd auto` picks an available implementation, `builtin` uses the anchored diff
from the Go standard library, and `git`/`diff` use those commands; `ignore_name` is a glob matched against only
the filename, and matching files/directories are skipped during the filesystem scan; `word_diff_mode` offers
`central` (byte-level central region) and `words` (word-level), with the `words` mode applying only when a
changed chunk has equal numbers of added and removed lines — which matches `Chunk.finalize`'s
`left_count == right_count` guard at `kittens/diff/patch.go:101-107`.

**Peripheral files** (consulted for completeness; not central to these questions): `kittens/diff/mouse.go`
(mouse selection / copy) and `kittens/diff/search.go` (in-diff search).

---

## Coverage pass — every named item, answered by name

| # | Item named in the question | Answered in | Status |
|---|----------------------------|-------------|--------|
| O1 | **Directories / directory pairing** (what belongs together) | [O1](#o1--directory-pairing-what-belongs-together) | ✅ answered |
| O2 | **Renames** — "recognize a rename instead of a deletion and a new file" | [O2](#o2--rename-detection-recognizing-a-rename-instead-of-a-delete--a-new-file) | ✅ answered |
| O3 | **Caching layers** — the seven caches | [O3](#o3--caching-layers-from-raw-file-contents-to-highlighted-output) | ✅ answered |
| O4 | **Cache efficiency** — "stay efficient" | [O4](#o4--cache-efficiency-stay-efficient) | ✅ answered |
| O5 | **Concurrency** — multiple files processed at once | [O5](#o5--concurrency-when-multiple-files-are-being-processed-at-once) | ✅ answered |
| O6 | **Parallel syntax highlighting** — "without stepping on itself" | [O6](#o6--parallel-highlight-safety-without-stepping-on-itself) | ✅ answered |
| O7a | **Binary files** | [O7](#o7--binary-files--images-what-changes-when-binary-files-or-images-show-up-alongside-plain-text) | ✅ answered |
| O7b | **Images** | [O7](#o7--binary-files--images-what-changes-when-binary-files-or-images-show-up-alongside-plain-text) | ✅ answered |
| O9 | **Matching regions / the diff algorithm** | [O9](#o9--matching-regions--the-diff-algorithm-and-the-diff_cmd-variants) | ✅ answered |
| O8 | **End-to-end runtime trace** — changes, renames, additions, removals fully understood | [O8](#o8--end-to-end-runtime-trace-from-two-directories-compared-to-changes-renames-additions-removals-fully-understood) | ✅ answered |
| O10 | **"Quietly keeps everything fast"** — perceived speed | [O10](#o10--perceived-speed-how-does-the-cache-quietly-keep-everything-fast) | ✅ answered |

---

## Reproduction appendix

All harnesses ran **outside** the repository, under `/tmp/obs` (module `obs`, with
`replace kitty => /tmp/blitzy/kitty/blitzy-e22ef2ce-f046-451f-b141-d3c834ec0465_cb4028`) and `/tmp/racetest`.
Each copied source was **byte-identical** to the original except, for the race harness, its `package` clause.
The source repository was verified unchanged throughout (`git status --porcelain` empty; no `blitzy/`
directory existed before this document).

Commands used (each prints lines tagged `O1`…`O10`, quoted verbatim above):

```
# Fidelity — the real upstream test, byte-identical:
go test ./diff/ -run TestDiffCollectWalk -v

# O1 directory pairing:
go test ./diff/ -run TestObsWalkPairing -v

# O2 rename detection (classification + MD5 cross-match):
go test ./diff/ -run 'TestObsCreateCollectionClassify|TestObsRenameHashCrossMatch' -v

# O3 seven caches (per-type population):
go test ./diff/ -run TestObsCachesByType -v

# O4 cache efficiency (read-once, eviction, cached-after-delete, lazy size_cache):
go test ./diff/ -run 'TestObsLRUReadOnceAndEviction|TestObsCaches' -v

# O5 concurrency + O6 exactly-once index distribution (real NumCPU scale):
go test ./diff/ -run TestObsParallel -v

# O6 the LRUCache.Set RLock-during-write race (intentionally crashes -> run in /tmp/racetest):
go test -race -run TestObserveSetRLockDuringWriteRACE -v

# O7 binary & image detection:
go test ./diff/ -run TestObsBinaryImage -v

# O8 end-to-end pipeline (classification + builtin patch):
go test ./diff/ -run 'TestObsCreateCollectionClassify|TestObsEndToEndPipeline' -v

# O9 anchored diff + tgs + no-newline + identical + diff_cmd variants:
go test ./diff/ -run 'TestObsAnchoredDiff|TestObsNoNewline|TestObsIdentical|TestObsDiffCmdVariants' -v

# O10 perceived speed (cached-after-delete + plain fallback):
go test ./diff/ -run TestObsCaches -v
```

**Fidelity result** (proves the harness exercises the real code):

```
=== RUN   TestDiffCollectWalk
--- PASS: TestDiffCollectWalk (0.00s)
PASS
ok  	obs/diff	0.019s
```

---

### Summary of the two "aha" mechanisms

- **Rename detection is not magic — it is content hashing.** A `removal` + `add` becomes a single `rename`
  only when a removed file and an added file share the same MD5 **and** their full bytes compare equal
  (`kittens/diff/collect.go:347-364`). Names never enter into it.
- **Everything feels fast because of read-once + async upgrade.** Each file's bytes are read exactly once into
  `data_cache` and reused by every derived cache (`kittens/diff/collect.go:65-70`); plain text is rendered
  immediately and asynchronously upgraded to syntax-highlighted text via `rerender_diff`
  (`kittens/diff/ui.go:272-273`), with `highlighted_lines_for_path` falling back to plain lines until Chroma is
  done (`kittens/diff/collect.go:148-157`).

And, reported exactly as observed and **not** altered: `LRUCache.Set` writes the underlying map while holding
only a read lock (`tools/utils/cache.go:32-37`), which the race detector flags as a data race and which aborts
with `fatal error: concurrent map writes` under `go test -race`.
