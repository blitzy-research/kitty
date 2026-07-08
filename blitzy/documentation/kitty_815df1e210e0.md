# `choose-fonts` Kitten — End-to-End Behavior & Persistence (Runtime-Verified Onboarding Q&A)

> **Repository:** `kovidgoyal/kitty`
> **Branch / commit anchored:** `kitty_815df1e210e0` @ `815df1e21`
> **Method:** Every behavioral claim below is backed by **unedited output** captured from the **real** `kitten choose-fonts` entry point, run inside the project's canonical build. Every factual claim carries a `file:line` citation naming the concrete function/struct. Nothing here is derived from a remote-control shortcut, a debug hook, importing internals, or calling `Patcher.Patch` directly.

---

## TL;DR — Is a font selection remembered across restarts?

**Yes.** When you press **Enter** at the final confirmation screen, the kitten **patches the persistent `kitty.conf` file on disk** — it inserts a sentinel-delimited managed font block and writes a `.bak` backup — so the choice is **durable and survives restarts**. It is *not* a session-only change. The decisive code is the Enter branch of `final_pane.on_key_event`, which builds a `config.Patcher{Write_backup: true}`, resolves `path := filepath.Join(utils.ConfigDir(), "kitty.conf")`, and calls `patcher.Patch(...)` [kittens/choose_fonts/final.go:L78-L97], delegating to `Patcher.Patch` [tools/config/api.go:L310-L350].

The kitten's single option, `--reload-in`, is a **separate concern**: it only controls whether *already-running* kitty instances are signalled (`SIGUSR1`) to live-reload their config. It does **not** gate the on-disk write — even `--reload-in none` still persists the choice to `kitty.conf`.

Direct proof (fresh processes, same temporary config dir; full transcript in **Q4**):

```
# A brand-new kitty reading the PERSISTED config resolves the chosen family:
$ KITTY_CONFIG_DIRECTORY="$CFG" kitty --debug-font-fallback bash -c true
[0.161]   Normal: FiraCodeRoman-Regular: /root/.local/share/fonts/FiraCode-VF.ttf:131072

# The SAME binary with default settings (--config NONE) does NOT get Fira Code:
$ kitty --config NONE --debug-font-fallback bash -c true
[0.159]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
```

The only difference between those two fresh launches is the persisted `kitty.conf` — hence the selection is remembered across restarts.

---

## How the interactive TUI was driven (methodology)

`choose-fonts` is an interactive full-screen TUI (arrow keys + Enter). To exercise the **real** keypress path non-interactively and capture output, it was driven through a small pseudo-terminal harness written with the Python standard library only (the container has no `tmux`, `pexpect`, or `pyte`, and no internet). The harness:

1. Spawns the real `kitty/launcher/kitten choose-fonts` on a PTY and makes that PTY the child's **controlling terminal** (`os.setsid()` + `ioctl(TIOCSCTTY)`); without this the kitten aborts with `open /dev/tty: no such device or address`.
2. Speaks kitty's keyboard protocol. The kitten enables progressive-enhancement **mode 29**, so keys are sent as CSI-u sequences: **Enter** = `\x1b[13u`, **Esc** = `\x1b[27u`, **Ctrl+c** = `\x1b[99;5u`, **s** = `\x1b[115;;115u`.
3. Answers the kitten's `XTGETTCAP` DCS queries (font size / DPI / fg / bg); if these go unanswered the Python backend aborts while parsing colors.
4. Renders the in-place-redrawing VT stream into clean 2-D snapshots via a minimal ANSI screen emulator, so the pane captures below are legible.

This is still the **real** entry point and the **real** keypress path — no `Patcher.Patch` call, no remote control, no debug bypass. Where a value could only be read from source (not observed at runtime) it is explicitly **labeled `inferred`** below.

All temporary artifacts (the harness, temporary `KITTY_CONFIG_DIRECTORY` dirs, generated `kitty.conf`/`.bak`) live outside the repository tree and are removed on completion; the working tree is left byte-for-byte unchanged apart from this document.

---

## Q1 — Build the repo and launch a single instance with default settings

**Direct answer:** Build with `./dev.sh build` (dev launcher) or `make` / `python3 setup.py`; this produces `kitty/launcher/kitty`. Launch a single instance with default settings using `kitty/launcher/kitty --config NONE`.

### Build

The dev launcher delegates the whole build to a Go program:

- `dev.sh:L9` is exactly `exec go run bypy/devenv.go "$@"` — so `./dev.sh build` runs the Go dev-environment builder.
- `make` runs the same thing through setup.py: the default `all:` target [Makefile:L12] executes `python3 setup.py $(VVAL)` [Makefile:L13].
- The build guide states you need a C compiler and the Go compiler, then `git clone … && cd kitty` and `./dev.sh build`, after which "You can run it as `kitty/launcher/kitty`" [docs/build.rst:L14-L22].

Runtime/toolchain floors from the manifests: **Python `>=3.8`** [pyproject.toml:L2 `requires-python = ">=3.8"`] and **Go `1.22`** [go.mod:L3 `go 1.22`]. The actual toolchain observed in the build container:

```
$ go version
go version go1.22.12 linux/amd64
$ python3 --version
Python 3.13.7
```

Running the canonical build reproduces the launcher (unedited tail):

```
$ ./dev.sh build --ignore-compiler-warnings
...
Build successful. Run kitty as: kitty/launcher/kitty
```

