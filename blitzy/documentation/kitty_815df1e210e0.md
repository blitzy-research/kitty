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

- **OBSERVED** — real output captured from a run and embedded verbatim next to its command:
  - the canonical build result (§3);
  - the canonical `kitten choose-fonts --help` (§5);
  - the real running kitten's **listing / faces / final** panes under `DISPLAY=:99` (§6.2, §7.1);
  - the real **`Enter`** write and its on-disk bytes, backup, exit code (§8);
  - each of the **`Esc` / `s`·`S` / `Ctrl+C`** branches, executed to completion (§9);
  - the **restart** in which a fresh kitty loads the persisted faces (§8.3).
- **CODE-DERIVED** — a claim read from source with an exact `file:line`, used only for internal
  control-flow (e.g., which struct field carries a value). No interactive screen output is
  fabricated.

**On screenshots.** The kitten's panes were captured as PNG screenshots under `DISPLAY=:99` using
ImageMagick `import`, and were **viewed** during the investigation. The single-file deliverable
rule permits only this `.md` in the repository, so the PNGs are not committed; their content is
transcribed faithfully in §6.2 and §7.1, and clearly marked as transcriptions of viewed
screenshots. The primary machine-checkable evidence — config bytes, SHA-256 hashes, STDOUT, exit
codes, and `ls -la` — is embedded verbatim.

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

The build log shows the Wayland backend being compiled and linked (excerpt):

```
Detected: CompilerType.gcc
gcc ... -D_GLFW_WAYLAND ... -o build/kitty/glfw-wayland.so ... \
    -lwayland-client -lwayland-cursor -lxkbcommon -ldbus-1
```

and the Go `kitten` binary (which contains the persistence code) being built:

```
/usr/local/go/bin/go build -v \
  -ldflags '-X kitty.VCSRevision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 -s -w' \
  -o kitty/launcher/kitten /app/tools/cmd
```

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

(equivalently `kitty +kitten choose-fonts`, once `kitty/launcher` is on `PATH`).

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
- Inside `main(_ *Options)` [`kittens/choose_fonts/main.go:L16-L68`], the parsed options are
  attached to the handler at `kittens/choose_fonts/main.go:L35`.
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
`DejaVu Sans Mono` and pressing `Enter` on the faces pane) was **OBSERVED** to display (family
name in cyan; option keys highlighted):

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

Every capture below was produced by driving the **real** `kitten choose-fonts` UI under
`DISPLAY=:99` with `xdotool`, inside the safe-isolation harness of §12. The pane sequence in each
run is: launch → (Return) faces → (Return) final → branch key.

### 8.1 Scenario A — a pre-existing `kitty.conf` (Enter → persist)

Seed a prior config, then drive the real UI and press **Enter** on the final pane. Commands and
complete, unedited output:

```bash
# (inside the harness; $ISO_CONF is the validated 0700 KITTY_CONFIG_DIRECTORY)
printf 'font_size 12.0\nfont_family Old Family Name\n' > "$ISO_CONF/kitty.conf"
sha256sum "$ISO_CONF/kitty.conf"
```

**BEFORE** (`cat "$ISO_CONF/kitty.conf"`), sha256
`7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e`:

```
font_size 12.0
font_family Old Family Name
```

Drive the UI (real keypresses) and capture the kitten's exit status and STDOUT:

```bash
# listing --Return--> faces --Return--> final --Return--> (Enter persists)
xdotool key --window "$WID" Return   # -> faces
xdotool key --window "$WID" Return   # -> final
xdotool key --window "$WID" Return   # -> Enter on final pane
```

```
kitten exit status : 0
kitten STDOUT       : (empty)
```

**AFTER** (`cat "$ISO_CONF/kitty.conf"`), sha256
`336b3aaee0ac6833d78684c47c69d9b5b47710781609d578ec53b232b25eae1e` (differs from BEFORE):

