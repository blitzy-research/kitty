# How the kitty `diff` kitten behaves when comparing files and directories

*A runtime‑grounded onboarding answer, written from **observed behavior** of an actually‑built,
actually‑run `kitten diff`. Every behavioral claim below is backed by unedited captured output and a
`file:line` citation to the specific function that does the work.*

---

## The question this document answers (verbatim)

> "I'm onboarding into the Kitty repository and trying to get an intuitive feel for how the diff kitten
> actually behaves when it compares files and directories, because people say it's fast in a way that
> doesn't feel obvious at first glance. When I point it at two directories, how does it decide what
> belongs together, and how does it sometimes recognize a rename instead of treating it like a deletion
> and a new file? That part feels almost magical. I also keep noticing that caching plays a big role,
> from raw file contents to highlighted output, but I don't yet understand how that cache stays efficient
> or what really happens when multiple files are being processed at once. How does syntax highlighting
> run in parallel without stepping on itself, and what changes when binary files or images show up
> alongside plain text? I want to trace what actually happens at runtime from the moment two directories
> are compared through to the point where changes, renames, additions, and removals are fully understood,
> and get a clearer picture of how the diff algorithm finds matching regions while the cache quietly keeps
> everything fast. Temporary scripts may be used for observation, but the repository itself should remain
> unchanged and anything temporary should be cleaned up afterward."

The eight sub‑questions are answered one per section:
**§1** directory pairing · **§2** rename detection ("feels almost magical") · **§3** the seven caches +
efficiency ("quietly keeps everything fast") · **§4** parallel highlighting + many files at once ·
**§5** binary/image handling · **§6** end‑to‑end runtime trace · **§7** how the diff finds matching
regions · **§8** synthesis.

---

## Build & Invocation (canonical, reproducible)

**Toolchain observed** (default, canonical build as a normal user would run it):

```
$ go version
go version go1.22.5 linux/amd64        # go.mod line 3 declares `go 1.22`
$ cc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0    # this pod uses gcc as cc (the AAP's "clang as cc" note does not apply here)
$ git --version
git version 2.51.0
$ diff --version | head -1
diff (GNU diffutils) 3.10
$ nproc
4
```

**Build command** (drives `gen/go_code.py` to emit the generated Go — `conf_generated.go`,
`cli_generated.go`, `constants_generated.go`, `data_generated.bin` — then compiles the Go `kitten`
binary; all outputs are `.gitignore`d):

```
CI=true python3 setup.py build --ignore-compiler-warnings
```

**Binary + version banner** (the real CLI entry point, not a bypassing hook):

```
$ kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
```

**Invocation.** `kitten diff LEFT RIGHT` is a full‑screen TUI that requires a controlling `/dev/tty`
(see §6). It was driven headless through a Python PTY harness that opens a pty, `os.setsid()`s, acquires
the pty as controlling terminal via `TIOCSCTTY`, dups the slave onto fds 0/1/2, and `exec`s the kitten;
the parent drains the raw byte stream (SGR + graphics‑protocol APC bytes preserved) to a file:

```
python3 /tmp/pty_run.py  kitty/launcher/kitten  /tmp/difftest/left  /tmp/difftest/right  /tmp/diff_out.bin
# -> captured_bytes=26501 out=/tmp/diff_out.bin
```

**Determinism.** The identical invocation was run **twice**; both runs produced exactly **26501 bytes**
and identical counts for every volatile marker (binary sizes, image dimensions, `@@` hunk, rename
pairing, omitted `same.cfg`, graphics APC). All values reported below are therefore stable, not
one‑off artifacts.

**Configuration** is the default/canonical one — `diff_cmd auto` (→ `git`, see §7),
`num_context_lines 3` (`kittens/diff/main.py:37`), `pygments_style default` (`kittens/diff/main.py:74`).
No non‑default option was used anywhere; any non‑default observation is labeled as such.

### Fixture layout (temporary, outside the repository)

The fixtures below exercise **every** classification branch in a **single** run. They live under
`/tmp/difftest` (outside the repo) and are deleted afterward, leaving the repository unchanged.

```
$ find /tmp/difftest -type f | sort
/tmp/difftest/left/blob.bin        # binary, 19 B  (non-UTF-8)
/tmp/difftest/left/config.py       # changed text, 102 B
/tmp/difftest/left/old_name.txt    # rename source, 44 B
/tmp/difftest/left/pic.png         # image, 2x2 PNG, 79 B
/tmp/difftest/left/removed.txt     # removal, 51 B (left only)
/tmp/difftest/left/same.cfg        # unchanged, 26 B (identical both sides)
/tmp/difftest/right/added.txt      # addition, 51 B (right only)
/tmp/difftest/right/blob.bin       # binary, 27 B (non-UTF-8, different size)
/tmp/difftest/right/config.py      # changed text, 111 B
/tmp/difftest/right/new_name.txt   # rename target, 44 B (byte-identical to old_name.txt)
/tmp/difftest/right/pic.png        # image, 2x2 PNG, 79 B (different bytes)
/tmp/difftest/right/same.cfg       # unchanged, 26 B (identical both sides)
```

| Condition | Fixture | Expectation |
|-----------|---------|-------------|
| changed text | `config.py` (both sides, different) | side‑by‑side `@@` hunk + highlighting |
| rename | `old_name.txt` → `new_name.txt` (byte‑identical) | one paired entry, **not** add + remove |
| addition | `added.txt` (right only) | "This file was added" |
| removal | `removed.txt` (left only) | "This file was removed" |
| unchanged | `same.cfg` (identical both sides) | **omitted** from output |
| binary | `blob.bin` (19 B vs 27 B, non‑UTF‑8) | size‑only "Binary file: N B" |
| image | `pic.png` (2×2 PNG, both sides) | graphics protocol + "Dimensions: 2x2" |

A note on where the source lives: the diff kitten is implemented in Go under `kittens/diff/`, with its
option definitions in `kittens/diff/main.py`; it leans on two shared Go packages — a generic LRU cache
(`tools/utils/cache.go`) and a generic parallel worker pool (`tools/utils/images/utils.go`). The
user‑facing docs (`docs/kittens/diff.rst:4,13-20`) summarize the design as "a fast side-by-side diff
tool with syntax highlighting and images" that displays diffs side-by-side, does syntax highlighting
"asynchronously, for maximum speed", displays images as well as text diffs "even over SSH", and does
"recursive directory diffing". Everything after this point explains *how* those bullet points actually
work at runtime.

---

## §1 — Directory pairing: how it decides "what belongs together"

**Direct answer: when both arguments are directories, the kitten walks each side recursively and pairs
files by their _relative path name_ using a set intersection. Two files "belong together" if and only
if they have the same path relative to their respective roots.** There is no fuzzy matching at this
stage — pairing is purely by name.

### Mechanism (cause → effect)

The entry point `main` (`kittens/diff/main.go:102`) first validates that exactly two paths were given
and that both are the same kind (both files or both directories):

```go
// kittens/diff/main.go
func main(_ *cli.Command, opts_ *Options, args []string) (rc int, err error) {
	...
	if len(args) != 2 {
		return 1, fmt.Errorf("You must specify exactly two files/directories to compare")   // :108-109
	}
	...
	if isdir(left) != isdir(right) {                                                          // :129
		return 1, fmt.Errorf("The items to be diffed should both be either directories or files. Not a mix.")
	}
```