> Note: `--ignore-compiler-warnings` is required in this environment's newer C toolchain; the build otherwise treats warnings as errors. The produced binaries `kitty/launcher/kitty` and `kitty/launcher/kitten` are git-ignored (they never appear in `git status`).

### Version banner (unedited)

```
$ kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

This matches the declared version `version: Version = Version(0, 35, 2)` [kitty/constants.py:L25]. The sibling `kitten` reports the same:

```
$ kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
```

### "Default settings" launch — `--config NONE`

The canonical way to launch with *no* user configuration (so only built-in defaults apply) is `--config NONE`. The CLI help documents this: `CONFIG_HELP` begins at [kitty/cli.py:L183]; the special value semantics — use `NONE` to not load any config file — are at [kitty/cli.py:L187-L188]; the `KITTY_CONFIG_DIRECTORY` behavior is described at [kitty/cli.py:L196-L197]. Launch command:

```
$ kitty/launcher/kitty --config NONE
```

Because the container is headless, the GUI was launched under `Xvfb`. It starts cleanly; the only output is a benign bus warning (unedited):

```
$ xvfb-run -a kitty/launcher/kitty --config NONE bash -c true
[0.156] Failed to open systemd user bus with error: Connection refused
```

That message is a non-fatal warning about the absence of a systemd user session in the container; the process starts and exits `0`. All font-resolution captures in **Q4** use this same headless launch pattern with `--debug-font-fallback` so the resolved fonts are printed at startup.

---

## Q2 — Invoke `choose-fonts` from inside the instance

**Direct answer:** Run `kitten choose-fonts`. From inside a running kitty window `kitten` is already on `PATH`; otherwise invoke the built binary directly as `kitty/launcher/kitten choose-fonts`.

### Why the invocation resolves

`choose-fonts` is wired into the `kitten` tool's command tree. `KittyToolEntryPoints(root)` [tools/cmd/tool/main.go:L35] registers every subcommand; the `// choose-fonts` comment is at [tools/cmd/tool/main.go:L81] and the registration call `choose_fonts.EntryPoint(root)` is at [tools/cmd/tool/main.go:L82]. That call reaches `EntryPoint` in the kitten's own package [kittens/choose_fonts/main.go:L74-L99] (traced in Q3a).

The `kitten` binary itself is built into the launcher directory: `dest = os.path.join(destination_dir or launcher_dir, 'kitten')` [setup.py:L1160], i.e. `kitty/launcher/kitten`.

### The kitten needs to find the `kitty` executable

The kitten spawns its Python enumeration backend by executing the `kitty` binary, which it locates via `KittyExe` (a `sync.OnceValue`) [tools/utils/paths.go:L69-L86]: it first tries the exe of `$KITTY_PID`'s parent process, then the sibling `kitty` next to the running executable (`kitty/launcher/kitty`), and finally the `$KITTY_PATH_TO_KITTY_EXE` fallback [tools/utils/paths.go:L85]. When driving the kitten outside a real kitty window, exporting `KITTY_PATH_TO_KITTY_EXE=<abs>/kitty/launcher/kitty` guarantees resolution.

### `--help` resolves (unedited)

```
$ kitty/launcher/kitten choose-fonts --help
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

### TUI progression: listing → faces → final

Driving the real TUI (see methodology) captures the three panes in order.

**Stage 1 — family listing** (the `>` marks the selected family; a live preview and a `Family:` search prompt are shown):

```
 Cascadia Code         ║                                        DejaVu Sans Mono
 Cascadia Code NF      ║
 Cascadia Code PL      ║ Styles: Bold, Bold Oblique, Book, Oblique
 Cascadia Mono         ║
 Cascadia Mono NF      ║ Press the Enter key to choose this family
 Cascadia Mono PL      ║
 Comfy Code            ║ ─────────────────────────────────────────── preview ───────────────────────────────────────────
>DejaVu Sans Mono      ║
 Fantasque Sans Mono   ║
 Fira Code             ║
 Hack                  ║
 IBM Plex Mono         ║
 Inconsolata           ║
 JetBrains Mono        ║
 JetBrains Mono NL     ║
 Liberation Mono       ║
 Noto Mono             ║
 Noto Sans SignWriting ║
 Source Code Pro       ║
 SourceCodeVF          ║
 Ubuntu Mono           ║
 Ubuntu Sans Mono      ║
                       ║
Family:
```

**Stage 2 — faces** (reached by pressing Enter on a family):

```
                                                    DejaVu Sans Mono

Press Enter to select this font, Esc to go back to the font list or any of the highlighted keys below to fine-tune the
appearance of the individual font styles.

Regular: DejaVuSansMono


Bold: DejaVuSansMono-Bold


Italic: DejaVuSansMono-Oblique


Bold-Italic: DejaVuSansMono-BoldOblique
```

**Stage 3 — final confirmation** (reached by pressing Enter on the faces pane):

```
You have chosen the DejaVu Sans Mono family

What would you like to do?

Enter to modify kitty.conf and use the new fonts

Esc to abort and return to font selection

s to write the new font settings to STDOUT