```
font_size 12.0
# font_family Old Family Name


# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTS
```

Observe, next to the code that produced each effect:

- the unrelated `font_size 12.0` is **preserved**;
- the prior `font_family Old Family Name` is **commented out** → `# font_family Old Family Name`
  (regex at [`tools/config/api.go:L325-L326`]);
- the four selected faces are written inside `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` (sentinel
  at [`tools/config/api.go:L330`]; the four keys are the arguments from
  [`kittens/choose_fonts/final.go:L82`]);
- the four written values are **exactly** the four faces shown in the UI (§6.2) — the same
  selection reached finalization unchanged.

**Backup + directory listing** (`ls -la "$ISO_CONF"` then `cat "$ISO_CONF/kitty.conf.bak"`). Since
there was prior non-empty content and `Write_backup` is set, a `.bak` is written
[`tools/config/api.go:L343-L344`]:

```
total 16
drwx------ 2 root root 4096 .. .
drwxrwxrwt 1 root root 4096 .. ..
-rw-r--r-- 1 root root  237 .. kitty.conf
-rw-r--r-- 1 root root   43 .. kitty.conf.bak
```

```
$ cat "$ISO_CONF/kitty.conf.bak"
font_size 12.0
font_family Old Family Name
```

### 8.2 Scenario B — no `kitty.conf` present (Enter → persist, no `.bak`)

Same real-UI `Enter`, but starting from an **empty** isolated directory. Complete output:

```
BEFORE: (no kitty.conf present)
kitten exit status : 0
kitten STDOUT      : (empty)
```

**AFTER** (`cat "$ISO_CONF/kitty.conf"`) — the file is *created* containing only the block:

```
# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTS
```

Directory listing shows **no `.bak`** — because the backup is guarded by "there was prior
content" (`len(raw) > 0` at [`tools/config/api.go:L343`]):

```
$ ls -la "$ISO_CONF"
total 12
drwx------ 2 root root 4096 .. .
drwxrwxrwt 1 root root 4096 .. ..
-rw-r--r-- 1 root root  190 .. kitty.conf
```

This proves persistence even from a clean slate: pressing `Enter` *creates* `kitty.conf` if absent
and writes the durable block, and no backup is produced when there was nothing to back up.

### 8.3 Restart proof — a fresh kitty loads the persisted faces (OBSERVED)

The critical persistence test: after the `Enter` write, the writing process has exited; a **fresh**
kitty process is then launched under the **same** isolated `KITTY_CONFIG_DIRECTORY` (this time
*without* `--config NONE`, so it reads the persisted config) and asked to report its resolved
fonts:

```bash
# fresh process; same $ISO_CONF as the write above
/app/kitty/launcher/kitty --debug-font-fallback -o font_size=14 \
    sh -c 'sleep 1; exit 0'
```

Complete, unedited relevant output — the restarted instance's **active** text fonts:

```
[0.194] Text fonts:
[0.194]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.194]   Bold: DejaVuSansMono-Bold: /root/.local/share/fonts/DejaVuSansMono-Bold.ttf:0
[0.194]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.194]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

The four faces the freshly-started instance loaded are **exactly** the four persisted lines, with
their resolved font files. This is runtime proof that the selection is active **after a restart**,
not merely that the file can be re-read.

### 8.4 Enter under the host kitty's DEFAULT config (canonical-configuration check)

To confirm the persistence is not an artifact of the harness launching the host kitty with
`--config NONE`, the `Enter` write was repeated with the host kitty using its **default** config
resolution (no `--config NONE`). Seeded BEFORE and captured AFTER:

```
BEFORE:
font_size 12.0

AFTER:
font_size 12.0


# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTS
```

The kitten's write path is identical with or without `--config NONE` on the host terminal: the
kitten resolves its own config directory from `KITTY_CONFIG_DIRECTORY` in the inherited environment
[`tools/utils/paths.go:L89-L91`], independent of the host kitty's `--config`. `--config NONE` is
therefore only a convenience so the host terminal does not try to load the seeded test font.

### 8.5 Idempotency (OBSERVED)

Pressing `Enter` a **second time** against the already-written directory is a no-op. Complete
evidence:

```
sha256 before 2nd Enter : 336b3aaee0ac6833d78684c47c69d9b5b47710781609d578ec53b232b25eae1e
sha256 after  2nd Enter : 336b3aaee0ac6833d78684c47c69d9b5b47710781609d578ec53b232b25eae1e
# BEGIN_KITTY_FONTS blocks in file : 1
kitten exit status : 0
```

The hashes are identical, exactly **one** `# BEGIN_KITTY_FONTS` block remains, and the `.bak` from
the first write is **not** rewritten. Mechanically, the regenerated bytes equal the existing bytes,
so the `bytes.Equal` guard at [`tools/config/api.go:L342`] short-circuits, `Patch` returns
`updated = false`, and no file (nor backup) is touched.

---

## 9. Exhaustive branch table (Enter / Esc / `s`·`S` / Ctrl+C)

Every mode the question implies was **executed to completion** in a fresh, isolated scenario, with
the trigger, the result (exit/STDOUT), and the byte-for-byte `kitty.conf` state captured. Per-branch
raw output follows the table.

| Key | Handler / anchor | Effect | `kitty.conf` before → after | Evidence |
|-----|------------------|--------|------------------------------|----------|
| **Enter** | `kittens/choose_fonts/final.go:L78-L94` → `tools/config/api.go:L310-L350` → `tools/utils/atomic-write.go:L79` | Writes sentinel block (+ conditional `.bak`), atomic replace; optional `SIGUSR1` reload | no block → **block present (PERSISTED)** | OBSERVED — §8 |
| **Esc** | `kittens/choose_fonts/final.go:L73-L76` | Sets `current_pane = &faces`; returns to faces pane; no write | unchanged → unchanged | OBSERVED — below |
| **`s` / `S`** | `kittens/choose_fonts/final.go:L101-L111` → STDOUT via `kittens/choose_fonts/main.go:L64-L66` | Serializes 4 lines to `output_on_exit`, quits, prints to **STDOUT**; never calls `Patch` | unchanged → unchanged | OBSERVED — below |
| **Ctrl+C (GUI key)** | `kittens/choose_fonts/ui.go:L195-L198` → `kittens/choose_fonts/main.go:L55-L57` | Handler intercepts the key, returns `"canceled by user"`; process exits **1**; no write | unchanged → unchanged | OBSERVED — below |
| **Ctrl+C (OS signal)** | loop signal path → `kittens/choose_fonts/main.go:L58-L62` | Prints `"Killed by signal:  interrupt"`, re-raises signal; shell status **130**; no write | unchanged → unchanged | OBSERVED — below |

**Only `Enter` mutates `kitty.conf`.** All other branches leave the file byte-for-byte unchanged
(verified by SHA-256 equality below).

### 9.1 Esc (OBSERVED)

Reach the final pane, press **Esc**. The UI returns to the **faces** pane (transcribed screenshot
matched the faces pane exactly), and the config is byte-identical:

```
sha256 before : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e
sha256 after  : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e
kitty.conf.bak : (absent)
```

### 9.2 `s` / `S` (OBSERVED)

Reach the final pane with STDOUT redirected to a file, press **s**. The kitten prints the four
serialized lines to STDOUT and quits; the config is unchanged:

```
kitten exit status : 0
kitten STDOUT:
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique

sha256 before : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e
sha256 after  : 7f9862c0e814651bafab5aec5d737669bb72574e19eeb212470e3a27902d936e
```

These are the same four lines `Enter` would write, but routed to STDOUT — the session/pipe path.
`S` is equivalent: both are handled by the same `case "s", "S":`
[`kittens/choose_fonts/final.go:L104`].