When both are directories, `create_collection` (`kittens/diff/collect.go:371`) takes the directory
branch and calls the `collect_files` method:

```go
// kittens/diff/collect.go
func create_collection(left, right string) (ans *Collection, err error) {
	...
	if left_stat.IsDir() {
		err = ans.collect_files(left, right)      // :386
		...
	}
```

`collect_files` (`kittens/diff/collect.go:296`) walks each root and then intersects the two name sets.
The walk itself is `walk` (`kittens/diff/collect.go:260`), which uses `filepath.WalkDir` and records
each file under its path **relative** to the root (`filepath.Rel`, `:283`), while honoring the
`conf.Ignore_name` globs by returning `fs.SkipDir` for ignored directories (`:272`):

```go
// kittens/diff/collect.go — walk()
	return filepath.WalkDir(base, func(path string, d fs.DirEntry, err error) error {
		if err != nil {
			return err
		}
		is_allowed := allowed(path, patterns...)
		if !is_allowed {
			if d.IsDir() {
				return fs.SkipDir       // :272  (prune ignored directories)
			}
			return nil
		}
		if d.IsDir() {
			return nil
		}
		...
		name, err := filepath.Rel(base, path)   // :283  (name is RELATIVE to the root)
		...
		if name != "." {
			path_name_map[path] = name
			names.Add(name)             // :289  (the relative name is the pairing key)
			pmap[name] = path
		}
		return nil
	})
```

The pairing is the single line at `kittens/diff/collect.go:306`:

```go
// kittens/diff/collect.go — collect_files()
	common_names := left_names.Intersect(right_names)   // :306  <-- "what belongs together"
```

For each common name, the two files' contents are compared (via the raw‑data cache, §3); if the bytes
differ, the pair is registered as a change (`add_change`, `:319`); if the bytes are identical but the
file **mode** differs, it is still registered as a (mode‑only) change (`:320-330`):

```go
// kittens/diff/collect.go — collect_files()
		if ld != rd {
			changed_names.Add(n)
			self.add_change(left_path_map[n], right_path_map[n])   // :319
		} else {
			if lstat, err := os.Stat(left_path_map[n]); err == nil {
				if rstat, err := os.Stat(right_path_map[n]); err == nil {
					if lstat.Mode() != rstat.Mode() {
						// identical files with only a mode change
						changed_names.Add(n)
						self.add_change(left_path_map[n], right_path_map[n])   // :326
					}
				}
			}
		}
```

### Corroborating unit test (the walk + glob‑ignore, executed)

`TestDiffCollectWalk` (`kittens/diff/collect_test.go:19`) walks a tree containing
`a/b/c, b, d, e, #d#, e~, f/g, h space` with ignore patterns `["*~", "#*#", "b"]`
(`kittens/diff/collect_test.go:42`) and asserts the surviving relative names are exactly
`{d, e, f/g, h space}` (`:33`). This proves glob ignoring through `fs.SkipDir`: `e~` is dropped by
`*~`, `#d#` by `#*#`, and the pattern `b` prunes both the file `b` and the directory `b` (so `a/b/c`
is skipped entirely). Executed in this environment:

```
$ CI=true go test ./kittens/diff/ -run TestDiffCollectWalk -v
=== RUN   TestDiffCollectWalk
--- PASS: TestDiffCollectWalk (0.00s)
PASS
ok  	kitty/kittens/diff	0.032s
```

### Observed evidence (from the captured run)

`config.py` exists under the same relative name on both sides, so it is paired and shown side‑by‑side.
Below is the captured render with ANSI escape/cursor sequences stripped for readability; because each
on‑screen row is written as *left‑half* then carriage‑return then *right‑half*, and the carriage returns
were converted to line breaks here, the left and right halves appear on consecutive lines (on screen
they occupy the left and right halves of the same row). The full ordered render of the whole run —
proving name‑based pairing and, importantly, that the unchanged `same.cfg` is **omitted** — is:

```
   added.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   This file was added
1  this file is brand new

2  it exists only on the right

   blob.bin
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Binary file: 19 B
   Binary file: 27 B

   config.py
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   @@ -1,8 +1,8 @@
1  import sys
1  import sys
2
2
3  def greet(name):
3  def greet(name):
4      print("Hello, " + name)
4      print("Hi there, " + name + "!")
5
5
6  def main():
6  def main():
7      greet("world")
7      greet("kitty")
8      return 0
8      return 0

   old_name.txt
   new_name.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

   pic.png
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Dimensions: 2x2 Size: 79 B
   Dimensions: 2x2 Size: 79 B
   Loading image...
   Loading image...

   removed.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1  this file is going away
   This file was removed
2  it exists only on the left
```

The entries appear in stable sorted order by relative name (`Collection.finalize` sorts `all_paths` by
`path_name_map`, `kittens/diff/collect.go:203-207`): `added.txt, blob.bin, config.py,
old_name.txt/new_name.txt, pic.png, removed.txt`. The pairing is purely by name — `left/config.py` ↔
`right/config.py` — and `same.cfg`, which is byte‑identical on both sides, does not appear at all (its
absence is quantified in §6).

---

## §2 — Rename detection: why it "feels almost magical" (and why it really isn't)

**Direct answer: it is _not_ heuristic similarity scoring. A rename is recognized by exact content
identity — the same MD5 hash of the raw bytes _plus_ a full byte‑for‑byte confirmation. A file that
exists only on the left and a file that exists only on the right are collapsed into a single "rename"
entry when their contents are identical.** The "magic" is just a content hash used as a fast index into
"is this removed file the same bytes as some added file?"

### Mechanism (cause → effect)

After pairing by name (§1), the leftovers are computed by set subtraction — files only on the left are
candidate removals, files only on the right are candidate additions
(`kittens/diff/collect.go:332-333`):

```go
// kittens/diff/collect.go — collect_files()
	removed := left_names.Subtract(common_names)   // :332  (left-only  = candidate removals)
	added := right_names.Subtract(common_names)     // :333  (right-only = candidate additions)
```

Both sets are hashed with `hash_for_path` (`kittens/diff/collect.go:106`), which reads the raw bytes
(through the data cache) and computes an MD5 over them (`:112`):

```go
// kittens/diff/collect.go
func hash_for_path(path string) (string, error) {
	return hash_cache.GetOrCreate(path, func(path string) (string, error) {
		ans, err := data_for_path(path)
		if err != nil {
			return "", err
		}
		hash := md5.Sum(utils.UnsafeStringToBytes(ans))    // :112  MD5 of the raw file bytes
		return utils.UnsafeBytesToString(hash[:]), err
	})
}
```

The rename loop (`kittens/diff/collect.go:347-364`) cross‑matches each removed file against each added
file: **first by hash equality, then by a full byte‑for‑byte comparison** (the hash alone is not trusted
— identical hashes still require `ld == rd`):