Ctrl+c to quit
```

---

## Q3a — How the subcommand is registered and its options are parsed

**Direct answer:** `EntryPoint` registers `choose-fonts` as a subcommand via `root.AddSubCommand`, declares exactly one option `--reload-in` (a `choices` option with values `parent, all, none`, default `parent`) via `ans.Add(cli.OptionSpec{...})`, and its `Run` closure parses the CLI into an `Options` struct with `cmd.GetOptionValues(&opts)`. A non-hidden clone alias `choose_fonts` (underscore) is also registered.

### Registration

`EntryPoint(root *cli.Command)` spans [kittens/choose_fonts/main.go:L74-L99]. It calls `root.AddSubCommand` with the name and description [kittens/choose_fonts/main.go:L75-L77]:

```go
ans := root.AddSubCommand(&cli.Command{
    Name:             "choose-fonts",
    ShortDescription: "Choose the fonts used in kitty",
```

### Option parsing → `Options`

The `Run` closure parses CLI values into the `Options` struct [kittens/choose_fonts/main.go:L78-L84]:

```go
Run: func(cmd *cli.Command, args []string) (rc int, err error) {
    opts := Options{}
    if err = cmd.GetOptionValues(&opts); err != nil {
        return 1, err
    }
    return main(&opts)
},
```

The `Options` struct has a single field [kittens/choose_fonts/main.go:L70-L72]:

```go
type Options struct {
    Reload_in string
}
```

### The single option — `--reload-in`

Registered via `ans.Add(cli.OptionSpec{...})` [kittens/choose_fonts/main.go:L86-L95]:

```go
ans.Add(cli.OptionSpec{
    Name:    "--reload-in",
    Dest:    "Reload_in",
    Type:    "choices",
    Choices: "parent, all, none",
    Default: "parent",
    Help: `By default, this kitten will signal only the parent kitty instance it is
running in to reload its config, after making changes. Use this option to
instead either not reload the config at all or in all running kitty instances.`,
})
```

**Exhaustive coverage of every `--reload-in` value** (causal reason for each is proven in Q4):

| Value | Parsed into | What it does at finalize | Persists to disk? |
|-------|-------------|--------------------------|-------------------|
| `parent` (default) | `opts.Reload_in = "parent"` | `case "parent": config.ReloadConfigInKitty(true)` — SIGUSR1 to the `$KITTY_PID` process only [final.go:L88-L89] | **Yes** |
| `all` | `opts.Reload_in = "all"` | `case "all": config.ReloadConfigInKitty(false)` — SIGUSR1 to every kitty GUI process [final.go:L90-L91] | **Yes** |
| `none` | `opts.Reload_in = "none"` | **no matching case** in the switch [final.go:L87-L92] → no live reload | **Yes** (still patched) |

The `choices` type is enforced at parse time. Passing an invalid value is rejected (unedited):

```
$ kitty/launcher/kitten choose-fonts --reload-in bogus --help
Error: bogus is not a valid value for --reload-in. Valid values: parent, all, none
```

…while the valid variants parse cleanly (each still shows the same choices line):

```
$ kitty/launcher/kitten choose-fonts --reload-in all --help
Usage: kitten choose-fonts
    Choices: parent, all, none
$ kitty/launcher/kitten choose-fonts --reload-in none --help
Usage: kitten choose-fonts
    Choices: parent, all, none
```

### The clone alias `choose_fonts` is **NOT hidden** (observed reality, not prose)

Immediately after registration, `EntryPoint` registers a clone [kittens/choose_fonts/main.go:L96-L98]:

```go
clone := root.AddClone(ans.Group, ans)
clone.Hidden = false
clone.Name = "choose_fonts"
```

Because `clone.Hidden = false` [kittens/choose_fonts/main.go:L97], the underscore alias `choose_fonts` is a **visible** alias, not a hidden one. Confirmed at runtime two ways (unedited):

```
$ kitty/launcher/kitten choose_fonts --help
Usage: kitten choose_fonts

Choose the fonts used in kitty
```

```
$ kitty/launcher/kitten --help   # (grep of the subcommand listing, with line numbers)
40:   choose-fonts
42:   choose_fonts
```

Both the hyphenated and underscored names resolve and both appear in the top-level `kitten --help` listing.


---

## Q3b — How option values flow through the program to the final step

**Direct answer:** The parsed `Options` (carrying `Reload_in`) is stored on the `handler` once, at construction; the family/style *selection* flows independently through a small pane state machine (listing → faces → final). `opts.Reload_in` is **not** consulted by the listing or faces panes — it is read **only** at finalize.

### `opts` enters the handler once

`main(opts *Options)` [kittens/choose_fonts/main.go:L16-L68] starts the Python backend, creates the loop, and builds the handler, wiring `opts` in at [kittens/choose_fonts/main.go:L35]:

```go
h := &handler{lp: lp, opts: opts}
```

### The pane state machine

Defined in [kittens/choose_fonts/ui.go]:

- `type State int` [ui.go:L17] with constants `SCANNING_FAMILIES` / `LISTING_FAMILIES` / `CHOOSING_FACES` [ui.go:L19-L23].
- The `pane` interface [ui.go:L33-L40].
- The `handler` struct [ui.go:L42-L62] holds `opts *Options` [ui.go:L43], plus the panes: `listing FontList` [ui.go:L55], `faces faces` [ui.go:L56], `face_pane face_panel` [ui.go:L57], `final_pane final_pane` [ui.go:L58], and the active `current_pane pane` [ui.go:L61].

### Listing → faces

In `FontList.on_key_event` [kittens/choose_fonts/list.go:L246-L254], pressing Enter — guarded by a non-empty current family — calls `faces.on_enter(family)` [kittens/choose_fonts/list.go:L250]:

```go
if event.MatchesPressOrRepeat("enter") {
    event.Handled = true
    if family := self.family_list.CurrentFamily(); family != "" {
        return self.handler.faces.on_enter(family)
    }
    self.handler.lp.Beep()
    return
}
```

### faces builds `faces_settings`

`faces.on_enter(family)` [kittens/choose_fonts/faces.go:L145-L159] populates `self.settings` (a `faces_settings`) from the resolved faces of the current `kitty.conf`, falling back to a default spec when the chosen family is not the one already configured [kittens/choose_fonts/faces.go:L152-L155]:

```go
d(r.Font_family, &self.settings.font_family, fmt.Sprintf(`family="%s"`, family))
d(r.Bold_font, &self.settings.bold_font, "auto")
d(r.Italic_font, &self.settings.italic_font, "auto")
d(r.Bold_italic_font, &self.settings.bold_italic_font, "auto")
```

The struct itself [kittens/choose_fonts/faces.go:L14-L16]:

```go
type faces_settings struct {
    font_family, bold_font, italic_font, bold_italic_font string
}
```

> This default rule (`family="…"` for the family; `auto` for the three styles) is exactly what the **Q4 Fira Code experiment** observes on disk when a not-yet-configured family is chosen — confirming this branch is the one exercised.

### faces → final

In `faces.on_key_event` [kittens/choose_fonts/faces.go:L112-L123], Enter calls `final_pane.on_enter(self.family, self.settings)` [kittens/choose_fonts/faces.go:L120]:

```go
if event.MatchesPressOrRepeat("enter") {
    event.Handled = true
    return self.handler.final_pane.on_enter(self.family, self.settings)
}
```

`final_pane.on_enter(family, settings)` [kittens/choose_fonts/final.go:L113-L118] stores `self.settings`/`self.family` and makes the final pane the current pane. **The option value `opts.Reload_in` plays no part until the Enter key is pressed on this final pane** — it is read from `self.handler.opts` inside the finalize handler (Q3c).

### Runtime confirmation of propagation

Driving listing → faces → final and reading the final pane proves the chosen family propagated end-to-end (the final screen names it):

```
You have chosen the DejaVu Sans Mono family
```

…and when the search box is used to pick a different family (Fira Code) the final pane names *that* one instead (from the Q4 experiment transcript):

```
FINAL: You have chosen the Fira Code family
```

---

## Q3c — What kitty does when the selection is finalized (output evidence)

**Direct answer:** Pressing **Enter** writes a **sentinel-delimited managed font block** into `kitty.conf` on disk (and a `.bak` backup when a prior file existed), committing atomically; it then optionally signals running instances to reload. The other final-screen keys do **not** write to disk. Proven byte-for-byte below.

### The final screen and its keys

`draw_screen` [kittens/choose_fonts/final.go:L29-L47] renders four actions — Enter [final.go:L38], Esc [final.go:L40], `s` [final.go:L42], Ctrl+c [final.go:L44]:

```go
fmt.Sprintf("%s to modify %s and use the new fonts", h("Enter"), s("italic", `kitty.conf`)),
"",
fmt.Sprintf("%s to abort and return to font selection", h("Esc")),
"",
fmt.Sprintf("%s to write the new font settings to %s", h("s"), s("italic", `STDOUT`)),
"",
fmt.Sprintf("%s to quit", h("Ctrl+c")),
```

### Enter → patch `kitty.conf`

The Enter branch of `on_key_event` [kittens/choose_fonts/final.go:L78-L97]:

```go
if event.MatchesPressOrRepeat("enter") {
    event.Handled = true
    patcher := config.Patcher{Write_backup: true}
    path := filepath.Join(utils.ConfigDir(), "kitty.conf")
    updated, err := patcher.Patch(path, "KITTY_FONTS", self.settings.serialized(), "font_family", "bold_font", "italic_font", "bold_italic_font")
    if err != nil {
        return err
    }
    if updated {
        switch self.handler.opts.Reload_in {
        case "parent":
            config.ReloadConfigInKitty(true)
        case "all":
            config.ReloadConfigInKitty(false)
        }
    }
    self.lp.Quit(0)
    return nil

}
```

Key facts: a `Patcher` with backups enabled [final.go:L80]; the write target `filepath.Join(utils.ConfigDir(), "kitty.conf")` [final.go:L81]; the `Patch` call with sentinel `"KITTY_FONTS"`, the serialized settings, and the four font keys to comment out [final.go:L82]; the `--reload-in` switch [final.go:L87-L92]; and `self.lp.Quit(0)` [final.go:L94].

### `serialized()` — the exact bytes written

`serialized()` [kittens/choose_fonts/final.go:L63-L70] joins four lines with `"\n"` (no trailing newline):

```go
func (self faces_settings) serialized() string {
    return strings.Join([]string{
        "font_family      " + self.font_family,
        "bold_font        " + self.bold_font,
        "italic_font      " + self.italic_font,
        "bold_italic_font " + self.bold_italic_font,
    }, "\n")
}
```

Note the byte-exact alignment: each key is padded to a **17-character field** before the value (`font_family` + 6 spaces, `bold_font` + 8, `italic_font` + 6, `bold_italic_font` + 1). The captured `kitty.conf` below reproduces this exactly (value begins at column 18).

### `Patcher.Patch` — the persistence mechanics

`Patcher.Patch` [tools/config/api.go:L310-L350] performs the on-disk mutation:

- **Comment out** any prior `font_family`/`bold_font`/`italic_font`/`bold_italic_font` lines by replacing them with `# $1` via `(?m)^\s*(<keys>)\b` [api.go:L325-L326].
- **Build** the managed block `# BEGIN_KITTY_FONTS\n<content>\n# END_KITTY_FONTS` [api.go:L330].
- **Replace in place** an existing block via `(?ms)^# BEGIN_KITTY_FONTS.+?# END_KITTY_FONTS` [api.go:L328, L331-L334] (idempotent), or **append** it (separated by `\n\n`) if absent [api.go:L335-L340].
- **Write a `.bak`** backup only when the pre-image is non-empty and backups are enabled: `if len(raw) > 0 && self.Write_backup` [api.go:L343-L345].
- **Commit atomically** via `utils.AtomicUpdateFile` [api.go:L347], defined at [tools/utils/atomic-write.go:L79-L89].

The write target honors the config-dir override: `utils.ConfigDir()` is a `sync.OnceValue` wrapping `ConfigDirForName("kitty.conf")` [tools/utils/paths.go:L132-L134], and `ConfigDirForName` returns `Abspath(Expanduser($KITTY_CONFIG_DIRECTORY))` when that variable is set [tools/utils/paths.go:L88-L90]. This is what makes the hermetic experiment in Q4 possible.

### OUTPUT EVIDENCE — before / after / diff / backup

Using a temporary `KITTY_CONFIG_DIRECTORY` and driving the real TUI to the final pane, then pressing Enter:

**Before** (no config file yet):

```
$ ls -la "$KITTY_CONFIG_DIRECTORY"
total 8
drwx--S---  2 root root 4096 Jul  8 04:44 .
drwxrwsrwx 10 root root 4096 Jul  8 04:44 ..
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"
cat: /tmp/cf_cfg.9eVwW1/kitty.conf: No such file or directory (os error 2)
```

**After Enter** (managed block written; note value column 18 matches `serialized()`):

```
$ ls -la "$KITTY_CONFIG_DIRECTORY"
total 12
drwx--S---  2 root root 4096 Jul  8 04:45 .
drwxrwsrwx 10 root root 4096 Jul  8 04:44 ..
-rw-r--r--  1 root root  190 Jul  8 04:45 kitty.conf
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"
# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTS
```

The four `font_*` lines are byte-for-byte what `serialized()` emits, wrapped in the `# BEGIN_KITTY_FONTS` / `# END_KITTY_FONTS` sentinels from `Patch` [api.go:L330]. **No `.bak` exists here** because the pre-image was absent (empty) — exactly the `len(raw) > 0` guard [api.go:L343]. The `.bak` *does* appear once a non-empty pre-image exists — see the idempotency run in Q4, which also shows the before/after `diff`.


---

## Q4 — Is the font choice remembered across restarts? (the runtime example)

**Direct answer: YES.** Pressing Enter writes durable on-disk state to `kitty.conf` [kittens/choose_fonts/final.go:L78-L97 → tools/config/api.go:L310-L350]; it is not a session-only change. A fresh kitty process launched later against the same config directory loads the persisted fonts. Everything below is captured from real runs.

### Why a temporary config dir makes this hermetic and canonical

Setting `KITTY_CONFIG_DIRECTORY` to a throwaway directory redirects the write target away from the developer's real `~/.config/kitty`, because `ConfigDir()` [tools/utils/paths.go:L132-L134] resolves through `ConfigDirForName`, which honors that env var [tools/utils/paths.go:L88-L90]. The experiment therefore observes the *real* code path while remaining fully isolated.

### The before / after / restart experiment

**State 1 — before** (config dir empty; no managed block):

```
$ cat $KITTY_CONFIG_DIRECTORY/kitty.conf
cat: /tmp/cf_cfg.sKyFE0/kitty.conf: No such file or directory (os error 2)
```

**State 2 — immediately after Enter** (drove `kitten choose-fonts`, searched `Fira Code`, Enter through faces, Enter at final). A distinctive family (Fira Code) is chosen deliberately so the restart contrast is unambiguous:

```
FINAL: You have chosen the Fira Code family
exit_after_ENTER: 0
$ ls -la $KITTY_CONFIG_DIRECTORY
total 12
drwx--S---  2 root root 4096 Jul  8 04:48 .
drwxrwsrwx 12 root root 4096 Jul  8 04:48 ..
-rw-r--r--  1 root root  139 Jul  8 04:48 kitty.conf
$ cat $KITTY_CONFIG_DIRECTORY/kitty.conf
# BEGIN_KITTY_FONTS
font_family      family="Fira Code"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
```

The `family="Fira Code"` + three `auto` values are exactly the defaults built by `faces.on_enter` for a family not already configured [kittens/choose_fonts/faces.go:L152-L155] — confirming the real value-flow path produced this file.

**State 3 — after a fresh relaunch** (new processes, same `KITTY_CONFIG_DIRECTORY`):

```
# (a) fresh cat AFTER the finalize process exited -> the block survived process death:
$ cat $KITTY_CONFIG_DIRECTORY/kitty.conf
# BEGIN_KITTY_FONTS
font_family      family="Fira Code"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS

# (b) a BRAND-NEW kitty process resolves fonts from the persisted config via its
#     own startup path (--debug-font-fallback) — i.e. the restart loads Fira Code:
$ KITTY_CONFIG_DIRECTORY="$CFG" kitty --debug-font-fallback bash -c true    (Xvfb)
[0.161] Text fonts:
[0.161]   Normal: FiraCodeRoman-Regular: /root/.local/share/fonts/FiraCode-VF.ttf:131072
[0.161]   Bold: FiraCodeRoman-SemiBold: /root/.local/share/fonts/FiraCode-VF.ttf:262144
[0.161]   Italic: FiraCodeRoman-Regular: /root/.local/share/fonts/FiraCode-VF.ttf:131072
[0.161]   Bold-Italic: FiraCodeRoman-SemiBold: /root/.local/share/fonts/FiraCode-VF.ttf:262144

# (c) CONTRAST — same fresh kitty with DEFAULT settings (--config NONE): NOT Fira Code
$ kitty --config NONE --debug-font-fallback bash -c true    (Xvfb)
[0.159] Text fonts:
[0.159]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.159]   Bold: DejaVuSansMono-Bold: /root/.local/share/fonts/DejaVuSansMono-Bold.ttf:0
[0.159]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.159]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

The only thing that differs between (b) and (c) is the persisted `kitty.conf`. A restart that reads the file gets **FiraCode-VF.ttf**; a restart with default settings gets **DejaVuSansMono**. **The choice is remembered across restarts.**

> Why `--debug-font-fallback`: there is no `--debug-config` flag in this build; `--debug-font-fallback` makes a genuinely fresh kitty process print the font files it resolved at startup, which is the canonical (non-bypassing) way to observe that a new process loaded the persisted config.

### Idempotency — the managed block is replaced in place, never duplicated

Running finalize a second (and third) time against the same file shows the `(?ms)` in-place replacement [tools/config/api.go:L328, L331-L334]. A bonus observation: a fresh `choose-fonts` run **pre-selects the currently-persisted family** (Run 2's final pane says "Fira Code" even though the default cursor was used), proving the config round-trips back into the kitten.

```
----- RUN 2 (idempotency): choose-fonts -> Enter x3 -----
# pre-image is now the non-empty Fira Code block, so a .bak MUST be created this time
FINAL: You have chosen the Fira Code family
exit_after_ENTER: 0
$ ls -la $KITTY_CONFIG_DIRECTORY
total 16
-rw-r--r--  1 root root  397 Jul  8 04:49 kitty.conf
-rw-r--r--  1 root root  139 Jul  8 04:49 kitty.conf.bak
$ grep -c '^# BEGIN_KITTY_FONTS' $KITTY_CONFIG_DIRECTORY/kitty.conf   # exactly 1 => not duplicated
1
$ cat $KITTY_CONFIG_DIRECTORY/kitty.conf.bak    # backup holds the PREVIOUS content
# BEGIN_KITTY_FONTS
font_family      family="Fira Code"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
$ diff $KITTY_CONFIG_DIRECTORY/kitty.conf.bak $KITTY_CONFIG_DIRECTORY/kitty.conf   # before(.bak) vs after
2,5c2,5
< font_family      family="Fira Code"
< bold_font        auto
< italic_font      auto
< bold_italic_font auto
---
> font_family      family='Fira Code' variable_name=FiraCodeRoman style=FiraCodeRoman-Light
> bold_font        family='Fira Code' variable_name=FiraCodeRoman style=FiraCodeRoman-Light
> italic_font      family='Fira Code' variable_name=FiraCodeRoman style=FiraCodeRoman-Light
> bold_italic_font family='Fira Code' variable_name=FiraCodeRoman style=FiraCodeRoman-Light
```

A third run choosing a *different* family (DejaVu Sans Mono) still leaves exactly one region, with the `.bak` now holding the previous Fira Code block:

```
----- RUN 3 (different family): choose-fonts -> search 'DejaVu Sans Mono' -> Enter x3 -----
FINAL: You have chosen the DejaVu Sans Mono family
exit_after_ENTER: 0
$ grep -c '^# BEGIN_KITTY_FONTS' kitty.conf   # STILL exactly 1 (in-place replacement)
1
$ cat kitty.conf
# BEGIN_KITTY_FONTS
font_family      family="DejaVu Sans Mono"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
```

This demonstrates the `.bak` guard (`len(raw) > 0` [api.go:L343]) — the backup appears only from Run 2 onward, once a non-empty pre-image exists.

### `--reload-in` is orthogonal to persistence

`--reload-in` only decides whether *already-running* instances receive `SIGUSR1` to live-reload, via `ReloadConfigInKitty` [tools/config/api.go:L352-L371]: `parent` signals only the `$KITTY_PID` process (`SendSignal(unix.SIGUSR1)` [api.go:L357]); `all` signals every kitty GUI process (`SIGUSR1` [api.go:L366]). Crucially there is **no `none` case** in the finalize switch [kittens/choose_fonts/final.go:L87-L92], so `--reload-in none` **skips live reload but still writes to disk**. Observed:

```
$ cat kitty.conf  (before)
cat: /tmp/cf_cfg.ippgt6/kitty.conf: No such file or directory (os error 2)
----- run: kitten choose-fonts --reload-in none -> Enter x3 -----
FINAL: You have chosen the DejaVu Sans Mono family
exit: 0
$ cat kitty.conf  (after --reload-in none)
# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTS
=> Patch happened regardless of --reload-in.
```

### The non-persisting final actions (contrast — each exercised, each shown)

**`s` / `S`** — `on_text` [kittens/choose_fonts/final.go:L101-L111] sets `output_on_exit = self.settings.serialized() + "\n"` [final.go:L105] and quits; `main()` prints it to STDOUT [kittens/choose_fonts/main.go:L64-L66]. **STDOUT only — nothing written to disk:**

```
----- run: kitten choose-fonts -> Enter,Enter (reach final) -> press 's' -----
FINAL: You have chosen the DejaVu Sans Mono family
exit_after_s: 0
---STDOUT (output_on_exit, serialized settings) BEGIN---
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
---STDOUT END---
$ ls -la $KITTY_CONFIG_DIRECTORY   (nothing written: no kitty.conf, no .bak)
total 8
drwx--S---  2 root root 4096 Jul  8 04:52 .
drwxrwsrwx 13 root root 4096 Jul  8 04:52 ..
$ cat $KITTY_CONFIG_DIRECTORY/kitty.conf
cat: /tmp/cf_cfg.V68GxE/kitty.conf: No such file or directory (os error 2)
```

**Esc** — `on_key_event` [kittens/choose_fonts/final.go:L73-L77] sets `current_pane = &self.handler.faces` and redraws — it returns to the faces pane, **no write:**

```
FINAL: You have chosen the DejaVu Sans Mono family
AFTER-ESC pane shows faces?: True ['Press Enter to select this font, Esc to go back to the font list ...']
$ ls -la $KITTY_CONFIG_DIRECTORY   (no kitty.conf written)
total 8
$ cat kitty.conf
cat: /tmp/cf_cfg.7wYWXQ/kitty.conf: No such file or directory (os error 2)
```

**Ctrl+c** — there is **no explicit `ctrl+c` handler** in `final.go` (`on_key_event` matches only `esc`/`enter`; `on_text` only `s`/`S`). It therefore falls through to the event loop's default quit (**this default-quit routing is `inferred` from the absence of a handler**; the **no-write result is observed**):

```
FINAL: You have chosen the DejaVu Sans Mono family
exit: 1
$ ls -la $KITTY_CONFIG_DIRECTORY   (no kitty.conf written)
total 8
$ cat kitty.conf
cat: /tmp/cf_cfg.bHAAhK/kitty.conf: No such file or directory (os error 2)
```

### Q4 summary

| Action at final screen | On-disk effect | Evidence |
|------------------------|----------------|----------|
| **Enter** | Writes/updates managed block in `kitty.conf` (+`.bak` if pre-image non-empty); **persists across restarts** | States 1/2/3, Fira Code restart contrast |
| **Enter, run again** | Block replaced **in place** (exactly one region); `.bak` holds prior content | Runs 2 & 3, `grep -c` = 1 |
| `--reload-in none` + Enter | **Still persists** (only live-reload skipped) | `--reload-in none` transcript |
| **`s`/`S`** | STDOUT only — **no disk write** | `s` transcript, empty dir |
| **Esc** | Returns to faces — **no write** | Esc transcript, empty dir |
| **Ctrl+c** | Quits — **no write** | Ctrl+c transcript, empty dir |


---

## Architecture note — Go TUI frontend ↔ Python enumeration backend

`choose-fonts` is a two-process feature. The **Go** frontend (the kitten) drives the TUI and owns the persistence path; a **Python** backend performs the actual font enumeration and sample rendering.

- The frontend spawns the backend by executing the `kitty` binary in `+runpy` mode: `start()` [kittens/choose_fonts/backend.go:L32-L63] runs `exec.Command(exe, "+runpy", "from kittens.choose_fonts.backend import main; main()")` [kittens/choose_fonts/backend.go:L41], where `exe` comes from `KittyExe()` [backend.go:L33].
- The two processes communicate over **newline-delimited JSON** across `os.Pipe` pipes [backend.go:L44, L48], decoded with `json.NewDecoder(k.from)` [backend.go:L52].
- The Python `main()` [kittens/choose_fonts/backend.py:L150-L168] reads stdin line-by-line (`for line in sys.stdin.buffer:` [backend.py:L153]) and dispatches actions: `list_monospaced_fonts` [backend.py:L156], `read_variable_data` [backend.py:L159], `render_family_samples` [backend.py:L164].
- Shared wire types live in [kittens/choose_fonts/types.go]: `VariableAxis` [types.go:L11], `ResolvedFace` [types.go:L73], `ResolvedFaces` [types.go:L78], `ListResult` [types.go:L85], `RenderedSampleTransmit` [types.go:L90]. The backend consumes kitty's font subsystem (`kitty/fonts/common.py`, `list.py`, `render.py`, `fontconfig.py`, `core_text.py`) for enumeration and rendering.

This architecture is *secondary* to the persistence question: the **finalize/persistence logic is entirely in the Go frontend** (`final.go` + `tools/config/api.go`); the Python backend only supplies the family list and previews shown in the listing/faces panes.

---

## Final coverage pass

Each named item, with its value, `file:line`, observed evidence, sibling variants, and causal reason. Items that could only be read from source (not observed at runtime) are marked **inferred**.

### Q1 — Build & default launch
- **Build command:** `./dev.sh build` → `exec go run bypy/devenv.go "$@"` [dev.sh:L9]; siblings `make` [Makefile:L12-L13] / `python3 setup.py`. Evidence: `Build successful. Run kitty as: kitty/launcher/kitty`. Reason: dev launcher wraps the Go builder.
- **Versions:** Python `>=3.8` [pyproject.toml:L2], Go `1.22` [go.mod:L3]; observed `go1.22.12`, `Python 3.13.7`. Reason: manifest floors vs container actuals.
- **Version banner:** `kitty 0.35.2 created by Kovid Goyal` — matches `Version(0, 35, 2)` [kitty/constants.py:L25]. Evidence: `--version` output.
- **Default launch:** `kitty/launcher/kitty --config NONE`; `NONE` semantics [kitty/cli.py:L183, L187-L188], `KITTY_CONFIG_DIRECTORY` [kitty/cli.py:L196-L197]. Evidence: headless Xvfb launch, exit 0.

### Q2 — Invocation
- **Command:** `kitten choose-fonts` (or `kitty/launcher/kitten choose-fonts`). Registration [tools/cmd/tool/main.go:L82]; kitten built to launcher [setup.py:L1160]; `KittyExe` resolution [tools/utils/paths.go:L69-L86]. Evidence: `--help` output.
- **TUI progression:** listing → faces → final. Evidence: three captured panes.

### Q3a — Registration & option parsing
- **Registration:** `EntryPoint` [main.go:L74-L99]; `AddSubCommand` name/desc [main.go:L75-L77].
- **`Run`/parse:** `opts := Options{}` → `GetOptionValues(&opts)` → `main(&opts)` [main.go:L78-L84].
- **`Options` struct:** single field `Reload_in string` [main.go:L70-L72].
- **`--reload-in` (every value):** `OptionSpec` [main.go:L86-L95]; `parent` (default) → `ReloadConfigInKitty(true)` [final.go:L88-L89]; `all` → `ReloadConfigInKitty(false)` [final.go:L90-L91]; `none` → no case, still persists [final.go:L87-L92]. Evidence: `--help`, variant `--help`s, invalid-value rejection. Reason: signal scope for live reload of running instances.
- **Clone alias:** `choose_fonts`, `clone.Hidden = false` [main.go:L96-L98]. Evidence: `kitten choose_fonts --help` resolves; both names appear in `kitten --help`. Reason: underscore alias intentionally visible (contrary to any "hidden clone" prose).

### Q3b — Value flow
- **`handler.opts`:** `h := &handler{lp: lp, opts: opts}` [main.go:L35]; handler holds `opts` [ui.go:L43].
- **State machine:** `State` [ui.go:L17], consts [ui.go:L19-L23], `pane` iface [ui.go:L33-L40], handler panes [ui.go:L42-L62].
- **listing→faces:** Enter → `faces.on_enter(family)` [list.go:L246-L254, call L250].
- **faces_settings:** struct [faces.go:L14-L16]; built in `on_enter` [faces.go:L145-L159, defaults L152-L155].
- **faces→final:** Enter → `final_pane.on_enter(family, settings)` [faces.go:L112-L123, call L120]; stored [final.go:L113-L118].
- **Evidence:** final pane names the chosen family (DejaVu / Fira Code). Reason: `opts.Reload_in` consumed only at finalize.

### Q3c — Finalization
- **Enter:** `Patcher{Write_backup: true}` [final.go:L80], `path` [final.go:L81], `Patch(...)` [final.go:L82], `Quit(0)` [final.go:L94].
- **`serialized()`:** four `font_*` lines, 17-char key field, joined by `\n` [final.go:L63-L70]. Evidence: on-disk bytes (value col 18).
- **`Patcher.Patch`:** comment-out [api.go:L325-L326], block build [api.go:L330], in-place replace [api.go:L328, L331-L334], append [api.go:L335-L340], `.bak` guard [api.go:L343-L345], `AtomicUpdateFile` [api.go:L347] / [tools/utils/atomic-write.go:L79-L89].
- **Managed block + `.bak`:** Evidence: before/after `kitty.conf`, `.bak` from Run 2, `diff`.

### Q4 — Persistence across restarts
- **Persistence = YES.** Enter → disk write [final.go:L78-L97 → api.go:L310-L350]. Evidence: State 1/2/3, fresh-process Fira Code vs `--config NONE` DejaVu contrast.
- **Temp-dir mechanism:** `ConfigDir` [paths.go:L132-L134] → `ConfigDirForName` honors `KITTY_CONFIG_DIRECTORY` [paths.go:L88-L90]. Evidence: writes land in the temp dir.
- **before/after/restart:** captured (States 1/2/3).
- **Idempotency:** in-place replace, one region, `.bak` holds prior [api.go:L328, L331-L334]. Evidence: Runs 2 & 3, `grep -c` = 1.
- **`--reload-in` ↔ persistence separation:** `ReloadConfigInKitty` [api.go:L352-L371]; no `none` case [final.go:L87-L92]. Evidence: `--reload-in none` still patched.
- **`s`/Esc/Ctrl+c non-persistence:** `s` → STDOUT via `output_on_exit` [final.go:L101-L111, L105; main.go:L64-L66]; Esc → faces [final.go:L73-L77]; Ctrl+c → default-quit (**inferred** from absence of handler), no-write **observed**. Evidence: three transcripts, empty config dirs.

### Explicitly labeled inferences
- The **Ctrl+c** finalize routing to the loop's default quit is **inferred** from the absence of a `ctrl+c` branch in `final.go` (`on_key_event` handles only `esc`/`enter`; `on_text` only `s`/`S`). The **outcome** (process quits, nothing written to `kitty.conf`) is **observed** (exit 1, empty config dir).
- The `parent`/`all` `SIGUSR1` *delivery* is read from `ReloadConfigInKitty` [tools/config/api.go:L352-L371] (**inferred** from source); what is **observed** at runtime is that the on-disk `Patch` occurs for `parent`, `all`, and `none` alike.