### 9.3 Ctrl+C — two distinct paths (OBSERVED)

Ctrl+C has **two** observable behaviors, depending on whether it arrives as a GUI **key** or as a
real **OS signal**. Both leave `kitty.conf` unchanged. (An earlier draft described only a single,
incorrect "exit 1 via Killed by signal" outcome; the two paths are distinct and are shown here as
executed.)

**(a) Ctrl+C as a GUI key on the final pane** — the handler intercepts it first:

```
kitten exit status : 1
kitten STDOUT      : (empty)
kitten STDERR      : Error: canceled by user
sha256 before/after: 7f9862c0…  ==  7f9862c0…  (unchanged)
```

Mechanism: the top-level handler `on_key_event` matches `ctrl+c` and returns
`fmt.Errorf("canceled by user")` [`kittens/choose_fonts/ui.go:L196-L198`] **before** the final
pane sees it; that error propagates out of the event loop and `main` returns `(1, err)`
[`kittens/choose_fonts/main.go:L55-L57`], which the CLI prints as `Error: canceled by user` with
exit status 1.

**(b) Ctrl+C as a real OS SIGINT** (`kill -INT <kitten-pid>`) — not a key event, so it takes the
loop's signal path:

```
kitten exit status : 130
kitten STDOUT      : Killed by signal:  interrupt
sha256 before/after: 7f9862c0…  ==  7f9862c0…  (unchanged)
```

Mechanism: the loop records the death signal; after `lp.Run()` returns, `main` sees a non-empty
`DeathSignalName()` [`tools/tui/loop/api.go:L198-L203`], prints `Killed by signal:  interrupt`, and
calls `lp.KillIfSignalled()` [`kittens/choose_fonts/main.go:L58-L62`]. `KillIfSignalled`
[`tools/tui/loop/api.go:L213-L217`] invokes `kill_self(self.death_signal)`, which **re-raises the
same signal on the process**, yielding the conventional shell status **130** (128 + SIGINT).

In both paths, `Patch` is never reached, so the file is untouched.

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

Every runtime capture used the harness below. It creates a **unique** temp directory with
`mktemp -d`, sets mode `0700`, validates ownership / non-symlink / expected prefix, exports
`KITTY_CONFIG_DIRECTORY` **before** any kitty process starts, and installs a **quoted** cleanup
trap — addressing predictable-path, TOCTOU, and symlink risks (CWE-20/22/367). It never touches
the developer's real config.

```bash
#!/usr/bin/env bash
set -u
KITTY_BIN=/app/kitty/launcher/kitty
KITTEN_BIN=/app/kitty/launcher/kitten

# Private runtime dir (0700) + display.
XDG_RUNTIME_DIR="$(mktemp -d /tmp/xdgrt.XXXXXX)"; chmod 700 "$XDG_RUNTIME_DIR"
export XDG_RUNTIME_DIR
export DISPLAY=:99

# Unique, validated isolated config dir (0700).
ISO_CONF="$(mktemp -d /tmp/kitty_iso.XXXXXX)"; chmod 700 "$ISO_CONF"
_validate_dir() {                       # abort unless every safety property holds
    local d="$1"
    [ -d "$d" ] && [ ! -L "$d" ] && [ -O "$d" ] || return 1   # real dir, not symlink, owned by us
    case "$d" in /tmp/*) : ;; *) return 1 ;; esac             # expected prefix
    [ "$(stat -c '%a' "$d")" = "700" ] || return 1            # mode 0700
}
_validate_dir "$ISO_CONF" || { echo "FATAL: unsafe temp dir"; exit 90; }

# Export BEFORE launching any kitty process; quoted, bounded cleanup trap.
export KITTY_CONFIG_DIRECTORY="$ISO_CONF"
trap 'rm -rf -- "$ISO_CONF" "$XDG_RUNTIME_DIR"' EXIT

# Launch one instance running the real kitten; capture the kitten's stdout/exit.
BEFORE_WINS="$(xdotool search --class kitty 2>/dev/null | sort -u)"
"$KITTY_BIN" --config NONE -o font_size=14 \
   sh -c "$KITTEN_BIN choose-fonts >/tmp/kitten.out 2>/tmp/kitten.err; echo \$? >/tmp/kitten.rc" \
   >/tmp/kitty.log 2>&1 &
KPID=$!

# Bind to the NEW window deterministically (set difference), then drive it.
WID=""
for _ in $(seq 1 40); do
  NOW="$(xdotool search --class kitty 2>/dev/null | sort -u)"
  WID="$(comm -13 <(printf '%s\n' "$BEFORE_WINS") <(printf '%s\n' "$NOW") | tail -1)"
  [ -n "$WID" ] && break; sleep 0.5
done
sleep 3
xdotool key --window "$WID" Return    # listing -> faces
xdotool key --window "$WID" Return    # faces   -> final
xdotool key --window "$WID" Return    # Enter on final -> persist (or: Escape / s / ctrl+c)

# PID teardown: stop only the process we launched.
if kill -0 "$KPID" 2>/dev/null; then kill "$KPID" 2>/dev/null || true; fi
# (trap removes $ISO_CONF and $XDG_RUNTIME_DIR on exit)
```