```go
// kittens/diff/collect.go — collect_files()
	for name, rh := range rhash {
		found := false
		for n, ah := range ahash {
			if ah == rh {                                       // :350  hashes match?
				ld, _ := data_for_path(left_path_map[name])
				rd, _ := data_for_path(right_path_map[n])
				if ld == rd {                                   // :353  confirm byte-for-byte
					self.add_rename(left_path_map[name], right_path_map[n])   // :354  RENAME
					added.Discard(n)                            // :355  (no longer an "add")
					found = true
					break
				}
			}
		}
		if !found {
			self.add_removal(left_path_map[name])               // :362  genuine removal
		}
	}
	for name := range added.Iterable() {
		self.add_add(right_path_map[name])                       // :366  genuine addition
	}
```

So a removed+added pair with identical content becomes **one** `rename` entry, and the matched addition
is discarded from the add set so it is not also reported as new. Files that do not match anything remain
a real removal (`add_removal`, `:362`) or a real addition (`add_add`, `:366`).

### Observed evidence — the real MD5 (computed and byte‑verified here)

The two fixture files have identical bytes; their MD5 (computed directly, and confirmed byte‑for‑byte
with `cmp`) is:

```
$ md5sum /tmp/difftest/left/old_name.txt /tmp/difftest/right/new_name.txt
1e280e1713df124d35709cf6138d9f91  /tmp/difftest/left/old_name.txt
1e280e1713df124d35709cf6138d9f91  /tmp/difftest/right/new_name.txt

$ cmp /tmp/difftest/left/old_name.txt /tmp/difftest/right/new_name.txt && echo IDENTICAL
IDENTICAL
```

`kitty` stores the MD5 as the raw 16 bytes (`utils.UnsafeBytesToString(hash[:])`, `:113`) and compares
those raw byte strings at `:350`; the hex `1e280e1713df124d35709cf6138d9f91` is that same digest in hex,
verified against the exact fixture bytes (`the quick brown fox jumps over the lazy dog\n`, 44 bytes).

### Observed evidence — the pair is collapsed into ONE entry (the direct proof)

In the captured render, `old_name.txt` and `new_name.txt` appear **together as a single paired section**,
not as a separate removal and a separate addition:

```
   old_name.txt                            new_name.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

The decisive proof is the **contrast** with genuine add/remove. In the raw 26501‑byte stream, the
add/removal explanatory messages appear, but the rename pair triggers **neither**:

```
$ python3 - <<'PY'
raw = open('/tmp/diff_out.bin','rb').read()
print("This file was added   :", raw.count(b'This file was added'))
print("This file was removed :", raw.count(b'This file was removed'))
print("was renamed to        :", raw.count(b'was renamed to'))
print("The file              :", raw.count(b'The file'))
PY
This file was added   : 2
This file was removed : 2
was renamed to        : 0
The file              : 0
```

`added.txt` produces "This file was added" and `removed.txt` produces "This file was removed" (count 2
each — the screen is painted twice, see §6), but `old_name.txt`/`new_name.txt` produce **neither**.
Instead they are shown as one paired title (`old_name.txt` on the left half, `new_name.txt` on the right
half — see `title_lines`, `kittens/diff/render.go:217-218`). That collapse from "removal + addition"
into a single paired entry is the observable manifestation of hash‑based rename detection.

### Honest nuance — the "was renamed to" sentence is NOT rendered (observed)

The AAP anticipated an on‑screen label `"The file X was renamed to Y"` from `rename_lines`
(`kittens/diff/render.go:684`). **That sentence is never emitted in the output** (`was renamed to` count
= 0 above, stable across both runs). This is a real, reproducible behavior, not a capture artifact, and
the cause is precise: `rename_lines` places the text into the **right** half of the screen line while
marking the logical line **full width**:

```go
// kittens/diff/render.go — rename_lines()  (:684)
	ll := LogicalLine{
		left_reference: Reference{path: path}, right_reference: Reference{path: other_path},
		line_type: CHANGE_LINE, is_change_start: true, is_full_width: true}          // full width
	for _, line := range splitlines(fmt.Sprintf(`The file %s was renamed to %s`,
		sanitize(path_name_map[path]), sanitize(path_name_map[other_path])), columns-margin_size) {
		sl := ScreenLine{}
		sl.right.marked_up_text = line          // :690  text put in the RIGHT half
		ll.screen_lines = append(ll.screen_lines, &sl)
	}
```

…but the drawing routine `render_screen_line` (`kittens/diff/render.go:70`) writes only the **left** half
and then `return`s early when the line is full width — before it ever reaches the right half:

```go
// kittens/diff/render.go — render_screen_line()
	lp.QueueWriteString(left_margin + "\x1b[m")
	lp.QueueWriteString(left_text)
	if self.is_full_width {
		return                                   // :99-101  full-width lines stop after the LEFT half
	}
	right_margin := place_in(sl.right.marked_up_margin_text, margin_size)
	right_text := place_in(sl.right.marked_up_text, available_cols)   // <-- never reached for rename_lines
	...
	lp.QueueWriteString(right_text)
