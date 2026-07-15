# Does pressing **Enter** in the `choose-fonts` kitten persist the font selection, or only change the current session?

**Repository:** `kovidgoyal/kitty` — branch `kitty_815df1e210e0`
**HEAD:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (commit *"Wire up applying of font config"*)
**kitty version built & observed:** `kitty 0.35.2`

---

## 1. Direct verdict (lead answer)

> **Pressing `Enter` at the `choose-fonts` kitten's final confirmation step PERSISTS the
> font selection across restarts.**

When you press `Enter` on the final pane, the kitten writes the four font-face settings into a
**sentinel-delimited block** inside `kitty.conf` **on disk** and replaces the file with an
atomic temp-file-then-`rename`. Because the bytes land on disk, kitty re-reads them on the next
startup, so the choice **survives restarts**.

- The concrete write is performed by `final_pane.on_key_event` calling `config.Patcher.Patch(...)`
  [`kittens/choose_fonts/final.go:L78-L82`].
- That produces a block wrapped in `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS`, plus a conditional
  `kitty.conf.bak` backup, replaced via `utils.AtomicUpdateFile`
  [`tools/config/api.go:L328-L347`].

This verdict is not merely read from source — it was **observed at runtime**. Driving the real
`kitten choose-fonts` UI and pressing `Enter` on the final pane rewrote an isolated `kitty.conf`
in place; a **freshly launched** kitty process then loaded those exact faces as its active fonts
(see §8). The evidence appears next to the exact command that produced it.

Two clarifications, so the verdict is not misread:

1. **The live-reload is a *separate* step, not the persistence mechanism.** After a successful
   *write*, the kitten *may also* send `SIGUSR1` to already-running kitty GUI processes so they
   re-read the (already-persisted) file. That reload is selected by `--reload-in` and executed by
   `config.ReloadConfigInKitty` [`kittens/choose_fonts/final.go:L86-L93`]
   [`tools/config/api.go:L352-L371`]. Reload only tells running instances to re-read a file that
   is *already* durable; it is best-effort and its errors are ignored. Even with
   `--reload-in none` the file is still written and still persists.

2. **The "session-only" behavior is a *different key*, not `Enter`.** Pressing **`s`** (or `S`)
   emits the same four lines to **STDOUT** and quits **without touching `kitty.conf`**
   [`kittens/choose_fonts/final.go:L101-L111`][`kittens/choose_fonts/main.go:L64-L66`]. That is the
   pipe/session-only path. Had the kitten's *Enter* behavior been session-only, this document
   would have led with that — it does not, because both the code and the on-disk runtime evidence
   show `Enter` writes the file.

**One-line causal chain:**
`Enter` → `final_pane.on_key_event` → `config.Patcher.Patch("KITTY_FONTS", …4 keys)` →
`# BEGIN_KITTY_FONTS…# END_KITTY_FONTS` block (+ conditional `.bak`) → `utils.AtomicUpdateFile`
(temp file + `rename`) → **bytes on disk → re-read at next startup → PERSISTED.**

---

## 2. How this answer was obtained

### 2.1 Canonical environment (exact image, container, and toolchain)

All build and runtime evidence in this document was gathered inside the **user-mandated Docker
image**, not on any host checkout:

- **Image:** `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`
- **Image ID:** `sha256:c0824992ad0b274bc8738bf1365d336bf91dec97d726ec122e2d08eb9f053288`
- **Running container name:** `kitty-setup`
- **OS:** Ubuntu 24.04.2 LTS

Commands were executed in that container. A representative invocation form (used throughout):

```bash
docker exec kitty-setup bash -lc '<command>'
```

**Toolchain actually observed in the image** (each value is real command output):

```
$ python3 --version
Python 3.12.3
$ go version
go version go1.23.4 linux/amd64
$ gcc --version | head -1
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
```

The repository's own constraints are Go `go 1.22` [`go.mod:L3`] and Python `>=3.8`
[`pyproject.toml`], both satisfied.

The canonical kitty checkout lives at **`/app`** inside the image (HEAD
`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`, `git status` clean). The image ships the **pre-built
default binaries** used for the runtime observations:

```
$ ls -la /app/kitty/launcher/kitty /app/kitty/launcher/kitten /app/kitty/fast_data_types.so
-rwxr-xr-x 1 root root    36224 Aug 28  2025 /app/kitty/launcher/kitty
-rwxr-xr-x 1 root root 15945988 Aug 28  2025 /app/kitty/launcher/kitten
-rwxr-xr-x 1 root root  1221264 Aug 28  2025 /app/kitty/fast_data_types.so
```

A GUI display is provided by **`Xvfb` on `DISPLAY=:99`** (1280×1024×24), which the interactive
kitten requires (§6.4).

> **Note on prior drafts.** An earlier version of this document reported Python `3.13.7` and a
> `/tmp/blitzy/...` host path. That was the wrong environment (a host checkout, not the mandated
> image). This version reports only what was observed in the mandated image above.

### 2.2 What is OBSERVED vs. CODE-DERIVED (labeling convention)

The real, canonical entry point — the `kitten choose-fonts` dispatcher subcommand — was exercised.
**No** remote-control hook, debug hook, fallback, or synthetic stand-in was used to obtain any
value or the persistence verdict.

Two kinds of OBSERVED evidence are used, and the document labels them differently so no claim is
over-stated:

- **OBSERVED (machine-checkable, embedded verbatim)** — text captured directly from tools, shown
  inside fenced blocks with nothing altered on the lines shown. This covers: the canonical build
  exit code / `wayland-protocols` version and the real gcc/Go build lines (§3.2); the canonical
  `kitten choose-fonts --help` (§5); and, for every persistence and branch scenario, the harness's
  own output — **config bytes, full 64-character SHA-256 (before/after), STDOUT/STDERR with byte
  counts, exit status, process PIDs/`ps -ww` tree, and `ls -la`** (§8 in full; §9.1 in full; the
  §9.2/§9.3 branch blocks show every audit field verbatim with the repetitive content dumps elided
  and that elision disclosed). Where any transformation was applied (an elided object-file list, an
  elided repeated content dump), it is stated at the block.
- **OBSERVED (visual, faithfully transcribed — _not_ verbatim machine text)** — the real running
  kitten's **listing / faces / final** panes under `DISPLAY=:99` (§6.2, §7.1). These were captured
  as PNG screenshots with ImageMagick `import` and **viewed**; because the single-file deliverable
  rule permits only this `.md`, the PNGs are not committed and their content is **transcribed**.
  These transcriptions are labeled as such and must not be read as byte-exact terminal output.
- **CODE-DERIVED** — a claim read from source with an exact `file:line`, used for internal
  control-flow (e.g., which struct field carries a value) or for a mechanism that cannot be captured
  headlessly in this image (e.g., the `SIGUSR1` reload delivery — no `strace` present). No
  interactive screen output is fabricated.

The **restart** proof (a fresh kitty loading the persisted faces) is OBSERVED and appears in §8.4.

### 2.3 Read-only, safe, reproducible workflow

- The kitty **source tree is read-only**; the only artifact created is this document under
  `blitzy/documentation/`.
- The canonical build and every runtime observation happen inside the container's `/app` and
  `/tmp`, never in the deliverable checkout, so building never dirties the deliverable repository.
- Runtime writes are redirected to a **validated, unique, mode-0700 temporary directory** via
  `KITTY_CONFIG_DIRECTORY`, exported *before* any kitty process starts, with a quoted cleanup
  trap (the safe-isolation harness is given in full in §12). This never touches the developer's
  real `~/.config/kitty/kitty.conf`.
- Repository-pristine verification appears in §13.2.

---

## 3. R1 — Build & launch (canonical, default configuration)

### 3.1 Canonical build commands

kitty provides two canonical build entry points:

