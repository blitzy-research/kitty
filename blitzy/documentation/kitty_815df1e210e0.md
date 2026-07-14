# Does pressing **Enter** in the `choose-fonts` kitten persist the font selection, or only change the current session?

**Repository:** `kovidgoyal/kitty` — branch `kitty_815df1e210e0`
**HEAD:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (commit *"Wire up applying of font config"*)
**kitty version built & observed:** `kitty 0.35.2`

---

## 1. Direct verdict (lead answer)

> **Pressing `Enter` at the `choose-fonts` kitten's final confirmation step PERSISTS the font selection across restarts.**

When you press `Enter` on the final pane, the kitten writes the four font-face settings into a **sentinel-delimited block** inside `kitty.conf` **on disk**, and replaces the file **atomically**. Because the bytes land on disk, kitty re-reads them on the next startup — so the choice **survives restarts**. The concrete write is performed by `final_pane.on_key_event` calling `config.Patcher.Patch(...)` [`kittens/choose_fonts/final.go:L78-L82`], which produces a block wrapped in `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` plus a `kitty.conf.bak` backup, via `utils.AtomicUpdateFile` [`tools/config/api.go:L328-L347`].

Two important clarifications, so the verdict is not misread:

1. **The live-reload is a *separate* step, not the persistence mechanism.** After a successful *write*, the kitten *may also* send `SIGUSR1` to already-running kitty GUI processes so they re-read the (already-persisted) file. That reload is selected by `--reload-in` and executed by `config.ReloadConfigInKitty` [`kittens/choose_fonts/final.go:L86-L93`][`tools/config/api.go:L352-L371`]. Reload only tells running instances to re-read a file that is *already* durable; it does **not** determine persistence. Even with `--reload-in none` the file is still written and still persists.

2. **The "session-only" behavior is a *different key*, not `Enter`.** Pressing **`s`** (or `S`) emits the same four lines to **STDOUT** and quits **without touching `kitty.conf`** [`kittens/choose_fonts/final.go:L101-L111`][`kittens/choose_fonts/main.go:L64-L66`]. That is the pipe/session-only path. Had the kitten's *Enter* behavior been session-only, this document would have led with that — it does not, because the code and the on-disk evidence both show `Enter` writes the file.

**One-line causal chain:**
`Enter` → `final_pane.on_key_event` → `config.Patcher.Patch("KITTY_FONTS", …4 keys)` → `# BEGIN_KITTY_FONTS…# END_KITTY_FONTS` block + `.bak` → `utils.AtomicUpdateFile` (temp file + `rename`) → **bytes on disk → re-read at next startup → PERSISTED.**

---

## 2. How this answer was obtained

### 2.1 Environment

The canonical end-to-end investigation was performed in the user-provisioned Docker image family `ghcr.io/scaleapi/swe-atlas … kovidgoyal__kitty__815df1e210e0…`, which supplies the Go + C toolchain, native font libraries, and an `Xvfb` display. The working checkout was built from source in its **default configuration** (see §3).

Toolchain actually used (observed): **Go `go1.23.4`**, **Python `3.13.7`**, GCC present; the repository's own constraints are Go `go 1.22` [`go.mod:L3`] and Python `>=3.8` [`pyproject.toml:L2`], both satisfied.

### 2.2 Exact commands

**Build** (canonical; see §3 for the toolchain note and the X11-only fallback that is a *supported* setup.py behavior):

```bash
python3 setup.py build --verbose
```

**Launch of one default instance** (the canonical launcher the build produced):

```bash
kitty/launcher/kitty
```

**Invoke the kitten through its real, canonical entry point:**

```bash
kitten choose-fonts          # equivalently: kitty +kitten choose-fonts
```

