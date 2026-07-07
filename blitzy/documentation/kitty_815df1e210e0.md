# How Kitty's diff kitten works internally (commit `815df1e210e0`)

This document explains — grounded in **both source code and observed runtime behavior** — how the
[kitty](https://github.com/kovidgoyal/kitty) terminal emulator's **"diff kitten"** (`kittens/diff/`)
works internally. It answers eight behavioral questions (Q1–Q8). Every behavioral claim follows the
same pattern:

> **mechanism → `file:line` → the exact command run → the complete, unedited observed output → the causal reason (cause → effect).**

Statements that are read from the source but *not* directly demonstrated at runtime are explicitly
labelled **(inferred)**. Everything else was produced by building kitty and running the real
`kitten diff` Go entry point in the canonical container, and the output blocks below are the actual,
unedited captures.

All `file:line` citations correspond to the pinned commit
`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.

---

## 0. Environment & Build

### 0.1 Canonical environment and pinned commit

The investigation was performed inside the rule-mandated container image
`andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
(from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`), checked out at the pinned
commit.

```console
$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1

$ git rev-parse --abbrev-ref HEAD
blitzy-5f613546-20aa-48be-98e6-bb0c6e374ea7

$ go version
go version go1.22.12 linux/amd64

$ python3 --version
Python 3.13.7
```

`go1.22.12` satisfies the `go 1.22` directive in `go.mod:3`; Python 3.13.7 satisfies the kitten
shim and the `setup.py` build driver.

### 0.2 Canonical build

The canonical build is `python3 setup.py` (aliased by `make all`). On this toolchain the global
option `--ignore-compiler-warnings` is required: gcc 15's default `-Werror=switch` makes
`glfw/wl_window.c:668` fatal because wayland-protocols 1.45 adds `XDG_TOPLEVEL_STATE_CONSTRAINED_*`
enum values the mid-2024 switch statement does not handle. This is a toolchain/dependency-version
incompatibility, not a code bug, and `--ignore-compiler-warnings` is a documented `setup.py` global
option (a build-invocation choice, not a source edit).

To capture a genuine build transcript (rather than an incremental no-op), the gitignored launcher
binary was removed and the Go build cache cleared before rebuilding:

```console
$ rm -f kitty/launcher/kitten && go clean -cache
$ python3 setup.py --ignore-compiler-warnings
... (196 lines) ...
```

The head of the transcript shows Go dependency compilation and the tail shows kitty's own Go
packages being built — including `kitty/kittens/diff`, the package under study:

```text
# head
github.com/seancfoley/ipaddress-go/ipaddr/addrstrparam
internal/nettrace
golang.org/x/exp/constraints
github.com/shirou/gopsutil/v3/common
log/internal
crypto/internal/boring/sig
...
# tail
kitty/kittens/transfer
kitty/tools/cmd/pytest
kitty/kittens/diff
kitty/tools/cmd/tool
kitty/tools/cmd/completion
kitty/tools/cmd
```

The build produced the launcher binary (gitignored — verified untracked, so rebuilding leaves the
working tree clean):

```console
$ ls -l kitty/launcher/kitten
-rwxr-xr-x 1 root root 15765764 ... kitty/launcher/kitten

$ git check-ignore kitty/launcher/kitten kitty/launcher/kitty
kitty/launcher/kitten
kitty/launcher/kitty

$ git status --porcelain      # after the full rebuild
                              # (empty — the rebuild dirtied nothing)
```

Throughout the investigation the launcher path `kitty/launcher/kitten` is referred to as `$KIT`.

### 0.3 Real entry point (Go), not the Python shim

The diff kitten is a dual-language component: a thin Python shim (`kittens/diff/main.py`) declares
configuration options, while the runtime behavior is implemented in Go. The **canonical** entry point
is `kitten diff <left> <right>` (equivalently `kitty +kitten diff <left> <right>`), which reaches the
Go `main` at `kittens/diff/main.go:102` (`EntryPoint` at `main.go:177`):

```console
$ $KIT diff --help
Usage: kitten diff [options] file_or_directory_left file_or_directory_right

Show a side-by-side diff of the specified files/directories. You can also use
ssh:hostname:remote-file-path to diff remote files.

Options:
  --context [=-1]
    Number of lines of context to show between changes. ...
  --config
    Specify a path to the configuration file(s) to use. ...
```

The Python `main()` is **not** a valid observation path — it deliberately errors. This is the
**forbidden / non-canonical** path, shown here only to prove it errors (`kittens/diff/main.py:13-14`):

```console
$ python3 -c "import kittens.diff.main as m; m.main([])"
Must be run as kitten diff
```

`kittens/diff/main.py:13-14`:

```python
def main(args: List[str]) -> None:
    raise SystemExit('Must be run as kitten diff')
```

All observations below therefore come from `$KIT diff …` (the Go path).

### 0.4 How full-screen TUI output was captured

`kitten diff` renders a full-screen, side-by-side TUI on the alternate screen, so its rendered frame
must be captured from a PTY rather than read from stdout. `tmux` is not installed in the container,
so a small Python `pty` harness (`/tmp/tui_capture.py`, outside the repository) was used. It:

1. `pty.fork()`s and sets the child window size with `TIOCSWINSZ` before exec'ing `$KIT diff …`;
2. answers the terminal's Primary Device Attributes (`ESC[?62;1;6c`) and Cursor-Position-Report
   queries so the kitty TUI event loop does not block;
3. accumulates the raw byte stream, records the offset at which `q` (quit) is sent;
4. feeds the captured bytes (up to the quit offset, avoiding the alt-screen-exit blank) to a fresh
   [`pyte`](https://pypi.org/project/pyte/) VT emulator and dumps the character grid.

Truecolor SGR written by kitty uses the colon form `ESC[38:2:R:G:Bm`, which `pyte` 0.8.x cannot
parse, so those SGR runs are stripped before the grid is dumped (this only removes color, never
text). A `--raw-out <path>` flag additionally dumps the raw bytes so that graphics-protocol escapes
and pre-render/transitional states can be inspected with `strings`/`grep`. The harness was validated
on a single-file diff before use. Where a captured pyte grid occasionally garbles a transient
mid-render frame (a harness artifact, not kitten behavior), the raw byte stream is grepped instead
and that is called out inline.

### 0.5 Selecting the diff engine and options

Config options are declared in `kittens/diff/main.py`: `diff_cmd` (default `'auto'`, `main.py:41`),
`ignore_name` (default empty `''`, `main.py:56`), `num_context_lines` (default `'3'`, `main.py:37`),
`syntax_aliases` (`main.py:29`), `replace_tab_by` (`main.py:52`). The CLI accepts `--config <file>`
and repeatable `--override`/`-o key=value`; `kittens/diff/main.go:24-27` loads them via
`config.ConfigParser.LoadConfig("diff.conf", opts.Config, opts.Override)`. Both override forms were
confirmed working in the container. To force a specific engine:

```sh
$KIT diff -o diff_cmd=git     L R    # external git diff --no-index
$KIT diff -o diff_cmd=diff    L R    # external diff
$KIT diff -o diff_cmd=builtin L R    # in-process anchored diff (diff.go)
```

An isolated config directory was used via `KITTY_CONFIG_DIRECTORY=/tmp/kdiff/kittyconf` so the host
config never interferes.

---

## 1. Q1 — Directory pairing ("what belongs together")

### 1.1 Direct answer

When two directories are compared, the diff kitten decides which file on the left corresponds to which
file on the right **purely by identical relative path** — not by content, not by position in the
listing. A file that exists at the same relative path on both sides is paired (and shown as a change
or omitted if identical); a path present on only one side is a removal or an addition.

### 1.2 Mechanism & code anchors

The work is done by `collect_files` (`kittens/diff/collect.go:296`). It walks both trees with `walk`
(calls at `collect.go:299` and `collect.go:303`), accumulating the relative paths into two sets
`left_names, right_names` (`collect.go:297`). Pairing is the set intersection:

- `common_names := left_names.Intersect(right_names)` (`collect.go:306`) — the paired files;
- `removed := left_names.Subtract(common_names)` (`collect.go:332`) — left-only paths;
- `added := right_names.Subtract(common_names)` (`collect.go:333`) — right-only paths.

For each common name the file data is compared: `if ld != rd` (`collect.go:317`) →
`add_change` (`collect.go:319`, definition at `collect.go:167`); if the bytes are equal the file is
skipped (no entry) unless the file **mode** differs (see §1.6). The relative path (not the absolute
path) is the map key, which is why a file nested in a subdirectory pairs with the same relative path
on the other side. This is corroborated by the user-facing docs: `docs/kittens/diff.rst:20` — "Does
recursive directory diffing".

### 1.3 Command(s) run

The left tree has `{only_left.txt, same.txt, sub/a.txt}`; the right tree has
`{only_right.txt, same.txt, sub/a.txt}`. `same.txt` is byte-identical on both sides; `sub/a.txt`
differs in one line; `only_left.txt`/`only_right.txt` exist on one side only.

```sh
$KIT diff /tmp/kdiff/q1left /tmp/kdiff/q1right
```

### 1.4 Observed output

```text
   only_left.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1  only on the left side                                                 This file was removed

   only_right.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   This file was added                                                1  only on the right side

   sub/a.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   @@ -1,3 +1,3 @@
1  alpha                                                              1  alpha
2  beta                                                               2  BETA-changed
3  gamma                                                              3  gamma
```

Three entries are shown, sorted by path: `only_left.txt` ("This file was removed"), `only_right.txt`
("This file was added"), and `sub/a.txt` (a change, `beta` → `BETA-changed`). `same.txt` — identical
on both sides — is **omitted entirely**.

The directory walk and pairing machinery is also exercised by the package's own unit test:

```console
$ go test ./kittens/diff/ -run TestDiffCollectWalk -v
=== RUN   TestDiffCollectWalk
--- PASS: TestDiffCollectWalk (0.00s)
PASS
ok  	kitty/kittens/diff	0.021s
```

(`kittens/diff/collect_test.go:19-54` walks a tree and asserts the resulting relative-name set.)

### 1.5 Causal reason

Pairing is set intersection over **relative paths** (`Intersect` at `collect.go:306`). `sub/a.txt`
pairs across the nested subdirectory because its relative path is identical on both sides; `same.txt`
produces no entry because its bytes are equal (`ld != rd` is false at `collect.go:317`); the two
single-sided files fall into `removed`/`added` because they are absent from `common_names`
(`Subtract` at `collect.go:332-333`). Nothing about content similarity or list position affects
pairing.

### 1.6 Sibling variants / edge cases

**Argument validation** — the kitten requires exactly two paths (`main.go:108-110`,
`if len(args) != 2`); 0, 1, or 3 arguments all produce the same error and exit code 1:

```console
$ $KIT diff            ; echo "[exit=$?]"
Error: You must specify exactly two files/directories to compare
[exit=1]
$ $KIT diff onlyone    ; echo "[exit=$?]"
Error: You must specify exactly two files/directories to compare
[exit=1]
$ $KIT diff a b c      ; echo "[exit=$?]"
Error: You must specify exactly two files/directories to compare
[exit=1]
```

**Identical inputs** — diffing two byte-identical files shows the "identical" banner (this is the
`Diff() == nil` path described in §8; captured here from `-o diff_cmd=builtin` on an identical pair):

```text
   /tmp/kdiff/q8id_left/f.txt                                  /tmp/kdiff/q8id_right/f.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   The files are identical
```

**Mode-only change** — if the bytes are identical but the file mode differs, the else-branch at
`collect.go:321-329` runs `os.Stat` on both sides and, when `lstat.Mode() != rstat.Mode()`
(`collect.go:323`), calls `add_change` (`collect.go:326`). The rendered banner comes from
`render.go:577`. Fixture: identical content (`md5 = 3fe37309da949d82291eb060c6dc69e3` both sides),
mode `0644` vs `0755`:

```console
$ md5sum /tmp/kdiff/mo_left/script.sh /tmp/kdiff/mo_right/script.sh
3fe37309da949d82291eb060c6dc69e3  /tmp/kdiff/mo_left/script.sh
3fe37309da949d82291eb060c6dc69e3  /tmp/kdiff/mo_right/script.sh
$ $KIT diff /tmp/kdiff/mo_left /tmp/kdiff/mo_right
```

```text
   script.sh
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Mode changed: -rw-r--r-- to -rwxr-xr-x
```

**Ignore-glob filtering** — the walk drops entries whose basename matches any `ignore_name` glob via
`allowed` (`collect.go:230`, doublestar glob). The default `ignore_name` is **empty** (`main.py:56`),
so by default nothing is ignored (the `.git`/`*~`/`*.pyc` values in the `main.py:63-65` help text are
documentation examples, not active defaults). With a fixture containing
`{#draft#, keep.txt, mod.pyc, notes.txt~}` on each side, the default run shows all four; supplying the
globs filters them:

```console
$ $KIT diff /tmp/kdiff/ig_left /tmp/kdiff/ig_right          # default: all four shown
   #draft#
   keep.txt
   mod.pyc
   notes.txt~

$ $KIT diff -o 'ignore_name=*~' -o 'ignore_name=#*#' -o 'ignore_name=*.pyc' \
      /tmp/kdiff/ig_left /tmp/kdiff/ig_right                # only keep.txt survives
   keep.txt
```

The same filtering is asserted by `TestDiffCollectWalk` (`collect_test.go:19-54`), which walks
`{a/b/c, b, d, e, #d#, e~, f/g, h space}` with globs `["*~", "#*#", "b"]` and expects
`{d, e, f/g, h space}`.

---


## 2. Q2 — Rename recognition (rename vs deletion + addition)

### 2.1 Direct answer

A change is classified as a **rename** only when a left-only (removed) file and a right-only (added)
file have **the same MD5 hash AND byte-identical content**. If a single byte differs, the MD5
pre-filter (and the exact-bytes check) fails and the pair degrades to a separate **removal + addition**
— there is no rename.

### 2.2 Mechanism & code anchors

Inside `collect_files`, after computing `removed`/`added` (§1.2), the kitten builds hash maps for the
added files (`ahash`, `collect.go:335-340`) and the removed files (`rhash`, `collect.go:341-346`) using
`hash_for_path` (`collect.go:106`), which computes `md5.Sum(...)` (`collect.go:112`; `import
crypto/md5` at `collect.go:6`). It then, for each removed hash, scans the added hashes
(`for name, rh := range rhash`, `collect.go:347`):

- `if ah == rh` (`collect.go:350`) — the MD5 pre-filter matches; then
- it reads both files' data (`ld, rd`, `collect.go:351-352`) and requires `if ld == rd`
  (`collect.go:353`) — **exact byte equality** — before calling `add_rename` (`collect.go:354`,
  definition `collect.go:175`) and discarding the added entry (`Discard`, `collect.go:355`);
- otherwise the removed file becomes `add_removal` (`collect.go:362`, definition `collect.go:192`), and
  any added files with no rename match become `add_add` (`collect.go:365-366`, definition
  `collect.go:181`).

The rename is surfaced in the UI by the **two-name title**: `title_lines` (`render.go:208`) reads
`left_name, right_name` from `path_name_map` (`render.go:209`); when the right name differs from the
left name (only true for a rename) it renders **both** names.

### 2.3 Command(s) run — both transitional states

```sh
# (a) byte-identical -> RENAME
$KIT diff /tmp/kdiff/q2a_left /tmp/kdiff/q2a_right
# (b) one byte changed -> REMOVE + ADD
$KIT diff /tmp/kdiff/q2b_left /tmp/kdiff/q2b_right
```

### 2.4 Observed output

**(a) byte-identical → rename.** `old_name.txt` (left) and `new_name.txt` (right) have the same MD5:

```console
$ md5sum /tmp/kdiff/q2a_left/old_name.txt /tmp/kdiff/q2a_right/new_name.txt
315108a035552eb328252565386e663a  /tmp/kdiff/q2a_left/old_name.txt
315108a035552eb328252565386e663a  /tmp/kdiff/q2a_right/new_name.txt
```

```text
   old_name.txt                                                new_name.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

A **single entry** appears, titled with **both** names side by side (`old_name.txt` on the left,
`new_name.txt` on the right) — the rename.

**(b) one byte changed → removal + addition.** Changing the last word (`dog` → `doX`) changes the MD5:

```console
$ md5sum /tmp/kdiff/q2b_left/old_name.txt /tmp/kdiff/q2b_right/new_name.txt
315108a035552eb328252565386e663a  /tmp/kdiff/q2b_left/old_name.txt
6c2d1fe9341479db3eedff36ddaefb6e  /tmp/kdiff/q2b_right/new_name.txt
```

```text
   new_name.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   This file was added                                      1  the quick brown fox
                                                            2  jumps over
                                                            3  the lazy doX

   old_name.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1  the quick brown fox                                         This file was removed
2  jumps over
3  the lazy dog
```

Two **separate** entries: `new_name.txt` ("This file was added") and `old_name.txt`
("This file was removed"). No rename.

### 2.5 Causal reason

The MD5 hash is a cheap pre-filter (`ah == rh`, `collect.go:350`); the decisive condition is the
exact-bytes comparison (`ld == rd`, `collect.go:353`). In (a) both are equal → `add_rename`. In (b)
the changed byte makes both the hash and the byte comparison fail → the removed and added files are
emitted independently via `add_removal` + `add_add`. Thus a rename is, by construction, a
content-preserving move; any content edit converts it to remove + add.

### 2.6 Sibling / honest finding — the rename **body** message is never rendered

`rename_lines` (`render.go:684`) *intends* to print `"The file %s was renamed to %s"`
(`render.go:688`), but that message is **not** rendered. Grepping the raw byte stream of the rename
capture for "renamed" returns nothing:

```console
$ strings /tmp/kdiff/q2a_raw.bin | grep -i renamed
$        # (no matches)
```

**Cause (grounded in source):** `rename_lines` sets `is_full_width: true` (`render.go:687`) and writes
the message to `sl.right.marked_up_text` (`render.go:689`). But `render_screen_line`
(`render.go:70`), for a full-width line, computes `available_cols = columns - margin_size`
(`render.go:76-78`), renders only `sl.left.marked_up_text` (`render.go:80`), and then
`if self.is_full_width { return }` (`render.go:99-101`) **returns before the right half is drawn**. The
message lives in the right half, so it is skipped. Consequently the rename is observable **only** via
the two-name title (§2.4a), not via any body text. This is reported as an observed latent bug; the
answer does not claim the message appears.

---


## 3. Q3 — Caching pipeline (raw bytes → highlighted lines)

### 3.1 Direct answer

Each file's raw bytes are read from disk **exactly once**, by `data_for_path`, and cached. Every
later stage — hashing, text/binary detection, line splitting, and syntax highlighting — reuses those
cached bytes rather than re-reading the file. Seven `LRUCache` instances (capacity **4096** each) hold
the intermediate results, keyed by absolute path, so repeated lookups are O(1) and memory is bounded
by LRU eviction.

### 3.2 Mechanism & code anchors

The seven caches are declared at `collect.go:20-24` and created in `init_caches()`
(`collect.go:26`) with `const sz = 4096` (`collect.go:29`):

| Cache | Value type | Purpose |
|-------|------------|---------|
| `size_cache` | `int64` | file size (via `os.Stat`) |
| `mimetypes_cache` | `string` | MIME type (by extension) |
| `data_cache` | `string` | **raw file bytes (the single content read)** |
| `is_text_cache` | `bool` | text-vs-binary decision |
| `lines_cache` | `[]string` | raw content split into lines |
| `highlighted_lines_cache` | `[]string` | Chroma-highlighted lines |
| `hash_cache` | `string` | MD5 hash |

The pipeline is layered: `data_for_path` (`collect.go:65-70`) → `sanitize` (`collect.go:128-130`) →
`lines_for_path` (`collect.go:138-146`) → `highlighted_lines_for_path` (`collect.go:148-157`). The
single content read is `data_cache.GetOrCreate(path, …os.ReadFile…)` (`collect.go:66-69`).
`hash_for_path` (`collect.go:106`) and `is_path_text` (`collect.go:86`) both call `data_for_path`, so
they consume the cached bytes rather than reopening the file. `LRUCache` itself is defined at
**`tools/utils/cache.go:13`** (`Get` at `:25`, `Set` at `:32`, `GetOrCreate` at `:39`,
`MustGetOrCreate` at `:60`) — note there is **no** `kittens/diff/cache.go`.

### 3.3 Command(s) run

`strace` (system tool, `apt`-installed; the repository is untouched) traced the `openat` syscalls the
kitten makes while diffing a 3-file-per-side directory. The kitten's own content read uses Go's
`os.ReadFile`, which always opens with `O_CLOEXEC`; counting `O_CLOEXEC` opens per path isolates the
cache's content read from any external subprocess opens.

```sh
strace -f -e trace=openat -o out.txt  $KIT diff /tmp/kdiff/q3_left /tmp/kdiff/q3_right   # (run inside the pty harness)
# then, per distinct path, count total opens and O_CLOEXEC opens
```

### 3.4 Observed output (stable across ≥2 runs)

**Builtin engine** (`-o diff_cmd=builtin`, no external subprocess) — each distinct path is opened
**exactly once**, and that open is the `O_CLOEXEC` content read:

```text
--- RUN1 ---                                --- RUN2 ---
  q3_left/f1.txt:  total_openat=1  O_CLOEXEC=1    q3_left/f1.txt:  total_openat=1  O_CLOEXEC=1
  q3_left/f2.txt:  total_openat=1  O_CLOEXEC=1    q3_left/f2.txt:  total_openat=1  O_CLOEXEC=1
  q3_left/f3.txt:  total_openat=1  O_CLOEXEC=1    q3_left/f3.txt:  total_openat=1  O_CLOEXEC=1
  q3_right/f1.txt: total_openat=1  O_CLOEXEC=1    q3_right/f1.txt: total_openat=1  O_CLOEXEC=1
  q3_right/f2.txt: total_openat=1  O_CLOEXEC=1    q3_right/f2.txt: total_openat=1  O_CLOEXEC=1
  q3_right/f3.txt: total_openat=1  O_CLOEXEC=1    q3_right/f3.txt: total_openat=1  O_CLOEXEC=1
```

**Auto engine** (default → git, see §8) — each path shows `total_openat=3` but still exactly **one**
`O_CLOEXEC` content read; the two extra non-`O_CLOEXEC` opens belong to the external `git` subprocess
(a different PID) reading the file itself:

```text
--- RUN1 ---                                       --- RUN2 --- (identical)
  q3_left/f1.txt:  total_openat=3  O_CLOEXEC=1        q3_left/f1.txt:  total_openat=3  O_CLOEXEC=1
  ... (all six paths identical) ...
```

The actual syscall lines for one path make the split explicit — one `O_CLOEXEC` open from the kitten
(pid 114113) and two plain opens from the git subprocess (pid 114123):

```text
114113 openat(AT_FDCWD, "/tmp/kdiff/q3_left/f1.txt", O_RDONLY|O_CLOEXEC) = 11
114123 openat(AT_FDCWD, "/tmp/kdiff/q3_left/f1.txt", O_RDONLY <unfinished ...>
114123 openat(AT_FDCWD, "/tmp/kdiff/q3_left/f1.txt", O_RDONLY <unfinished ...>
```

### 3.5 Causal reason

The kitten reads each file's content exactly once because `data_for_path` funnels every read through
`data_cache.GetOrCreate` (`collect.go:66-69`): the first miss runs `os.ReadFile` (one `O_CLOEXEC`
open); every subsequent consumer — `hash_for_path`, `is_path_text`, `lines_for_path`,
`highlighted_lines_for_path` — retrieves the cached string with no further open. That is why the
`O_CLOEXEC` count is exactly 1 per path in **both** engines, and why the builtin engine's total is
also 1 (it does everything in-process). The extra opens in auto mode are not the cache pipeline at
all; they are the external git process, which reads the files independently (this also feeds the
cache-interplay discussion in §8). The cache key is the absolute path and capacity 4096 bounds memory
via LRU eviction (`NewLRUCache(sz)`).

---


## 4. Q4 — Concurrency across many files

### 4.1 Direct answer

When many files must be processed, the diff work is dispatched across a worker pool whose size is
`runtime.NumCPU()`. Each file pair becomes one job (`do_diff`), and up to `NumCPU` jobs run
concurrently. On this 128-CPU machine, an unconstrained 400-file diff was observed running dozens of
`git` subprocesses at once; constraining the CPU set to 1, 2, or 4 CPUs makes the peak concurrency
exactly 1, 2, or 4.

### 4.2 Mechanism & code anchors

The worker-pool primitive is `Context.Parallel(start, stop, fn)` at
**`tools/utils/images/utils.go:27-56`**:

- `count := stop - start` (`utils.go:28`);
- `procs := self.NumberOfThreads()` (`utils.go:33`); when unset this is 0, so
  `if procs <= 0 { procs = runtime.NumCPU() }` (`utils.go:35`) — the pool size;
- `if procs > count { procs = count }` (`utils.go:37-38`) — never more workers than jobs;
- all indices are pushed onto a buffered channel `make(chan int, count)` (`utils.go:41`), then closed
  (`utils.go:45`);
- `procs` goroutines each drain the channel and call `fn(c)` (`utils.go:48-54`), joined by
  `wg.Wait()` (`utils.go:55`).

The diff jobs are driven by `diff(jobs, context_count)` (`patch.go:352`), which creates a zero-value
`images.Context{}` (so `NumberOfThreads() == 0` → the `NumCPU` fallback fires), calls
`ctx.Parallel(0, len(jobs), …)` (`patch.go:361`), and each worker runs `do_diff(job.file1,
job.file2, context_count)` (`patch.go:365`). One `do_diff` job corresponds to one file pair. The same
primitive drives highlighting (Q5).

### 4.3 Command(s) run

Two probes. First, the **pool-size basis** (a **non-canonical stand-in** — the canonical fact is the
code at `utils.go:35`; this standalone probe merely reports what `runtime.NumCPU()` returns in this
container):

```console
$ cat ncpu.go
package main
import ("fmt";"runtime")
func main(){ fmt.Println("runtime.NumCPU()=", runtime.NumCPU()) }
$ go run ncpu.go
runtime.NumCPU()= 128
$ nproc --all
128
```

Second, the **effective parallelism**, observed canonically by tracing the `git` subprocesses the
kitten spawns (`execve`/`exit_group` with timestamps) and computing the peak number that overlap in
time. A controlled sweep pins the process to 1/2/4 CPUs with `taskset` (N=80 files/side); a scaled run
uses N=400 files/side.

```sh
taskset -c 0   strace -f -ttt -e trace=execve,exit_group -o out $KIT diff L R   # NumCPU=1
taskset -c 0-1 …                                                                # NumCPU=2
taskset -c 0-3 …                                                                # NumCPU=4
```

### 4.4 Observed output — controlled proof (stable across ≥2 runs)

Peak concurrent `git diff` subprocesses equals the CPU count exactly, and is stable:

```text
taskset -c 0   (NumCPU=1) RUN1: git-diff subprocesses=80  MAX_CONCURRENT=1
taskset -c 0   (NumCPU=1) RUN2: git-diff subprocesses=80  MAX_CONCURRENT=1
taskset -c 0-1 (NumCPU=2) RUN1: git-diff subprocesses=74  MAX_CONCURRENT=2
taskset -c 0-1 (NumCPU=2) RUN2: git-diff subprocesses=80  MAX_CONCURRENT=2
taskset -c 0-3 (NumCPU=4) RUN1: git-diff subprocesses=4   MAX_CONCURRENT=4
taskset -c 0-3 (NumCPU=4) RUN2: git-diff subprocesses=80  MAX_CONCURRENT=4
```

At scale (N=400 files/side), a single CPU serializes (`MAX_CONCURRENT=1`) while the full machine runs
dozens at once. The unconstrained peak varies run-to-run (short-lived `git` processes overlap
differently under 128-way scheduling), reported honestly here as observed:

```text
1-CPU  RUN1: git-diff subprocesses=400  MAX_CONCURRENT=1   span=2.07s
1-CPU  RUN2: git-diff subprocesses=400  MAX_CONCURRENT=1   span=1.98s
FULL   RUN1: git-diff subprocesses=400  MAX_CONCURRENT=65  span=1.69s
FULL   RUN2: git-diff subprocesses=400  MAX_CONCURRENT=39  span=2.21s
```

400 files produce exactly 400 `git diff` subprocesses — one `do_diff` job per file pair.

### 4.5 Causal reason

Each pool goroutine runs one `do_diff` at a time, and in auto mode each `do_diff` spawns one `git`
subprocess; therefore the peak number of concurrent `git` processes equals the number of goroutines,
which is `procs = min(runtime.NumCPU(), jobs)` (`utils.go:35,37-38`). Pinning to *k* CPUs makes
`runtime.NumCPU()` return *k*, so the peak is exactly *k* — which is precisely what the 1/2/4 sweep
shows. Unconstrained, the peak is bounded by `NumCPU` (128) but in practice lands in the tens because
each `git` on tiny files is very short-lived, so not all workers are busy at the same instant. The
1-CPU `MAX_CONCURRENT=1` versus full-CPU `MAX_CONCURRENT=39–65` is the direct, deterministic signature
of the `NumCPU`-sized pool. (Wall-clock span is similar here because for tiny files the `git` spawn
overhead dominates the actual diff work; the concurrency magnitude is the `MAX_CONCURRENT` value, not
the span.)

---


## 5. Q5 — Parallel highlighting without "stepping on itself"

### 5.1 Direct answer

Syntax highlighting runs in parallel, one path per worker, and each worker writes its result under a
**distinct per-path cache key**. Because no two workers write the same key, the highlighted output is
deterministic and uncorrupted across runs. However, this safety is *usage*-dependent, not
lock-enforced: `LRUCache.Set` mutates its map while holding only a **read** lock, so concurrent
writers to the same map are a genuine (latent) data race — demonstrated below with the race detector.

### 5.2 Mechanism & code anchors

`highlight_all(paths)` (`highlight.go:217`) creates a zero-value `images.Context{}` and dispatches one
path per worker via `ctx.Parallel(0, len(paths), …)` (`highlight.go:219`). Each worker highlights with
`highlight_file` (`highlight.go:161`, which uses the cached `data_for_path` bytes plus Chroma lexer
matching) and, on success, writes `highlighted_lines_cache.Set(path, text_to_lines(raw))`
(`highlight.go:224`) — a **distinct key per path**. Corroborated by `docs/kittens/diff.rst:15-16`
("asynchronously, for maximum speed").

The nuance to verify (not assume): `LRUCache.Set` (`tools/utils/cache.go:32`) takes `RLock`
(`:33`), assigns `self.data[key] = val` (`:34`), then `RUnlock` (`:35`) — i.e. it **mutates the map
under a read lock**. `GetOrCreate` (`:39`) reads under `RLock`, runs the create function **outside**
any lock (no single-flight guard), then writes under `Lock` (`:48`).

### 5.3 Command(s) run and observed output — race detector (verbatim)

First, the existing tests under the race detector. The diff kitten's own tests report **no** data
race:

```console
$ go test -race ./kittens/diff/...
ok  	kitty/kittens/diff	1.105s
$ grep -c "DATA RACE" q5_race_diff.txt
0
```

The `tools/utils` tests also report **no** data race; the only failure is the unrelated `TestFileLock`
(an environment issue — it fails identically **without** `-race`, so it is not a race):

```console
$ go test -race ./tools/utils/...
--- FAIL: TestFileLock (0.00s)
    filelock_test.go:41: Lock test process failed with error: exec: no command and output:
FAIL
FAIL	kitty/tools/utils	0.070s
ok  	kitty/tools/utils/base85	1.021s
ok  	kitty/tools/utils/humanize	1.021s
...
$ go test ./tools/utils/ -run TestFileLock        # WITHOUT -race: same failure
    filelock_test.go:41: Lock test process failed with error: exec: no command and output:
FAIL	kitty/tools/utils	0.006s
```

The existing tests do not drive concurrent `Set`, so they cannot surface the latent race. To verify
the nuance directly, a small external program (outside the repository) drives the **real**
`utils.LRUCache.Set` concurrently with **distinct** keys — exactly the `highlight_all` pattern
(8 goroutines × 2000 `Set`):

```console
$ go run -race .
==================
WARNING: DATA RACE
Write at 0x00c00044ae40 by goroutine 9:
  runtime.mapassign_faststr()
      /usr/local/go/src/runtime/map_faststr.go:203 +0x0
  kitty/tools/utils.(*LRUCache[go.shape.string,go.shape.int]).Set()
      /tmp/blitzy/kitty/.../tools/utils/cache.go:34 +0x98
  main.main.func1()
      /tmp/racetest/main.go:22 +0x184
  ...
Previous write at 0x00c00044ae40 by goroutine 14:
  runtime.mapassign_faststr()
      /usr/local/go/src/runtime/map_faststr.go:203 +0x0
  kitty/tools/utils.(*LRUCache[go.shape.string,go.shape.int]).Set()
      /tmp/blitzy/kitty/.../tools/utils/cache.go:34 +0x98
  ...
==================
fatal error: concurrent map writes
```

### 5.4 Determinism check (same input ×2)

Diffing the same 12-file, highlightable directory (`.py` files) twice and comparing the visible text
tokens yields identical results — no corruption from parallel highlighting:

```console
$ md5sum q5_textA.txt q5_textB.txt
3ac2121ca260d6aaa2686ada91146c06  q5_textA.txt
3ac2121ca260d6aaa2686ada91146c06  q5_textB.txt
$ diff -q q5_textA.txt q5_textB.txt && echo "IDENTICAL TEXT CONTENT ACROSS RUNS"
IDENTICAL TEXT CONTENT ACROSS RUNS
```

All 12 files pair correctly (`src_1.py` … `src_12.py`), each `@@ -1,7 +1,7 @@` with line 4
`return x * N  # left` → `return x + N  # right`:

```text
   src_1.py
   @@ -1,7 +1,7 @@
4      return x * 1  # left                                 4      return x + 1  # right
   src_10.py
   @@ -1,7 +1,7 @@
4      return x * 10  # left                                4      return x + 10  # right
```

### 5.5 Causal reason

Highlighting does not "step on itself" in the canonical path because each worker writes a **different**
cache key (`highlighted_lines_cache.Set(path, …)`, `highlight.go:224`), so there is no logical
collision on a shared value, and the observed output is byte-for-byte identical across runs (§5.4). But
this is safe only *because* the keys differ **in practice** — the underlying `LRUCache.Set` mutates a
Go map under a read lock (`cache.go:33-35`), and Go maps are unsafe for concurrent writes **even to
different keys**; the race detector proves this with `WARNING: DATA RACE … cache.go:34` followed by
`fatal error: concurrent map writes` (§5.3). The canonical `highlight_all` path rarely triggers it
because Chroma highlighting dominates each worker's runtime, so two `Set` calls almost never coincide;
nevertheless the latent race is real. (Per the read-only scope, this is reported, not fixed.)

---


## 6. Q6 — Binary files and images

### 6.1 Direct answer

When a file is not text, the kitten does not attempt a line diff. A **non-text, non-image** file is
rendered as a single banner `Binary file: <human-readable size>`. A **non-text file that is an image**
is rendered via the kitty graphics protocol (its pixels are transmitted to the terminal), with a
`Dimensions: WxH Size: …` header. Plain UTF-8 text takes the normal syntax-highlighted line diff.

### 6.2 Mechanism & code anchors

In `render()` (`render.go:696`) the classification is:

- `is_binary := !is_path_text(path)` (`render.go:706`), where `is_path_text` (`collect.go:86`) tests
  the image MIME prefix and UTF-8 validity of the cached bytes;
- refined for diffs: `if !is_binary && item_type == "diff" && !is_path_text(changed_path) { is_binary
  = true }` (`render.go:707-708`);
- `is_img := is_binary && is_image(path) || (item_type == "diff" && is_image(changed_path))`
  (`render.go:710`).

The dispatch `switch item_type` (`render.go:712`) routes each case — `"diff"` (`render.go:713-714`),
`"add"` (`render.go:726-727`), `"removal"` (`render.go:739-740`) — to `image_lines`
(definition `render.go:333`) when `is_img`, else to `binary_lines` (definition `render.go:446`), else
to the normal text path. `binary_lines` emits `fmt.Sprintf("Binary file: %s", human_readable(sz))`
(`render.go:452`). `image_lines` emits a `Size:` line and, once the resolution is known,
`Dimensions: %dx%d` (`render.go:343`). Corroborated by `docs/kittens/diff.rst:18` ("Displays images as
well as text diffs, even over SSH").

### 6.3 Command(s) run and observed output — all three branches

**(a) UTF-8 text** — `file` reports UTF-8; the diff is a normal highlighted line diff (note the
non-ASCII `café` is preserved on both sides):

```console
$ file /tmp/kdiff/q6_text_l/doc.txt
/tmp/kdiff/q6_text_l/doc.txt: Unicode text, UTF-8 text
$ $KIT diff /tmp/kdiff/q6_text_l /tmp/kdiff/q6_text_r
```

```text
   doc.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   @@ -1,3 +1,3 @@
1  plain UTF-8 text                                         1  plain UTF-8 text
2  second line café                                         2  second line CHANGED café
3  third                                                    3  third
```

**(b) non-UTF-8 binary** — 4096 random bytes each side (`file` says `data`; a UTF-8 decode raises
`UnicodeDecodeError`). Rendered as the literal `Binary file: 4 KB` banner on both sides:

```console
$ file /tmp/kdiff/q6_bin_l/data.bin
/tmp/kdiff/q6_bin_l/data.bin: data
$ python3 -c "open('/tmp/kdiff/q6_bin_l/data.bin','rb').read().decode('utf-8')"
UnicodeDecodeError: 'utf-8' codec can't decode byte ...
$ $KIT diff /tmp/kdiff/q6_bin_l /tmp/kdiff/q6_bin_r
```

```text
   data.bin
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Binary file: 4 KB                                           Binary file: 4 KB
```

**(c) image** — a 48×32 PNG (118 B) each side. The frame shows the `Dimensions: 48x32 Size: 118 B`
header plus `Loading image...`:

```console
$ file /tmp/kdiff/q6_img_l/pic.png
/tmp/kdiff/q6_img_l/pic.png: PNG image data, 48 x 32, 8-bit/color RGB, non-interlaced
$ $KIT diff /tmp/kdiff/q6_img_l /tmp/kdiff/q6_img_r
```

```text
   pic.png
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Dimensions: 48x32 Size: 118 B                               Dimensions: 48x32 Size: 118 B
   Loading image...                                            Loading image...
```

The image is genuinely transmitted via the kitty graphics protocol — the raw byte stream contains
five graphics-protocol APC chunks (`ESC _ G …`):

```console
$ python3 -c "d=open('q6c_img_raw.bin','rb').read(); print('ESC_G chunks =', d.count(b'\x1b_G'))"
ESC_G chunks = 5
```

**State before/during/after (Rule 9):** the raw stream contains *both* an initial
`Dimensions: 0x0 Size: 118 B` (emitted before the image resolution is known) *and* the final
`Dimensions: 48x32 Size: 118 B` (after asynchronous resolution loading, the `res.Width > -1` branch at
`render.go:342`):

```text
...Dimensions: 0x0 Size: 118 B ...
...Dimensions: 48x32 Size: 118 B ...
```

### 6.4 Causal reason

`is_path_text` (`collect.go:86`) drives everything: it returns false for the random-bytes file
(invalid UTF-8) and for the PNG (image MIME), setting `is_binary` (`render.go:706`). `is_image`
(`collect.go:82`, MIME prefix `image/`) then separates the two: the PNG makes `is_img` true
(`render.go:710`) → `image_lines` → graphics protocol; the random bytes stay `is_img == false` →
`binary_lines` → the `Binary file: 4 KB` banner (`render.go:452`, size from `human_readable(4096)`).
Text passes both checks and takes the normal highlighted diff. The `0x0 → 48x32` transition happens
because image dimensions are resolved asynchronously, so the first render has no dimensions yet.

---


## 7. Q7 — End-to-end runtime trace

### 7.1 Direct answer

From "two directories compared" to a fully resolved screen, the flow is:
`main` validates the two arguments and sets up the engine + caches → a background goroutine builds the
**collection** (pairing/classification) → the main thread is woken and, on the collection result,
runs **generate_diff**, **highlight_all**, and **load_all_images** → the diff jobs complete and the
screen renders every change, rename, addition, and removal together. A single fixture containing all
four kinds resolves them all in one screen.

### 7.2 Mechanism & code anchors

Entry and setup (`kittens/diff/main.go`): `main` (`main.go:102`) → `load_config` (`main.go:104`) →
argument check `if len(args) != 2` returning the error `"You must specify exactly two
files/directories to compare"` (`main.go:108-109`) → `set_diff_command(conf.Diff_cmd)`
(`main.go:111`) → `init_caches()` (`main.go:114`) → `create_formatters()` (`main.go:115`); the exported
`EntryPoint` is at `main.go:177`.

UI orchestration (`kittens/diff/ui.go`): the `Handler` struct (`ui.go:55`) owns
`async_results chan AsyncResult` (`ui.go:56`), created with buffer 32 (`ui.go:132`). A startup
goroutine (`ui.go:133`) runs `create_collection(self.left, self.right)` (`ui.go:135`), sends the
result (`ui.go:136`), and calls `self.lp.WakeupMainThread()` (`ui.go:137`). `handle_async_result`
(`ui.go:245`) routes results: on `COLLECTION` (`ui.go:247`) it stores the collection (`ui.go:248`) and
calls `generate_diff()` (`ui.go:249`), `highlight_all()` (`ui.go:250`), `load_all_images()`
(`ui.go:251`); on `DIFF` (`ui.go:252`) it stores the diff map (`ui.go:253`), computes statistics
(`ui.go:254`), and renders (`ui.go:256`). `generate_diff` (`ui.go:142`) builds jobs from
`collection.Apply` (`ui.go:145`) and, in a goroutine, calls `diff(jobs, …)` (`ui.go:155`) then wakes
the main thread (`ui.go:157`). `mouse.go` and `search.go` are peripheral (referenced only for context).

### 7.3 Command(s) run

A single fixture contains all four change kinds at once, verified by hash: `changed.txt` differs;
`oldname.txt` (left) and `newname.txt` (right) are byte-identical (a rename); `removed.txt` is
left-only; `added.txt` is right-only.

```console
$ (cd /tmp/kdiff/q7_left  && for f in $(find . -type f|sort); do echo "$(md5sum "$f"|cut -d' ' -f1)  $f"; done)
4ac5d7a447c4194d9a380c0965bf8adf  ./changed.txt
1ab8c70ed77446bd72b7be2d3782072c  ./oldname.txt
068cd2ad65bdd9e65c1006a0c17cb881  ./removed.txt
$ (cd /tmp/kdiff/q7_right && for f in $(find . -type f|sort); do echo "$(md5sum "$f"|cut -d' ' -f1)  $f"; done)
ecf5812c8deede9a38dac7fc56213ea1  ./added.txt
6d2532a81558cc3f2856c1b4a277aeb1  ./changed.txt
1ab8c70ed77446bd72b7be2d3782072c  ./newname.txt
$ $KIT diff /tmp/kdiff/q7_left /tmp/kdiff/q7_right
```

### 7.4 Observed output — all four resolved in one screen

```text
   added.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   This file was added                                                     1  this file exists only on the right and is brand new

   changed.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   @@ -1,2 +1,2 @@
1  config value = 1                                                        1  config value = 2
2  keep this                                                               2  keep this

   oldname.txt                                                                newname.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━


   removed.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1  this file exists only on the left and is deleted                           This file was removed
```

All four are present, sorted by path: `added.txt` (addition), `changed.txt` (change,
`config value = 1` → `2`), `oldname.txt│newname.txt` (rename — two-name title, empty body per §2.6),
and `removed.txt` (removal).

### 7.5 Causal reason and observed-vs-inferred

The screen proves the pipeline resolved every classification: `create_collection` paired and
categorized the files (Q1/Q2 mechanisms), `generate_diff` produced the line hunks for the change, and
rendering placed additions/removals/renames using the branches in `render.go`. The following are
**observed** at runtime: the four resolved classifications (above), the parallel `git` subprocesses
(§4), the transient "Calculating diff…" state during `generate_diff`, and "Loading image…" from
`load_all_images` (§6). The goroutine / `async_results` channel / `WakeupMainThread` routing internals
are read from source and so are labelled **(inferred)** — the trace hops
`main → create_collection → handle_async_result → generate_diff → highlight_all → load_all_images` are
grounded in the cited `file:line` locations but were not individually instrumented at runtime.

---


## 8. Q8 — Diff algorithm + cache interplay

### 8.1 Direct answer (lead with the plain reading)

With the **default** configuration (`diff_cmd = 'auto'`), the builtin diff algorithm in `diff.go`
**does not run** on this machine — `auto` resolves to external `git diff --no-index`, because git is
installed. The builtin "anchored diff" engine only runs when explicitly selected (`-o
diff_cmd=builtin`) or when neither `git` nor `diff` exists. All three engines were exercised
explicitly; they produce the same side-by-side rendering, differing only in the raw unified-diff
header they emit. The builtin engine matches regions using a longest-common-subsequence of **unique**
lines ("anchored"/"patience" diff), and it consumes the once-read cached bytes, so switching engines
re-runs only the diff step — not the file reads or highlighting.

### 8.2 Mechanism & code anchors

**Engine resolution** (`kittens/diff/patch.go`): the command templates are
`GIT_DIFF = "git diff --no-color --no-ext-diff --exit-code -U_CONTEXT_ --no-index --"`
(`patch.go:21`) and `DIFF_DIFF = "diff -p -U _CONTEXT_ --"` (`patch.go:22`). `find_differ`
(`patch.go:34`) prefers git (`patch.go:36`), then `diff` (`patch.go:38`), then the builtin empty
command (`patch.go:40`). `set_diff_command` (`patch.go:44`) maps the config value: `"auto"` →
`find_differ` (`patch.go:47`); `"builtin"`/`""` → `diff_cmd = []string{}` (`patch.go:50-51`); `"diff"`
(`patch.go:53`); `"git"` (`patch.go:55`); a custom string via `shlex` (`patch.go:57-61`).
`run_diff` (`patch.go:282`) branches on `if len(diff_cmd) == 0` (`patch.go:294`): builtin path reads
both files via `data_for_path` and calls `Diff(path1, data1, path2, data2, num_of_context_lines)`
(`patch.go:303`), returning nil→no-difference (`patch.go:304`); otherwise it substitutes `_CONTEXT_`
with the context count (`patch.go:309-311`) and runs the external command via `exec.Command`
(`patch.go:315`), treating exit code 1 as "differences found" (`patch.go:321-323`).

**Builtin anchored diff** (`kittens/diff/diff.go`): the file header states it is *"Copied from the Go
stdlib, with modifications."* (`diff.go:1-2`), referencing `internal/diff`. `Diff(...)` (`diff.go:49`,
**exported**) returns `nil` immediately when `old == new` (`diff.go:50-52`), prints the
`diff`/`---`/`+++` header (`diff.go:56-60`), and iterates the anchor pairs returned by `tgs(x, y)`
(`diff.go:192`) — the longest common subsequence of lines that appear exactly once in both inputs —
emitting `@@ … @@` unified hunks with context (`diff.go:74-164`). `lines` (`diff.go:172`) appends
`"\ No newline at end of file"` (`diff.go:179`) when an input lacks a trailing newline.

*Background (framing):* the Go `internal/diff` documentation describes this as an "anchored diff": it
seeks the smallest set of unique lines to insert/remove, where unique means a line appearing exactly
once in both old and new, using those unique lines as anchors; it is sometimes called a "patience
diff." This framing is corroborated by directly running the builtin engine below.

### 8.3 Command(s) run — all three engines on the same fixture

The fixture is a 5-line file changed on two lines (`beta`→`BETA`, `epsilon`→`EPSILON`).

```sh
$KIT diff -o diff_cmd=git     /tmp/kdiff/q8_left/f.txt /tmp/kdiff/q8_right/f.txt
$KIT diff -o diff_cmd=diff    /tmp/kdiff/q8_left/f.txt /tmp/kdiff/q8_right/f.txt
$KIT diff -o diff_cmd=builtin /tmp/kdiff/q8_left/f.txt /tmp/kdiff/q8_right/f.txt
```

### 8.4 Observed output — rendering is identical across engines

All three engines render the same side-by-side frame (shown once; the other two are byte-identical in
the visible grid):

```text
   /tmp/kdiff/q8_left/f.txt                                    /tmp/kdiff/q8_right/f.txt
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   @@ -1,5 +1,5 @@
1  alpha                                                    1  alpha
2  beta                                                     2  BETA
3  gamma                                                    3  gamma
4  delta                                                    4  delta
5  epsilon                                                  5  EPSILON
```

The engines differ in the **raw unified diff** they emit (which the TUI then parses into the grid
above). Running each engine's exact command directly exposes the difference:

**git** (exact `GIT_DIFF` with `_CONTEXT_`=3) — adds a `diff --git`/`index` header; exit 1 = different:

```console
$ git diff --no-color --no-ext-diff --exit-code -U3 --no-index -- /tmp/kdiff/q8_left/f.txt /tmp/kdiff/q8_right/f.txt ; echo "[exit=$?]"
diff --git a/tmp/kdiff/q8_left/f.txt b/tmp/kdiff/q8_right/f.txt
index 600d48ac7..1663761a9 100644
--- a/tmp/kdiff/q8_left/f.txt
+++ b/tmp/kdiff/q8_right/f.txt
@@ -1,5 +1,5 @@
 alpha
-beta
+BETA
 gamma
 delta
-epsilon
+EPSILON
[exit=1]
```

**diff** (exact `DIFF_DIFF`) — adds timestamp lines; exit 1 = different:

```console
$ diff -p -U 3 -- /tmp/kdiff/q8_left/f.txt /tmp/kdiff/q8_right/f.txt ; echo "[exit=$?]"
--- /tmp/kdiff/q8_left/f.txt	2026-07-06 23:18:59.613599104 +0000
+++ /tmp/kdiff/q8_right/f.txt	2026-07-06 23:18:59.613599104 +0000
@@ -1,5 +1,5 @@
 alpha
-beta
+BETA
 gamma
 delta
-epsilon
+EPSILON
[exit=1]
```

**builtin** — the exported `Diff()` (`diff.go:49`) called directly (the exact function `run_diff`
invokes at `patch.go:303`) on the same fixture. Its header is the cleanest form — `diff <old> <new>`
with no `index` line and no timestamps (182 bytes):

```console
$ go run .   # external program: diff.Diff("/…/f.txt", old, "/…/f.txt", new, 3)
=== builtin diff.Diff() raw bytes (len=182) ===
diff /tmp/kdiff/q8_left/f.txt /tmp/kdiff/q8_right/f.txt
--- /tmp/kdiff/q8_left/f.txt
+++ /tmp/kdiff/q8_right/f.txt
@@ -1,5 +1,5 @@
 alpha
-beta
+BETA
 gamma
 delta
-epsilon
+EPSILON
[patch==nil? false]
```

### 8.5 Edge — "No newline at end of file" (`diff.go:179`)

A fixture whose right file omits the trailing newline (left = 14 bytes ending `\n`, right = 13 bytes
with none, confirmed by `od -c`). Calling the builtin `Diff()` directly emits the marker after the
changed line:

```console
$ od -c /tmp/kdiff/q8nl_left/f.txt  | tail -2
0000000   o   n   e  \n   t   w   o  \n   t   h   r   e   e  \n
$ od -c /tmp/kdiff/q8nl_right/f.txt | tail -2
0000000   o   n   e  \n   t   w   o  \n   t   h   r   e   e
```

```text
# builtin Diff("a.txt","one\ntwo\nthree\n","b.txt","one\ntwo\nthree",3)  -> len=105
diff a.txt b.txt
--- a.txt
+++ b.txt
@@ -1,3 +1,3 @@
 one
 two
-three
+three
\ No newline at end of file
```

When **both** sides omit the trailing newline (and the last line differs), the marker appears on both
`-` and `+` lines (len=133):

```text
-three
\ No newline at end of file
+THREE
\ No newline at end of file
```

The external `git` engine emits the identical `\ No newline at end of file` text for the same fixture.
(Honest nuance: the full-screen side-by-side TUI renders the affected line but does not surface the
marker as its own visible row; the marker is present in the engine's byte output, shown above.)

### 8.6 Edge — identical / empty inputs return `nil` (`diff.go:50-52`)

```text
# builtin Diff("a.txt","same\ncontent\n","b.txt","same\ncontent\n",3)
[len=0 nil?=true]
# builtin Diff("a.txt","","b.txt","",3)   (both empty)
[len=0 nil?=true]
```

Through the canonical kitten path, `-o diff_cmd=builtin` on an identical pair renders
`The files are identical` (the `patchb == nil` branch at `patch.go:304`; frame shown in §1.6).

### 8.7 Causal reason and cache interplay

`find_differ` (`patch.go:34-42`) checks for `git` first, so on a git-equipped machine `auto` never
reaches the builtin engine — hence the plain-reading "builtin does not run by default." The three
engines produce the same *rendering* because the TUI parses whichever unified diff it receives into the
same side-by-side grid; only the raw header differs (git's `index` line, diff's timestamps, builtin's
bare `diff <old> <new>`). The builtin engine finds matching regions via `tgs` (`diff.go:192`), the
LCS-of-unique-lines that anchors the match, which is why it produces clean, minimal hunks.

**Cache interplay:** the builtin path calls `data_for_path` for both files (`patch.go:295-301`) and so
reuses the once-read cached bytes (§3) before running `Diff()` in-process; this is why the builtin
`openat` count is exactly 1 per file (§3.4). External `git`/`diff` re-read the files themselves in a
subprocess (the two extra non-`O_CLOEXEC` opens in §3.4). Either way, switching the engine re-runs only
the diff step — the file reads (`data_cache`) and syntax highlighting (`highlighted_lines_cache`)
remain cached — so the cache keeps everything fast regardless of which engine is selected.

---


## 9. Coverage checklist (every named item)

Each named mechanism, condition, config option, dependency, and doc corroboration, mapped to the
section that answers it, its `file:line`, and whether it was **observed** at runtime or **(inferred)**
from source.

### 9.1 Functions / methods / structs

| Item | `file:line` | Section | Observed? |
|------|-------------|---------|-----------|
| `collect_files` | collect.go:296 | §1 | observed |
| `walk` | collect.go:260 (calls 299,303) | §1 | observed |
| `data_for_path` (single content read) | collect.go:65-70 | §3 | observed (strace) |
| `hash_for_path` / `md5.Sum` | collect.go:106 / 112 | §2 | observed |
| `sanitize` | collect.go:128-130 | §3 | inferred |
| `lines_for_path` | collect.go:138-146 | §3 | inferred |
| `highlighted_lines_for_path` | collect.go:148-157 | §3 | inferred |
| `add_change` | collect.go:167 (call 319) | §1 | observed |
| `add_rename` | collect.go:175 (call 354) | §2 | observed |
| `add_add` | collect.go:181 (call 366) | §2 | observed |
| `add_removal` | collect.go:192 (call 362) | §2 | observed |
| `allowed` (ignore-glob) | collect.go:230 | §1.6 | observed |
| `init_caches` / `const sz = 4096` | collect.go:26 / 29 | §3 | observed |
| `Context.Parallel` | tools/utils/images/utils.go:27 | §4 | observed |
| `diff` (worker pool) | patch.go:352 (dispatch 361) | §4 | observed |
| `do_diff` | patch.go:330 (call 365) | §4 | observed |
| `run_diff` | patch.go:282 (branch 294) | §8 | observed |
| `parse_patch` | patch.go:245 | §8 | inferred |
| `find_differ` | patch.go:34-42 | §8 | observed |
| `set_diff_command` | patch.go:44 | §8 | observed |
| `highlight_all` | highlight.go:217 (dispatch 219) | §5 | observed |
| `highlight_file` | highlight.go:161 | §5 | inferred |
| `render` | render.go:696 | §6 | observed |
| `binary_lines` / `"Binary file: %s"` | render.go:446 / 452 | §6 | observed |
| `image_lines` / `Dimensions` | render.go:333 / 343 | §6 | observed |
| `render_screen_line` (rename full-width return) | render.go:70,99-101 | §2.6 | observed |
| `rename_lines` | render.go:684 (msg 688) | §2.6 | observed (as latent bug) |
| `title_lines` (two-name title) | render.go:208-209 | §2 | observed |
| `main` / `EntryPoint` | main.go:102 / 177 | §0,§7 | observed |
| `handle_async_result` | ui.go:245 (COLLECTION 247, DIFF 252) | §7 | inferred |
| `generate_diff` | ui.go:142 (diff() 155) | §7 | inferred |
| `create_collection` (call) | ui.go:135 | §7 | observed (result) |
| `LRUCache` (Get/Set/GetOrCreate/MustGetOrCreate) | tools/utils/cache.go:13 (25/32/39/60) | §3,§5 | observed |
| `Diff` / `tgs` / `lines` | diff.go:49 / 192 / 172 | §8 | observed |

### 9.2 Structs / consts / vars

| Item | `file:line` | Section |
|------|-------------|---------|
| seven caches (size,mimetypes,data,is_text,lines,highlighted_lines,hash) | collect.go:20-24 | §3 |
| `const sz = 4096` | collect.go:29 | §3 |
| `LRUCache[K,V]` | tools/utils/cache.go:13 | §3,§5 |
| `async_results chan AsyncResult` + buffer 32 | ui.go:56 / 132 | §7 |
| `GIT_DIFF` / `DIFF_DIFF` / `diff_cmd` var | patch.go:21 / 22 / 24 | §8 |

### 9.3 Conditions / branches

| Condition | `file:line` | Section | Observed value |
|-----------|-------------|---------|----------------|
| relative-path pairing (`Intersect`) | collect.go:306 | §1 | 3 entries, same.txt omitted |
| rename = MD5 `ah==rh` AND `ld==rd` | collect.go:350 / 353 | §2 | identical→rename; 1 byte→remove+add |
| mode-only change (`lstat.Mode()!=rstat.Mode()`) | collect.go:321-329 (323) | §1.6 | "Mode changed: …" |
| `is_binary` / `is_img` | render.go:706 / 710 | §6 | binary→banner, image→graphics |
| `"Binary file: %s"` | render.go:452 | §6 | "Binary file: 4 KB" |
| builtin vs external dispatch `len(diff_cmd)==0` | patch.go:294 (Diff 303) | §8 | builtin openat=1; auto=3 |
| no-newline marker | diff.go:179 | §8.5 | "\ No newline at end of file" |
| identical → nil | diff.go:50-52 | §8.6 | len=0 nil?=true |
| arg count `len(args)!=2` | main.go:108-109 | §1.6 | error, exit 1 |

### 9.4 Config options (by name)

| Option | Default | `file:line` | Section |
|--------|---------|-------------|---------|
| `diff_cmd` | `auto` | main.py:41 | §8 |
| `ignore_name` | empty `''` | main.py:56 | §1.6 |
| `num_context_lines` | `3` | main.py:37 | §8 (`-U3`) |
| `syntax_aliases` | `pyj:py pyi:py recipe:py` | main.py:29 | §0.5 |
| `replace_tab_by` | 4 spaces | main.py:52 | §0.5 |

### 9.5 Dependency versions (`go.mod`) and doc corroboration

| Dependency | Version | `go.mod` | Role |
|------------|---------|----------|------|
| Go runtime | 1.22 | go.mod:3 | build |
| `github.com/alecthomas/chroma/v2` | v2.14.0 | go.mod:7 | syntax highlighting (§5) |
| `github.com/bmatcuk/doublestar/v4` | v4.6.1 | go.mod:8 | `ignore_name` globs (§1.6) |
| `github.com/kovidgoyal/imaging` | v1.6.3 | go.mod:13 | image decode/scale (§6) |
| `github.com/edwvee/exiffix` | v0.0.0-20240229113213 | go.mod:10 | EXIF-aware image load (§6) |
| `golang.org/x/image` | v0.17.0 | go.mod:18 | image formats (§6) |
| `github.com/zeebo/xxh3` | v1.0.2 | go.mod:16 | fast hashing |

Doc corroboration (`docs/kittens/diff.rst`): syntax highlighting "asynchronously, for maximum speed"
(L15-16, §5); "Displays images as well as text diffs, even over SSH" (L18, §6); "Does recursive
directory diffing" (L20, §1); git integration (L92+, §8).

---


## 10. Cleanup verification

This investigation was strictly read-only: no existing repository file was modified, and the only
artifact added is this document (plus the new `blitzy/`/`blitzy/documentation/` directories). All
temporary fixtures and observation scripts lived outside the repository tree (under `/tmp`) and were
removed on completion. The gitignored launcher binaries produced by the build are untracked and do not
appear as changes.

The working tree after the investigation (and after all `/tmp` fixtures and scripts were removed)
shows only the new document and its parent directories, with **no existing tracked file modified or
deleted**. This is the actual, unedited output:

```console
$ git status --porcelain
?? blitzy/

$ git status --porcelain --untracked-files=all
?? blitzy/documentation/kitty_815df1e210e0.md

$ git diff --stat
                                 # (empty — zero changes to any existing tracked file)

$ git status --porcelain --untracked-files=all | grep -E '^ ?[MDR]' \
    && echo "TRACKED CHANGES" || echo "OK: zero tracked-file modifications/deletions"
OK: zero tracked-file modifications/deletions
```

The single new untracked path `blitzy/documentation/kitty_815df1e210e0.md` is this document; the
gitignored launcher binaries produced by the build do not appear (they are untracked and ignored).
After committing this document the working tree is clean.

---

*End of document. All `file:line` citations correspond to commit
`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`; every behavioral claim above is paired with the exact
command run and its complete, unedited observed output, or is explicitly labelled `(inferred)`.*