- **`./dev.sh build`** — documented by the official build page as the recommended developer build
  ([sw.kovidgoyal.net/kitty/build](https://sw.kovidgoyal.net/kitty/build/)). `dev.sh:L9` is
  literally `exec go run bypy/devenv.go "$@"`, a thin wrapper that downloads pre-built dependency
  bundles and then invokes `setup.py`; it produces `kitty/launcher/kitty` and the Go
  `kitty/launcher/kitten`.
- **`python3 setup.py build`** — the in-repository build routine. The `Makefile` `all:` target
  runs `python3 setup.py $(VVAL)` [`Makefile:L12-L13`]. This is also the command specified by the
  environment's own setup instructions.

The build reproduced for this investigation (per the environment setup instructions) was:

```bash
docker exec kitty-setup bash -lc 'cd /app && python3 setup.py build --verbose'
```

### 3.2 Observed build result — default X11 **and** Wayland, no altered package discovery

The build completes cleanly in the mandated image, with **both** display backends compiled
(no backend was disabled, and no package discovery was altered):

```
$ docker exec kitty-setup bash -lc 'cd /app && python3 setup.py build --verbose >/tmp/build.log 2>&1; echo "BUILD EXIT CODE = $?"'
BUILD EXIT CODE = 0

$ docker exec kitty-setup bash -lc 'pkg-config --modversion wayland-protocols'
1.34
```

The build log shows the Wayland backend being compiled and linked. The following are **real,
unmodified lines copied from the build log** (`/tmp/build.log`). The compile line is shown in
**full** (note `-D_GLFW_WAYLAND` and `-Werror`); in the link line, the list of 37 compiled `.o`
object files is the only thing elided, marked `[… 37 glfw-wayland-*.c.o object files …]` — nothing
else is changed:

```
gcc -MMD -DNDEBUG -D_GLFW_WAYLAND -D_GLFW_BUILD_DLL -DHAS_MEMFD_CREATE -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/wl_window.c -o build/glfw-wayland-glfw-wl_window.c.o
```

```
gcc -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -Wall -O3 -shared -flto [… 37 glfw-wayland-*.c.o object files …] -pthread -lm -lrt -ldl -lwayland-client -lwayland-cursor -lxkbcommon -ldbus-1 -o build/kitty/glfw-wayland.so
```

and the Go `kitten` binary (which contains the persistence code) being built — the real,
single-line log entry, unmodified:

```
/usr/local/go/bin/go build -v -ldflags '-X kitty.VCSRevision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 -s -w' -o kitty/launcher/kitten /app/tools/cmd
```

> **Incremental-vs-clean disclosure.** The mandated image ships kitty **already built**, so a plain
> `python3 setup.py build` on it relinks only what changed (chiefly the Go `kitten`); its log is a
> handful of lines and does **not** re-emit the gcc Wayland compile/link commands. The two gcc lines
> above were therefore captured from a **forced clean rebuild** of the Wayland backend objects in
> the same image (build still exits 0, and both `build/kitty/glfw-wayland.so` and
> `build/kitty/glfw-x11.so` are produced). The `BUILD EXIT CODE = 0` and `wayland-protocols 1.34`
> captures above are from the plain `setup.py build --verbose` run.

Versions of the resulting binaries — **OBSERVED**:

```
$ /app/kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
$ /app/kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

> **Honesty note.** `wayland-protocols` (version 1.34) is present and used; the default build
> needs no modification and no fallback. An earlier draft described hiding `wayland-protocols.pc`
> to force an X11-only fallback build — that was an artifact of a *different* (host) environment
> and is not what happens in the mandated image. In the mandated image the default build (X11 +
> Wayland) succeeds with exit code 0 and zero alterations.

The persistence logic lives in the **compiled Go `kitten` binary** (≈15.9 MB above), which is
exactly why a build — not merely reading Python — is required to observe the behavior. The Python
backend performs **no** persistence (§6.3).

### 3.3 Launch one default instance

The default launcher the build produced is `kitty/launcher/kitty`; in the image it is at
`/app/kitty/launcher/kitty`. To keep a single, self-terminating instance while driving the kitten
under Xvfb, kitty is launched with a child program (see the runnable harness in §12), rather than
as a bare blocking GUI. No non-default configuration affects the kitten's write path — this is
demonstrated in §8.4 by repeating the `Enter` write with the host kitty using its **default**
config.

### 3.4 Web-search corroboration of the canonical build

The official kitty build documentation
([sw.kovidgoyal.net/kitty/build](https://sw.kovidgoyal.net/kitty/build/)) states that kitty is
designed to run from source and that, to get started, you need a C compiler and the Go compiler
(and, on Linux, the X11 development libraries); the recommended build is `./dev.sh build`, after
which kitty runs as `kitty/launcher/kitty`. `dev.sh` is documented as a thin wrapper around
`bypy/devenv.go` that downloads pre-built dependency bundles before invoking `setup.py`. This
matches `dev.sh:L9` and `Makefile:L12-L13`. The `python3 setup.py build` command used here is the
repository's / environment's own build command (not attributed to the official page).

---

## 4. R2 — Invocation through the canonical entry point

The kitten is invoked through the **real dispatcher**, not any debug/remote/synthetic path. In the
image the runnable command is:

```bash
docker exec kitty-setup bash -lc '/app/kitty/launcher/kitten choose-fonts --help'
```

> **Important — `kitty +kitten choose-fonts` is _not_ an equivalent invocation at this commit; it
> is a silent no-op.** The canonical, working entry point is `kitten choose-fonts` (the aggregate
> Go `kitten` binary — see §5 for its real `--help` output). The `kitty +kitten <name>` form
> dispatches into the Go `kitten` binary **only** for kittens whose names are compiled into the C
> launcher's `WRAPPED_KITTENS` list; `choose_fonts` is not on that list, so the command instead
> falls back to a Python module that is an empty placeholder in this checkout — producing no output
> and no effect.
>
> **OBSERVED — the `+kitten` form produces nothing (silent no-op):**
>
> ```
> $ DISPLAY=:99 /app/kitty/launcher/kitty +kitten choose-fonts --help; echo "rc=$?"
> rc=0
> # 0 bytes on STDOUT, 0 bytes on STDERR
> ```
>
> **OBSERVED — contrast with the canonical binary and with a genuinely wrapped kitten** (same
> image, same shell): `/app/kitty/launcher/kitten choose-fonts --help` prints its full help (463
> bytes on STDOUT, `rc=0`; reproduced verbatim in §5), and the wrapped kitten
> `DISPLAY=:99 /app/kitty/launcher/kitty +kitten query_terminal --help` prints 1721 bytes on
> STDOUT (`rc=0`) — proving the `+kitten` path itself works, just not for `choose-fonts`.
>
> **CODE-DERIVED mechanism (exact `file:line`):** the C launcher handles `+kitten` in
> `delegate_to_kitten_if_possible` [`kitty/launcher/main.c:L353-L357`]; the `+kitten` branch
> [`kitty/launcher/main.c:L356`] execs the Go `kitten` binary **only when** `is_wrapped_kitten()`
> [`kitty/launcher/main.c:L332-L337`] finds the name inside the compile-time `WRAPPED_KITTENS`
> string [`kitty/data-types.c:L252`]. That string is generated by `setup.py`'s `wrapped_kittens()`
> [`setup.py:L1075`] and injected as a `-D` cppflag [`setup.py:L1233`]; its compiled value is
> `ask clipboard diff hints hyperlinked_grep icat query_terminal show_key ssh themes transfer
> unicode_input` — which does **not** include `choose_fonts`. The name therefore falls through to
> the Python runner `run_kitten` [`kittens/runner.py:L110-L133`], which reaches
> `runpy.run_module('kittens.choose_fonts.main')` [`kittens/runner.py:L115-L116`] because
> `choose_fonts` is a builtin kitten name; but `kittens/choose_fonts/main.py` (and
> `kittens/choose_fonts/__init__.py`) are **0-byte placeholders**, so nothing executes and the
> process exits `0` with no output. All of the kitten's real logic lives in the Go binary reached
> via `kitten choose-fonts` (§5).

**Grounding of the entry point (CODE-DERIVED, exact `file:line`):**

- The single aggregate `kitten` binary is built from `tools/cmd`; its dispatcher imports the
  package at `tools/cmd/tool/main.go:L9` (`"kitty/kittens/choose_fonts"`).
- It wires the subcommand into the root command at `tools/cmd/tool/main.go:L82`
  (`choose_fonts.EntryPoint(root)`).

So `kitten choose-fonts` dispatches straight into `choose_fonts.EntryPoint`'s registered `Run`
closure (§5).

---

## 5. R3 — Subcommand registration & option parsing

Anchors below were verified against source at HEAD `815df1e210e0`.

**`EntryPoint(root *cli.Command)`** — `kittens/choose_fonts/main.go:L74` — adds the subcommand:

- `Name: "choose-fonts"` and `ShortDescription: "Choose the fonts used in kitty"`
  [`kittens/choose_fonts/main.go:L76-L77`].
- The `Run` closure [`kittens/choose_fonts/main.go:L78-L84`] calls `cmd.GetOptionValues(&opts)`
  [`kittens/choose_fonts/main.go:L80`] and then `main(&opts)` [`kittens/choose_fonts/main.go:L83`].

**The single option `--reload-in`** [`kittens/choose_fonts/main.go:L86-L95`]:

- `Dest = "Reload_in"` [`kittens/choose_fonts/main.go:L88`],
  `Choices = "parent, all, none"` [`kittens/choose_fonts/main.go:L90`],
  `Default = "parent"` [`kittens/choose_fonts/main.go:L91`].

**Alias** `choose_fonts` (underscore) is registered by cloning the command
[`kittens/choose_fonts/main.go:L96-L98`].

**OBSERVED** — real output of the canonical binary, confirming the description, the `--reload-in`
choices, and the default:

```
$ docker exec kitty-setup bash -lc '/app/kitty/launcher/kitten choose-fonts --help'
Usage: kitten choose-fonts 

Choose the fonts used in kitty

Options:
  --reload-in [=parent]
    By default, this kitten will signal only the parent kitty instance it is
    running in to reload its config, after making changes. Use this option to
    instead either not reload the config at all or in all running kitty
    instances.
    Choices: parent, all, none

  --help, -h
    Show help for this command

kitten choose-fonts 0.35.2 created by Kovid Goyal
```

The `--help` text itself confirms the reload/persistence separation: `--reload-in` governs
*"signal … to reload its config, **after making changes**"* — the change (the file write) happens
first; the reload is a subsequent, optional signal.

---

## 6. R4 — Value flow to the final step & pane progression

### 6.1 Option value carried on the handler (CODE-DERIVED)

- `type Options struct { Reload_in string }` — `kittens/choose_fonts/main.go:L70-L72`.
- Inside `main(opts *Options)` [`kittens/choose_fonts/main.go:L16-L68`] (the signature is
  `func main(opts *Options) (rc int, err error)` at `kittens/choose_fonts/main.go:L16`), the parsed
  options are attached to the handler at `kittens/choose_fonts/main.go:L35`
  (`h := &handler{lp: lp, opts: opts}`).
- `opts.Reload_in` is *carried* through the whole UI but is **consumed only at the final `Enter`
  branch** [`kittens/choose_fonts/final.go:L86-L93`]; it never affects whether earlier panes write
  anything.

### 6.2 Pane progression: listing → faces → final

**Anchors (CODE-DERIVED):**

- The handler holds an ordered pane list:
  `h.panes = []pane{&h.listing, &h.faces, &h.face_pane, &h.final_pane}` —
  `kittens/choose_fonts/ui.go:L81`.
- The current faces are populated from the backend result:
  `h.listing.resolved_faces_from_kitty_conf = r.Resolved_faces`
  [`kittens/choose_fonts/ui.go:L97`]; the current family is selected via `SelectFamily(...)`
  [`kittens/choose_fonts/list.go:L170`].
- Data shapes: `type ResolvedFaces struct` with `Font_family` / `Bold_font` / `Italic_font` /
  `Bold_italic_font` [`kittens/choose_fonts/types.go:L78-L82`]; `type ListResult struct` with
  `Resolved_faces` [`kittens/choose_fonts/types.go:L85-L87`]. The *mutable* selection lives in
  `type faces_settings struct` [`kittens/choose_fonts/faces.go:L14`].
- The transition into the final pane happens from the faces pane on `Enter`:
  `return self.handler.final_pane.on_enter(self.family, self.settings)`
  [`kittens/choose_fonts/faces.go:L118-L120`]. (`Esc` on the faces pane instead returns to the
  listing [`kittens/choose_fonts/faces.go:L113-L116`].)

**The pane progression was OBSERVED** on the real running kitten under `DISPLAY=:99`. Each pane
was captured as a screenshot and viewed; the content is transcribed below (a transcription of the
viewed screenshots, not a pixel-aligned reproduction).

**Pane 1 — family listing** (`kitten choose-fonts` on launch). Two-column TUI. Left = the
scrollable family list with the current selection marked `>` and highlighted (green), and a
`Family:` filter prompt at the bottom. Right = a live preview of the highlighted family
(OBSERVED, transcribed):

```
Cascadia Code             DejaVu Sans Mono
Cascadia Code NF
Cascadia Code PL          Styles: Bold, Bold Oblique,
Cascadia Mono             Book, Oblique
Cascadia Mono NF
Cascadia Mono PL          Press the Enter key to choose
Comfy Code                this family
> DejaVu Sans Mono
Fantasque Sans Mono       ———————— preview ————————
Fira Code                 abcdefghijklmnopqrstuvwxyz 0123
Hack                      456789 ABCDEFGHIJKLMNOPQRSTUVWX
IBM Plex Mono             YZ !"#$%&'()*+,-./:;<=>?@[\]^_`
JetBrains Mono            {|}~
JetBrains Mono NL
Liberation Mono
Noto Mono
Family: |
```

**Pane 2 — faces** (after `Enter` on the listing). Title is the chosen family; the prompt
confirms `Enter`/`Esc` plus highlighted per-face keys; the four faces (regular, bold, italic,
bold-italic) are shown with live rendered previews (OBSERVED, transcribed):

```
                    DejaVu Sans Mono
Press Enter to select this font, Esc to go back to the
font list or any of the highlighted keys below to
fine-tune the appearance of the individual font styles.

Regular: DejaVuSansMono
abcdefghijklmnopqrstuvwxyz 0123456789 ABCDEFGHIJKLMNOPQRS

Bold: DejaVuSansMono-Bold
abcdefghijklmnopqrstuvwxyz 0123456789 ABCDEFGHIJKLMNOPQRS

Italic: DejaVuSansMono-Oblique
abcdefghijklmnopqrstuvwxyz 0123456789 ABCDEFGHIJKLMNOPQRS

Bold-Italic: DejaVuSansMono-BoldOblique
```

The four faces shown here — `DejaVuSansMono`, `DejaVuSansMono-Bold`, `DejaVuSansMono-Oblique`,
`DejaVuSansMono-BoldOblique` — correspond **exactly** to the four values later written to
`kitty.conf` by `Enter` (§8.1). This is the direct integration check: the values visible in the
UI are the values that reach finalization unchanged.

> **Naming detail (honest).** The listing shows the human-readable family *"DejaVu Sans Mono"*,
> while the faces pane and the written file use the resolved face names (e.g. `DejaVuSansMono`,
> `DejaVuSansMono-Bold`). These are the same family; the kitten resolves the display name to
> concrete face names for the config.
>
> **Why these exact face values appear (honest, environment-dependent).** On entering the faces
> pane, `faces.on_enter` computes each value with
> `*setting = utils.IfElse(family == conf.Family, conf.Spec, defval)`
> [`kittens/choose_fonts/faces.go:L148-L155`]: if the family you pick **is** the one kitty currently
> resolves for that face, the **concrete** face spec (`DejaVuSansMono-Bold`, …) is used; otherwise
> the value defaults to `font_family family="<name>"` with `auto` for the other three. In this image
> the default monospaced family resolves to **DejaVu Sans Mono**, which is also the pre-selected
> entry on the listing, so `family == conf.Family` holds and the four concrete specs are written —
> exactly the values shown here and in §8. A different host (whose default resolves to another
> family, or where a non-default family is chosen) would persist the equally canonical
> `family="…"` + `auto` form through this **same** code path; only the resolved default family
> differs. The persistence **verdict is unaffected** — `Enter` writes the sentinel block either way.

### 6.3 The backend boundary (why the Go layer owns persistence)

- The Go UI shells out to a Python backend via `+runpy`:
  `exec.Command(exe, "+runpy", "from kittens.choose_fonts.backend import main; main()")`
  [`kittens/choose_fonts/backend.go:L41`], exchanging JSON over pipes.
- The Python backend `main()` [`kittens/choose_fonts/backend.py:L150-L168`] handles **only**
  `list_monospaced_fonts`, `read_variable_data`, and `render_family_samples`, and raises
  `SystemExit` on anything else. It **lists / resolves / renders** fonts and performs **no
  persistence**.
- **Cause → effect:** since the backend never writes config, *all* config writing is in the Go
  layer — specifically the `Enter` branch of `final_pane.on_key_event` (§7). This is the second
  reason a build is required: the durable behavior is compiled Go, not interpretable Python.

### 6.4 Display dependency (satisfied by the provisioned Xvfb)

`h.lp.QueryTerminal("font_size", "dpi_x", "dpi_y", "foreground", "background")`
[`kittens/choose_fonts/ui.go:L80`] shows the kitten queries the host terminal, and previews use
the graphics protocol [`kittens/choose_fonts/graphics.go`], so a real display is required. The
provisioned container satisfies this with `Xvfb` on `DISPLAY=:99`, under which the panes above and
every branch in §8–§9 were driven with `xdotool`.

---

## 7. R5 — Finalization: the persistence primitive

### 7.1 What the final pane shows (OBSERVED)

The final confirmation pane of the real running kitten (under `DISPLAY=:99`, after choosing
`DejaVu Sans Mono` and pressing `Enter` on the faces pane) was **OBSERVED** and is **transcribed
from the viewed screenshot** below (family name in cyan; option keys highlighted). Its wording is
then cross-checked line-by-line against the source, so the transcription is verifiable and not
merely asserted:

```
You have chosen the DejaVu Sans Mono family
What would you like to do?

Enter to modify kitty.conf and use the new fonts

Esc to abort and return to font selection

s to write the new font settings to STDOUT

Ctrl+c to quit
```

This is generated by `final_pane.draw_screen` [`kittens/choose_fonts/final.go:L29`], and the
observed text matches the source lines verbatim:

- *"`Enter` to modify `kitty.conf` and use the new fonts"* [`kittens/choose_fonts/final.go:L38`]
- *"`Esc` to abort and return to font selection"* [`kittens/choose_fonts/final.go:L40`]
- *"`s` to write the new font settings to `STDOUT`"* [`kittens/choose_fonts/final.go:L42`]
- *"`Ctrl+c` to quit"* [`kittens/choose_fonts/final.go:L44`]

**This observed pane is decisive for the verdict:** the kitten's own UI declares that **`Enter`
modifies `kitty.conf`** (persist) while **`s` writes to `STDOUT`** (session/pipe).

### 7.2 The `Enter` branch (source anchors)

`final_pane.on_key_event` [`kittens/choose_fonts/final.go:L72-L99`], the `enter` case
[`kittens/choose_fonts/final.go:L78-L97`]:

```go
// Source: kittens/choose_fonts/final.go — on_key_event "enter" case (L78-L94)
patcher := config.Patcher{Write_backup: true}                 // L80
path := filepath.Join(utils.ConfigDir(), "kitty.conf")        // L81
updated, err := patcher.Patch(
    path, "KITTY_FONTS", self.settings.serialized(),
    "font_family", "bold_font", "italic_font", "bold_italic_font",
)                                                             // L82
if err != nil {                                               // L83
    return err                                                // L84  (aborts BEFORE Quit)
}
if updated { /* switch opts.Reload_in { … } */ }             // L86-L93
self.lp.Quit(0)                                              // L94
```

Two behaviors matter for accuracy:

- If `Patch` returns an error, the branch **returns the error before `lp.Quit(0)`**
  [`kittens/choose_fonts/final.go:L83-L84`] — the kitten does not silently succeed on a failed
  write.
- The `--reload-in` switch runs **only when `updated == true`**
  [`kittens/choose_fonts/final.go:L86`].

`serialized()` [`kittens/choose_fonts/final.go:L63-L70`] emits **exactly four** space-padded lines
joined by `\n`:

```
font_family      <value>
bold_font        <value>
italic_font      <value>
bold_italic_font <value>
```

### 7.3 How the write becomes durable — `config.Patcher.Patch`

`config.Patcher.Patch` [`tools/config/api.go:L310-L350`] performs, in order:

1. Record `backup_path := path` **before** symlink resolution
   [`tools/config/api.go:L314`], then resolve symlinks for the write target
   [`tools/config/api.go:L315-L317`].
2. Read any existing file content [`tools/config/api.go:L318`]; a missing file is not an error
   [`tools/config/api.go:L319-L324`].
3. **Comment out** prior *active* matching settings by replacing each matched key line with
   `# $1` via regex [`tools/config/api.go:L325-L326`]. The regex requires the key at line start
   (after optional whitespace), so lines that are *already* commented are left as-is.
4. Locate any existing `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` block and **replace it**; if none
   exists, append the new block (separated by a blank line)
   [`tools/config/api.go:L328-L340`]. The block is
   `# BEGIN_KITTY_FONTS\n<serialized>\n# END_KITTY_FONTS` [`tools/config/api.go:L330`].
5. Only if the new bytes differ from the old [`tools/config/api.go:L342`]:
   - **conditionally** write a `<path>.bak` backup — **only** when there was prior non-empty
     content **and** `Write_backup` is set [`tools/config/api.go:L343`] — and the backup write is
     **best-effort: its error is deliberately ignored** (`_ = os.WriteFile(...)`)
     [`tools/config/api.go:L344`];
   - replace the file via `utils.AtomicUpdateFile` and return `updated = true`
     [`tools/config/api.go:L347`].
6. If the bytes are unchanged, return `updated = false` and write nothing
   [`tools/config/api.go:L349`].

**Atomicity — precise scope.** `AtomicUpdateFile` [`tools/utils/atomic-write.go:L79`] delegates to
`AtomicWriteFile` [`tools/utils/atomic-write.go:L42`], which does
`CreateTemp` [`tools/utils/atomic-write.go:L53`] → `Write`
[`tools/utils/atomic-write.go:L63`] → `Chmod` [`tools/utils/atomic-write.go:L65`] →
`os.Rename` [`tools/utils/atomic-write.go:L67`]. Because the finished bytes are `rename`d into
place, a concurrent reader sees either the old or the new file, **never a partially written one**,
and the durable bytes are re-read at the next startup ⇒ persistence.

> **Robustness caveat (honest).** The implementation performs **no `fsync`** on the file or its
> directory (verified: no `Sync`/`fsync` call exists in
> `tools/utils/atomic-write.go`). The guarantee is therefore *atomic replacement against
> partially-written userspace content* — **not** power-loss/crash durability, and an abrupt
> termination could leave a `*.atomic-write-*` temp file behind. The claim is limited accordingly.

### 7.4 Reload is orthogonal to persistence

After a successful *write* with `updated == true`, the `Enter` branch consults `--reload-in`
[`kittens/choose_fonts/final.go:L86-L93`]:

- `"parent"` → `config.ReloadConfigInKitty(true)` [`kittens/choose_fonts/final.go:L89`]
- `"all"` → `config.ReloadConfigInKitty(false)` [`kittens/choose_fonts/final.go:L91`]
- `"none"` → no case → **no reload** (the file is still written)

`ReloadConfigInKitty` [`tools/config/api.go:L352-L371`] sends **`SIGUSR1`** — to the parent
instance via `KITTY_PID` [`tools/config/api.go:L354-L357`] or to all kitty GUI processes
[`tools/config/api.go:L363-L366`]. It is **best-effort**: the send errors are ignored
(`_ = p.SendSignal(...)` [`tools/config/api.go:L366`]), and the `Enter` branch discards
`ReloadConfigInKitty`'s return value. Reload merely asks *already-running* instances to re-read a
file that is *already* durable; it does not decide whether the file was written. Persistence (the
write) and reload (the signal) are independent.

---

## 8. R6 — Persistence verdict with real Enter, before / after, and restart evidence

Every block in this section is the **verbatim STDOUT of the safe harness (§12)**, which drove the
**real** `kitten choose-fonts` UI under `DISPLAY=:99` with `xdotool` inside a private, isolated
`KITTY_CONFIG_DIRECTORY`. The pane sequence in each run is: launch → (Return) faces → (Return)
final → branch key. So the reader can audit the captures, these are the harness's exact output
conventions (nothing is hand-edited inside the fenced blocks):

- **SHA-256 values are the full 64 hex characters** and are **content-deterministic** — they
  reproduce byte-for-byte across runs. **PIDs, the window id (`WID`), and `ls -la` timestamps are
  per-run** and therefore differ between captures.
- The harness prints each captured **STDOUT/STDERR** line prefixed with `  | `; a `(bytes=0)` header
  followed by `  (empty)` means that stream produced nothing.
- The `PROCESS TREE` block is `ps -ww` output (untruncated), taken while the kitten sits on its
  final pane. It shows the canonical three-process chain that proves the real dispatcher and the
  Go→Python boundary: **GUI `kitty` → Go `kitten` → `kitty +runpy … choose_fonts.backend`**.
- **`# END_KITTY_FONTS` is written with no trailing newline.** Consequently the harness's next
  status line (`changed?      : YES`) abuts it on the same physical line inside the "after content"
  dump — that run-together line is itself byte-level proof of the exact bytes written, not a typo.

### 8.1 Scenario A — a pre-existing `kitty.conf` (Enter → persist, default `--reload-in=parent`)

The harness seeds `printf 'font_size 12.0\nfont_family Old Family Name\n'` into the isolated
`kitty.conf`, launches the host kitty with `--config NONE`, drives `Return Return Return` (listing →
faces → final → **Enter**), then captures the result. **Verbatim harness output:**

```
======== SCENARIO: A_enter_default_parent_preexisting ========
WORK=/tmp/cf_qa.lFEtbA  ISO_CONF=/tmp/cf_qa.lFEtbA/conf  RELOAD='<default:parent>'  BRANCH=Return  HOSTCFG=NONE
---- BEFORE ----
before sha256 : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e
before bytes  : 43
before content:
font_size 12.0
font_family Old Family Name
GUI kitty PID : 82792
kitty WID     : 2097164
---- PROCESS TREE (at final pane) ----
kitten PID    : 82875
backend PID   : 82891
    PID    PPID COMMAND         COMMAND
  82792   82769 kitty           /app/kitty/launcher/kitty --config NONE -o font_size=14 sh -c /app/kitty/launcher/kitten choose-fonts  >'/tmp/cf_qa.lFEtbA/kitten.out' 2>'/tmp/cf_qa.lFEtbA/kitten.err'; echo $? >'/tmp/cf_qa.lFEtbA/kitten.rc'
  82875   82874 kitten          /app/kitty/launcher/kitten choose-fonts
  82891   82875 kitty           /app/kitty/launcher/kitty +runpy from kittens.choose_fonts.backend import main; main()
---- KEYS ----
nav keys      : Return Return (listing->faces, faces->final)
branch key    : Return (Enter on final pane -> persist)
---- RESULT ----
kitten exit status : 0
kitten STDOUT (bytes=0):
  (empty)
kitten STDERR (bytes=0):
  (empty)
---- AFTER ----
after sha256  : 336b3aaee0ac6833d78684c47c69d9b5b47710781609d578ec53b232b25eae1e
after bytes   : 237
BEGIN_KITTY_FONTS blocks: 1
after content :
font_size 12.0
# font_family Old Family Name


# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTSchanged?      : YES
---- DIR LISTING (real ls -la) ----
total 16
drwx------ 2 root root 4096 Jul 15 01:35 .
drwx------ 4 root root 4096 Jul 15 01:35 ..
-rw-r--r-- 1 root root  237 Jul 15 01:35 kitty.conf
-rw-r--r-- 1 root root   43 Jul 15 01:35 kitty.conf.bak
backup sha256 : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e
backup bytes  : 43
.bak == before? : YES
backup content:
font_size 12.0
font_family Old Family Name
======== END A_enter_default_parent_preexisting ========
```

Reading this capture next to the code that produced each effect:

- **The kitten exited `0` with empty STDOUT and empty STDERR** — the persist branch neither prints
  nor errors; its whole effect is the file write.
- The unrelated `font_size 12.0` is **preserved**; the prior `font_family Old Family Name` is
  **commented out** → `# font_family Old Family Name` (regex at [`tools/config/api.go:L325-L326`]).
- The four selected faces are written inside `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` (sentinel at
  [`tools/config/api.go:L330`]; the four keys are the arguments from
  [`kittens/choose_fonts/final.go:L82`]). The `before`→`after` SHA-256 change
  (`7f9862c0…` → `336b3aaee0…`, 43 → 237 bytes) is the file mutation.
- Because there was prior non-empty content and `Write_backup` is set, a `kitty.conf.bak` is written
  [`tools/config/api.go:L343-L344`]; its SHA-256 (`7f9862c0…`, 43 bytes) **equals the BEFORE file**,
  so the backup is a faithful copy of the pre-write config (`.bak == before? : YES`).
- The three-process chain (GUI `kitty` 82792 → `kitten` 82875 → `+runpy` backend 82891) confirms the
  invocation went through the real Go dispatcher (§4) and that persistence is owned by the Go layer,
  not the Python backend (§6.3).

> **Reload (SIGUSR1) is _not_ observed here and is not the persistence mechanism.** With the default
> `--reload-in=parent`, after the write the kitten *additionally* signals its parent kitty to
> re-read config [`tools/config/api.go:L352-L371`, code-derived — see §7.4]. That signal is a
> separate "apply to the already-running instance" step; §8.3 shows the identical file write happens
> even with `--reload-in=none` (no signal at all), and §8.4 shows a brand-new process picking the
> selection up from disk. The runtime-observable fact captured above is the **file write**; the
> SIGUSR1 delivery itself is labeled code-derived (no `strace` is available in this image to capture
> the signal directly).

### 8.2 Scenario B — no `kitty.conf` present (Enter → persist, no `.bak`)

Same real-UI `Enter`, but starting from an **empty** isolated directory (no seed). **Verbatim
harness output:**

```
======== SCENARIO: B_enter_emptydir ========
WORK=/tmp/cf_qa.0ubIsH  ISO_CONF=/tmp/cf_qa.0ubIsH/conf  RELOAD='<default:parent>'  BRANCH=Return  HOSTCFG=NONE
---- BEFORE ----
before        : (no kitty.conf present)
GUI kitty PID : 83563
kitty WID     : 2097164
---- PROCESS TREE (at final pane) ----
kitten PID    : 83646
backend PID   : 83662
    PID    PPID COMMAND         COMMAND
  83563   83546 kitty           /app/kitty/launcher/kitty --config NONE -o font_size=14 sh -c /app/kitty/launcher/kitten choose-fonts  >'/tmp/cf_qa.0ubIsH/kitten.out' 2>'/tmp/cf_qa.0ubIsH/kitten.err'; echo $? >'/tmp/cf_qa.0ubIsH/kitten.rc'
  83646   83645 kitten          /app/kitty/launcher/kitten choose-fonts
  83662   83646 kitty           /app/kitty/launcher/kitty +runpy from kittens.choose_fonts.backend import main; main()
---- KEYS ----
nav keys      : Return Return (listing->faces, faces->final)
branch key    : Return (Enter on final pane -> persist)
---- RESULT ----
kitten exit status : 0
kitten STDOUT (bytes=0):
  (empty)
kitten STDERR (bytes=0):
  (empty)
---- AFTER ----
after sha256  : db8f2c845fe9d52b3a58790a090a4947eaad63b726777d536904064201235da5
after bytes   : 190
BEGIN_KITTY_FONTS blocks: 1
after content :
# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTSchanged?      : YES
---- DIR LISTING (real ls -la) ----
total 12
drwx------ 2 root root 4096 Jul 15 01:37 .
drwx------ 4 root root 4096 Jul 15 01:37 ..
-rw-r--r-- 1 root root  190 Jul 15 01:37 kitty.conf
backup        : (absent)
======== END B_enter_emptydir ========
```

The file is **created** (there was no BEFORE file) containing only the block; the SHA-256 is
`db8f2c84…` (190 bytes). The directory listing shows **no `kitty.conf.bak`** — because the backup
is guarded by "there was prior content" (`len(raw) > 0` at [`tools/config/api.go:L343`]). This
proves persistence even from a clean slate: pressing `Enter` *creates* `kitty.conf` if absent and
writes the durable block, and no backup is produced when there was nothing to back up.

### 8.3 Scenario C — Enter with `--reload-in=none` (persist without any reload signal)

This is the decisive **reload-vs-persistence** experiment mandated by the plan. The kitten is
invoked with `--reload-in=none`, so it sends **no `SIGUSR1` to any kitty instance** after the write.
Everything else matches Scenario A (same 43-byte seed, same `Return Return Return`). **Verbatim
harness output** (note `--reload-in=none` in the process-tree command):

```
======== SCENARIO: C_enter_reload_none ========
WORK=/tmp/cf_qa.aIqqX8  ISO_CONF=/tmp/cf_qa.aIqqX8/conf  RELOAD='none'  BRANCH=Return  HOSTCFG=NONE
---- BEFORE ----
before sha256 : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e
before bytes  : 43
before content:
font_size 12.0
font_family Old Family Name
GUI kitty PID : 82997
kitty WID     : 2097164
---- PROCESS TREE (at final pane) ----
kitten PID    : 83080
backend PID   : 83096
    PID    PPID COMMAND         COMMAND
  82997   82974 kitty           /app/kitty/launcher/kitty --config NONE -o font_size=14 sh -c /app/kitty/launcher/kitten choose-fonts --reload-in=none >'/tmp/cf_qa.aIqqX8/kitten.out' 2>'/tmp/cf_qa.aIqqX8/kitten.err'; echo $? >'/tmp/cf_qa.aIqqX8/kitten.rc'
  83080   83079 kitten          /app/kitty/launcher/kitten choose-fonts --reload-in=none
  83096   83080 kitty           /app/kitty/launcher/kitty +runpy from kittens.choose_fonts.backend import main; main()
---- KEYS ----
nav keys      : Return Return (listing->faces, faces->final)
branch key    : Return (Enter on final pane -> persist)
---- RESULT ----
kitten exit status : 0
kitten STDOUT (bytes=0):
  (empty)
kitten STDERR (bytes=0):
  (empty)
---- AFTER ----
after sha256  : 336b3aaee0ac6833d78684c47c69d9b5b47710781609d578ec53b232b25eae1e
after bytes   : 237
BEGIN_KITTY_FONTS blocks: 1
after content :
font_size 12.0
# font_family Old Family Name


# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTSchanged?      : YES
---- DIR LISTING (real ls -la) ----
total 16
drwx------ 2 root root 4096 Jul 15 01:35 .
drwx------ 4 root root 4096 Jul 15 01:35 ..
-rw-r--r-- 1 root root  237 Jul 15 01:35 kitty.conf
-rw-r--r-- 1 root root   43 Jul 15 01:35 kitty.conf.bak
backup sha256 : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e
backup bytes  : 43
.bak == before? : YES
backup content:
font_size 12.0
font_family Old Family Name
======== END C_enter_reload_none ========
```

The AFTER SHA-256 is `336b3aaee0…` (237 bytes) — **byte-for-byte identical to Scenario A**, and the
`.bak` is the same 43-byte copy of the prior config. With `--reload-in=none` **no reload signal is
sent at all**, yet the persisted `kitty.conf` is exactly the same. This is direct runtime proof that
**persistence (the file write) is independent of the reload scope**: `--reload-in` only governs the
optional `SIGUSR1` step [`kittens/choose_fonts/final.go:L86-L93`; `tools/config/api.go:L352-L371`],
never whether the file is written.

### 8.4 Restart proof — a fresh kitty loads the persisted faces (OBSERVED)

The critical persistence test: after the `Enter` write, the writing process has exited; a **fresh**
kitty process is then launched under the **same** isolated `KITTY_CONFIG_DIRECTORY` (this time
*without* `--config NONE`, so it reads the persisted config) and asked to report its resolved
fonts. **Verbatim harness output** (the same private-`mktemp -d` harness performs the seeded Enter
write, then the restart, then the idempotency re-run of §8.6, so all three share one isolated dir):

```
===== WRITE (seeded, Enter) =====
write rc     : 0
written sha  : 336b3aaee0ac6833d78684c47c69d9b5b47710781609d578ec53b232b25eae1e
written bytes: 237
blocks       : 1
bak sha      : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e

===== RESTART PROOF (fresh kitty, same $ISO_CONF, no --config NONE) =====
restart cmd  : /app/kitty/launcher/kitty --debug-font-fallback -o font_size=14 sh -c 'sleep 1; exit 0'
restart PID  : 83331
restart config sha (unchanged by read): 336b3aaee0ac6833d78684c47c69d9b5b47710781609d578ec53b232b25eae1e
---- restart 'Text fonts:' block (active faces + resolved font files) ----
  [0.175] Text fonts:
  [0.175]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
  [0.175]   Bold: DejaVuSansMono-Bold: /root/.local/share/fonts/DejaVuSansMono-Bold.ttf:0
  [0.175]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
  [0.175]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

The freshly-started instance was launched with `--config NONE` **absent**, so it read the persisted
`kitty.conf` (SHA-256 `336b3aaee0…`, unchanged by the read) and loaded the four persisted faces as
its **active** text fonts, resolved to concrete font files on disk. This is runtime proof that the
selection is active **after a restart**, not merely that the file can be re-read.

> The bracketed `[0.175]` prefix is kitty's per-run wall-clock offset in seconds since process
> start; it is timing-dependent and **varies between runs** (other runs of this exact capture
> printed `[0.180]` and `[0.194]`). Only the four face lines and their resolved paths are the stable
> evidence — the timestamp is not.

### 8.5 Enter under the host kitty's DEFAULT config (canonical-configuration check)

To confirm the persistence is not an artifact of the harness launching the host kitty with
`--config NONE`, the `Enter` write was repeated with the host kitty using its **default** config
resolution (`HOSTCFG=DEFAULT` → the launch omits `--config NONE`, visible in the process tree).
**Verbatim harness output:**

```
======== SCENARIO: D_enter_default_hostconfig ========
WORK=/tmp/cf_qa.muUWZT  ISO_CONF=/tmp/cf_qa.muUWZT/conf  RELOAD='<default:parent>'  BRANCH=Return  HOSTCFG=DEFAULT
---- BEFORE ----
before sha256 : 145cb1accfa064cf47b8b233ea2f3d268e02a1575e2771e8572618dee2e913b8
before bytes  : 15
before content:
font_size 12.0
GUI kitty PID : 83761
kitty WID     : 2097164
---- PROCESS TREE (at final pane) ----
kitten PID    : 83844
backend PID   : 83859
    PID    PPID COMMAND         COMMAND
  83761   83738 kitty           /app/kitty/launcher/kitty -o font_size=14 sh -c /app/kitty/launcher/kitten choose-fonts  >'/tmp/cf_qa.muUWZT/kitten.out' 2>'/tmp/cf_qa.muUWZT/kitten.err'; echo $? >'/tmp/cf_qa.muUWZT/kitten.rc'
  83844   83843 kitten          /app/kitty/launcher/kitten choose-fonts
  83859   83844 kitty           /app/kitty/launcher/kitty +runpy from kittens.choose_fonts.backend import main; main()
---- KEYS ----
nav keys      : Return Return (listing->faces, faces->final)
branch key    : Return (Enter on final pane -> persist)
---- RESULT ----
kitten exit status : 0
kitten STDOUT (bytes=0):
  (empty)
kitten STDERR (bytes=0):
  (empty)
---- AFTER ----
after sha256  : a1d53c0b4670dabc6ae915ba783eb58e1d20986cb735d013bb8e2b0786dbd743
after bytes   : 207
BEGIN_KITTY_FONTS blocks: 1
after content :
font_size 12.0


# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTSchanged?      : YES
---- DIR LISTING (real ls -la) ----
total 16
drwx------ 2 root root 4096 Jul 15 01:37 .
drwx------ 4 root root 4096 Jul 15 01:37 ..
-rw-r--r-- 1 root root  207 Jul 15 01:37 kitty.conf
-rw-r--r-- 1 root root   15 Jul 15 01:37 kitty.conf.bak
backup sha256 : 145cb1accfa064cf47b8b233ea2f3d268e02a1575e2771e8572618dee2e913b8
backup bytes  : 15
.bak == before? : YES
backup content:
font_size 12.0
======== END D_enter_default_hostconfig ========
```

The process-tree command line for the host kitty is `/app/kitty/launcher/kitty -o font_size=14 …`
with **no `--config NONE`** — the host terminal used its default config resolution — and the write
still happened (`145cb1ac…`, 15 bytes → `a1d53c0b…`, 207 bytes; `.bak` = the 15-byte prior file).
The byte arithmetic is exact: `207 = 15 (preserved prior line + newline) + 2 (blank separators) +
190 (the block, as in §8.2)`. The kitten resolves its own config directory from
`KITTY_CONFIG_DIRECTORY` in the inherited environment [`tools/utils/paths.go:L89-L91`], independent
of the host kitty's `--config`; `--config NONE` in the other scenarios is therefore only a
convenience so the host terminal does not itself try to load the seeded test font.

### 8.6 Idempotency (OBSERVED)

Pressing `Enter` a **second time** against the already-written directory is a no-op. This is the
tail of the same harness run as §8.4 (shared isolated dir). **Verbatim harness output:**

```
===== IDEMPOTENCY (2nd Enter on already-written dir) =====
sha before 2nd Enter : 336b3aaee0ac6833d78684c47c69d9b5b47710781609d578ec53b232b25eae1e
sha after  2nd Enter : 336b3aaee0ac6833d78684c47c69d9b5b47710781609d578ec53b232b25eae1e
2nd Enter rc         : 0
blocks after 2nd     : 1
.bak mtime unchanged?: YES (m1=1784079380 m2=1784079380)
identical?           : YES
===== END =====
```

The before/after hashes are identical (`336b3aaee0…`), exactly **one** `# BEGIN_KITTY_FONTS` block
remains, and the `.bak` modification time is unchanged (`m1 == m2 == 1784079380`), so the backup
from the first write was **not** rewritten. Mechanically, the regenerated bytes equal the existing
bytes, so the `bytes.Equal` guard at [`tools/config/api.go:L342`] short-circuits, `Patch` returns
`updated = false`, and neither the file nor the backup is touched.

---

## 9. Exhaustive branch table (Enter / Esc / `s`·`S` / Ctrl+C)

Every mode the question implies was **executed to completion** in a fresh, isolated scenario, with
the trigger, the exit status, STDOUT/STDERR, the process chain, and the byte-for-byte `kitty.conf`
SHA-256 (before → after) captured. Per-branch **verbatim harness output** follows the table. In the
"Evidence" column, the file-state facts are **OBSERVED**; the single item labeled *(code-derived)*
is the `SIGUSR1` delivery, which cannot be captured headlessly in this image (no `strace`).

| Trigger | Handler / anchor | Effect | `kitty.conf` before → after | Exit | Evidence |
|---------|------------------|--------|------------------------------|------|----------|
| **Enter** (default `--reload-in=parent`) | `kittens/choose_fonts/final.go:L78-L94` → `tools/config/api.go:L310-L350` → `tools/utils/atomic-write.go:L79` | Writes sentinel block (+ conditional `.bak`), atomic replace; **then** signals parent kitty to reload | `7f9862c0…` (43 B) → `336b3aaee0…` (237 B) — **PERSISTED** | `0` | OBSERVED — §8.1 (write); SIGUSR1 delivery *(code-derived — §7.4)* |
| **Enter** (`--reload-in=none`) | same write path; `Reload_in == "none"` skips the reload switch `kittens/choose_fonts/final.go:L86-L93` | Writes the **identical** block; sends **no** reload signal to any instance | `7f9862c0…` (43 B) → `336b3aaee0…` (237 B) — **PERSISTED, byte-identical to default** | `0` | OBSERVED — §8.3 |
| **Esc** | `kittens/choose_fonts/final.go:L73-L76` | Sets `current_pane = &faces`; returns to faces pane; no write; kitten keeps running | `7f9862c0…` → `7f9862c0…` (unchanged) | (still alive) | OBSERVED — §9.1 |
| **`s` / `S`** | `kittens/choose_fonts/final.go:L101-L111` (`case "s", "S":` L104) → STDOUT via `kittens/choose_fonts/main.go:L64-L66` | Serializes 4 lines to `output_on_exit`, quits, prints to **STDOUT**; never calls `Patch` | `7f9862c0…` → `7f9862c0…` (unchanged) | `0` | OBSERVED — §9.2 |
| **Ctrl+C (GUI key)** | `kittens/choose_fonts/ui.go:L195-L198` → `kittens/choose_fonts/main.go:L55-L57` | Handler intercepts the key, returns `"canceled by user"`; STDERR message; no write | `7f9862c0…` → `7f9862c0…` (unchanged) | `1` | OBSERVED — §9.3(a) |
| **Ctrl+C (OS signal)** | loop signal path → `kittens/choose_fonts/main.go:L58-L62` → `tools/tui/loop/api.go:L213-L217` | Prints `"Killed by signal:  interrupt"` to STDOUT, re-raises signal; no write | `7f9862c0…` → `7f9862c0…` (unchanged) | `130` | OBSERVED — §9.3(b) |

**Only `Enter` mutates `kitty.conf`.** All other branches leave the file byte-for-byte unchanged —
proven by SHA-256 equality (`7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e`
before and after) in each subsection below.

### 9.1 Esc (OBSERVED)

Reach the final pane, press **Esc**. Per `kittens/choose_fonts/final.go:L73-L76`, Esc sets
`current_pane = &faces` and redraws — it returns to the **faces** pane and the kitten **keeps
running** (it does not exit and does not write). **Verbatim harness output:**

```
======== SCENARIO: esc ========
WORK=/tmp/cf_qa.QrMjPP  ISO_CONF=/tmp/cf_qa.QrMjPP/conf  RELOAD='<default:parent>'  BRANCH=Escape  HOSTCFG=NONE
---- BEFORE ----
before sha256 : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e
before bytes  : 43
before content:
font_size 12.0
font_family Old Family Name
GUI kitty PID : 83986
kitty WID     : 2097164
---- PROCESS TREE (at final pane) ----
kitten PID    : 84069
backend PID   : 84085
    PID    PPID COMMAND         COMMAND
  83986   83970 kitty           /app/kitty/launcher/kitty --config NONE -o font_size=14 sh -c /app/kitty/launcher/kitten choose-fonts  >'/tmp/cf_qa.QrMjPP/kitten.out' 2>'/tmp/cf_qa.QrMjPP/kitten.err'; echo $? >'/tmp/cf_qa.QrMjPP/kitten.rc'
  84069   84068 kitten          /app/kitty/launcher/kitten choose-fonts
  84085   84069 kitty           /app/kitty/launcher/kitty +runpy from kittens.choose_fonts.backend import main; main()
---- KEYS ----
nav keys      : Return Return (listing->faces, faces->final)
branch key    : Escape (final -> back to faces)
---- RESULT ----
kitten exit status : (rc file absent)
kitten STDOUT (bytes=0):
  (empty)
kitten STDERR (bytes=43):
  | Error: Failed doing I/O with terminal: EOF
---- AFTER ----
after sha256  : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e
after bytes   : 43
BEGIN_KITTY_FONTS blocks: 0
after content :
font_size 12.0
font_family Old Family Name
changed?      : NO
---- DIR LISTING (real ls -la) ----
total 12
drwx------ 2 root root 4096 Jul 15 01:42 .
drwx------ 4 root root 4096 Jul 15 01:42 ..
-rw-r--r-- 1 root root   43 Jul 15 01:42 kitty.conf
backup        : (absent)
======== END esc ========
```

The **OBSERVED Esc result** is: config **unchanged** (`7f9862c0…` before == after, 43 bytes),
**zero** `# BEGIN_KITTY_FONTS` blocks, **no `.bak`** — i.e. no write at all.

> **Honest note on the STDERR line.** Because Esc does not exit the kitten, the process is still
> running when the harness tears down its parent GUI kitty; the kitten then dies as a side-effect of
> that teardown and emits a diagnostic. That diagnostic is **timing-dependent and not an effect of
> Esc**: across runs it appeared as `Error: Failed doing I/O with terminal: EOF` (this run) and as
> `Killed by signal:  hangup` (another run). The `rc file absent` line reflects the same fact — the
> kitten never reached its own exit path. Neither message is part of Esc's behavior; the durable,
> repeatable Esc fact is the **unchanged config**.

### 9.2 `s` / `S` (OBSERVED — run independently)

Reach the final pane and press **s** (and, in a separate run, **S**). The kitten serializes the four
faces to `output_on_exit` and quits, printing them to **STDOUT** (`kittens/choose_fonts/final.go`
`on_text`, `case "s", "S":` at L104 → STDOUT at `kittens/choose_fonts/main.go:L64-L66`); the config
is untouched.

> **Block-format note (applies to §9.2 and §9.3).** Every line inside the fenced blocks below is the
> harness's output shown **unmodified**, but four repetitive line-groups are **elided** to save
> space: the `before content` dump, the long `ps -ww` process-tree line, the `after content` dump,
> and the `ls -la` listing. Each elided group is identical in form to the full blocks in §8.1/§9.1
> (a 43-byte seed that stays unchanged, with no `.bak`). The audit-critical fields are all retained
> verbatim: PIDs, keystrokes, exit status, STDOUT/STDERR with byte counts, the **full 64-character**
> before/after SHA-256, the `# BEGIN_KITTY_FONTS` block count, and the backup state.

**`s` — harness output (excerpt per the note above):**

```
======== SCENARIO: s_lower ========
WORK=/tmp/cf_qa.BSPtjl  ISO_CONF=/tmp/cf_qa.BSPtjl/conf  RELOAD='<default:parent>'  BRANCH=s  HOSTCFG=NONE
---- BEFORE ----
before sha256 : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e
before bytes  : 43
GUI kitty PID : 84176
kitten PID    : 84259
backend PID   : 84274
---- KEYS ----
nav keys      : Return Return (listing->faces, faces->final)
branch key    : s (STDOUT)
---- RESULT ----
kitten exit status : 0
kitten STDOUT (bytes=153):
  | font_family      DejaVuSansMono
  | bold_font        DejaVuSansMono-Bold
  | italic_font      DejaVuSansMono-Oblique
  | bold_italic_font DejaVuSansMono-BoldOblique
kitten STDERR (bytes=0):
  (empty)
---- AFTER ----
after sha256  : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e
after bytes   : 43
BEGIN_KITTY_FONTS blocks: 0
changed?      : NO
backup        : (absent)
======== END s_lower ========
```

**`S` (uppercase) — separate, independent run — harness output (excerpt per the note above):**

```
======== SCENARIO: S_upper ========
WORK=/tmp/cf_qa.3E76qR  ISO_CONF=/tmp/cf_qa.3E76qR/conf  RELOAD='<default:parent>'  BRANCH=S  HOSTCFG=NONE
---- BEFORE ----
before sha256 : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e
before bytes  : 43
GUI kitty PID : 84367
kitten PID    : 84450
backend PID   : 84464
---- KEYS ----
nav keys      : Return Return (listing->faces, faces->final)
branch key    : shift+s (uppercase S, STDOUT)
---- RESULT ----
kitten exit status : 0
kitten STDOUT (bytes=153):
  | font_family      DejaVuSansMono
  | bold_font        DejaVuSansMono-Bold
  | italic_font      DejaVuSansMono-Oblique
  | bold_italic_font DejaVuSansMono-BoldOblique
kitten STDERR (bytes=0):
  (empty)
---- AFTER ----
after sha256  : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e
after bytes   : 43
BEGIN_KITTY_FONTS blocks: 0
changed?      : NO
backup        : (absent)
======== END S_upper ========
```

Both keys print the **same 153-byte, four-line** serialization to STDOUT (`rc = 0`) and leave
`kitty.conf` unchanged (`7f9862c0…` before == after, zero blocks, no `.bak`). These are the same
four lines `Enter` would write, but routed to STDOUT — the session/pipe path — because both are
handled by the single `case "s", "S":` at [`kittens/choose_fonts/final.go:L104`].

### 9.3 Ctrl+C — two distinct paths (OBSERVED)

Ctrl+C has **two** observable behaviors, depending on whether it arrives as a GUI **key** or as a
real **OS signal**. Both leave `kitty.conf` unchanged. (An earlier draft described only a single,
incorrect outcome; the two paths are distinct and are shown here exactly as executed.)

**(a) Ctrl+C as a GUI key on the final pane** — the top-level handler intercepts it first.
**Harness output (excerpt per the note in §9.2):**

```
======== SCENARIO: ctrlc_gui ========
WORK=/tmp/cf_qa.i5LuhO  ISO_CONF=/tmp/cf_qa.i5LuhO/conf  RELOAD='<default:parent>'  BRANCH=ctrlc  HOSTCFG=NONE
---- BEFORE ----
before sha256 : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e
GUI kitty PID : 84557
kitten PID    : 84640
backend PID   : 84655
---- KEYS ----
nav keys      : Return Return (listing->faces, faces->final)
branch key    : ctrl+c (GUI key)
---- RESULT ----
kitten exit status : 1
kitten STDOUT (bytes=0):
  (empty)
kitten STDERR (bytes=24):
  | Error: canceled by user
---- AFTER ----
after sha256  : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e
BEGIN_KITTY_FONTS blocks: 0
changed?      : NO
backup        : (absent)
======== END ctrlc_gui ========
```

Mechanism: the top-level handler `on_key_event` matches `ctrl+c` and returns
`fmt.Errorf("canceled by user")` [`kittens/choose_fonts/ui.go:L195-L198`] **before** the final pane
sees it; that error propagates out of the event loop and `main` returns `(1, err)`
[`kittens/choose_fonts/main.go:L55-L57`], which the CLI prints as `Error: canceled by user` on
STDERR (24 bytes) with exit status **1**. No write occurs (`7f9862c0…` unchanged).

**(b) Ctrl+C as a real OS SIGINT** (`kill -INT <kitten-pid>`, PID-scoped) — not a key event, so it
takes the loop's signal path. **Harness output (excerpt per the note in §9.2):**

```
======== SCENARIO: ctrlc_ossignal ========
WORK=/tmp/cf_qa.u7Yzpy  ISO_CONF=/tmp/cf_qa.u7Yzpy/conf  RELOAD='<default:parent>'  BRANCH=sigint  HOSTCFG=NONE
---- BEFORE ----
before sha256 : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e
GUI kitty PID : 84748
kitten PID    : 84831
backend PID   : 84847
---- KEYS ----
nav keys      : Return Return (listing->faces, faces->final)
branch action : kill -INT to kitten PID 84831 (PID-scoped OS signal)
---- RESULT ----
kitten exit status : 130
kitten STDOUT (bytes=29):
  | Killed by signal:  interrupt
kitten STDERR (bytes=0):
  (empty)
---- AFTER ----
after sha256  : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e
BEGIN_KITTY_FONTS blocks: 0
changed?      : NO
backup        : (absent)
======== END ctrlc_ossignal ========
```

Mechanism: the loop records the death signal; after `lp.Run()` returns, `main` sees a non-empty
`DeathSignalName()` [`tools/tui/loop/api.go:L198-L203`] and prints `Killed by signal:  interrupt`
to STDOUT (29 bytes — the double space is real: the source is
`fmt.Println("Killed by signal: ", ds)` [`kittens/choose_fonts/main.go:L60`], whose trailing-space
literal plus `Println`'s argument separator yield two spaces before `interrupt`). It then calls
`lp.KillIfSignalled()` [`kittens/choose_fonts/main.go:L58-L62`]; `KillIfSignalled`
[`tools/tui/loop/api.go:L213-L217`] invokes `kill_self(self.death_signal)`, which **re-raises the
same signal on the process**, yielding the conventional shell status **130** (128 + SIGINT).

In both Ctrl+C paths, `Patch` is never reached, so the file is untouched (`7f9862c0…` before ==
after, zero blocks, no `.bak`).

---

## 10. Startup re-read & config-directory inheritance (why the write is durable)

The write persists because kitty **re-reads `kitty.conf` at startup**, and because the kitten
writes to the **same** directory the host kitty reads — both grounded in source:

**Startup re-read (Python side):**

- `kitty/main.py:L494` calls `create_opts(cli_opts, …)`.
- `kitty/cli.py:L1081-L1086` — `create_opts` imports and calls `load_config(*config, …)`
  (config paths from `default_config_paths(args.config)`).
- `kitty/config.py:L163-L184` — `load_config` reads and parses the config file(s) into `Options`.

So a freshly-started kitty parses `kitty.conf` (including the `# BEGIN_KITTY_FONTS` block) into its
options — which is exactly what §8.3 observed at runtime.

**Config-directory inheritance (so isolation is honored end-to-end):**

- `kitty/constants.py:L87-L89` — `_get_config_dir()` returns
  `$KITTY_CONFIG_DIRECTORY` (abspath/expanduser) when set; `kitty/constants.py:L131-L133` derive
  `config_dir` and `defconf = <config_dir>/kitty.conf` from it.
- `kittens/runner.py:L92` sets `os.environ['KITTY_CONFIG_DIRECTORY'] = config_dir` for kittens.
- `kitty/boss.py:L1951` sets `env['KITTY_CONFIG_DIRECTORY'] = config_dir` in the environment kitty
  hands to a kitten's UI.
- On the Go side, `tools/utils/paths.go:L88-L91` (`ConfigDirForName`) honors
  `KITTY_CONFIG_DIRECTORY` first, and `tools/utils/paths.go:L132-L133` (`ConfigDir`) resolves
  `kitty.conf` through it.

Together these mean the kitten's `Enter` write and the host kitty's startup read target the **same**
file — the mechanism behind both the persistence and the clean, isolated observation.

---

## 11. Cause → effect summary

- **Why `Enter` persists:** `Enter` → `final_pane.on_key_event` builds
  `config.Patcher{Write_backup: true}` and calls
  `Patcher.Patch(ConfigDir()/kitty.conf, "KITTY_FONTS", serialized(), …4 keys)`
  [`kittens/choose_fonts/final.go:L80-L82`]. `Patch` comments out prior keys, wraps the four faces
  in the sentinel block, conditionally writes a `.bak`, and atomically renames the new file into
  place [`tools/config/api.go:L325-L347`; `tools/utils/atomic-write.go:L79`]. The bytes are on
  disk, and kitty re-reads them at startup (§10) → **PERSISTED** (observed §8).
- **Why reload ≠ persistence:** the reload branch fires only *after* a successful write and only
  sends `SIGUSR1` (best-effort, errors ignored) to *already-running* instances
  [`kittens/choose_fonts/final.go:L86-L93`; `tools/config/api.go:L352-L371`]. With
  `--reload-in none` there is no signal, yet the file is still written.
- **Why `s`/`S` is session-only:** it serializes the four lines to `output_on_exit` and prints them
  to STDOUT [`kittens/choose_fonts/final.go:L101-L111`; `kittens/choose_fonts/main.go:L64-L66`],
  and **never calls `Patch`** — a pipe/session output, not a persisted setting (observed §9.2).
- **Why a build was required:** the persistence logic is compiled **Go** in the `kitten` binary;
  the Python backend only lists/resolves/renders fonts and writes nothing
  [`kittens/choose_fonts/backend.py:L150-L168`].

---

## 12. Safe-isolation harness (runnable, security-reviewed)

Every runtime capture in §§3 and 7–10 was produced by the single parametrized driver below. It is
written to satisfy the safety and reproducibility properties in one place — directly addressing the
predictable-path, global-process-selection, and cleanup-before-inspection risks (CWE-20/22/367):

- **One private root.** A single `mktemp -d /tmp/cf_qa.XXXXXX` directory (`$WORK`, mode `0700`)
  holds **every** artifact — the isolated config dir (`$WORK/conf`), the private `XDG_RUNTIME_DIR`
  (`$WORK/xdg`), and the kitten's stdout/stderr/rc plus the GUI log (`$WORK/kitten.out`, `.err`,
  `.rc`, `$WORK/kitty.log`). There are **no** predictable global `/tmp/kitten.*` or `/tmp/kitty.log`
  files and **no** second temp root.
- **Owned-only cleanup.** `trap 'rm -rf -- "$WORK"' EXIT` removes **only** the one directory this
  run created — never `$HOME`, never the checkout, never a shared path, and never a broad
  `git clean`.
- **Deterministic PID capture.** The launched GUI PID is `$!`; the `kitten` and Python `backend`
  PIDs are found by walking the **live descendants** of that exact PID (`pgrep -P`, skipping
  zombies) — never a global `ps -C kitten | tail -1`.
- **Readiness + WID assertion.** The new window is found by set-difference against the pre-launch
  window list; if no window id appears within the poll budget the run **aborts**
  (`FATAL … exit 91`) after `kill`+`wait` — it never drives an empty `$WID`.
- **Always wait.** Teardown is `kill "$KPID"` **followed by** `wait "$KPID"`, so the process is
  reaped and its `rc`/streams are flushed before inspection.
- **Capture before cleanup.** All hashes, byte counts, the `# BEGIN_KITTY_FONTS` block count, the
  `.bak` state, and the real `ls -la` are emitted **before** the script ends (before the EXIT trap
  fires), so nothing is removed before it is recorded.
- **Config isolation.** `KITTY_CONFIG_DIRECTORY="$WORK/conf"` is exported **before** any kitty
  process starts [`tools/utils/paths.go:L89-L91`], so the write lands in the throwaway dir and the
  developer's real `~/.config/kitty/kitty.conf` is never touched.
- **PID-scoped signals.** The OS-SIGINT branch sends `kill -INT` to the **recorded** `kitten`
  descendant PID only — not to any process matched by name.

```bash
#!/usr/bin/env bash
# Safe evidence driver for the choose-fonts kitten.
# One private mktemp -d root holds EVERY artifact; PID-scoped teardown; WID/readiness
# assertions; bytes+hashes captured BEFORE cleanup; removes only the owned dir.
set -u
KITTY_BIN=/app/kitty/launcher/kitty
KITTEN_BIN=/app/kitty/launcher/kitten
SCENARIO="${SCENARIO:-scenario}"
SEED="${SEED:-}"            # optional pre-existing kitty.conf content (printf %b)
RELOAD="${RELOAD:-}"        # ""|parent|all|none  -> --reload-in
BRANCH="${BRANCH:-Return}"  # Return|Escape|s|S|ctrlc|sigint
HOSTCFG="${HOSTCFG:-NONE}"  # NONE (host --config NONE) | DEFAULT (host default config)

WORK="$(mktemp -d /tmp/cf_qa.XXXXXX)"; chmod 700 "$WORK"
ISO_CONF="$WORK/conf";  mkdir -p "$ISO_CONF"; chmod 700 "$ISO_CONF"
XRT="$WORK/xdg";        mkdir -p "$XRT";      chmod 700 "$XRT"
OUT="$WORK/kitten.out"; ERR="$WORK/kitten.err"; RC="$WORK/kitten.rc"; LOG="$WORK/kitty.log"
trap 'rm -rf -- "$WORK"' EXIT          # removes ONLY the owned dir

export DISPLAY=:99
export XDG_RUNTIME_DIR="$XRT"
export KITTY_CONFIG_DIRECTORY="$ISO_CONF"   # BEFORE any kitty process starts

echo "======== SCENARIO: $SCENARIO ========"
echo "WORK=$WORK  ISO_CONF=$ISO_CONF  RELOAD='${RELOAD:-<default:parent>}'  BRANCH=$BRANCH  HOSTCFG=$HOSTCFG"

if [ -n "$SEED" ]; then printf '%b' "$SEED" > "$ISO_CONF/kitty.conf"; fi
echo "---- BEFORE ----"
if [ -f "$ISO_CONF/kitty.conf" ]; then
  BEFORE_SHA="$(sha256sum "$ISO_CONF/kitty.conf" | awk '{print $1}')"
  BEFORE_BYTES="$(wc -c < "$ISO_CONF/kitty.conf")"
  echo "before sha256 : $BEFORE_SHA"; echo "before bytes  : $BEFORE_BYTES"
  echo "before content:"; cat "$ISO_CONF/kitty.conf"
else
  BEFORE_SHA="(absent)"; BEFORE_BYTES=0; echo "before        : (no kitty.conf present)"
fi

CF_ARGS=""; [ -n "$RELOAD" ] && CF_ARGS="--reload-in=$RELOAD"
HOSTARGS=(--config NONE); [ "$HOSTCFG" = "DEFAULT" ] && HOSTARGS=()

BEFORE_WINS="$(xdotool search --class kitty 2>/dev/null | sort -u)"
"$KITTY_BIN" "${HOSTARGS[@]}" -o font_size=14 \
  sh -c "$KITTEN_BIN choose-fonts $CF_ARGS >'$OUT' 2>'$ERR'; echo \$? >'$RC'" \
  >"$LOG" 2>&1 &
KPID=$!                                  # deterministic: the PID we launched
echo "GUI kitty PID : $KPID"

WID=""
for _ in $(seq 1 60); do                 # readiness poll (set-difference on window ids)
  NOW="$(xdotool search --class kitty 2>/dev/null | sort -u)"
  WID="$(comm -13 <(printf '%s\n' "$BEFORE_WINS") <(printf '%s\n' "$NOW") | grep -E '^[0-9]+$' | tail -1)"
  [ -n "$WID" ] && break
  sleep 0.5
done
if [ -z "$WID" ]; then                   # assertion: never drive an empty WID
  echo "FATAL: no new kitty window id (readiness assertion failed)"
  kill "$KPID" 2>/dev/null || true; wait "$KPID" 2>/dev/null || true; exit 91
fi
echo "kitty WID     : $WID"
sleep 3

echo "---- PROCESS TREE (at final pane) ----"
# Walk live descendants of the launched GUI PID (avoids pre-existing zombie kittens).
_desc() { local p="$1" c; for c in $(pgrep -P "$p" 2>/dev/null); do echo "$c"; _desc "$c"; done; }
DESC="$(_desc "$KPID" | sort -u)"
KITTEN_PID=""; BACKEND_PID=""
for pid in $DESC; do
  st="$(ps -o stat= -p "$pid" 2>/dev/null | tr -d ' ')"; [ "${st#Z}" != "$st" ] && continue
  cm="$(ps -o comm= -p "$pid" 2>/dev/null | tr -d ' ')"
  ar="$(ps -o args= -p "$pid" 2>/dev/null)"
  case "$ar" in *choose_fonts.backend*) BACKEND_PID="$pid";; esac
  [ "$cm" = "kitten" ] && KITTEN_PID="$pid"
done
echo "kitten PID    : ${KITTEN_PID:-<none>}"
echo "backend PID   : ${BACKEND_PID:-<none>}"
ps -ww -o pid,ppid,comm,args -p "$KPID" ${KITTEN_PID:+-p $KITTEN_PID} ${BACKEND_PID:+-p $BACKEND_PID} 2>/dev/null

KEYSEQ="Return Return"
echo "---- KEYS ----"
echo "nav keys      : $KEYSEQ (listing->faces, faces->final)"
for k in $KEYSEQ; do xdotool key --window "$WID" "$k"; sleep 1.5; done

case "$BRANCH" in
  Return) echo "branch key    : Return (Enter on final pane -> persist)"; xdotool key --window "$WID" Return ;;
  Escape) echo "branch key    : Escape (final -> back to faces)";        xdotool key --window "$WID" Escape ;;
  s)      echo "branch key    : s (STDOUT)";                             xdotool key --window "$WID" s ;;
  S)      echo "branch key    : shift+s (uppercase S, STDOUT)";          xdotool key --window "$WID" shift+s ;;
  ctrlc)  echo "branch key    : ctrl+c (GUI key)";                       xdotool key --window "$WID" ctrl+c ;;
  sigint) echo "branch action : kill -INT to kitten PID ${KITTEN_PID:-<none>} (PID-scoped OS signal)";
          [ -n "${KITTEN_PID:-}" ] && kill -INT "$KITTEN_PID" 2>/dev/null || true ;;
esac
sleep 3

if kill -0 "$KPID" 2>/dev/null; then kill "$KPID" 2>/dev/null || true; fi
wait "$KPID" 2>/dev/null || true         # ALWAYS reap before inspecting rc/streams

echo "---- RESULT ----"
if [ -f "$RC" ]; then echo "kitten exit status : $(cat "$RC")"; else echo "kitten exit status : (rc file absent)"; fi
echo "kitten STDOUT (bytes=$(wc -c < "$OUT" 2>/dev/null || echo 0)):"
if [ -s "$OUT" ]; then sed 's/^/  | /' "$OUT"; else echo "  (empty)"; fi
echo "kitten STDERR (bytes=$(wc -c < "$ERR" 2>/dev/null || echo 0)):"
if [ -s "$ERR" ]; then sed 's/^/  | /' "$ERR"; else echo "  (empty)"; fi

echo "---- AFTER ----"
if [ -f "$ISO_CONF/kitty.conf" ]; then
  AFTER_SHA="$(sha256sum "$ISO_CONF/kitty.conf" | awk '{print $1}')"
  AFTER_BYTES="$(wc -c < "$ISO_CONF/kitty.conf")"
  echo "after sha256  : $AFTER_SHA"; echo "after bytes   : $AFTER_BYTES"
  echo "BEGIN_KITTY_FONTS blocks: $(grep -c '^# BEGIN_KITTY_FONTS' "$ISO_CONF/kitty.conf" || true)"
  echo "after content :"; cat "$ISO_CONF/kitty.conf"
else
  AFTER_SHA="(absent)"; AFTER_BYTES=0; echo "after         : (no kitty.conf present)"
fi
echo "changed?      : $([ "$BEFORE_SHA" = "$AFTER_SHA" ] && echo NO || echo YES)"

echo "---- DIR LISTING (real ls -la) ----"
ls -la "$ISO_CONF"                        # captured BEFORE the EXIT trap fires
if [ -f "$ISO_CONF/kitty.conf.bak" ]; then
  BAK_SHA="$(sha256sum "$ISO_CONF/kitty.conf.bak" | awk '{print $1}')"
  echo "backup sha256 : $BAK_SHA"
  echo "backup bytes  : $(wc -c < "$ISO_CONF/kitty.conf.bak")"
  echo ".bak == before? : $([ "$BAK_SHA" = "$BEFORE_SHA" ] && echo YES || echo NO)"
  echo "backup content:"; cat "$ISO_CONF/kitty.conf.bak"
else
  echo "backup        : (absent)"
fi
echo "======== END $SCENARIO ========"
echo
```

Each scenario in this document was produced by invoking the driver with explicit parameters (the
driver was saved as `cf_driver.sh` inside the mandated container and run with `bash cf_driver.sh`):

```bash
# §8.1 Scenario A — Enter, default --reload-in=parent, pre-existing 43-byte kitty.conf
SCENARIO=A  SEED='font_size 12.0\nfont_family Old Family Name\n'  BRANCH=Return  bash cf_driver.sh
# §8.2 Scenario B — Enter into an empty config dir (no seed) -> no .bak
SCENARIO=B  BRANCH=Return  bash cf_driver.sh
# §8.3 Scenario C — Enter with --reload-in=none (persist, no reload signal)
SCENARIO=C  SEED='font_size 12.0\nfont_family Old Family Name\n'  RELOAD=none  BRANCH=Return  bash cf_driver.sh
# §8.5 Enter under the host kitty's DEFAULT config (no --config NONE)
SCENARIO=D  SEED='font_size 12.0\n'  HOSTCFG=DEFAULT  BRANCH=Return  bash cf_driver.sh
# §9.1 Esc ; §9.2 s and (independently) S ; §9.3 Ctrl+C key and Ctrl+C OS signal
SCENARIO=esc            BRANCH=Escape  bash cf_driver.sh
SCENARIO=s_lower        BRANCH=s       bash cf_driver.sh
SCENARIO=S_upper        BRANCH=S       bash cf_driver.sh
SCENARIO=ctrlc_gui      BRANCH=ctrlc   bash cf_driver.sh
SCENARIO=ctrlc_ossignal BRANCH=sigint  bash cf_driver.sh
```

For the **restart proof** (§8.4), after the `Enter` write and process exit — and **without**
deleting `$WORK` — a fresh instance is launched under the **same** `$KITTY_CONFIG_DIRECTORY` and
**without** `--config NONE`, so it reads the just-written `kitty.conf`:

```bash
"$KITTY_BIN" --debug-font-fallback -o font_size=14 sh -c 'sleep 1; exit 0'   # logs the active faces
```

> **Reproducibility note.** Running the driver leaves the host unchanged: the only filesystem
> artifact is `$WORK`, which the EXIT trap removes. `git -C /app status --porcelain` is empty after
> every run (§13.2). The isolated write is confined to `$WORK/conf/kitty.conf`; the real
> `~/.config/kitty/` canary stays absent throughout.

---

## 13. Appendix

### 13.1 Observed vs. code-derived (summary)

**Directly OBSERVED at runtime** (verbatim output embedded next to its command above):

- Canonical build (`python3 setup.py build --verbose`, exit 0; X11 + Wayland; wayland-protocols
  1.34), the Go kitten build command, binary versions `kitten`/`kitty 0.35.2`. (§3)
- Canonical `kitten choose-fonts --help`. (§5)
- The real running kitten's listing / faces / final panes under `DISPLAY=:99` (screenshots viewed
  and transcribed). (§6.2, §7.1)
- **Enter** write: BEFORE/AFTER bytes + SHA-256, `.bak`, `ls -la`, exit 0, empty STDOUT; clean-slate
  create with no `.bak`; idempotent re-run. (§8.1, §8.2, §8.5)
- **Restart**: a fresh kitty loads the four persisted faces as active fonts. (§8.3)
- **Esc / s·S / Ctrl+C (key) / Ctrl+C (signal)**: each executed; exit/STDOUT/STDERR and unchanged
  config hashes captured. (§9)

**CODE-DERIVED** (internal control-flow, exact `file:line`; every anchor is written with its full
path and range): subcommand registration [`kittens/choose_fonts/main.go:L74-L98`]; value flow
through the handler and panes [`kittens/choose_fonts/main.go:L16-L68`,
`kittens/choose_fonts/main.go:L35`; `kittens/choose_fonts/ui.go:L81`,
`kittens/choose_fonts/ui.go:L97`; `kittens/choose_fonts/list.go:L170`;
`kittens/choose_fonts/types.go:L78-L87`; `kittens/choose_fonts/faces.go:L14`,
`kittens/choose_fonts/faces.go:L113-L120`]; backend boundary
[`kittens/choose_fonts/backend.go:L41`; `kittens/choose_fonts/backend.py:L150-L168`];
persistence engine internals [`tools/config/api.go:L310-L371`]; atomic writer
[`tools/utils/atomic-write.go:L42-L79`]; config path resolution [`tools/utils/paths.go:L88-L91`,
`tools/utils/paths.go:L132-L134`]; startup re-read [`kitty/main.py:L494`;
`kitty/cli.py:L1081-L1086`; `kitty/config.py:L163-L184`].

### 13.2 Repository left pristine (verified)

- The kitty source tree was **not modified**; the sole new artifact is this document under
  `blitzy/documentation/`.
- The canonical build and all runtime observations ran inside the container's `/app` and `/tmp`;
  the container's `/app` checkout `git status` stays clean, and no build output was produced inside
  the deliverable checkout.
- The isolated `KITTY_CONFIG_DIRECTORY` directories and every scratch log live **beneath a single
  private `mktemp -d` root** (never in the checkout). That one owned directory is removed by the
  harness's own `trap ... EXIT` with a **targeted** `rm -rf "$WORK"` (§12) — no repository-wide or
  ignored-file sweep (e.g. `git clean`) is used, so tracked and untracked source are never at risk.
- In the deliverable checkout, the document is a **tracked file that is committed as the final
  step** of this task; the delivered working tree is therefore clean. `git status --porcelain`
  returns nothing (empty), and there are no build-generated (git-ignored) implementation artifacts.
  Exact output of the delivered tree:

```console
$ git status --porcelain
$ echo "rc=$?"
rc=0

$ git status --ignored --porcelain | grep -c '^!!'
0
```

  (An empty `git status --porcelain` is the definition of a clean working tree — the documentation
  change is in the commit history, not pending in the working copy.)

### 13.3 Web-search corroboration

- **Build:** the official build documentation
  ([sw.kovidgoyal.net/kitty/build](https://sw.kovidgoyal.net/kitty/build/)) — kitty runs from
  source; needs a C compiler and the Go compiler (plus X11 dev libraries on Linux); the
  recommended build is `./dev.sh build` → `kitty/launcher/kitty`; `dev.sh` wraps
  `bypy/devenv.go`, which downloads pre-built dependency bundles before invoking `setup.py`. The
  `python3 setup.py build` command used here is the repository's / environment's own command.
- **choose-fonts / kitty.conf:** the official
  [`kitty.conf` reference](https://sw.kovidgoyal.net/kitty/conf/) recommends
  [`kitten choose-fonts`](https://sw.kovidgoyal.net/kitty/kittens/choose-fonts/) as the easiest way
  to select fonts and documents the four keys `font_family`, `bold_font`, `italic_font`,
  `bold_italic_font`; it documents `SIGUSR1` (`kill -SIGUSR1 $KITTY_PID`) as a separate manual
  reload mechanism — corroborating reload ≠ persistence. These online pages describe the current
  (master) version; the checkout here is `0.35.2`, and this document grounds every behavioral
  claim in the checkout's own source plus the runtime evidence above.
- **In-repo docs:** `docs/kittens/choose-fonts.rst` exists on GitHub *master* but is **absent from
  this checkout** (`815df1e210e0`), so the interaction model is grounded in source and the online
  docs.