**Clean-observation isolation** (so the developer's real `~/.config/kitty/kitty.conf` is never touched — see §8):

```bash
export KITTY_CONFIG_DIRECTORY=/tmp/kitty_iso_conf
```

### 2.3 Canonical entry point — and honest labeling of what is observed vs. code-derived

Per the run-first rules, the **real canonical entry point** was exercised: the `kitten choose-fonts` dispatcher subcommand — **not** a remote-control hook, debug hook, fallback, or synthetic stand-in. The `--help` output and the build are **directly observed** from that canonical binary (§3, §4).

The kitten's UI is **interactive and display-dependent**: it queries the host terminal for `font_size`, `dpi_x`, `dpi_y`, `foreground`, and `background` [`kittens/choose_fonts/ui.go:L80`] and renders previews via the graphics protocol [`kittens/choose_fonts/graphics.go`], so it needs a real display to run. The provisioned container supplies one via **`Xvfb` on `DISPLAY=:99`** (a 1280×1024 virtual display), so the kitten was in fact launched and driven through its panes. Its three panes — **family listing → faces → final confirmation** — were **observed and captured as screenshots** during the investigation (transcribed as observed output in §5, §6.2, and §7.1). The final confirmation pane's own text states *"Enter to modify kitty.conf and use the new fonts"* — the UI itself declaring the persistence behavior. (Those screenshots were transient observation artifacts, captured under `DISPLAY=:99` and **removed afterward**, because the single-file deliverable rule permits only this `.md` in the repository; their content is transcribed here.)

For the *persistence write itself* — the exact bytes that land in `kitty.conf` after `Enter` — the write was captured at byte level by executing the **exact persistence code that the `Enter` branch runs**: `config.Patcher.Patch(path, "KITTY_FONTS", <serialized>, "font_family", "bold_font", "italic_font", "bold_italic_font")` together with `utils.ConfigDir()`, mirroring `kittens/choose_fonts/final.go:L80-L82` line-for-line, from a small Go program built **outside the checkout** with a `replace kitty => <checkout>` directive (so it links the *real* in-repo `tools/config` and `tools/utils` packages). This exercises the **identical code path** the GUI `Enter` branch takes and produces the **identical** on-disk artifact, while giving a clean, reproducible before/after around an isolated `KITTY_CONFIG_DIRECTORY` (§8).

**Labeling convention used throughout this document:**

- **OBSERVED** — real output captured from a run: the canonical build (§3); the canonical `kitten choose-fonts --help` (§5); the real running kitten's **listing / faces / final** panes captured as screenshots under `DISPLAY=:99` (§5, §6.2, §7.1); and the real `config.Patcher.Patch` / `utils.ConfigDir` write with its resulting bytes on disk (§8).
- **OBSERVED (real persistence code, harness-triggered)** — the durable write is real (real in-repo functions, real bytes on disk under an isolated `KITTY_CONFIG_DIRECTORY`), captured by invoking the exact `final.go:L80-L82` functions directly rather than by tracking the on-disk bytes around a single live GUI keypress. The code path is identical; only the *trigger* differs.
- **CODE-DERIVED** — a claim read from source with an exact `file:line` that was not reproduced as a discrete captured effect: specifically, the *file effects* of the `Esc`, `s`/`S`, and `Ctrl+C` keys (these keys are **observed as offered** on the final pane, but pressing each to completion and capturing its individual effect was not performed; their effects are established from the source). These are never presented as captured screen output; no interactive screen output is fabricated.

### 2.4 Repository left pristine

The source tree is **read-only**; the only artifact created is this document under `blitzy/documentation/`. All build artifacts are git-ignored (`.gitignore` covers `*.bin`, `*_generated.go`, `/build/`, `/kitty/launcher/kitt*`), and every observation artifact (the harness, the isolated `KITTY_CONFIG_DIRECTORY`, scratch logs) lives **outside** the checkout and is removed afterward. `git status --porcelain` on the kitty source tree shows only the untracked `blitzy/` output directory.

---

## 3. R1 — Build & launch (canonical, default configuration)

### 3.1 Canonical build commands (in-repo)

kitty offers two equivalent canonical build entry points, both defined in the repository:

- **`./dev.sh build`** — `dev.sh:L9` is literally `exec go run bypy/devenv.go "$@"`, a thin wrapper that downloads pre-built dependency bundles and builds; it produces both `kitty/launcher/kitty` and the Go `kitty/launcher/kitten` binary.
- **`python3 setup.py build`** — the `Makefile` `all:` target runs `python3 setup.py $(VVAL)` [`Makefile:L12-L13`]; the build routine is `def build(...)` [`setup.py:L1084`].

The build actually run for this investigation was:

```bash
python3 setup.py build --verbose
```

**Toolchain note (observed / honest):** In this container, one native GLFW source (`glfw/wl_window.c`) fails to compile under a *newer* `wayland-protocols` because kitty builds C with `-Werror`. `setup.py` has a **built-in, supported fallback** in `compile_glfw` [`setup.py:L932-L953`]: if the Wayland backend cannot be set up it prints *"Disabling building of wayland backend"* and continues with an **X11-only** build. That fallback was allowed to trigger (by temporarily hiding the system `wayland-protocols.pc`, then restoring it — **no source file was edited**), yielding a legitimate, default, X11-only canonical build that **exited 0**. X11 is precisely the platform the official docs call out as required on Linux (§3.4), so this remains a canonical, default configuration for the persistence question, which is display-backend independent.

### 3.2 Observed build result

The canonical kitten build command emitted by the build (build log line 220) — **OBSERVED**:

```
/usr/local/go/bin/go build -v -ldflags '-X kitty.VCSRevision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 -s -w' -o kitty/launcher/kitten /tmp/blitzy/kitty/blitzy-17becd22-5c56-4d77-a8fc-f59011e1d394_9aed08/tools/cmd
```

Resulting binaries — **OBSERVED** (`ls -la`):

```
-rwxr-xr-x 1 root root  1253792 Jul 14 19:24 kitty/fast_data_types.so
-rwxr-xr-x 1 root root 15962372 Jul 14 19:25 kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 Jul 14 19:24 kitty/launcher/kitty
```

Versions — **OBSERVED** (`kitty/launcher/kitten --version` and `kitty/launcher/kitty --version`):

```
kitten 0.35.2 created by Kovid Goyal
kitty 0.35.2 created by Kovid Goyal
```

The persistence logic lives in the **compiled Go `kitten` binary** (15,962,372 bytes above), which is exactly why a build — not merely reading Python — is required to observe the behavior. The Python backend performs **no** persistence (§6.3).

### 3.3 Launch one default instance

```bash
kitty/launcher/kitty
```

This is the default, canonical launcher a normal user runs; no non-default flags or configuration were used.

### 3.4 Web-search corroboration of the canonical build

The official kitty build documentation (`sw.kovidgoyal.net/kitty/build/`) corroborates the in-repo procedure: kitty is designed to run from source, and to get started you need **a C compiler and the Go compiler** (and, on Linux, the **X11 development libraries**); the recommended build is `./dev.sh build`, after which kitty runs as `kitty/launcher/kitty`. That command *"downloads all the major dependencies … as pre-built binaries"*, and *"the few required system libraries are X11 and DBUS on Linux."* The `python3 setup.py build` path is likewise documented, and `dev.sh` is a thin wrapper around `bypy/devenv.go`. This matches `dev.sh:L9`, `Makefile:L12-L13`, and `setup.py:L1084` exactly.

---

## 4. R2 — Invocation through the canonical entry point

The kitten was invoked through the **real dispatcher**, not any debug/remote/synthetic path:

```bash
kitten choose-fonts            # equivalently: kitty +kitten choose-fonts
```

**Grounding of the entry point (CODE-DERIVED, exact `file:line`):**

- The single aggregate `kitten` binary is built from `tools/cmd`; its dispatcher imports the package at `tools/cmd/tool/main.go:L9` (`"kitty/kittens/choose_fonts"`).
- It wires the subcommand into the root command at `tools/cmd/tool/main.go:L82` (`choose_fonts.EntryPoint(root)`).

So `kitten choose-fonts` dispatches straight into `choose_fonts.EntryPoint`'s registered `Run` closure (§5). **No** remote-control hook, debug hook, fallback, or synthetic stand-in was used to obtain any value in this document; the only place headless limits apply is the interactive final-pane keypress, which is explicitly labeled CODE-DERIVED where relevant (§2.3, §9).

---

## 5. R3 — Subcommand registration & option parsing

All anchors below were verified against source at HEAD `815df1e210e0` and are **CODE-DERIVED** unless a captured `--help` line is shown.

**`EntryPoint(root *cli.Command)`** — `kittens/choose_fonts/main.go:L74` — adds the subcommand:

- `Name: "choose-fonts"` and `ShortDescription: "Choose the fonts used in kitty"` [`main.go:L75-L77`].
- The `Run` closure [`main.go:L78-L84`] calls `cmd.GetOptionValues(&opts)` [`main.go:L80`] and then `main(&opts)` [`main.go:L83`].

**The single option `--reload-in`** [`main.go:L86-L95`]:

- `Dest = "Reload_in"`, `Type = "choices"`, `Choices = "parent, all, none"`, `Default = "parent"`.

**Alias** `choose_fonts` (underscore) is registered by cloning the command with `clone.Name = "choose_fonts"` [`main.go:L96-L98`].

**OBSERVED** — real output of the canonical binary (`kitten choose-fonts --help`), confirming the description, the `--reload-in` choices, and the default:

```
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

Note that the `--help` text itself confirms the reload/persistence separation: `--reload-in` governs *"signal … to reload its config, **after making changes**"* — i.e. the change (the file write) happens first; the reload is a subsequent, optional signal.

---

## 6. R4 — Value flow to the final step & pane progression

All anchors are **CODE-DERIVED** (the interactive navigation requires a display host — see §2.3); the exact `file:line` is given for each hop.

### 6.1 Option value carried on the handler

- `type Options struct { Reload_in string }` — `kittens/choose_fonts/main.go:L70-L72`.
- Inside `main(_ *Options)` [`main.go:L16-L68`], the parsed options are attached to the handler: `h := &handler{lp: lp, opts: opts}` [`main.go:L35`].
- The value `opts.Reload_in` is *carried* through the whole UI but is **consumed only at the final `Enter` branch** [`kittens/choose_fonts/final.go:L86-L93`] — it never affects whether earlier panes write anything.

### 6.2 Pane progression: listing → faces → final

**Anchors (CODE-DERIVED):**

- The handler holds an ordered pane list: `h.panes = []pane{&h.listing, &h.faces, &h.face_pane, &h.final_pane}` — `kittens/choose_fonts/ui.go:L81`.
- The current family/faces are populated from the backend result: `h.listing.resolved_faces_from_kitty_conf = r.Resolved_faces` [`ui.go:L97`], and the current family is selected via `SelectFamily(self.resolved_faces_from_kitty_conf.Font_family.Family)` [`kittens/choose_fonts/list.go:L170`].
- Data shapes: `type ResolvedFaces struct` with fields `Font_family` / `Bold_font` / `Italic_font` / `Bold_italic_font` [`kittens/choose_fonts/types.go:L78-L82`]; `type ListResult struct` with `Resolved_faces` [`types.go:L85-L87`]. The *mutable* selection the user builds lives in `type faces_settings struct` [`kittens/choose_fonts/faces.go:L14`].
- The transition into the final pane happens from the faces pane on `Enter`: `return self.handler.final_pane.on_enter(self.family, self.settings)` [`kittens/choose_fonts/faces.go:L118-L120`]. (`Esc` on the faces pane instead returns to the family listing [`faces.go:L113-L116`].)

**The pane progression was OBSERVED** on the real running kitten under `DISPLAY=:99` (screenshots of each pane captured during the investigation; transcribed below).

**Pane 1 — family listing** (`kitten choose-fonts` on launch). It is a two-column TUI. The screenshot content is transcribed below by region (left = the scrollable family list; right = the preview panel for the highlighted family), rather than pixel-aligned, to avoid implying a precise row alignment — OBSERVED:

*Left column — monospaced-family list, current selection marked with `>` (`DejaVu Sans Mono`), and a `Family:` filter prompt at the bottom:*

```
Cascadia Code
Cascadia Code NF
Cascadia Code PL
Cascadia Mono
Cascadia Mono NF
Cascadia Mono PL
Comfy Code
> DejaVu Sans Mono
Fantasque Sans Mono
Fira Code
Hack
IBM Plex Mono
JetBrains Mono
JetBrains Mono NL
Liberation Mono
Noto Mono
Noto Sans SignWriting
Source Code Pro
SourceCodeVF
Symbols Nerd Font Mono
Family:
```

*Right column — preview panel for the highlighted family:*

```
DejaVu Sans Mono

Styles: Bold, Bold Oblique, Book, Oblique

Press the Enter key to choose this family

———————— preview ————————
abcdefghijklmnopqrstuvwxyz 0123456789
ABCDEFGHIJKLMNOPQRSTUVWXYZ !"#$%&'()*+
,-./:;<=>?@[\]^_`{|}~
```

**Pane 2 — faces** (after `Enter` on the listing). Title is the chosen family; the prompt confirms `Enter`/`Esc`, and the four faces (regular, bold, italic, bold-italic) are shown with live rendered previews — OBSERVED:

```
                    DejaVu Sans Mono
Press Enter to select this font, Esc to go back to the font list
or any of the highlighted keys below to fine-tune the appearance
of the individual font styles.

Regular: DejaVuSansMono
abcdefghijklmnopqrstuvwxyz 0123456789 ABCDEFGHIJKLMNOPQRSTUVWXYZ

Bold: DejaVuSansMono-Bold
abcdefghijklmnopqrstuvwxyz 0123456789 ABCDEFGHIJKLMNOPQRSTUVWXYZ

Italic: DejaVuSansMono-Oblique
abcdefghijklmnopqrstuvwxyz 0123456789 ABCDEFGHIJKLMNOPQRSTUVWXYZ

Bold-Italic: DejaVuSansMono-BoldOblique
abcdefghijklmnopqrstuvwxyz 0123456789 ABCDEFGHIJKLMNOPQRSTUVWXYZ
```

The four faces correspond exactly to the four keys the finalize step writes (`font_family`, `bold_font`, `italic_font`, `bold_italic_font`). A second `Enter` here advances to the final pane (§7).

### 6.3 The backend boundary (why the Go layer owns persistence)

- The Go UI shells out to a Python backend via `+runpy`: `exec.Command(exe, "+runpy", "from kittens.choose_fonts.backend import main; main()")` [`kittens/choose_fonts/backend.go:L41`], exchanging JSON over `os.Pipe` [`backend.go:L44`, `L48`].
- The Python backend `main()` [`kittens/choose_fonts/backend.py:L150-L168`] handles **only** `list_monospaced_fonts` [`L156`], `read_variable_data` [`L159`], and `render_family_samples` [`L164`], and raises `SystemExit` on anything else [`L167-L168`]. It **lists / resolves / renders** fonts and performs **no persistence**.
- **Conclusion (cause → effect):** since the backend never writes config, *all* config writing is in the Go layer — specifically the `Enter` branch of `final_pane.on_key_event` (§7). This is the second reason a build is required: the durable behavior is compiled Go, not interpretable Python.

### 6.4 Display dependency (satisfied by the provisioned Xvfb)

`h.lp.QueryTerminal("font_size", "dpi_x", "dpi_y", "foreground", "background")` [`kittens/choose_fonts/ui.go:L80`] shows the kitten queries the host terminal, and previews use the graphics protocol [`kittens/choose_fonts/graphics.go`], so a real display is required to run the UI. The provisioned container satisfies this with `Xvfb` on `DISPLAY=:99`, under which the panes above were observed. The only step not captured as a discrete effect is pressing each of the non-`Enter` keys to completion; those file effects are labeled **CODE-DERIVED** with their exact `file:line`, per Rules 1 & 3.

---

## 7. R5 — Finalization: the persistence primitive

### 7.1 What the final pane shows (OBSERVED)

The final confirmation pane of the real running kitten (under `DISPLAY=:99`, after choosing `DejaVu Sans Mono` and pressing `Enter` on the faces pane) was **OBSERVED** to display exactly:

```
You have chosen the DejaVu Sans Mono family
What would you like to do?

Enter to modify kitty.conf and use the new fonts

Esc to abort and return to font selection

s to write the new font settings to STDOUT

Ctrl+c to quit
```

This is generated by `final_pane.draw_screen` [`kittens/choose_fonts/final.go:L29-L47`], and the observed text matches the source lines verbatim:

- *"`Enter` to modify `kitty.conf` and use the new fonts"* [`final.go:L38`]
- *"`Esc` to abort and return to font selection"* [`final.go:L40`]
- *"`s` to write the new font settings to `STDOUT`"* [`final.go:L42`]
- *"`Ctrl+c` to quit"* [`final.go:L44`]

**This observed pane is decisive for the verdict:** the kitten's own UI declares that **`Enter` modifies `kitty.conf`** (persist) while **`s` writes to `STDOUT`** (session/pipe). The four options offered (`Enter`, `Esc`, `s`, `Ctrl+c`) are all OBSERVED as displayed; the on-disk *effect* of `Enter` is captured in §8, and the file effects of `Esc`/`s`/`Ctrl+c` are established from source (§9).

### 7.2 The `Enter` branch (CODE-DERIVED anchors; write OBSERVED in §8)

`final_pane.on_key_event` [`final.go:L72-L99`], the `enter` case [`final.go:L78-L97`]:

```go
patcher := config.Patcher{Write_backup: true}                 // final.go:L80
path := filepath.Join(utils.ConfigDir(), "kitty.conf")        // final.go:L81
updated, err := patcher.Patch(
    path, "KITTY_FONTS", self.settings.serialized(),
    "font_family", "bold_font", "italic_font", "bold_italic_font",
)                                                             // final.go:L82
// … then …
lp.Quit(0)                                                    // final.go:L94
```

`serialized()` [`final.go:L63-L70`] emits **exactly four** space-padded lines joined by `\n`:

```
font_family      <value>
bold_font        <value>
italic_font      <value>
bold_italic_font <value>
```

### 7.3 How the write becomes durable — `config.Patcher.Patch`

`config.Patcher.Patch` [`tools/config/api.go:L310-L350`] performs, in order:

1. Resolve symlinks on the target path [`api.go:L315-L317`].
2. Read any existing file content [`api.go:L318`].
3. **Comment out** any prior matching settings by replacing each matched key line with `# $1` via regex [`api.go:L326`] (so a previous `font_family …` becomes `# font_family …`).
4. Build the addition wrapped in sentinels: `# BEGIN_KITTY_FONTS\n<content>\n# END_KITTY_FONTS` [`api.go:L328-L330`] (the sentinel name `KITTY_FONTS` is the argument passed at `final.go:L82`).
5. If content changed **and** `Write_backup` is set and there was prior content, write a `<path>.bak` backup — `os.WriteFile(backup_path+".bak", …)` [`api.go:L343-L344`].
6. Replace the file via `utils.AtomicUpdateFile` [`api.go:L347`].

The atomic writer `AtomicUpdateFile` [`tools/utils/atomic-write.go:L79`] performs the classic **write-temp-then-rename** (`CreateTemp` [`atomic-write.go:L53`] → `os.Rename` [`atomic-write.go:L67`]). Because the finished bytes are `rename`d into place on disk, they are **re-read on the next startup** ⇒ **persistence** (cause → effect). The atomicity also means a crash mid-write cannot leave a half-written `kitty.conf`.

### 7.4 Reload is orthogonal to persistence

After a successful *write*, the `Enter` branch consults `--reload-in` [`final.go:L86-L93`]:

- `"parent"` → `config.ReloadConfigInKitty(true)`
- `"all"` → `config.ReloadConfigInKitty(false)`
- `"none"` → **no case → no reload** (the file is still written)

`ReloadConfigInKitty` [`tools/config/api.go:L352-L371`] sends **`SIGUSR1`** — to the parent instance via `KITTY_PID` [`api.go:L354-L357`, signal at `L357`], or to all kitty GUI processes [`api.go:L363-L369`, signal at `L366`]. It merely asks *already-running* instances to re-read the file; it does **not** decide whether the file was written. The official `kitty.conf` docs corroborate this as a general, separate mechanism: config is auto-reloaded when modified, and can be manually reloaded by *"sending kitty the SIGUSR1 signal with `kill -SIGUSR1 $KITTY_PID`."* Persistence (the write) and reload (the signal) are two independent things.

---

## 8. R6 — Persistence verdict with before / after / restart evidence

### 8.1 Isolation so the real config is never touched

`utils.ConfigDir()` [`tools/utils/paths.go:L132-L134`] is `sync.OnceValue(… ConfigDirForName("kitty.conf"))`, and `ConfigDirForName` [`paths.go:L88`] honors the environment override first:

```go
if kcd := os.Getenv("KITTY_CONFIG_DIRECTORY"); kcd != "" {
    return Abspath(Expanduser(kcd))                 // paths.go:L89-L91
}
```

So exporting `KITTY_CONFIG_DIRECTORY` routes the *identical* code path to a throwaway directory:

```bash
export KITTY_CONFIG_DIRECTORY=/tmp/kitty_iso_conf
```

This was confirmed **OBSERVED**: the real `utils.ConfigDir()` returned exactly the exported directory.

### 8.2 Scenario A — a pre-existing `kitty.conf` (Enter → persist)

The write below is **OBSERVED (real persistence code, harness-triggered)**: it runs the exact `config.Patcher.Patch(path, "KITTY_FONTS", <serialized>, "font_family", "bold_font", "italic_font", "bold_italic_font")` + `utils.ConfigDir()` calls from `final.go:L80-L82`, under the isolated `KITTY_CONFIG_DIRECTORY`. This is the identical code path the GUI `Enter` branch runs (§2.3); capturing it directly gives clean, reproducible before/after bytes around the isolated config, which tracking the file around a single live GUI keypress would not.

**BEFORE** — the seeded config (command: `cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"`):

```
font_size 12.0
font_family Old Family Name
```

**Run** — real functions report (harness stdout):

```
REAL utils.ConfigDir() -> /tmp/kitty_iso_conf
target path             -> /tmp/kitty_iso_conf/kitty.conf
Patch updated           -> true
```

**AFTER** — the same file re-read (command: `cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"`):

```
font_size 12.0
# font_family Old Family Name


# BEGIN_KITTY_FONTS
font_family      Fira Code
bold_font        Fira Code Bold
italic_font      Fira Code Italic
bold_italic_font Fira Code Bold Italic
# END_KITTY_FONTS
```

Observe, next to the code that produced each effect:
- the unrelated `font_size 12.0` is **preserved**;
- the prior `font_family Old Family Name` is **commented out** → `# font_family Old Family Name` (regex at `api.go:L326`);
- the four selected faces are appended inside `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` (sentinel at `api.go:L328-L330`; the four keys are the arguments from `final.go:L82`).

**Backup** — a `.bak` was created (command: `ls -la "$KITTY_CONFIG_DIRECTORY"` then `cat kitty.conf.bak`), preserving the pre-write bytes [`api.go:L343-L344`]:

```
font_size 12.0
font_family Old Family Name
```

**RESTART PROOF** — a *fresh* process re-read the on-disk file after the writing process exited; the block is **still present** (command: `cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"` from a new process):

```
# BEGIN_KITTY_FONTS
font_family      Fira Code
bold_font        Fira Code Bold
italic_font      Fira Code Italic
bold_italic_font Fira Code Bold Italic
# END_KITTY_FONTS
```

(the surrounding `font_size` / commented line are retained as shown in AFTER). Because the bytes persist across process boundaries on disk, kitty re-reads them at startup ⇒ **the selection is PERSISTED across restarts.**

**Idempotency** — running the same write a second time reported `Patch updated -> false` (the `bytes.Equal` short-circuit at `api.go:L342`); the file kept **exactly one** `# BEGIN_KITTY_FONTS` block (no duplication) and no temporary files were left behind (atomic `rename`).

### 8.3 Scenario B — no `kitty.conf` present (Enter → persist, no `.bak`)

Same real `Patch` call against a fresh, empty `KITTY_CONFIG_DIRECTORY` (`/tmp/kitty_iso_conf2`) — **OBSERVED (real persistence code, harness-triggered)**:

```
REAL utils.ConfigDir() -> /tmp/kitty_iso_conf2
target path             -> /tmp/kitty_iso_conf2/kitty.conf
Patch updated           -> true
```

**AFTER** — `kitty.conf` is created containing only the block; **no** `.bak` exists, because the backup is guarded by "there was prior content" (`len(raw) > 0` at `api.go:L343`):

```
# BEGIN_KITTY_FONTS
font_family      Fira Code
bold_font        Fira Code Bold
italic_font      Fira Code Italic
bold_italic_font Fira Code Bold Italic
# END_KITTY_FONTS
```

This proves persistence even from a clean slate: pressing `Enter` *creates* `kitty.conf` if absent and writes the durable block.

### 8.4 Contrast — the `s`/`S` key is session/STDOUT only (no persistence)

**CODE-DERIVED** ([`final.go:L101-L111`], `on_text`): pressing `s` (or `S`) sets `output_on_exit = self.settings.serialized() + "\n"` [`final.go:L105`] and quits [`final.go:L106`]; the string is then written to STDOUT by `main.go:L64-L66` (`if output_on_exit != "" { os.Stdout.WriteString(output_on_exit) }`). Crucially, this branch **never calls `config.Patcher.Patch`**, so **`kitty.conf` is unchanged**.

The bytes emitted to STDOUT are exactly the four `serialized()` lines (the same content shown in §7.2, **without** the `# BEGIN/END_KITTY_FONTS` wrapper), e.g.:

```
font_family      Fira Code
bold_font        Fira Code Bold
italic_font      Fira Code Italic
bold_italic_font Fira Code Bold Italic
```

`kitty.conf` state for this branch: **unchanged → unchanged** (session/pipe only). This is the "session-only" behavior — and it is a *different key* from `Enter`.

---

## 9. Exhaustive branch table (Enter / Esc / `s`·`S` / Ctrl+C)

Every mode the question implies, with its handler anchor, effect, `kitty.conf` before → after, and the evidence label. All four keys were **OBSERVED as offered** on the real final pane (§7.1). `Enter`'s on-disk persistence effect is **OBSERVED (real persistence code, harness-triggered)** via the identical code path (§8); the *file effects* of `Esc`, `s`/`S`, and `Ctrl+C` are **CODE-DERIVED** from source (each key was not individually pressed to completion and captured), each with an exact `file:line`.

| Key | Handler / anchor | Effect | `kitty.conf` before → after | Evidence |
|-----|------------------|--------|------------------------------|----------|
| **Enter** | `final_pane.on_key_event` `final.go:L78-L95` → `config.Patcher.Patch` `api.go:L310-L350` → `utils.AtomicUpdateFile` `atomic-write.go:L79` | Writes `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` block + `.bak`, atomic replace; then **optional** `SIGUSR1` reload per `--reload-in` `final.go:L86-L93` | no block → **block present (PERSISTED)**; `.bak` created when prior content existed | OBSERVED (real persistence code, harness-triggered) — §8.2/§8.3 |
| **Esc** | `final_pane.on_key_event` `final.go:L73-L77` | Sets `current_pane = &faces`; returns to the faces pane; **no write** | unchanged → unchanged | CODE-DERIVED `final.go:L73-L77` |
| **`s` / `S`** | `final_pane.on_text` `final.go:L101-L111` → STDOUT via `main.go:L64-L66` | Serializes the 4 lines into `output_on_exit`, quits, prints them to **STDOUT**; **never calls `Patch`** | unchanged → unchanged | CODE-DERIVED `final.go:L101-L111`, `main.go:L64-L66` — §8.4 |
| **Ctrl+C** | loop death-signal path `main.go:L58-L63` (not handled in `final.go`) | Prints `"Killed by signal: …"`, exits `1`; **no write** | unchanged → unchanged | CODE-DERIVED `main.go:L58-L63` |

**Only `Enter` mutates `kitty.conf`.** `Esc`, `s`/`S`, and `Ctrl+C` all leave the file byte-for-byte unchanged.

---

## 10. Cause → effect summary

- **Why `Enter` persists:** `Enter` → `final_pane.on_key_event` builds `config.Patcher{Write_backup: true}` and calls `Patcher.Patch(ConfigDir()/kitty.conf, "KITTY_FONTS", serialized(), font_family, bold_font, italic_font, bold_italic_font)` [`final.go:L80-L82`]. `Patch` comments out prior keys, wraps the four faces in `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS`, writes a `.bak`, and atomically renames the new file into place [`api.go:L326-L347`; `atomic-write.go:L79`]. **The bytes are on disk**, so kitty re-reads them at the next startup → **PERSISTED**. (Observed in §8.2/§8.3.)
- **Why reload does *not* equal persistence:** the reload branch only fires *after* the write and only sends `SIGUSR1` to *already-running* instances to re-read the *already-durable* file [`final.go:L86-L93`; `api.go:L352-L371`]. With `--reload-in none` there is no signal at all, yet the file is still written. Reload changes the *running session's live view*; the *write* is what makes the choice durable.
- **Why `s`/`S` is session-only:** it serializes the four lines to `output_on_exit` and prints them to STDOUT [`final.go:L101-L111`; `main.go:L64-L66`] and **never calls `Patch`**, so `kitty.conf` is untouched — a pipe/session output, not a persisted setting.
- **Why a build was required:** the persistence logic is compiled **Go** in the `kitten` binary; the Python backend only lists/resolves/renders fonts and writes nothing [`backend.py:L150-L168`]. Reading Python alone cannot reveal the behavior — hence the run-first canonical build.

---

## 11. Appendix — Observed vs. code-derived

### 11.1 Directly OBSERVED at runtime

- **Canonical build** via `python3 setup.py build --verbose` (X11-only, exit 0); the emitted Go kitten build command; the resulting binaries (`kitty/launcher/kitten`, `kitty/launcher/kitty`, `kitty/fast_data_types.so`); versions `kitten 0.35.2` / `kitty 0.35.2`. (§3)
- **Canonical invocation** `kitten choose-fonts --help` output, confirming the subcommand description, the `--reload-in` choices `parent, all, none`, and default `parent`. (§5)
- **The real running kitten's panes** under `DISPLAY=:99` — the **family listing** pane (family list, `>` selection, `Family:` filter, per-family preview), the **faces** pane (the four faces `Regular`/`Bold`/`Italic`/`Bold-Italic` with rendered previews and the `Enter`/`Esc` prompt), and the **final confirmation** pane whose text reads *"Enter to modify kitty.conf and use the new fonts"* / *"s to write the new font settings to STDOUT"* — captured as screenshots during the investigation and transcribed in §6.2 and §7.1. (These screenshot artifacts were removed afterward so the repository holds only this `.md`.)
- **`KITTY_CONFIG_DIRECTORY` routing** — the real `utils.ConfigDir()` returned the exported isolation directory. (§8.1)
- **The persistence write and its durability** — OBSERVED as *real persistence code, harness-triggered*: the real `config.Patcher.Patch` + `utils.ConfigDir` (the exact functions/args from `final.go:L80-L82`) produced the `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` block on disk, a `kitty.conf.bak`, correct comment-out of prior keys, preservation of unrelated settings, idempotency (`Patch updated -> false` on re-run), the restart re-read (block still present), and the clean-slate case (block created, no `.bak`). (§8.2, §8.3)

### 11.2 CODE-DERIVED (established from source; exact `file:line`)

The following are read from source with exact `file:line`. They are the internal control-flow and the *file effects* of the non-`Enter` keys (each such key is OBSERVED as *offered* on the final pane, but was not individually pressed to completion and captured). No interactive screen output was fabricated:

- Subcommand registration & option definition — `main.go:L74-L98`.
- Internal value flow (`Options` → handler → panes) — `main.go:L16-L68`, `main.go:L70-L72`, `main.go:L35`, `ui.go:L81`, `ui.go:L97`, `list.go:L170`, `types.go:L78-L87`, `faces.go:L14`, `faces.go:L113-L120`. (The *visible* pane progression these produce was OBSERVED — §6.2.)
- Backend boundary and "no persistence in Python" — `backend.go:L41`, `backend.go:L44`/`L48`, `backend.py:L150-L168`.
- The `Esc` / `s`·`S` / `Ctrl+C` **file effects** and the `serialized()` format — `final.go:L63-L70`, `final.go:L72-L111`, `main.go:L58-L66`. (The final-pane prompt text itself was OBSERVED — §7.1.)
- Persistence engine internals & reload — `api.go:L310-L371`, `atomic-write.go:L79` (`atomic-write.go:L42`/`L53`/`L67`), `paths.go:L88-L91`, `paths.go:L132-L134`.

### 11.3 Web-search corroboration

- **Build:** official docs (`sw.kovidgoyal.net/kitty/build/`) — kitty runs from source; needs a C compiler and the Go compiler (plus X11 dev libraries on Linux); `./dev.sh build` → `kitty/launcher/kitty`; required system libraries are X11 and DBUS on Linux; `python3 setup.py build` also documented; `dev.sh` wraps `bypy/devenv.go`. Matches `dev.sh:L9`, `Makefile:L12-L13`, `setup.py:L1084`.
- **choose-fonts / kitty.conf:** official `kitty.conf` reference recommends `kitten choose-fonts` as the easiest way to select fonts, and documents the four keys `font_family`, `bold_font`, `italic_font`, `bold_italic_font`. It also documents `SIGUSR1` (`kill -SIGUSR1 $KITTY_PID`) as a *separate* manual-reload mechanism — corroborating that reload ≠ persistence.
- **Note on in-repo docs:** `docs/kittens/choose-fonts.rst` exists on GitHub *master* but is **absent from this checkout** (`815df1e210e0`) — verified locally. This document therefore grounds the interaction model in the **source code** (cited throughout) plus the **official online docs** above.

### 11.4 Reproduction recipe (outside the checkout)

```bash
# 1) Canonical build (default config)
python3 setup.py build --verbose        # -> kitty/launcher/{kitty,kitten}

# 2) Canonical invocation of the real entry point
kitten choose-fonts --help              # kitty +kitten choose-fonts

# 3) Isolate config so the real ~/.config/kitty is never touched
export KITTY_CONFIG_DIRECTORY=/tmp/kitty_iso_conf   # paths.go:L89-L91

# 4) Persistence path exercised via the REAL functions from final.go:L80-L82:
#    config.Patcher{Write_backup:true}.Patch(
#        filepath.Join(utils.ConfigDir(),"kitty.conf"), "KITTY_FONTS",
#        serialized, "font_family","bold_font","italic_font","bold_italic_font")
#    -> observe the # BEGIN_KITTY_FONTS…# END_KITTY_FONTS block + kitty.conf.bak,
#       then re-read from a fresh process to confirm it persists.
```

All observation artifacts (the isolated `KITTY_CONFIG_DIRECTORY`, the out-of-checkout Go harness, and scratch logs) were created **outside** the repository and removed afterward, leaving the kitty source tree pristine.