For the **real OS SIGINT** case (§9.3b), replace the branch key with a signal to the running
kitten process only:

```bash
SIGPID=$(ps -C kitten -o pid=,stat= | awk '$2 !~ /Z/ {print $1}' | tail -1 | tr -d ' ')
kill -INT "$SIGPID"
```

For the **restart proof** (§8.3), after the `Enter` write and process exit, launch a fresh instance
under the **same** `$ISO_CONF` **without** `--config NONE`:

```bash
"$KITTY_BIN" --debug-font-fallback -o font_size=14 sh -c 'sleep 1; exit 0'  # prints active fonts
```

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

**CODE-DERIVED** (internal control-flow, exact `file:line`): subcommand registration
[`kittens/choose_fonts/main.go:L74-L98`]; value flow through the handler and panes
[`kittens/choose_fonts/main.go:L16-L68`, `:L35`; `kittens/choose_fonts/ui.go:L81`, `:L97`;
`kittens/choose_fonts/list.go:L170`; `kittens/choose_fonts/types.go:L78-L87`;
`kittens/choose_fonts/faces.go:L14`, `:L113-L120`]; backend boundary
[`kittens/choose_fonts/backend.go:L41`; `kittens/choose_fonts/backend.py:L150-L168`];
persistence engine internals
[`tools/config/api.go:L310-L371`]; atomic writer [`tools/utils/atomic-write.go:L42-L79`]; config
path resolution [`tools/utils/paths.go:L88-L91`, `:L132-L133`]; startup re-read
[`kitty/main.py:L494`; `kitty/cli.py:L1081-L1086`; `kitty/config.py:L163-L184`].

### 13.2 Repository left pristine (verified)

- The kitty source tree was **not modified**; the sole new artifact is this document under
  `blitzy/documentation/`.
- The canonical build and all runtime observations ran inside the container's `/app` and `/tmp`;
  the container's `/app` checkout `git status` stays clean, and no build output was produced inside
  the deliverable checkout.
- The isolated `KITTY_CONFIG_DIRECTORY` directories and scratch logs live under `/tmp` (never in
  the checkout) and are removed by the harness cleanup trap.
- In the deliverable checkout, `git status --porcelain` reports only the tracked documentation
  change, and no build-generated (git-ignored) implementation artifacts remain. Any artifacts
  produced during implementation were removed with `git clean -Xdf` (ignored-only; never touches
  tracked source), and the scratch screenshot directory was deleted. Exact final output:

```console
$ git status --porcelain
 M blitzy/documentation/kitty_815df1e210e0.md

$ git status --ignored --porcelain | grep -c '^!!'
0
```

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

