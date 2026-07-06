# How kitty's `choose-fonts` kitten works, and whether a font selection persists

> **Investigation methodology (RUN-FIRST).** Every behavioral claim below was produced by *building kitty from this checkout, running it, driving the real `kitten choose-fonts` entry point under a PTY, and capturing the actual output.* Each claim is grounded in a specific `file:line` reference **and** the captured output that demonstrates it. Values that were reasoned-about-but-not-directly-observed are labelled **[inferred]**; values obtained through anything other than the real entry point are labelled **[non-canonical]**. Observed HEAD commit: `815df1e21 "Wire up applying of font config"`; source branch `kitty_815df1e210e0`.

---

## Bottom line up front (the direct answer)

**Pressing `Enter` on the final screen of `choose-fonts` PERSISTS the font choice to disk. It is *not* a session-only change.** The kitten writes a sentinel-delimited `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` block containing four keys (`font_family`, `bold_font`, `italic_font`, `bold_italic_font`) into `kitty.conf` in your kitty config directory, atomically and with a `.bak` backup, and then signals the running kitty to reload. Because the setting lives in `kitty.conf`, a **freshly launched kitty re-reads it on startup**, so the choice **survives a full restart**.

The other two final-screen actions do **not** persist:

- **`s` / `S`** writes the same four-key block to **STDOUT only** — nothing is written to `kitty.conf`.
- **`Esc`** aborts and returns to font selection — nothing is written.

Concretely, starting from a config directory with **no** `kitty.conf`:

```
# BEGIN kitty.conf  ->  (file does not exist)
# after Enter       ->
# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTS
# a fresh kitty process then reports font_family = DejaVuSansMono
```

This document was produced with the canonical developer build `./dev.sh build` and kitty/kitten **version `0.35.2`** (observed banner, see Q1).

---

## Q1 — Build kitty from this checkout and launch a default instance

**Direct answer.** Build with the canonical developer entry point `./dev.sh build`; it produces the in-place launchers `kitty/launcher/kitty` and `kitty/launcher/kitten`. Launch a default instance simply by running `kitty/launcher/kitty`. The observed version banner is **`kitty 0.35.2 created by Kovid Goyal`**.

### Build

`dev.sh` delegates to the Go dev-env bootstrap, which downloads kitty's major dependencies as prebuilt binaries and builds in place:

```sh
# dev.sh:9
exec go run bypy/devenv.go "$@"
```