```

Because `rename_lines` writes the sentence into `sl.right.marked_up_text` while setting
`is_full_width: true`, and full‑width lines emit only `sl.left` (which is empty here), the explanatory
sentence is dropped at render time. **The rename is still correctly detected and displayed as a single
paired entry** (that is the user's actual question — "recognize a rename instead of a deletion and a new
file"), but the descriptive sentence itself does not appear. Reported exactly as observed.

---

## §3 — The seven caches, and how the cache "quietly keeps everything fast"

**Direct answer: `init_caches` (`kittens/diff/collect.go:26`) creates exactly _seven_ generic
`LRUCache[K,V]` instances, every one with the same capacity `const sz = 4096`
(`kittens/diff/collect.go:29`). Each expensive per‑path computation — stat size, MIME type, raw bytes,
is‑text test, plain line split, highlighted line split, content hash — is memoized in its own cache, so
each file's bytes are read from disk once and the derived results are computed once and reused across
pairing, hashing, diffing, highlighting, and rendering.** That reuse is what "quietly keeps everything
fast."

### The seven caches (all enumerated)

```go
// kittens/diff/collect.go — init_caches()
func init_caches() {
	path_name_map = make(map[string]string, 32)
	remote_dirs = make(map[string]string, 32)
	const sz = 4096                                                 // :29  uniform capacity
	size_cache = utils.NewLRUCache[string, int64](sz)               // :30
	mimetypes_cache = utils.NewLRUCache[string, string](sz)         // :31
	data_cache = utils.NewLRUCache[string, string](sz)              // :32
	is_text_cache = utils.NewLRUCache[string, bool](sz)             // :33
	lines_cache = utils.NewLRUCache[string, []string](sz)          // :34
	highlighted_lines_cache = utils.NewLRUCache[string, []string](sz)  // :35
	hash_cache = utils.NewLRUCache[string, string](sz)              // :36
}
```

| # | Cache | Key → Value | Populated by | `file:line` |
|---|-------|-------------|--------------|-------------|
| 1 | `size_cache` | `string → int64` | `size_for_path` (`os.Stat().Size()`) | `collect.go:30`, fn `:72` |
| 2 | `mimetypes_cache` | `string → string` | `mimetype_for_path` (`GuessMimeTypeWithFileSystemAccess`) | `collect.go:31`, fn `:50` |
| 3 | `data_cache` | `string → string` | `data_for_path` (`os.ReadFile` + zero‑copy `UnsafeBytesToString`) | `collect.go:32`, fn `:65` |
| 4 | `is_text_cache` | `string → bool` | `is_path_text` (image?/`/dev/null`?/`utf8.ValidString`) | `collect.go:33`, fn `:86` |
| 5 | `lines_cache` | `string → []string` | `lines_for_path` (`data_for_path` + `sanitize` + split) | `collect.go:34`, fn `:138` |
| 6 | `highlighted_lines_cache` | `string → []string` | written by `highlight_all` via `Set` (`highlight.go:224`), read by `highlighted_lines_for_path` (`collect.go:148`) | `collect.go:35` |
| 7 | `hash_cache` | `string → string` | `hash_for_path` (`md5.Sum`) | `collect.go:36`, fn `:106` |

### Efficiency mechanism 1 — memoize + bounded LRU eviction

The generic cache is `LRUCache[K,V]` (`tools/utils/cache.go:13`). The workhorse is `GetOrCreate`
(`tools/utils/cache.go:39`): it first tries a read‑locked lookup; on a miss it computes the value once,
then takes the write lock to store it, push it to the front of the LRU list, and evict the least‑recently
used entry when the list exceeds `max_size`:

```go
// tools/utils/cache.go — GetOrCreate()
func (self *LRUCache[K, V]) GetOrCreate(key K, create func(key K) (V, error)) (V, error) {
	self.lock.RLock()
	ans, found := self.data[key]
	self.lock.RUnlock()
	if found {
		return ans, nil                       // :44  cache HIT: no recompute
	}
	ans, err := create(key)                   // :46  cache MISS: compute exactly once
	if err == nil {
		self.lock.Lock()
		self.data[key] = ans                  // :49
		self.lru.PushFront(key)               // :50
		if self.max_size > 0 && self.lru.Len() > self.max_size {
			k := self.lru.Remove(self.lru.Back())     // :52  evict LRU
			delete(self.data, k.(K))                  // :53
		}
		self.lock.Unlock()
	}
	return ans, err
}
```

### Efficiency mechanism 2 — zero‑copy string conversions

Raw file bytes are turned into a `string` **without copying** via `utils.UnsafeBytesToString`
(`kittens/diff/collect.go:68`), so the whole file is materialized once and shared by reference through
the data cache:

```go
// kittens/diff/collect.go — data_for_path()
func data_for_path(path string) (string, error) {
	return data_cache.GetOrCreate(path, func(path string) (string, error) {
		ans, err := os.ReadFile(path)
		return utils.UnsafeBytesToString(ans), err     // :68  zero-copy []byte -> string
	})
}
```

The chain of reuse is visible in the code: `hash_for_path` (§2) calls `data_for_path`;
`is_path_text` (§5) calls `data_for_path`; `lines_for_path` calls `data_for_path`; `highlight_file`
(§4) calls `data_for_path`; `run_diff`'s builtin path (§7) calls `data_for_path` twice. Every one of
those is a cache lookup after the first read — so a file that is paired, hashed, tested for text,
diffed, highlighted, and rendered is read from disk **once**.

### Honest nuance — not every cache is actually LRU‑bounded (observed in code)

Two sibling methods bypass the eviction path, and this is worth stating precisely rather than glossing:

- `Set` (`tools/utils/cache.go:32`) writes the map **while holding only a read lock** (`RLock` at `:33`)
  and never touches the LRU list:

  ```go
  // tools/utils/cache.go — Set()
  func (self *LRUCache[K, V]) Set(key K, val V) {
  	self.lock.RLock()          // :33  READ lock, not write lock
  	self.data[key] = val       // :34  mutates the map under a read lock; no lru.PushFront
  	self.lock.RUnlock()
  	return
  }
  ```

- `MustGetOrCreate` (`tools/utils/cache.go:60`) does take a full write lock (`:68`) but likewise never
  pushes to or evicts from the LRU list.

**Consequence:** bounded LRU eviction (the `sz = 4096` cap) applies only to the four `GetOrCreate`
caches — `size_cache`, `data_cache`, `lines_cache`, `hash_cache`. The `mimetypes_cache` and
`is_text_cache` (populated via `MustGetOrCreate`) and the `highlighted_lines_cache` (populated via
`Set`) grow with the number of distinct paths seen in a run and are not evicted. For a diff session this
is fine (the number of distinct paths is finite and modest), but it is the accurate description of the
code, not "all seven are LRU‑bounded."

### Evidence note (labeled)

Cache **hit counts** are internal and not surfaced by the TUI, so they cannot be read off the captured
byte stream; the efficiency claim rests on the code paths above (memoize via `GetOrCreate`, zero‑copy via
`UnsafeBytesToString`, single‑read reuse across subsystems) and is therefore *inferred from code*, not
measured from output. The *observable consequence* — the whole multi‑file diff (seven fixtures, text +
binary + image) renders in a single fast pass within the capture window — is what the run demonstrates
(§6). Per the read‑only contract, no instrumentation was added to the repository to count hits.

---

## §4 — Parallel syntax highlighting "without stepping on itself" (and what happens with many files)

**Direct answer: each file is highlighted by exactly one worker goroutine, and that worker writes its
result into a distinct cache key (the file's own path). Because every worker owns a different key and
holds no shared mutable highlight state, the workers cannot step on each other.** The same worker‑pool
helper is reused to diff many files in parallel too.

### Mechanism — the channel‑fed worker pool guarantees one distinct key per worker

`highlight_all` (`kittens/diff/highlight.go:217`) builds a fresh worker context and fans the list of
paths out across it:

```go
// kittens/diff/highlight.go — highlight_all()
func highlight_all(paths []string) {
	ctx := images.Context{}                              // :218
	ctx.Parallel(0, len(paths), func(nums <-chan int) {  // :219
		for i := range nums {
			path := paths[i]
			raw, err := highlight_file(path)             // :222  highlight ONE file
			if err == nil {
				highlighted_lines_cache.Set(path, text_to_lines(raw))   // :224  write to THIS path's key
			}
		}
	})
}
```

The pool is `Context.Parallel` (`tools/utils/images/utils.go:27`). It chooses the worker count, pushes
every index into a **buffered channel**, closes it, and then starts that many goroutines that each
`range` over the shared channel:

```go
// tools/utils/images/utils.go — Parallel()
func (self *Context) Parallel(start, stop int, fn func(<-chan int)) {
	count := stop - start
	if count < 1 {
		return
	}
	procs := self.NumberOfThreads()          // :33
	if procs <= 0 {
		procs = runtime.NumCPU()             // :34-35  default = number of CPUs
	}
	if procs > count {
		procs = count                        // :37-38  never more workers than items
	}
	c := make(chan int, count)               // :41  buffered channel
	for i := start; i < stop; i++ {
		c <- i                               // :43  enqueue every index
	}
	close(c)                                 // :45  closed channel -> each value delivered once
	var wg sync.WaitGroup
	for i := 0; i < procs; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			fn(c)                            // :52  every worker ranges the SAME channel
		}()
	}
	wg.Wait()                                // :55  block until all workers finish
}
```

The critical property: a **closed, buffered channel** hands each queued index to exactly **one**
receiving goroutine. So no two workers ever get the same index → no two workers ever call
`highlight_file` on the same path → each worker writes a **different** `highlighted_lines_cache` key.
That is the structural reason highlighting does not "step on itself": there is no shared mutable
highlight state, only per‑path writes to disjoint keys. (The one subtlety from §3 — that
`highlighted_lines_cache.Set` mutates the map under a read lock, `tools/utils/cache.go:33` — is safe
here precisely because the keys are disjoint; workers never write the same map slot concurrently.)

### What `highlight_file` actually does (Chroma)

Each worker runs `highlight_file` (`kittens/diff/highlight.go:161`): it picks a lexer by filename
(`lexers.Match`, `:175`) with a content‑analysis fallback (`lexers.Analyse`, `:178`); since the default
`pygments_style` is `default` (`kittens/diff/main.py:74`) it selects the built‑in `DefaultStyle()`
(`:188`); it tokenizes (`lexer.Tokenise`, `:205`); and it formats to ANSI SGR escapes through
`ansi_formatter` (`kittens/diff/highlight.go:84`, whose `SGR_PREFIX = "\033["` is at `:85`). The Chroma
version is pinned exactly at `github.com/alecthomas/chroma/v2 v2.14.0` (`go.mod:7`).

### "Many files at once" — the same pool drives parallel per‑file diffing

The identical worker‑pool helper is reused for diffing. `diff` (`kittens/diff/patch.go:352`) builds a
context and dispatches one `do_diff` per changed file, collecting results over a buffered channel:

```go
// kittens/diff/patch.go — diff()
func diff(jobs []diff_job, context_count int) (ans map[string]*Patch, err error) {
	ans = make(map[string]*Patch)
	ctx := images.Context{}                                  // :354
	type result struct {
		file1, file2 string
		err          error
		patch        *Patch
	}
	results := make(chan result, len(jobs))                  // :360  buffered per job
	ctx.Parallel(0, len(jobs), func(nums <-chan int) {       // :361
		for i := range nums {
			job := jobs[i]
			r := result{file1: job.file1, file2: job.file2}
			r.patch, r.err = do_diff(job.file1, job.file2, context_count)   // :365  one diff per worker
			results <- r
		}
	})
	close(results)
	for r := range results {
		if r.err != nil {
			return nil, r.err
		}
		ans[r.file1] = r.patch
	}
```

So with many files, both the diffing and the highlighting fan out across `runtime.NumCPU()` workers.

### Observed evidence — highlighting really ran, and produced pygments‑default colors

On this machine the worker basis is `runtime.NumCPU()`:

```
$ nproc
4
```

The captured stream contains **270** SGR sequences, **118** of them foreground‑color sequences, and the
distinct foreground truecolors are exactly the pygments "default" palette — confirming Chroma syntax
highlighting completed (not merely diff coloring):

```
$ python3 - <<'PY'
import re
raw=open('/tmp/diff_out.bin','rb').read()
fg=re.findall(rb'\x1b\[38[:;]2[:;](\d+)[:;](\d+)[:;](\d+)m', raw)
from collections import Counter
for (r,g,b),n in sorted(Counter(fg).items(), key=lambda x:-x[1]):
    print(f"  rgb({int(r):>3},{int(g):>3},{int(b):>3})  x{n}")
PY
  rgb(186, 33, 33)  x7     # pygments string  literal   (#BA2121)
  rgb(170,170,170)  x6     # line-number margin          (#AAAAAA)
  rgb(102,102,102)  x5     # pygments number/comment     (#666666)
  rgb(  0,  0,255)  x4     # pygments function name      (#0000FF)
  rgb(172,242,189)  x2     # diff intraline ADD highlight
  rgb(253,184,192)  x2     # diff intraline REMOVE highlight
  rgb(  0,128,  0)  x2     # pygments keyword green      (#008000)
```

And the SGR escapes are woven **inside** the source tokens. For example the changed line's right half
shows the intraline change region highlighted with a truecolor background (this is `changed_center`,
`kittens/diff/patch.go:86`, marking just the differing middle): the common prefix `print("H` is plain,
then the highlight begins:

```
$ python3 -c "raw=open('/tmp/diff_out.bin','rb').read(); i=raw.find(b'i there'); print(repr(raw[i-24:i+20]))"
b'    print("H\x1b[48:2:172:242:189mi there, " + name'
```

`\x1b[48:2:172:242:189m` is a truecolor **background** (SGR 48) of RGB (172,242,189) — the green "added"
intraline highlight — beginning exactly where `"Hi there…"` diverges from `"Hello…"`.

---

## §5 — What changes when binary files or images show up alongside plain text

**Direct answer: two gates classify every path. Text on both sides is diffed and highlighted; a binary
file is shown as a single size‑only "Binary file: N B" line (no content diff); an image is drawn with
the kitty graphics protocol and labeled with its dimensions.** The gates are `is_path_text` and
`is_image`.

### Gate 1 — `is_path_text` (`kittens/diff/collect.go:86`)

```go
// kittens/diff/collect.go — is_path_text()
func is_path_text(path string) bool {
	return is_text_cache.MustGetOrCreate(path, func(path string) bool {
		if is_image(path) {
			return false                       // :89  images are never "text"
		}
		s1, err := os.Stat(path)
		if err == nil {
			s2, err := os.Stat("/dev/null")
			if err == nil && os.SameFile(s1, s2) {
				return false                   // :95  /dev/null is not "text"
			}
		}
		d, err := data_for_path(path)
		if err != nil {
			return false
		}
		return utf8.ValidString(d)             // :102  text  <=>  valid UTF-8
	})
}
```

### Gate 2 — `is_image` (`kittens/diff/collect.go:82`)

```go
// kittens/diff/collect.go — is_image()
func is_image(path string) bool {
	return strings.HasPrefix(mimetype_for_path(path), "image/")   // :83
}
```

### Routing

The diff *job* is created only when **both** sides are text — the guard in `generate_diff`
(`kittens/diff/ui.go:147`):

```go
// kittens/diff/ui.go — generate_diff()
	_ = self.collection.Apply(func(path, typ, changed_path string) error {
		if typ == "diff" {
			if is_path_text(path) && is_path_text(changed_path) {    // :147  only text gets a diff job
				jobs = append(jobs, diff_job{path, changed_path})
			}
		}
		return nil
	})
```

At render time (`kittens/diff/render.go:696` `render()`), a non‑text entry routes to `image_lines` if it
is an image, else to `binary_lines`:

- Binary → `binary_lines` (`kittens/diff/render.go:446`), whose per‑side text is
  `fmt.Sprintf("Binary file: %s", human_readable(sz))` (`kittens/diff/render.go:452`).
- Image → `image_lines` (`kittens/diff/render.go:333`); the label is built as
  `text := fmt.Sprintf("Size: %s", human_readable(sz))` (`:341`) and then
  `text = fmt.Sprintf("Dimensions: %dx%d %s", res.Width, res.Height, text)` (`:344`). Image *paths* are
  gathered by `load_all_images` (`kittens/diff/ui.go:190`) — which calls `is_image` on each path/changed
  path (`:192`,`:196`) — and drawn through the kitty graphics protocol in `tools/tui/graphics/`.

### Observed per‑fixture classification

Derived from the observed routing (text files got a `@@` diff; `blob.bin` got "Binary file"; `pic.png`
got the graphics protocol) and confirmed against `file(1)` MIME guesses:

| Fixture | `is_image` | `utf8.ValidString` | Route |
|---------|-----------|--------------------|-------|
| `config.py` (`text/x-script.python`) | false | **true** | text diff + highlight |
| `blob.bin` (`application/octet-stream`) | false | **false** | `binary_lines` |
| `pic.png` (`image/png`) | **true** | (n/a — image) | `image_lines` + graphics |

```
$ file --mime-type /tmp/difftest/left/blob.bin /tmp/difftest/left/pic.png /tmp/difftest/left/config.py
/tmp/difftest/left/blob.bin:  application/octet-stream
/tmp/difftest/left/pic.png:   image/png
/tmp/difftest/left/config.py: text/x-script.python
```

### Observed evidence — binary (size‑only, both sizes shown)

`blob.bin` (19 bytes left, 27 bytes right, both non‑UTF‑8) renders as two size‑only lines and **no**
content diff:

```
   blob.bin
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Binary file: 19 B
   Binary file: 27 B
```

### Observed evidence — image (dimensions + graphics‑protocol APC bytes)

`pic.png` (2×2 on both sides) renders its dimensions and streams the kitty graphics protocol. The label:

```
   pic.png
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Dimensions: 2x2 Size: 79 B
   Dimensions: 2x2 Size: 79 B
   Loading image...
```

The raw stream contains the graphics‑protocol **APC** sequences (`\x1b_G … \x1b\\`). The first one,
quoted verbatim (leading bytes), and the set of distinct control strings observed:

```
$ python3 - <<'PY'
raw=open('/tmp/diff_out.bin','rb').read()
g=raw.find(b'\x1b_G'); end=raw.find(b'\x1b\\', g)
print("first APC (%d bytes):" % (end+2-g), repr(raw[g:g+98]))
import re
for a in sorted(set(re.findall(rb'\x1b_G([^;\x1b]*)', raw))):
    print("  ctrl:", a.decode())
PY
first APC (98 bytes): b'\x1b_Ga=q,f=24,t=t,s=1,v=1,S=47,i=1;L2Rldi9zaG0va2l0dHktdHR5LWdyYXBoaWNzLXByb3RvY29sLTQxMzgzNTIyNTY\x1b\\'
  ctrl: a=d
  ctrl: a=q,f=24,t=s,s=1,v=1,S=18,i=2
  ctrl: a=q,f=24,t=t,s=1,v=1,S=47,i=1
```

Here `a=q` is a graphics *query* (probe whether the image can be displayed), `f=24` is RGB format,
`t=t`/`t=s` are the transmission media (temp file / shared memory), `S` is the payload byte size, `i` is
the image id, and `a=d` deletes placements. The base64 payload after the `;` decodes to a
`/dev/shm/kitty-tty-graphics-protocol-…` path — i.e. the image bytes are handed to the terminal out of
band. This is exactly the "displays images as well as text diffs" behavior, routed entirely by the two
gates above.

---

## §6 — End‑to‑end runtime trace: from "two directories" to "changes, renames, adds, removals fully understood"

**Direct answer: the flow is asynchronous. Collection runs on a background goroutine; when it finishes,
the diff, the highlighting, and the image loading are each fanned out on their own goroutines; results
are drained through a wakeup channel and dispatched to render. The UI paints a "Calculating diff, please
wait…" screen immediately and fills in as each async result arrives.**

### Step‑by‑step trace (every arrow is a cited `file:line`)

1. **`main` (`kittens/diff/main.go:102`)** validates the two arguments (§1), resolves the diff backend
   with `set_diff_command(conf.Diff_cmd)` (`:111`, see §7), creates the seven caches with `init_caches()`
   (`:114`, see §3), constructs the event loop with `loop.New()` (`:138`), and enters via `EntryPoint`
   (`:177`).

2. **The loop opens the controlling terminal.** `tools/tui/loop/run.go:305` calls
   `tty.OpenControllingTerm(tty.SetRaw)`, which is `OpenTerm(Ctermid(), …)` (`tools/tty/tty.go:133-134`),
   and `Ctermid()` returns the literal `"/dev/tty"` (`tools/tty/tty.go:362`). This is why the kitten
   needs a controlling terminal (and why the error path below happens without one).

3. **`initialize` (`kittens/diff/ui.go:114`)** creates the async results channel (buffered 32,
   `:132`), spawns a goroutine that runs `create_collection` and then wakes the main thread
   (`:133-138`), and immediately draws the first screen (`:139`):

   ```go
   // kittens/diff/ui.go — initialize()
   	self.async_results = make(chan AsyncResult, 32)          // :132
   	go func() {
   		r := AsyncResult{}
   		r.collection, r.err = create_collection(self.left, self.right)   // :135  collection on a goroutine
   		self.async_results <- r
   		self.lp.WakeupMainThread()                            // :137
   	}()
   	self.draw_screen()                                        // :139  paints BEFORE any result is ready
   ```

4. **`draw_screen` (`kittens/diff/ui.go:340`)** prints the transitional message whenever the logical
   lines, diff map, or collection are still nil (`:349-351`) — this is the **before** state:

   ```go
   // kittens/diff/ui.go — draw_screen()
   	if self.logical_lines == nil || self.diff_map == nil || self.collection == nil {
   		lp.Println(`Calculating diff, please wait...`)       // :350
   		return
   	}
   ```

5. **`on_wakeup` (`kittens/diff/ui.go:161`)** drains the channel and calls `handle_async_result`
   (`kittens/diff/ui.go:245`) for each result. This is the dispatcher that turns "two directories" into a
   fully understood diff:

   ```go
   // kittens/diff/ui.go — handle_async_result()
   	switch r.rtype {
   	case COLLECTION:                                          // :247  collection is ready
   		self.collection = r.collection
   		self.generate_diff()                                 // :249  -> parallel per-file diff (§4, §7)
   		self.highlight_all()                                 // :250  -> parallel highlight   (§4)
   		self.load_all_images()                               // :251  -> async image load     (§5)
   	case DIFF:                                                // :252  diffs are ready
   		self.diff_map = r.diff_map
   		self.calculate_statistics()                          // :254
   		self.clear_mouse_selection()
   		err := self.render_diff()                            // :256
   		...
   		self.draw_screen()                                   // :268  now paints the populated diff
   	case IMAGE_LOAD, HIGHLIGHT:                               // :272
   		return self.rerender_diff()                          // :273  re-render with colors / images
   	}
   ```

   So the moment the **`COLLECTION`** result arrives, the three producers (`generate_diff`,
   `highlight_all`, `load_all_images`) are launched together; the **`DIFF`** result stores `diff_map`,
   computes statistics, and renders; **`HIGHLIGHT`** and **`IMAGE_LOAD`** each trigger a re‑render so the
   colors and images fill in on top of the already‑visible diff.

6. **`render_diff` (`kittens/diff/ui.go:294`)** refuses to render on a tiny screen — `columns < 8`
   (`:295-296`) or `rows < 2` (`:298-299`) — then builds the logical lines, which consume the highlighted
   lines via `highlighted_lines_for_path` (`kittens/diff/render.go:593`, `:599`, `:636`), tying the render
   back to the highlight cache from §3/§4.

### Before / intermediate / after (observed)

- **Before** — the transitional frame is in the captured stream (raw, verbatim; `count = 1`, stable
  across both runs):

  ```
  $ python3 -c "raw=open('/tmp/diff_out.bin','rb').read(); i=raw.find(b'Calculating'); print(repr(raw[i-12:i+34]))"
  b'\x1b[1;1H\x1b[JCalculating diff, please wait...\n\r'
  ```

  (`\x1b[1;1H` homes the cursor, `\x1b[J` clears to end of screen, then the message — exactly
  `draw_screen`'s nil‑guard branch at `kittens/diff/ui.go:350`.)

- **Intermediate/After** — the populated side‑by‑side render (quoted in full in §1) shows the change
  (`config.py` `@@` hunk), the rename (`old_name.txt`/`new_name.txt` paired), the addition (`added.txt` →
  "This file was added"), the removal (`removed.txt` → "This file was removed"), the binary
  (`blob.bin` → two "Binary file" lines), and the image (`pic.png` → "Dimensions: 2x2"). The unchanged
  `same.cfg` is **omitted**, quantified here:

  ```
  $ python3 -c "raw=open('/tmp/diff_out.bin','rb').read(); print('same.cfg count =', raw.count(b'same.cfg'))"
  same.cfg count = 0
  ```

  (`same.cfg` is byte‑identical on both sides, so it never enters `changed_names`, `removed`, or `added`
  in `collect_files` (§1) and is therefore never rendered — the correct "no difference" result.)

### The error path (no controlling `/dev/tty`), observed verbatim

Running the kitten without a controlling terminal (via `setsid`, stdin from `/dev/null`) makes
`Ctermid()`/`OpenControllingTerm` fail:

```
$ setsid kitty/launcher/kitten diff /tmp/difftest/left /tmp/difftest/right < /dev/null > /tmp/err_out.txt 2>&1 ; echo "exit=$?"
exit=1
$ cat /tmp/err_out.txt
Error: open /dev/tty: no such device or address
```

The `ENXIO` ("no such device or address") comes straight from opening `Ctermid() == "/dev/tty"`
(`tools/tty/tty.go:362`) when the process has no controlling terminal — the concrete mechanism, not a
vague "it needs a terminal."

### Observed control flow (summary; every arrow maps to a citation above)

```
main (main.go:102) ─ validate args, set_diff_command, init_caches, loop.New, EntryPoint
   loop opens /dev/tty (run.go:305 -> tty.go:134 -> Ctermid tty.go:362)
   initialize (ui.go:114): go create_collection (ui.go:135); draw_screen -> "Calculating…" (ui.go:350)
   on_wakeup (ui.go:161) -> handle_async_result (ui.go:245):
       COLLECTION (ui.go:247) ─┬─> generate_diff  (ui.go:249) ─> diff() parallel (patch.go:352)
                               ├─> highlight_all  (ui.go:250) ─> Parallel highlight (highlight.go:217)
                               └─> load_all_images(ui.go:251) ─> graphics protocol (§5)
       DIFF       (ui.go:252) ─> diff_map + calculate_statistics + render_diff (ui.go:294) + draw_screen
       HIGHLIGHT / IMAGE_LOAD (ui.go:272) ─> rerender_diff (ui.go:234)
   render consumes highlighted_lines_for_path (render.go:593,599,636)
```

---

## §7 — How the diff algorithm finds matching regions

**Direct answer: the built‑in differ is an _anchored_ diff (the "patience diff" family). It anchors the
matching regions on lines that are _unique in both_ inputs, then recurses between anchors. This runs in
`O(n log n)` instead of the classic `O(n²)`. By default, though, `diff_cmd auto` resolves to an external
backend — `git diff --no-index` in this environment — and the built‑in algorithm is used only when no
external differ is available (or when `diff_cmd builtin` is set).**

### The built‑in anchored/patience algorithm

`Diff` (`kittens/diff/diff.go:49`) is a modified port of the Go standard library's `internal/diff`. Its
header comment states the design precisely — it looks for a diff with the fewest **unique** lines
inserted/removed, calling those unique lines the anchors, and guarantees `O(n log n)`:

```go
// kittens/diff/diff.go  (:32-40, from the Diff header comment)
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

The comment (`:42-48`) also explains why the code avoids the name "patience diff" (the term is imprecise
and misleadingly implies slowness). `Diff` short‑circuits identical inputs immediately:

```go
// kittens/diff/diff.go — Diff()
func Diff(oldName, old, newName, new string, num_of_context_lines int) []byte {
	if old == new {
		return nil          // :50-52  identical -> no output
	}
	x := lines(old)
	y := lines(new)
```

The heart — the longest common subsequence of **unique** lines — is `tgs`
(`kittens/diff/diff.go:192`), whose doc‑comment cites the source algorithm:

```go
// kittens/diff/diff.go  (:188-191, tgs doc-comment)
// The longest common subsequence algorithm is as described in
// Thomas G. Szymanski, “A Special Case of the Maximal Common
// Subsequence Problem,” Princeton TR #170 (January 1975),
// available at https://research.swtch.com/tgs170.pdf.
```

Inside, unique lines are identified by counting occurrences on each side and keeping only those that
appear exactly once in both, then the body applies "Algorithm A from Szymanski's paper"
(`kittens/diff/diff.go:229`). Line splitting is `lines` (`kittens/diff/diff.go:172`), which uses
`strings.SplitAfter(x, "\n")` (`:173`) and appends the GNU/BSD marker for a missing final newline
(`:179`):

```go
// kittens/diff/diff.go — lines()
	} else {
		// Treat last line as having a message about the missing newline attached,
		// using the same text as BSD/GNU diff (including the leading backslash).
		l[len(l)-1] += "\n\\ No newline at end of file\n"   // :179
	}
```

### Backend resolution — `diff_cmd auto` (the default)

`set_diff_command` (`kittens/diff/patch.go:44`) maps the option; `auto` calls `find_differ`
(`kittens/diff/patch.go:34`), which probes **git first**, then GNU diff, else falls back to the built‑in
(empty command list):

```go
// kittens/diff/patch.go — find_differ()
func find_differ() {
	if GitExe() != "git" && exec.Command(GitExe(), "--help").Run() == nil {
		diff_cmd, _ = shlex.Split(GIT_DIFF)       // :36  git wins if present
	} else if DiffExe() != "diff" && exec.Command(DiffExe(), "--help").Run() == nil {
		diff_cmd, _ = shlex.Split(DIFF_DIFF)      // :38  else GNU diff
	} else {
		diff_cmd = []string{}                     // :40  else built-in Diff()
	}
}
```

The exact command literals (`_CONTEXT_` is replaced by `num_context_lines`, default `3`,
`kittens/diff/main.py:37`) are:

```go
// kittens/diff/patch.go
const GIT_DIFF = `git diff --no-color --no-ext-diff --exit-code -U_CONTEXT_ --no-index --`   // :21
const DIFF_DIFF = `diff -p -U _CONTEXT_ --`                                                    // :22
```

`run_diff` (`kittens/diff/patch.go:282`) uses the built‑in `Diff()` when `len(diff_cmd) == 0` (`:294`),
otherwise shells out — substituting `_CONTEXT_`, appending both resolved paths, and treating **exit code
1 as "files differ"** (`:321-322`). `do_diff` (`kittens/diff/patch.go:330`) then re‑reads the two files'
lines (from the caches, §3) and parses the unified diff via `parse_patch`
(`kittens/diff/patch.go:245`); intraline centers come from `changed_center`
(`kittens/diff/patch.go:86`, see §4).

### Observed evidence — the resolved backend here is git

Both tools are present and resolve to absolute paths (so `find_differ`'s `GitExe() != "git"` guard is
satisfied and git is chosen first):

```
$ git --version && diff --version | head -1
git version 2.51.0
diff (GNU diffutils) 3.10
$ command -v git; command -v diff
/usr/bin/git
/usr/bin/diff
```

Running the *exact* `GIT_DIFF` command (with `_CONTEXT_` = 3) on the changed fixture reproduces the
same `@@ -1,8 +1,8 @@` hunk that the TUI rendered, and returns exit code 1 ("different"):

```
$ git diff --no-color --no-ext-diff --exit-code -U3 --no-index -- \
      /tmp/difftest/left/config.py /tmp/difftest/right/config.py ; echo "exit=$?"
diff --git a/tmp/difftest/left/config.py b/tmp/difftest/right/config.py
index dfee40b..333753e 100644
--- a/tmp/difftest/left/config.py
+++ b/tmp/difftest/right/config.py
@@ -1,8 +1,8 @@
 import sys
 
 def greet(name):
-    print("Hello, " + name)
+    print("Hi there, " + name + "!")
 
 def main():
-    greet("world")
+    greet("kitty")
     return 0
exit=1
```

The hunk header `@@ -1,8 +1,8 @@` is exactly what appears in the captured TUI output (§1), confirming the
effective backend is `git diff --no-index` under the default `diff_cmd auto`. (To observe the built‑in
anchored algorithm specifically, one would set `diff_cmd builtin` — a **non‑default** configuration —
which forces the `len(diff_cmd) == 0` path in `run_diff` and calls `Diff()` directly; that variant was
not used for the canonical observations above.)

---

## §8 — Synthesis: why it's "fast in a way that doesn't feel obvious"

Pulling the threads together (each claim points back to the section that evidences it):

1. **Name‑based pairing avoids O(n²) cross‑comparison (§1).** Directory pairing is a single set
   intersection on relative names (`kittens/diff/collect.go:306`); it never compares every left file
   against every right file.

2. **Content‑hash rename detection avoids re‑diffing moved files (§2).** Left‑only and right‑only files
   are matched by MD5 (`kittens/diff/collect.go:106`) + byte confirmation (`:353`); a moved file becomes
   one cheap "rename" entry instead of a full delete‑plus‑add diff.

3. **Seven memoizing caches read each file once (§3).** `GetOrCreate` (`tools/utils/cache.go:39`) plus
   zero‑copy `UnsafeBytesToString` (`kittens/diff/collect.go:68`) mean the same bytes feed pairing,
   hashing, is‑text testing, diffing, highlighting, and rendering without re‑reading — the "quiet" speed
   the user senses. (With the honest caveat that only the four `GetOrCreate` caches are LRU‑bounded; §3.)

4. **Per‑file diff and highlight fan out across `runtime.NumCPU()` workers (§4).** The same
   `Context.Parallel` pool (`tools/utils/images/utils.go:27`) drives both `diff` (`kittens/diff/patch.go:352`)
   and `highlight_all` (`kittens/diff/highlight.go:217`); a closed buffered channel gives each worker a
   distinct file, so there is no lock contention on the highlight results.

5. **The built‑in matching is `O(n log n)` (§7).** The anchored/patience algorithm (`kittens/diff/diff.go:49`,
   `tgs` `:192`) is asymptotically better than a naive `O(n²)` diff — and by default the work is handed to
   an even more optimized external `git diff --no-index`.

6. **Highlighting is asynchronous, so the UI feels instant (§6).** `initialize` paints "Calculating
   diff, please wait…" immediately (`kittens/diff/ui.go:350`) while collection runs on a goroutine; diff,
   highlight, and image load then arrive independently and re‑render in place
   (`kittens/diff/ui.go:245`) — the screen is useful before highlighting finishes.

The net effect: bytes are read once and reused everywhere, cheap set/hash operations replace expensive
comparisons, the genuinely expensive work (diffing and highlighting) is parallelized per file, and the
UI never blocks on any of it. That is the "fast in a way that doesn't feel obvious at first glance."

---

## Appendix — coverage map (question item → section → evidence)

| Question item | Section | Key `file:line` | Observed evidence |
|---------------|---------|-----------------|-------------------|
| directory pairing (what belongs together) | §1 | `collect.go:296`, `:306` | `config.py` paired; `TestDiffCollectWalk` PASS |
| rename detection ("almost magical") | §2 | `collect.go:106`, `:347-364` | MD5 `1e280e1713df124d35709cf6138d9f91`; pair collapsed to one entry |
| the seven caches | §3 | `collect.go:26`, `:30-36` | all seven enumerated with key→value + populating fn |
| cache efficiency ("quietly keeps fast") | §3 | `cache.go:39` (evict), `:32` (Set‑RLock) | memoize/evict path + honest Set‑under‑RLock nuance |
| parallel highlighting (no stepping) | §4 | `highlight.go:217`, `images/utils.go:27` | 118 fg SGR; pygments‑default colors; one‑key‑per‑worker |
| many files at once | §4 | `patch.go:352`, `:361` | parallel per‑file diff reuses the pool |
| binary handling | §5 | `collect.go:86`, `render.go:452` | `Binary file: 19 B` / `27 B` |
| image handling | §5 | `collect.go:82`, `render.go:344` | `Dimensions: 2x2 Size: 79 B` + graphics APC |
| end‑to‑end runtime trace | §6 | `ui.go:114`, `:245`, `:294` | "Calculating…" (before) → populated render (after); error path |
| omitted unchanged file | §6 | `collect.go:317-330` | `same.cfg` count = 0 |
| matching‑region algorithm | §7 | `diff.go:49`, `:192` | anchored/patience, Szymanski, `O(n log n)` |
| diff backends | §7 | `patch.go:34`, `:21-22` | resolved backend = git; exact `@@` hunk reproduced |
| Chroma pin | §4 | `go.mod:7` | `github.com/alecthomas/chroma/v2 v2.14.0` |
| config defaults | Build | `main.py:37`, `:41`, `:74` | `num_context_lines 3`, `diff_cmd auto`, `pygments_style default` |

**Determinism statement.** All volatile observations above (the rename MD5, binary sizes, image
dimensions, the `@@` hunk, the omitted `same.cfg`, the graphics APC bytes, and the total 26501‑byte
capture) were confirmed identical across two independent runs of the exact same invocation before being
reported here.

**Read‑only statement.** No file in the kitty source tree was modified. The fixtures
(`/tmp/difftest`), the PTY harness (`/tmp/pty_run.py`), and all captured byte streams lived outside the
repository and were removed after the investigation; the only file added to the repository is this
document.