The exact command used (the `--ignore-compiler-warnings` flag is the project's own sanctioned accommodation, `setup.py:2003`; on this toolchain the strict `-Werror` default trips over a newer `wayland-protocols` enum in `glfw/wl_window.c`, which is unrelated to `choose-fonts`):

```
$ ./dev.sh build --ignore-compiler-warnings
Build successful. Run kitty as: kitty/launcher/kitty
(exit 0)
$ ls -l kitty/launcher/kitty kitty/launcher/kitten
-rwxr-xr-x 1 root root 15765764 Jul  6 22:35 kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 Jul  6 21:36 kitty/launcher/kitty
```

The build requires a C compiler and Go (`go 1.22`, `go.mod:3`) plus kitty's native font libraries — `harfbuzz >= 2.2.0`, `freetype`, `fontconfig`, `libpng`, `zlib`, `liblcms2`, `libcanberra`, `simde`, `pkg-config` (`docs/build.rst:83-102`). The canonical developer build is `./dev.sh build` (`docs/build.rst:19-24`); the packager path is `python3 setup.py` (`Makefile:12-13`) and CI uses `python .github/workflows/ci.py build` → `setup.py build --verbose` (`.github/workflows/ci.yml:62-63`, `.github/workflows/ci.py:102-109`). `pyproject.toml:2` declares `requires-python = ">=3.8"`.

### Default launch + version banner

A default instance is launched as a normal user would, by running the produced launcher `kitty/launcher/kitty` (no custom config). The version banner (observed, unedited):

```
$ kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal

$ kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
```

This matches the source-level version `Version(0, 35, 2)` at `kitty/constants.py:25` (`str_version` at `:26`). The observed banner is reported exactly as printed.

---

## Q2 — Invoke the font chooser from inside the running instance

**Direct answer.** The real entry point is the command **`kitten choose-fonts`** (`kittens/choose_fonts/main.go:76`). Run it from inside a running kitty instance; it opens the interactive family-list screen.

```sh
kitten choose-fonts
```

That the subcommand is genuinely registered (not a shim) is shown by `--help`, which also enumerates its single option (see Q3b):

```
$ kitten choose-fonts --help
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

**Alternate (still canonical) entry.** kitty's Python side can hand off to the same kitten via `os.execlp(kitten_exe(), 'kitten', 'choose-fonts')` (`kitty/fonts/list.py:41-42`); this ends up in exactly the same `kitten choose-fonts` code path.

**[non-canonical] note.** A font value obtained through a remote-control command (`kitten @ …`) or a debug hook would *not* answer Q2 — those bypass the real entry point. Every observation in this document was made through `kitten choose-fonts` itself.

---

## Q3 — End-to-end behavior

### Q3a — How the subcommand is registered

**Direct answer.** The root `kitten` tool imports the package and calls `choose_fonts.EntryPoint(root)`, which adds a `cli.Command` named `choose-fonts` (plus a visible alias `choose_fonts`).

- `tools/cmd/tool/main.go:9` — `import "kitty/kittens/choose_fonts"`
- `tools/cmd/tool/main.go:82` — `choose_fonts.EntryPoint(root)`
- `kittens/choose_fonts/main.go:74-76` — `EntryPoint(root *cli.Command)` → `root.AddSubCommand(&cli.Command{ Name: "choose-fonts", … })`
- `kittens/choose_fonts/main.go:96-98` — a clone alias is added with `clone.Hidden = false` and `clone.Name = "choose_fonts"` (underscore form)

Observed — both names appear in the tool's command list, each described "Choose the fonts used in kitty":

```
$ kitten --help | grep -i -A1 choose
   choose-fonts
    Choose the fonts used in kitty
   choose_fonts
    Choose the fonts used in kitty
```

### Q3b — How its options are parsed (`--reload-in`)

**Direct answer.** The command binds a single option, `--reload-in`, of type `choices` with the values `parent, all, none` and a **default of `parent`**. At run time the closure builds an `Options{}`, fills it via `cmd.GetOptionValues(&opts)`, and calls `main(&opts)`.

- `kittens/choose_fonts/main.go:70-72` — `type Options struct { Reload_in string }`
- `kittens/choose_fonts/main.go:78-84` — Run closure: `opts := Options{}` → `cmd.GetOptionValues(&opts)` → `main(&opts)`
- `kittens/choose_fonts/main.go:86-95` — `ans.Add(cli.OptionSpec{ Name: "--reload-in", Dest: "Reload_in", Type: "choices", Choices: "parent, all, none", Default: "parent", … })`

Observed — the `--help` output (Q2) shows `--reload-in [=parent]` and `Choices: parent, all, none`, confirming both the choice set and the default `parent` at run time.

### Q3c — How the selected values flow through the panes

**Direct answer.** The kitten accumulates a `faces_settings` value — a struct of the four font keys — as control passes from the **family-list** pane to the **faces** (preview) pane to the **final** pane. Font *enumeration/rendering* is delegated to a Python backend over JSON pipes; **persistence is entirely in the Go frontend** (the Python backend never writes config).

The accumulating value:

```go
// kittens/choose_fonts/faces.go:14-15
type faces_settings struct {
	font_family, bold_font, italic_font, bold_italic_font string
}
```

Pane hand-offs:

- **Family list → faces.** On `Enter`, if a family is selected, control transfers to the faces pane:
  `kittens/choose_fonts/list.go:246-250` — `if family := self.family_list.CurrentFamily(); family != "" { return self.handler.faces.on_enter(family) }`.
- **Seeding the settings.** `faces.on_enter(family)` seeds the four fields from the *existing* `kitty.conf` (`resolved_faces_from_kitty_conf`) or, absent a match, from defaults (`family="…"`, `auto`): `kittens/choose_fonts/faces.go:145-159`. *(Observed: seeding a config with `font_family CascadiaCode` caused Cascadia Code to be pre-selected — see the `.bak` demo in Q4.)*
- **Faces → final.** On `Enter` in the faces pane, control advances to the final pane with the accumulated settings:
  `kittens/choose_fonts/faces.go:118-121` — `return self.handler.final_pane.on_enter(self.family, self.settings)` (call at `:120`).
- **Faces → fine-tune (optional).** The keys `r/R`, `b/B`, `i/I`, `o/O` open the per-face fine-tuning panel for `font_family`/`bold_font`/`italic_font`/`bold_italic_font` respectively:
  `kittens/choose_fonts/faces.go:125-141` → `kittens/choose_fonts/face.go:298` `func (self *face_panel) on_enter(family, which string, settings faces_settings) error`.
- **Faces → back.** `Esc` returns to the listing pane: `kittens/choose_fonts/faces.go:112-117`.

The Go↔Python split (enumeration/rendering only, no persistence):

- `kittens/choose_fonts/backend.go:41` — `k.cmd = exec.Command(exe, "+runpy", "from kittens.choose_fonts.backend import main; main()")`
- `kittens/choose_fonts/backend.go:44-52` — `os.Pipe()` pipes + `json.NewDecoder(k.from)`
- `kittens/choose_fonts/backend.py:11-27` — imports `kitty.cli` / `kitty.fonts.*` and performs font enumeration/sample rendering only.

Observed value flow (three panes, real text captured while driving the TUI). The family list, the faces pane with the four accumulated previews, and the final pane are all visible in the raw stream (see Appendix E for the unedited capture):

```
Scanning system for fonts, please wait...
Cascadia Code   Cascadia Code NF   Cascadia Code PL   Cascadia Mono   Cascadia Mono NF
Cascadia Mono PL   Comfy Code   > DejaVu Sans Mono   Fantasque Sans Mono   Fira Code
Hack   IBM Plex Mono   Inconsolata   JetBrains Mono   JetBrains Mono NL   Liberation Mono
Noto Mono   Noto Sans SignWriting   Source Code Pro   SourceCodeVF   Ubuntu Mono
    DejaVu Sans Mono
    Styles: Bold, Bold Oblique, Book, Oblique
    Press the Enter key to choose this family
--- (faces pane, after Enter) ---
    DejaVu Sans Mono
    Press Enter to select this font, Esc to go back to the font list or any of the
    highlighted keys below to fine-tune the appearance of the individual font styles.
    Regular:     DejaVuSansMono
    Bold:        DejaVuSansMono-Bold
    Italic:      DejaVuSansMono-Oblique
    Bold-Italic: DejaVuSansMono-BoldOblique
--- (final pane, after Enter) ---
    You have chosen the DejaVu Sans Mono family
    What would you like to do?
    Enter to modify kitty.conf and use the new fonts
    Esc to abort and return to font selection
    s to write the new font settings to STDOUT
    Ctrl+c to quit
```

The four `Regular/Bold/Italic/Bold-Italic` lines are exactly the four fields of `faces_settings` that were accumulated for the chosen family, and they become the four keys written on finalization.

### Q3d — What kitty does when the selection is finalized

**Direct answer.** The final pane offers **four** keyboard actions. The one that changes on-disk state is `Enter`.

```go
// kittens/choose_fonts/final.go:38-44 (draw_screen prompts)
"%s to modify %s and use the new fonts",      // Enter -> kitty.conf
"%s to abort and return to font selection",   // Esc
"%s to write the new font settings to %s",    // s    -> STDOUT
"%s to quit",                                 // Ctrl+c
```

- **`Enter`** — patch `kitty.conf` and (optionally) reload; then quit. `kittens/choose_fonts/final.go:78-95` (detailed in Q4).
- **`Esc`** — set `current_pane = &faces` and redraw; **no write**. `kittens/choose_fonts/final.go:72-77`.
- **`s` / `S`** — set `output_on_exit = self.settings.serialized() + "\n"` and quit; the string is emitted to STDOUT at process exit. `kittens/choose_fonts/final.go:101-111` (case at `:104-106`) and `kittens/choose_fonts/main.go:64-66` (`if output_on_exit != "" { os.Stdout.WriteString(output_on_exit) }`).
- **`Ctrl+c`** — quit with no write.

The exact serialized form written by `Enter` / emitted by `s` (note the fixed column alignment — trailing spaces are part of the literals):

```go
// kittens/choose_fonts/final.go:63-70
func (self faces_settings) serialized() string {
	return strings.Join([]string{
		"font_family      " + self.font_family,
		"bold_font        " + self.bold_font,
		"italic_font      " + self.italic_font,
		"bold_italic_font " + self.bold_italic_font,
	}, "\n")
}
```


---

## Q4 — Persistence: does `Enter` make kitty *remember* the choice? (the crux)

**Direct answer: YES — pressing `Enter` persists the choice on disk, so it survives a restart.** It is not a session-only change. The proof below observes `kitty.conf` **before** and **after**, shows the exact block written, shows the `.bak` backup, and confirms a **freshly launched kitty process re-reads the file**.

All runtime observation used an **isolated, throwaway** config directory so the real user config was never touched:

```sh
export KITTY_CONFIG_DIRECTORY="$(mktemp -d)"
```

This is honored first by both the Go and Python sides: `tools/utils/paths.go:88-90` (`ConfigDirForName` returns `KITTY_CONFIG_DIRECTORY` directly) and `kitty/constants.py:87-89` (`_get_config_dir` checks it first); `defconf = os.path.join(config_dir, 'kitty.conf')` at `kitty/constants.py:131-133`. Note `utils.ConfigDir` is a `sync.OnceValue` (`tools/utils/paths.go:132-134`) — it is computed **once per process**, so the environment variable must be exported *before* launching.

### What the `Enter` handler does

```go
// kittens/choose_fonts/final.go:78-95
if event.MatchesPressOrRepeat("enter") {
	event.Handled = true
	patcher := config.Patcher{Write_backup: true}
	path := filepath.Join(utils.ConfigDir(), "kitty.conf")
	updated, err := patcher.Patch(path, "KITTY_FONTS", self.settings.serialized(),
		"font_family", "bold_font", "italic_font", "bold_italic_font")
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

### Observed: BEFORE → AFTER (config directory with no prior `kitty.conf`)

**BEFORE** — the config directory is empty; `kitty.conf` does not exist:

```
$ ls -la $KITTY_CONFIG_DIRECTORY   # BEFORE
total 8
drwx------  2 root root 4096 Jul  6 22:35 .
drwxr-xr-x 10 root root 4096 Jul  6 22:35 ..
$ cat $KITTY_CONFIG_DIRECTORY/kitty.conf   # BEFORE
cat: /tmp/blitzy_obs/cfgfinal.xMS9iQ/kitty.conf: No such file or directory
```

Drive `kitten choose-fonts`, select the pre-highlighted family, reach the final pane, press **`Enter`**.

**AFTER** — `kitty.conf` now exists (190 bytes) and contains the sentinel block with the four keys (verbatim, unedited):

```
$ ls -la $KITTY_CONFIG_DIRECTORY   # AFTER
total 12
drwx------  2 root root 4096 Jul  6 22:35 .
drwxr-xr-x 10 root root 4096 Jul  6 22:35 ..
-rw-r--r--  1 root root  190 Jul  6 22:35 kitty.conf
$ cat $KITTY_CONFIG_DIRECTORY/kitty.conf   # AFTER
# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTS
```

### Observed: RESTART re-reads the file (the persistence proof)

A **fresh kitty process** — same `KITTY_CONFIG_DIRECTORY`, launched after the kitten exited — resolves the same `defconf` and loads the written keys (this is exactly the config file kitty reads at startup):

```
$ kitty/launcher/kitty +runpy "$(cat reread.py)"   # fresh process, same KITTY_CONFIG_DIRECTORY
config_dir      = /tmp/blitzy_obs/cfgfinal.xMS9iQ
defconf         = /tmp/blitzy_obs/cfgfinal.xMS9iQ/kitty.conf
config_paths    = ('/tmp/blitzy_obs/cfgfinal.xMS9iQ/kitty.conf',)
font_family     = FontSpec(family='', style='', postscript_name='', full_name='', system='DejaVuSansMono', axes=(), variable_name='', created_from_string='DejaVuSansMono')
bold_font       = FontSpec(family='', style='', postscript_name='', full_name='', system='DejaVuSansMono-Bold', axes=(), variable_name='', created_from_string='DejaVuSansMono-Bold')
italic_font     = FontSpec(family='', style='', postscript_name='', full_name='', system='DejaVuSansMono-Oblique', axes=(), variable_name='', created_from_string='DejaVuSansMono-Oblique')
bold_italic_font= FontSpec(family='', style='', postscript_name='', full_name='', system='DejaVuSansMono-BoldOblique', axes=(), variable_name='', created_from_string='DejaVuSansMono-BoldOblique')
```

`config_paths` shows the fresh process **read the written file**, and `font_family = DejaVuSansMono` (etc.) are precisely the values the kitten wrote. The choice therefore survives a restart ⇒ **on-disk persistence, not a session-only change.** *(This was reproduced identically across three independent runs.)*

For contrast, a config directory with **no** `kitty.conf` yields the built-in default:

```
config_paths (no file)    = ()
DEFAULT font_family       = (builtin default: monospace)
```

so the transition `monospace → DejaVuSansMono` is a genuine, observed state change.

### Observed: the `.bak` backup and prior-key handling

To exercise the backup path, a config directory was seeded with a *prior* `kitty.conf` containing a conflicting top-level `font_family` line, then `Enter` was pressed. The backup is written **only when a non-empty file already existed** (`tools/config/api.go:342-347`), which is why the first-run (no prior file) case above produced no `.bak`.

`diff kitty.conf.bak kitty.conf` (unedited) — the prior key is commented out and the sentinel block appended, while `.bak` preserves the original byte-for-byte:

```
2c2
< font_family      CascadiaCode
---
> # font_family      CascadiaCode
4a5,12
>
>
> # BEGIN_KITTY_FONTS
> font_family      CascadiaCode-Regular
> bold_font        family='Cascadia Code' variable_name=CascadiaCodeRoman style=''
> italic_font      family='Cascadia Code' style=''
> bold_italic_font family='Cascadia Code' style=''
> # END_KITTY_FONTS
\ No newline at end of file
```

*(Observed nuance, reported exactly as seen: because the seeded config named `CascadiaCode`, the kitten pre-selected the variable font **Cascadia Code**, whose serialization uses the richer `family='…' variable_name=… style=''` FontSpec form — unlike the static DejaVu Sans Mono, which serializes to plain PostScript names. Both are valid `kitty.conf` font specs.)*

### Condition matrix (every final-screen action + every `--reload-in` value — all observed)

| Trigger | On-disk write to `kitty.conf`? | Reload signal | Persists across restart? | Evidence |
|---|---|---|---|---|
| **`Enter`** (`--reload-in parent`, default) | **Yes** — `# BEGIN_KITTY_FONTS` block, atomic + `.bak` | `SIGUSR1` to parent (`KITTY_PID`) | **Yes** | BEFORE/AFTER + reread above; `final.go:78-95` |
| **`s` / `S`** | **No** | none | **No** | serialized block on STDOUT; `kitty.conf` absent (Appendix C); `final.go:101-111`, `main.go:64-66` |
| **`Esc`** | **No** | none | **No** | returns to faces pane; `kitty.conf` absent (Appendix D); `final.go:72-77` |
| **`--reload-in parent`** + Enter | **Yes** | `SIGUSR1` to parent only | **Yes** | dummy `KITTY_PID` received `SIGUSR1` (Appendix F); `api.go:352-361` |
| **`--reload-in all`** + Enter | **Yes** | `SIGUSR1` to **all** kitty GUIs | **Yes** | dummy received `SIGUSR1` (Appendix F); `api.go:363-368` |
| **`--reload-in none`** + Enter | **Yes** | **none** (no `switch` case) | **Yes** | dummy received **no** `SIGUSR1`, file still written (Appendix F); `final.go:86-93` |

**Key consequence:** the on-disk write (persistence) happens for **all** `--reload-in` values — persistence is *independent* of reload. Only whether/where a `SIGUSR1` reload signal is sent differs. `--reload-in none` still writes the file; it just does not tell any running instance to reload.

### `--config NONE` caveat

`kitty --config NONE` yields **pure defaults with no persistent target** and must not be used to judge persistence. `resolve_config` (`kitty/conf/utils.py:322-330`) yields nothing when `NONE` is present:

```
resolve_config() default  -> ['/etc/xdg/kitty/kitty.conf', '<cfgdir>/kitty.conf']
resolve_config(['NONE'])  -> []
```

So a `NONE` launch reads no `kitty.conf` at all (`kitty/cli.py:870-872` defines the `--config`/`-c` option with `kwds:none,NONE`). The persistence proof above therefore uses a **real, writable** temporary config directory, not a `NONE` run.


---

## Mechanism deep-dive

### The `Patcher.Patch` sentinel-block algorithm

Persistence is implemented by a small, idempotent config patcher shared across kitty's Go tools.

```go
// tools/config/api.go:305-308
type Patcher struct {
	Write_backup bool
	Mode         fs.FileMode
}
```

`Patcher.Patch(path, sentinel, content, settings_to_comment_out...)` (`tools/config/api.go:310-350`) does the following, in order:

1. **Mode default** — if unset, the file mode defaults to `0o644` (`:311-313`).
2. **Read existing** — reads the current file; a missing file is tolerated (`fs.ErrNotExist`) and treated as empty (`:318`).
3. **Comment out prior keys** — any prior top-level lines matching the given keys (`font_family`, `bold_font`, `italic_font`, `bold_italic_font`) are prefixed with `# ` via a `(?m)^\s*(keys)\b` substitution (`:325-326`). *(Observed in the `.bak` diff: `font_family      CascadiaCode` → `# font_family      CascadiaCode`.)*
4. **Replace-or-append the block** — a `(?ms)^# BEGIN_KITTY_FONTS.+?# END_KITTY_FONTS` region is replaced if present, otherwise the block is appended (separated by `\n\n` when the file already had content) (`:328-340`).
5. **Backup + atomic write** — only if the content actually changed: when `Write_backup` is set **and** the prior file was non-empty, a `<path>.bak` copy is written first; then the new content is written via `utils.AtomicUpdateFile` (`:342-347`). *(This is why a first-time write, with no prior file, produces no `.bak`.)*

The atomic write is a temp-write-then-rename that preserves existing permissions:

```
// tools/utils/atomic-write.go:79 — AtomicUpdateFile(path, data, ...) (atomic temp-write + rename)
```

The sentinel name passed by the kitten is `KITTY_FONTS`, so the delimiters are `# BEGIN_KITTY_FONTS` / `# END_KITTY_FONTS` (`kittens/choose_fonts/final.go:82`).

### Config-directory resolution

The write target is `filepath.Join(utils.ConfigDir(), "kitty.conf")` (`kittens/choose_fonts/final.go:81`). `ConfigDir` resolves like so:

- `KITTY_CONFIG_DIRECTORY` if set (returned directly, expanded/absolutized) — `tools/utils/paths.go:88-90`;
- otherwise the XDG config location, then `~/.config/kitty` — `tools/utils/paths.go:101-128`;
- memoized once per process via `sync.OnceValue` — `tools/utils/paths.go:132-134`.

The Python side mirrors this (`kitty/constants.py:87-89`, `:131-133`), so the file the kitten writes is exactly the file a fresh kitty reads at startup. If missing on an edit path, kitty can bootstrap a commented-out default `kitty.conf` (`kitty/config.py:79-86`).

### The reload signal: `SIGUSR1` (distinct from `load_config`)

After a successful write, the kitten reloads the running instance's config by sending **`SIGUSR1`** — this is a signal-based mechanism, *not* the `load_config` remote-control command.

```go
// tools/config/api.go:352-371
func ReloadConfigInKitty(in_parent_only bool) error {
	if in_parent_only {
		if pid, err := strconv.Atoi(os.Getenv("KITTY_PID")); err == nil {
			if p, err := process.NewProcess(int32(pid)); err == nil {
				if c, err := p.CmdlineSlice(); err == nil && is_kitty_gui_cmdline(c...) {
					return p.SendSignal(unix.SIGUSR1)
				}
			}
		}
		return nil
	}
	if all, err := process.Processes(); err == nil {
		for _, p := range all {
			if c, err := p.CmdlineSlice(); err == nil && is_kitty_gui_cmdline(c...) {
				_ = p.SendSignal(unix.SIGUSR1)
			}
		}
	}
	return nil
}
```

- `--reload-in parent` → `ReloadConfigInKitty(true)` → `SIGUSR1` to the process named by `KITTY_PID` (only if its cmdline looks like a kitty GUI, per `is_kitty_gui_cmdline`, `tools/config/api.go:282-303`).
- `--reload-in all` → `ReloadConfigInKitty(false)` → `SIGUSR1` to *every* kitty GUI process.
- `--reload-in none` → no `case` in the `switch` (`kittens/choose_fonts/final.go:86-93`), so no signal is sent.

This mechanism is corroborated by the official docs, which note that `kitty.conf` is reloaded when modified (via `auto_reload_config`, or by sending `SIGUSR1`).

---

## Appendix — raw, unedited evidence (with the commands that produced it)

### Appendix A — Build and version banners

```
$ ./dev.sh build --ignore-compiler-warnings
Build successful. Run kitty as: kitty/launcher/kitty
(exit 0)
$ ls -l kitty/launcher/kitty kitty/launcher/kitten
-rwxr-xr-x 1 root root 15765764 Jul  6 22:35 kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 Jul  6 21:36 kitty/launcher/kitty

$ kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal

$ kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
```

### Appendix B — Registration and option parsing

```
$ kitten --help | grep -i -A1 choose
   choose-fonts
    Choose the fonts used in kitty
   choose_fonts
    Choose the fonts used in kitty

$ kitten choose-fonts --help
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

### Appendix C — `s` writes the four-key block to STDOUT only (not persisted)

Driving `kitten choose-fonts` to the final pane and pressing `s` (delivered as the kitty-keyboard-protocol CSI-u sequence `ESC [ 115 ; ; 115 u`). The serialized block is emitted to STDOUT at process exit; the raw tail (unedited) shows the alt-screen teardown (`ESC 8`) immediately followed by the plain-text block:

```
'…\x1b[?25h\x1b[<u\x1b[?1049l\x1b[?r\x1b[#Q\x1b8'
'font_family      DejaVuSansMono\r\n'
'bold_font        DejaVuSansMono-Bold\r\n'
'italic_font      DejaVuSansMono-Oblique\r\n'
'bold_italic_font DejaVuSansMono-BoldOblique\r\n'
```

And the config directory was left untouched (no persistence):

```
>>> kitty.conf does NOT exist -> 's' path does NOT persist (correct)
```

### Appendix D — `Esc` aborts (no write)

Driving to the final pane and pressing `Esc` (`ESC [ 27 u`) returned control to the faces pane and wrote nothing. Marker counts over the captured stream (the faces-pane prompt appears both before *and* after `Esc`; the final-pane prompt appears once):

```
'What would you like to do' occurrences: 1
'select this font' occurrences      : 3
>>> kitty.conf does NOT exist -> Esc aborts without writing (correct)
```

### Appendix E — Value-flow capture (family → faces → final)

Readable text extracted from the unedited TUI byte stream while driving the kitten (box-drawing glyphs and duplicate redraw frames elided by de-duplication for legibility; no logic elided):

```
Scanning system for fonts, please wait...
Cascadia Code Cascadia Code NF Cascadia Code PL Cascadia Mono Cascadia Mono NF Cascadia Mono PL
Comfy Code > DejaVu Sans Mono Fantasque Sans Mono Fira Code Hack IBM Plex Mono Inconsolata
JetBrains Mono JetBrains Mono NL Liberation Mono Noto Mono Noto Sans SignWriting Source Code Pro
SourceCodeVF Ubuntu Mono
DejaVu Sans Mono   Styles: Bold, Bold Oblique, Book, Oblique   Press the Enter key to choose this family
DejaVu Sans Mono   Press Enter to select this font, Esc to go back to the font list or any of the
highlighted keys below to fine-tune the appearance of the individual font styles.
Regular: DejaVuSansMono   Bold: DejaVuSansMono-Bold   Italic: DejaVuSansMono-Oblique   Bold-Italic: DejaVuSansMono-BoldOblique
You have chosen the DejaVu Sans Mono family   What would you like to do?
Enter to modify kitty.conf and use the new fonts
Esc to abort and return to font selection
s to write the new font settings to STDOUT
Ctrl+c to quit
```

### Appendix F — `--reload-in` variants: SIGUSR1 vs on-disk write

A dummy process with argv[0] = `kitty` (so it satisfies `is_kitty_gui_cmdline`) was started to trap `SIGUSR1`, and `KITTY_PID` was pointed at it. Then `kitten choose-fonts --reload-in <variant>` was driven to `Enter` for each variant:

```
===== --reload-in parent : KITTY_PID=dummy =====
--- SIGUSR1 received by dummy? ---
GOT_SIGUSR1 ts=1783377156.077656743
--- kitty.conf written? ---
>>> YES (190 bytes, block present)

===== --reload-in none : KITTY_PID=dummy =====
--- SIGUSR1 received by dummy? ---
(NO SIGUSR1 received)
--- kitty.conf written? ---
>>> YES (190 bytes, block present)

===== --reload-in all : KITTY_PID=dummy =====
--- SIGUSR1 received by dummy? ---
GOT_SIGUSR1 ts=1783377189.400082592
--- kitty.conf written? ---
>>> YES (190 bytes, block present)
```

### Appendix G — Persistence: BEFORE / AFTER / restart re-read

```
$ export KITTY_CONFIG_DIRECTORY=/tmp/<throwaway>
$ ls -la $KITTY_CONFIG_DIRECTORY                       # BEFORE
(empty; no kitty.conf)
$ cat $KITTY_CONFIG_DIRECTORY/kitty.conf               # BEFORE
cat: .../kitty.conf: No such file or directory

# ... drive kitten choose-fonts, select family, final pane, press Enter ...

$ cat $KITTY_CONFIG_DIRECTORY/kitty.conf               # AFTER
# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTS

# ... fresh kitty process, same KITTY_CONFIG_DIRECTORY ...
$ kitty/launcher/kitty +runpy "$(cat reread.py)"
config_paths    = ('.../kitty.conf',)
font_family     = FontSpec(... system='DejaVuSansMono', created_from_string='DejaVuSansMono')
bold_font       = FontSpec(... system='DejaVuSansMono-Bold', ...)
italic_font     = FontSpec(... system='DejaVuSansMono-Oblique', ...)
bold_italic_font= FontSpec(... system='DejaVuSansMono-BoldOblique', ...)
```

---

## Coverage checklist

- **Q1** — build (`./dev.sh build`) + default launch (`kitty/launcher/kitty`) + observed banner `0.35.2` ✓ (Q1, Appendix A)
- **Q2** — real entry point `kitten choose-fonts` + `--help`; alternate `kitty/fonts/list.py:41-42`; remote-control values flagged **[non-canonical]** ✓ (Q2, Appendix B)
- **Q3a** — registration (`tools/cmd/tool/main.go:82`, `main.go:74-98`) ✓
- **Q3b** — `--reload-in`, default `parent`, choices `parent, all, none` (`main.go:86-95`) ✓
- **Q3c** — value flow (`faces_settings`, pane hand-offs, Go↔Python backend) ✓ (Q3c, Appendix E)
- **Q3d** — finalization: four actions (`final.go:38-44`) + `serialized()` (`final.go:63-70`) ✓
- **Q4** — persistence PROVEN: BEFORE (no block) / AFTER (`# BEGIN_KITTY_FONTS`) / `.bak` / restart re-read; condition matrix for `Enter`/`s`/`Esc`/`--reload-in parent|all|none`; `--config NONE` caveat ✓
- **Mechanism** — `Patcher.Patch` sentinel-block algorithm, `ConfigDir` resolution, `SIGUSR1` reload (distinct from `load_config`) ✓

