# kitty `choose-fonts`: Does a Font Selection Persist Across Restarts? — A Code-Grounded Q&A

| | |
|---|---|
| **Repository** | `kovidgoyal/kitty` |
| **Commit** | `815df1e21` — *"Wire up applying of font config"* |
| **Branch** | `kitty_815df1e210e0` |
| **Scope** | Read-only investigation (no source files modified) |
| **Architecture** | Go-native kitten with a Python backend worker over JSON IPC |
| **Kitten built/observed** | `kitten 0.35.2` (`python3 setup.py build`) |

> ## TL;DR — Direct Answer
>
> **A font selection made through `choose-fonts` is _persisted_, not session-only.** Pressing **`Enter`** at the final confirmation step writes the four font settings into **`kitty.conf` on disk**, so the choice **is remembered across restarts**. The `SIGUSR1` "reload" that follows is a *separate, conditional* step that only makes the already-persisted change take effect **immediately** in the running session — it is **not** what makes the choice durable. The alternative **`s`** action writes the settings to **STDOUT only** (manual/session use; `kitty.conf` is left untouched). Even **`--reload-in none`** still persists to disk — it merely skips the live reload.

**Methodology.** Every behavioral claim below is grounded in the source code with a `[path:locator]` citation and a short snippet; **the code is the source of truth**. Where the current upstream documentation diverges from this commit, the code prevails and the divergence is flagged explicitly (see [§6](#section-6--rationale-methodology--version-drift)). The runtime outputs shown are **confirming evidence produced by running the actual production code** in this environment (the real `kitten` binary and a direct call into the real `tools/config.Patcher.Patch`); the full GUI build/run is also reproducible in the user-provided Docker image (see [Appendix B](#appendix-b--reproducible-runtime-experiment-isolated-self-cleaning)). All runtime experiments are isolated to a throwaway `KITTY_CONFIG_DIRECTORY`; the user's real `~/.config/kitty/kitty.conf` is never touched, and the kitty source tree is left byte-for-byte unchanged.

---

## Contents

1. [Build & run a single default instance, then invoke the kitten (R1)](#section-1--build--run-a-single-default-instance-then-invoke-the-kitten)
2. [Subcommand registration & option parsing (R2)](#section-2--subcommand-registration--option-parsing)
3. [Option/selection value flow to the final step (R3)](#section-3--optionselection-value-flow-to-the-final-step)
4. [Finalization behavior, with output evidence (R4)](#section-4--finalization-behavior-with-output-evidence)
5. [Persistence across restarts — the core answer (R5)](#section-5--persistence-across-restarts--the-core-answer)
6. [Rationale, methodology & version-drift note](#section-6--rationale-methodology--version-drift)
- [Appendix A — Architecture: Go kitten + Python backend](#appendix-a--architecture-go-kitten--python-backend)
- [Appendix B — Reproducible runtime experiment (isolated, self-cleaning)](#appendix-b--reproducible-runtime-experiment-isolated-self-cleaning)
- [Appendix C — Live runtime capture (real production code)](#appendix-c--live-runtime-capture-real-production-code)

---

## Section 1 — Build & run a single default instance, then invoke the kitten

> **Answers R1:** how to build the source tree, launch a single default-settings instance, and invoke `choose-fonts` from inside it.

### 1.1 Build

The canonical build entry point is **`python3 setup.py`**. The `Makefile`'s default target just delegates to it:

```make
all:
	python3 setup.py $(VVAL)
```
`[Makefile:L12-L13]`

There is also a developer convenience wrapper, `./dev.sh`, which is a one-liner that execs the Go dev-environment runner:

```sh
exec go run bypy/devenv.go "$@"
```
`[dev.sh:L9]`

### 1.2 Toolchain (consumed unchanged)

| Requirement | Version | Evidence |
|---|---|---|
| Go | `1.22` | `module kitty` / `go 1.22` `[go.mod:L1-L3]` |
| Python | `>=3.8` | `requires-python = ">=3.8"` `[pyproject.toml:L2]` |
| CI Python (matrix) | `"3.10"` | `[.github/workflows/ci.yml:L30]` |
| CI Python (linux-package) | `"3.11"` | `[.github/workflows/ci.yml:L85]` |
| CI Go | from `go.mod` | `go-version-file: go.mod` `[.github/workflows/ci.yml:L60]` |

Building and running also requires native font libraries (harfbuzz, freetype, fontconfig) plus the platform GUI/windowing libraries; these are linked by `setup.py` when it compiles kitty's C extensions.

### 1.3 What the build produces

`setup.py` writes the launcher executable to **`kitty/launcher/kitty`**:

```python
dest = os.path.join(launcher_dir, 'kitty')
```
`[setup.py:L1295]`

and the Go **`kitten`** binary alongside it:

```python
dest = os.path.join(destination_dir or launcher_dir, 'kitten')
```
`[setup.py:L1160]`

A real build in this environment produced both binaries (`kitty/launcher/kitty` and `kitty/launcher/kitten`, reporting `kitten 0.35.2`).

### 1.4 Launch a single default-settings instance

```bash
kitty/launcher/kitty --config NONE
```

The `--config NONE` flag makes kitty **ignore all config files** and start from built-in defaults. This is stated directly in the CLI help text for `--config`:

> "Use the special value `NONE` to not load any config file."

`[kitty/cli.py:L187-L188]`

### 1.5 Invoke the kitten from *inside* the running instance

At the shell prompt of that kitty window, run:

```bash
kitten choose-fonts
```

Running it *inside* the instance matters: the kitten reads the parent's process id from `$KITTY_PID` to deliver its post-write reload signal (see [§4.3.1](#431-where-kittyconf-lives--utilsconfigdir) and [§4.4](#44-reload-semantics--configreloadconfiginkitty)).

### 1.6 Rationale ("thinking")

`setup.py` is the single source-of-truth build entry — both the `Makefile` and the documented build flow route through it `[Makefile:L12-L13]`, so describing any other path would risk drift. `--config NONE` is chosen deliberately: it guarantees a **clean default baseline** with no ambient `kitty.conf` influencing the experiment `[kitty/cli.py:L187-L188]`. And the kitten must be run *inside* the target instance because that is what wires the kitten's reload signal to that specific parent process via `$KITTY_PID` — a detail that becomes important when we examine the reload step.

---

## Section 2 — Subcommand registration & option parsing

> **Answers R2:** how `choose-fonts` is registered with the `kitten` CLI and how its options are declared and parsed.

### 2.1 CLI wiring

The `kitten` tool's root command imports the package and registers the kitten via its `EntryPoint`:

```go
import "kitty/kittens/choose_fonts"   // [tools/cmd/tool/main.go:L9]
// ...
choose_fonts.EntryPoint(root)         // [tools/cmd/tool/main.go:L82]
```

### 2.2 `EntryPoint`

`EntryPoint(root *cli.Command)` `[kittens/choose_fonts/main.go:L74]` adds the subcommand and wires its `Run` closure:

```go
ans := root.AddSubCommand(&cli.Command{
	Name:             "choose-fonts",
	ShortDescription: "Choose the fonts used in kitty",
	Run: func(cmd *cli.Command, args []string) (rc int, err error) {
		opts := Options{}
		if err = cmd.GetOptionValues(&opts); err != nil {
			return 1, err
		}
		return main(&opts)
	},
})
```
`[kittens/choose_fonts/main.go:L75-L85]`

The `Run` closure constructs `opts := Options{}`, parses the command line into it with `cmd.GetOptionValues(&opts)`, and dispatches to `main(&opts)` `[kittens/choose_fonts/main.go:L79-L83]`.

### 2.3 The `Options` struct — exactly one field

```go
type Options struct {
	Reload_in string
}
```
`[kittens/choose_fonts/main.go:L70-L72]`

### 2.4 The single declared option: `--reload-in`

```go
ans.Add(cli.OptionSpec{
	Name:    "--reload-in",
	Dest:    "Reload_in",
	Type:    "choices",
	Choices: "parent, all, none",
	Default: "parent",
	Help:    `...signal only the parent kitty instance ... to reload its config...`,
})
```
`[kittens/choose_fonts/main.go:L86-L95]`

This is the **only** option the kitten declares. Its three choices map directly to the reload behavior analyzed in [§4.4](#44-reload-semantics--configreloadconfiginkitty).

**Real runtime confirmation** — `kitten choose-fonts --help` prints exactly this option set:

```text
Usage: kitten choose-fonts

Choose the fonts used in kitty

Options:
  --reload-in [=parent]
    By default, this kitten will signal only the parent kitty instance it is
    running in to reload its config, after making changes. ...
    Choices: parent, all, none

  --help, -h
    Show help for this command
```

### 2.5 Version-drift correction: the `choose_fonts` clone is **visible**, not hidden

After defining the subcommand, the code registers a second, underscore-named clone and explicitly makes it **visible**:

```go
clone := root.AddClone(ans.Group, ans)
clone.Hidden = false
clone.Name = "choose_fonts"
```
`[kittens/choose_fonts/main.go:L96-L98]`

`clone.Hidden = false` `[kittens/choose_fonts/main.go:L97]` means the alias is **VISIBLE** in the CLI — it is *not* a hidden alias. This is confirmed at runtime: `kitten --help` lists **both** `choose-fonts` **and** `choose_fonts`, each described "Choose the fonts used in kitty". Any characterization of `choose_fonts` as a "hidden" alias contradicts the code and is corrected here.

### 2.6 Rationale ("thinking")

Option parsing is **declarative** via the `kitty/tools/cli` framework: a single `OptionSpec` describes `--reload-in`, and `cmd.GetOptionValues(&opts)` populates the `Options` struct `[kittens/choose_fonts/main.go:L79-L95]`. The deliberate design point is that the *only* command-line knob is `--reload-in` (default `parent`); **everything about the actual font choice is gathered interactively**, not via flags. That is why the question "does my selection persist?" cannot be answered from the option set alone — it is answered by what the interactive flow does at finalization, which the next sections trace.

---

## Section 3 — Option/selection value flow to the final step

> **Answers R3:** how the user's interactive choices flow through the program to the final confirmation pane.

### 3.1 The handler and the pane state machine

`main(&opts)` builds an event loop and a `handler`. The `handler` struct holds the parsed `opts`, the loop `lp`, the four panes, and a pointer to the currently active pane:

```go
type handler struct {
	opts         *Options
	lp           *loop.Loop
	// ...
	listing      FontList
	faces        faces
	face_pane    face_panel
	final_pane   final_pane
	panes        []pane
	current_pane pane
}
```
`[kittens/choose_fonts/ui.go:L42-L62]`

The four panes are collected into an ordered slice, establishing the navigable UI:

```go
h.panes = []pane{&h.listing, &h.faces, &h.face_pane, &h.final_pane}
```
`[kittens/choose_fonts/ui.go:L81]`

On startup, a worker goroutine queries the Python backend for the monospaced family list, and — crucially — stores the **currently resolved faces from the existing `kitty.conf`**:

```go
// worker goroutine
r := ... query("list_monospaced_fonts" ...)            // [ui.go:L95]
h.listing.resolved_faces_from_kitty_conf = r.Resolved_faces  // [ui.go:L97]
```

### 3.2 Global Ctrl+c, otherwise delegate to the current pane

Key events are handled globally for Ctrl+c (cancel), and otherwise delegated to whichever pane is current:

```go
if event.MatchesPressOrRepeat("ctrl+c") {
	// ...
	return fmt.Errorf("canceled by user")
}
// else: dispatch to h.current_pane
```
`[kittens/choose_fonts/ui.go:L195-L199]`

### 3.3 Family list → faces

In the family listing pane, pressing **`Enter`** on a highlighted family transitions to face selection by calling `faces.on_enter` with the chosen family:

```go
if event.MatchesPressOrRepeat("enter") {
	// ...
	self.handler.faces.on_enter(self.fonts[...].family)  // [list.go:L250]
}
```
`[kittens/choose_fonts/list.go:L246-L250]`

### 3.4 Faces pre-population — the kitten *reads the persisted config* (key insight)

When the faces pane is entered, it **seeds the editable settings from the currently persisted configuration**. It reads the `resolved_faces_from_kitty_conf` captured in §3.1 and, for each of the four faces, uses the current `kitty.conf` spec when the family matches, otherwise a default:

```go
r := self.handler.listing.resolved_faces_from_kitty_conf            // [faces.go:L148]
d := func(conf ResolvedFace, setting *string, defval string) {
	*setting = utils.IfElse(family == conf.Family, conf.Spec, defval) // [faces.go:L150]
}
d(r.Font_family, &self.settings.font_family, fmt.Sprintf(`family="%s"`, family)) // [faces.go:L152]
d(r.Bold_font,   &self.settings.bold_font,   "auto")                 // [faces.go:L153]
d(r.Italic_font, &self.settings.italic_font, "auto")                 // [faces.go:L154]
d(r.Bold_italic_font, &self.settings.bold_italic_font, "auto")       // [faces.go:L155]
```
`[kittens/choose_fonts/faces.go:L145-L159]`

The default for `font_family` is `family="<name>"`; the bold/italic/bold-italic variants default to `auto` `[faces.go:L152-L155]`. **The fact that the kitten reads `kitty.conf` to pre-fill its UI is early, independent evidence that `kitty.conf` is the durable store of font choices** — a point we return to in [§5](#section-5--persistence-across-restarts--the-core-answer).

### 3.5 The `faces_settings` struct — the accumulator

The user's four choices accumulate in a single value:

```go
type faces_settings struct {
	font_family, bold_font, italic_font, bold_italic_font string
}
```
`[kittens/choose_fonts/faces.go:L14-L16]`

### 3.6 Fine-tuning a face

In the faces pane, typing `r`/`R`, `b`/`B`, `i`/`I`, or `o`/`O` opens the per-face fine-tuning pane for `font_family`, `bold_font`, `italic_font`, or `bold_italic_font` respectively, by dispatching to `face_pane.on_enter`:

```go
// on_text in faces pane (one of four similar cases)
self.handler.face_pane.on_enter(self.family, which, self.settings)  // [faces.go:L139]
```
`[kittens/choose_fonts/faces.go:L125-L143]`

The fine-tuning pane (`face.go`) edits styles, font features, and variable-axis sliders. On **`Enter`** it commits the edited value back into the faces pane's settings and returns; on **`Esc`** it discards:

```go
if event.MatchesPressOrRepeat("enter") {
	// ...
	self.handler.faces.settings = self.settings   // [face.go:L288]  (commit)
	self.handler.current_pane = &self.handler.faces
}
```
`[kittens/choose_fonts/face.go:L285-L289]`  (entry point `on_enter` at `[face.go:L298-L303]`)

### 3.7 Faces → final

Pressing **`Enter`** in the faces pane routes the accumulated selection to the final pane:

```go
if event.MatchesPressOrRepeat("enter") {
	// ...
	self.handler.final_pane.on_enter(self.family, self.settings)  // [faces.go:L118-L120]
}
```
`[kittens/choose_fonts/faces.go:L118-L120]`

The final pane stores them and becomes the current pane:

```go
func (self *final_pane) on_enter(family string, settings faces_settings) error {
	self.settings = settings
	self.family = family
	self.handler.current_pane = self
	// ...
}
```
`[kittens/choose_fonts/final.go:L113-L117]`

### 3.8 Compact flow

```text
listing  --Enter on family-->  faces  (settings pre-filled from current kitty.conf)
faces    --r/b/i/o----------->  face_pane (fine-tune)  --Enter-->  faces (commit)
faces    --Enter------------->  final_pane  (holds faces_settings)
final    --Enter/s/Esc/Ctrl+c-> persist to kitty.conf / STDOUT / back / quit
```

### 3.9 Rationale ("thinking")

The user's choices are threaded through a single `faces_settings` value that flows `listing → faces → (face_pane) → final_pane` `[faces.go:L14-L16,L118-L120][final.go:L113-L117]`. Fine-tuning round-trips through `face_pane` but always commits back into `faces.settings` `[face.go:L288]`, so the faces pane remains the single accumulator. By the time the user reaches the confirmation screen, `final_pane.settings` **fully represents the selection** and is ready to be serialized — which is exactly what the finalization step does next.

---

## Section 4 — Finalization behavior, with output evidence

> **Answers R4:** exactly what kitty does when the selection is finalized, supported by concrete output evidence.

This is the evidentiary heart of the document. We examine each final-step key and the persistence machinery it drives.

### 4.0 The on-screen prompt is itself strong evidence

`final_pane.draw_screen` renders the confirmation text. The wording explicitly states that **Enter modifies `kitty.conf`**:

```go
// final_pane.draw_screen  [final.go:L29-L47]
// "You have chosen the <family> family"                     [L34]
// "Enter to modify kitty.conf and use the new fonts"        [L38]
// "Esc to abort and return to font selection"               [L40]
// "s to write the new font settings to STDOUT"              [L42]
// "Ctrl+c to quit"                                          [L44]
```
`[kittens/choose_fonts/final.go:L29-L47]`

The line at `[final.go:L38]` — **"Enter to modify `kitty.conf` and use the new fonts"** — distinguishes the durable on-disk edit ("modify `kitty.conf`") from the live effect ("use the new fonts"), foreshadowing the two-part mechanism below.

### 4.1 Serialization

The chosen faces are serialized into four config lines, joined by newlines, with column padding baked into the literals:

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
`[kittens/choose_fonts/final.go:L63-L70]`

### 4.2 The Enter branch — the central snippet

```go
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
`[kittens/choose_fonts/final.go:L78-L97]`

Three facts are decisive here:

1. **The target file is HARD-CODED** to `kitty.conf` under `utils.ConfigDir()` `[final.go:L81]`. There is no per-invocation file override at this commit (see [§6](#section-6--rationale-methodology--version-drift)).
2. **The patch happens FIRST** `[final.go:L82]` — the on-disk write is unconditional (subject only to whether content actually changed; see §4.3).
3. **The reload is SECOND and CONDITIONAL** — gated on both `updated` *and* the `--reload-in` value `[final.go:L86-L93]`. Persistence does not depend on the reload.

### 4.2.1 The other final-step keys

- **`Esc`** returns to the faces pane without any change:
  ```go
  self.handler.current_pane = &self.handler.faces   // [final.go:L73-L77]
  ```
- **`s` / `S`** writes the serialized settings to a package-level variable and quits — it never touches `kitty.conf`:
  ```go
  case "s", "S":
  	output_on_exit = self.settings.serialized() + "\n"  // [final.go:L105]
  	self.lp.Quit(0)
  ```
  `[kittens/choose_fonts/final.go:L101-L111]`
  That buffer is flushed to STDOUT by `main()` on exit:
  ```go
  if output_on_exit != "" {
  	os.Stdout.WriteString(output_on_exit)   // [main.go:L64-L66]
  }
  ```
  `[kittens/choose_fonts/main.go:L64-L66]`
- **`Ctrl+c`** quits with "canceled by user" (handled globally in `ui.go` `[ui.go:L195-L199]`).

### 4.3 The persistence mechanism — `config.Patcher.Patch`

`Patch` performs the durable, safe edit of `kitty.conf` `[tools/config/api.go:L305-L351]`:

```go
type Patcher struct {
	Write_backup bool
	Mode         fs.FileMode
}
```
`[tools/config/api.go:L305-L308]`

The algorithm (with verified line ranges):

1. **Default mode** `0o644` if unset `[api.go:L311-L313]`.
2. **Read the existing file**, tolerating "does not exist" (treated as empty content) `[api.go:L318-L323]`.
3. **Comment out** any pre-existing target lines via a multiline regex, turning e.g. `font_family ...` into `# font_family ...`:
   ```go
   // (?m)^\s*(font_family|bold_font|italic_font|bold_italic_font)\b  ->  "# $1"
   ```
   `[api.go:L325-L326]`
4. **Insert or replace** a sentinel-delimited block. The sentinel regex is `(?ms)^# BEGIN_KITTY_FONTS.+?# END_KITTY_FONTS` `[api.go:L328]`; the inserted block is `# BEGIN_KITTY_FONTS\n<serialized>\n# END_KITTY_FONTS` `[api.go:L330]`. An existing block is replaced in place `[api.go:L331-L335]`; otherwise the block is appended, separated by a blank line `[api.go:L336-L340]`.
5. **Backup + atomic write — only if content actually changed**:
   ```go
   if !bytes.Equal(raw, nraw) {
   	if len(raw) > 0 && self.Write_backup {
   		// write "<path>.bak"
   	}
   	return true, utils.AtomicUpdateFile(path, nraw, self.Mode)  // [api.go:L347]
   }
   return false, nil                                              // [api.go:L349]
   ```
   `[tools/config/api.go:L342-L349]`
   - A `.bak` backup is written **only when the file had prior content and `Write_backup` is set** `[api.go:L343-L345]`.
   - The update itself is **atomic** via `utils.AtomicUpdateFile` `[api.go:L347]`.
   - If nothing changed, `Patch` returns `(false, nil)` `[api.go:L349]` — so the Enter branch's `if updated` guard skips both the backup and the reload (idempotent).

### 4.3.1 Where `kitty.conf` lives — `utils.ConfigDir()`

`Patch` writes to `filepath.Join(utils.ConfigDir(), "kitty.conf")`. `ConfigDir` resolves the directory, honoring `$KITTY_CONFIG_DIRECTORY` **first**:

```go
func ConfigDirForName(name string) (config_dir string) {
	if kcd := os.Getenv("KITTY_CONFIG_DIRECTORY"); kcd != "" {
		return Abspath(Expanduser(kcd))     // [paths.go:L89-L90]
	}
	// ... XDG_CONFIG_HOME / XDG_CONFIG_DIRS / ~/.config/kitty fallback ...
}

var ConfigDir = sync.OnceValue(func() (config_dir string) {
	return ConfigDirForName("kitty.conf")   // [paths.go:L132-L134]
})
```
`[tools/utils/paths.go:L88-L134]`

This `$KITTY_CONFIG_DIRECTORY`-first resolution `[paths.go:L88-L90]` is precisely the redirection point the runtime experiment uses to isolate all writes into a throwaway directory (see [Appendix B](#appendix-b--reproducible-runtime-experiment-isolated-self-cleaning)).

### 4.4 Reload semantics — `config.ReloadConfigInKitty`

```go
func ReloadConfigInKitty(in_parent_only bool) error {
	if in_parent_only {
		// read $KITTY_PID, verify it is a kitty GUI process,
		// then send unix.SIGUSR1 to it                       // [api.go:L353-L357]
	}
	// else: iterate all processes, send SIGUSR1 to every kitty GUI  // [api.go:L363-L368]
}
```
`[tools/config/api.go:L352-L371]`

Mapping to `--reload-in`:

| `--reload-in` | Call | Effect |
|---|---|---|
| `parent` (default) | `ReloadConfigInKitty(true)` | `SIGUSR1` to the `$KITTY_PID` parent only `[api.go:L353-L357]` |
| `all` | `ReloadConfigInKitty(false)` | `SIGUSR1` to **every** kitty GUI process `[api.go:L363-L368]` |
| `none` | *(no matching `switch` case)* | **No reload signal** — but `Patch` already wrote to disk |

The critical observation: `--reload-in none` falls through the `switch` in the Enter branch with no case `[final.go:L86-L93]`, so **no signal is sent — yet the on-disk patch has already happened**. This cleanly separates "persistence" (always, via the write) from "live reload" (optional).

### 4.5 Output evidence — algorithm-accurate simulation of `Patcher.Patch`

The blocks below are an **algorithm-accurate simulation** of `tools/config.Patcher.Patch`: they trace, step by step, the exact transformations the `Patch` algorithm performs — comment-out of any prior `font_*` keys, the sentinel-delimited `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` block, the `.bak` backup, and the atomic write `[api.go:L310-L350]`. For the **live runtime capture** produced by executing the real `Patcher.Patch` (plus the real reload signal and the real next-launch read) from this commit, see [Appendix C](#appendix-c--live-runtime-capture-real-production-code).

**Case A — fresh `kitty.conf` (file absent, as with `--config NONE`).** `Patch` returns `updated=true`; **no `.bak`** is written because there was no prior content:

```text
# BEGIN_KITTY_FONTS
font_family      family="Fira Code"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
```
*(`kitty.conf.bak` absent — confirmed.)*

**Case B — pre-existing `kitty.conf` with user font lines.** The prior `font_family`/`bold_font` lines are **commented out**, the sentinel block is **appended** (separated by a blank line), and a `kitty.conf.bak` **IS** written:

```text
# my config
# font_family monospace
# bold_font auto
font_size 12.0


# BEGIN_KITTY_FONTS
font_family      family="Fira Code"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
```
*(`kitty.conf.bak` exists, containing the original unmodified content.)* The double blank line between `font_size 12.0` and the sentinel block is the `"\n\n"` separator from the append path `[api.go:L336-L340]`.

**Case C — idempotent re-run with the identical selection.** `Patch` returns `updated=false`; **no `.bak`, no reload** — there is nothing to change `[api.go:L349]`.

```text
updated=false   ->  no backup written, Enter branch skips reload
```

### 4.6 Rationale ("thinking")

Four independent code facts converge on the same conclusion: (1) the **on-screen prompt** literally says Enter modifies `kitty.conf` `[final.go:L38]`; (2) the target is a **hard-coded** `kitty.conf` path `[final.go:L81]`; (3) the write is **atomic and on-disk** via `Patch` → `AtomicUpdateFile` `[api.go:L347]`; and (4) a **`.bak` backup + sentinel-delimited block** are the hallmarks of a durable, repeatable file edit `[api.go:L325-L345]`. Together they prove Enter's effect is a **durable file edit**. The reload is a *separate, conditional* step that only governs whether the change is also applied to the live session — it does not affect persistence.

---

## Section 5 — Persistence across restarts — the core answer

> **Answers R5:** does pressing Enter at the final step persist the choice for the next launch, or only change the running session?

### 5.1 The answer

**Pressing `Enter` PERSISTS the font selection by writing it into `kitty.conf` on disk; therefore it IS remembered across restarts.** The selection is durable the instant the file is written, independent of any reload.

### 5.2 The reasoning chain (grounded in code)

1. **Enter writes to disk.** The Enter branch calls `patcher.Patch(<ConfigDir>/kitty.conf, "KITTY_FONTS", serialized(), …)` `[final.go:L80-L82]`, and `Patch` performs an **atomic on-disk write** `[api.go:L347]`. This happens **before and independently of** any reload.
2. **Reload is conditional and only affects the live session.** The `SIGUSR1` reload is gated on `if updated` and on `--reload-in` `[final.go:L86-L93]`, and `ReloadConfigInKitty` merely signals running processes `[api.go:L352-L371]`. It makes the change take effect **immediately** in the current session; it is **not** what makes the choice durable.
3. **The next launch reads `kitty.conf`.** On a *normal* startup, kitty resolves its config directory in **Python**: `KITTY_CONFIG_DIRECTORY` (when set) is used verbatim, otherwise the XDG search applies `[kitty/constants.py:L87-L89]`, yielding `defconf = <config dir>/kitty.conf` `[kitty/constants.py:L131-L133]`. The startup resolver `default_config_paths` then yields `SYSTEM_CONF` followed by `defconf` `[kitty/cli.py:L1064-L1069][kitty/conf/utils.py:L322-L329]`, and `load_config` parses those onto the builtin defaults `[kitty/config.py:L163-L167]`. So the persisted `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` block is read and the chosen fonts are in effect **after restart**. (The Go `utils.ConfigDir()` `[paths.go:L132-L134]` is the *write* side used by the kitten; it resolves the **same** directory, which is precisely why the kitten writes where the next launch reads.) **Exception — `--config NONE`:** that special value deliberately loads *no* config file, so `resolve_config` yields nothing extra when `NONE` is present `[kitty/cli.py:L187-L188][kitty/conf/utils.py:L322-L329]`; a restart used to *verify persistence* must therefore omit `--config NONE` (runtime-confirmed in [Appendix B](#appendix-b--reproducible-runtime-experiment-isolated-self-cleaning)).
4. **The kitten itself trusts `kitty.conf` as the store.** `faces.on_enter` pre-fills its UI from `resolved_faces_from_kitty_conf` `[faces.go:L145-L159]` — i.e., the kitten reads the *persisted* config to recover the user's previous choices. This is independent corroboration that `kitty.conf` is the durable source of truth.

### 5.3 Contrast: the `s` path is session/manual only

Pressing **`s`** writes the four settings to **STDOUT only** `[final.go:L101-L111]`, flushed by `main()` on exit `[main.go:L64-L66]`; `kitty.conf` is left untouched. The user could redirect or paste that output themselves, but nothing is persisted automatically — this is the explicit session/manual alternative to Enter.

### 5.4 `--reload-in none` still persists

With `--reload-in none`, the Enter branch's `switch` has no matching case, so **no `SIGUSR1` is sent** `[final.go:L86-L93]` — **but `Patch` already wrote the block to disk** `[api.go:L347]`. The selection is therefore **still persisted across restarts**; only the *immediate* live reload is skipped. This is the cleanest demonstration that persistence (the write) and live reload (the signal) are distinct.

### 5.5 Edge case: identical selection is idempotent

If the new selection equals what is already in `kitty.conf`, `Patch` returns `updated=false` `[api.go:L349]`, so **no `.bak` is written and no reload is sent** — there is nothing to do because the value is already persisted.

### 5.6 Rationale ("thinking")

The single most important distinction this document makes is: **persistence == the on-disk write; reload == immediacy.** The common misconception is to conflate the two and conclude that because a reload happens, the change is "only" applied to the running session. The code refutes this: the write to `kitty.conf` is unconditional (modulo idempotency) and atomic `[final.go:L82][api.go:L347]`, while the reload is conditional and merely propagates the already-durable change into the live process `[final.go:L86-L93][api.go:L352-L371]`. Hence the font choice is remembered across restarts.

---

## Section 6 — Rationale, methodology & version-drift

### 6.1 Methodology recap

The code at commit `815df1e21` is authoritative. Upstream documentation and man pages are used only as **cross-checks**, and the runtime outputs are **confirming evidence**. Every experiment is isolated to a throwaway `KITTY_CONFIG_DIRECTORY`, and the kitty source tree is never modified.

### 6.2 Version-drift findings (code is truth)

Three places where the **current upstream documentation diverges from this commit**, with the code prevailing:

1. **Only `--reload-in` exists here.** The current upstream `kitten-choose-fonts` man page additionally documents a `--config-file-name [=kitty.conf]` option — described there as "the name or path to the config file to edit" (https://www.mankier.com/1/kitten-choose_fonts). **That option does NOT exist at this commit** — the kitten declares only `--reload-in` `[kittens/choose_fonts/main.go:L86-L95]`, and the real `--help` output confirms it (see [§2.4](#24-the-single-declared-option---reload-in)).
2. **The target file is hard-coded.** Because `--config-file-name` is absent, the edited file is hard-coded to `kitty.conf` under the config directory `[kittens/choose_fonts/final.go:L81]`; there is no per-invocation override at this commit.
3. **The `choose_fonts` clone is visible, not hidden.** `clone.Hidden = false` `[kittens/choose_fonts/main.go:L97]`, and `kitten --help` lists both `choose-fonts` and `choose_fonts` at runtime (see [§2.5](#25-version-drift-correction-the-choose_fonts-clone-is-visible-not-hidden)).

### 6.3 Cross-checks with upstream documentation (non-authoritative, corroborating)

The following corroborate — but are not the basis of — the behavioral claims above:

- The **choose-fonts kitten guide** describes the same UI flow: filter the family list by typing, press Enter to select a family, view regular/bold/italic previews, fine-tune the regular face with the `R` key, and use a slider for variable axes (https://sw.kovidgoyal.net/kitty/kittens/choose-fonts/). This matches the pane flow in [§3](#section-3--optionselection-value-flow-to-the-final-step).
- The **`kitty.conf` reference** documents the four keys `font_family`, `bold_font`, `italic_font`, `bold_italic_font` and recommends the kitten as, in its words, "the easiest way to select fonts" (https://sw.kovidgoyal.net/kitty/conf/); it also documents reload via `SIGUSR1` / `kill -SIGUSR1 $KITTY_PID`, exactly the signal `ReloadConfigInKitty` sends `[api.go:L353-L357]`.
- The **`kitten-choose-fonts` man page** documents the `--reload-in` choices `parent, all, none` (https://www.mankier.com/1/kitten-choose_fonts), matching the `OptionSpec` in [§2.4](#24-the-single-declared-option---reload-in).

All quoted phrases above are short and attributed; the code remains the sole authority for behavior.

---

## Appendix A — Architecture: Go kitten + Python backend

This appendix is supporting context (it does not affect the persistence answer). It documents where font enumeration and preview rendering come from, so a new contributor understands the full picture.

The `choose-fonts` kitten is **Go-native** but delegates font enumeration and preview rendering to a **Python worker** over JSON IPC.

- **Go side spawns the Python worker.** `backend.go`'s `start()` resolves the kitty executable via `utils.KittyExe()` (falling back to `Which("kitty")`; the error message mentions `KITTY_PATH_TO_KITTY_EXE`), then runs it with `+runpy` to launch the Python entry point:
  ```go
  exe := utils.KittyExe()                       // [backend.go:L33]  (fallback Which("kitty") [L34])
  // error text references KITTY_PATH_TO_KITTY_EXE  [backend.go:L38]
  cmd := exec.Command(exe, "+runpy",
  	"from kittens.choose_fonts.backend import main; main()")  // [backend.go:L41]
  ```
  `[kittens/choose_fonts/backend.go:L32-L41]`
  Communication is **JSON over pipes**; `query()` sets the action field on the request:
  ```go
  cmd["action"] = action   // [backend.go:L106]
  ```
  `[kittens/choose_fonts/backend.go:L100-L106]`

- **Python worker.** `backend.py` imports `kitty.fonts.*` and dispatches actions in `main()`:
  ```python
  # main() dispatch  [backend.py:L150-L168]
  #   "list_monospaced_fonts"  -> [L156]
  #   "read_variable_data"     -> [L159]
  #   "render_family_samples"  -> [L164]
  #   else: Unknown action     -> [L168]
  ```
  `[kittens/choose_fonts/backend.py:L150-L168]`
  The helper `resolved_faces(opts)` `[backend.py:L137]` is what produces the resolved-faces data that the Go side stores as `resolved_faces_from_kitty_conf` and uses for pre-population (see [§3.4](#34-faces-pre-population--the-kitten-reads-the-persisted-config-key-insight)).

- **Go-native confirmation.** `kittens/choose_fonts/main.py` and `__init__.py` are **0-byte stubs**, confirming the kitten's logic lives entirely in Go.

- **(Aside) The broader Python config model.** kitty's general Python configuration pipeline lives in `kitty/config.py` (`atomic_save` `[kitty/config.py:L32]`, `load_config` `[kitty/config.py:L163]`), but `choose-fonts` writes via the **Go** `tools/config.Patcher`, not this Python path.

---

## Appendix B — Reproducible runtime experiment (isolated, self-cleaning)

This procedure proves persistence end-to-end. It must run in the user-provided Docker image `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (which supplies the Go/C toolchain, native font libraries, and a display); the analysis sandbox lacks a GUI. It **never touches** the real `~/.config/kitty`, and it cleans up after itself.

```bash
# 0) Build (once), from the kitty source root
python3 setup.py            # or: ./dev.sh build  -> produces kitty/launcher/kitty + kitten

# 1) Isolate config so the real ~/.config/kitty is never touched
export KITTY_CONFIG_DIRECTORY="$(mktemp -d)"
echo "Using throwaway config dir: $KITTY_CONFIG_DIRECTORY"

# 2) Launch ONE *default-settings baseline* instance. --config NONE makes kitty
#    ignore ALL config files, so the kitten starts from pristine defaults.
#    NOTE: this is the BASELINE launch only; it is NOT the restart check (step 5).
kitty/launcher/kitty --config NONE &

# 3) Inside that kitty window, run the kitten, pick a family, reach the final
#    screen, then press Enter:
kitten choose-fonts

# 4) ENTER-PATH evidence (font selection persisted to disk):
cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"     # contains # BEGIN_KITTY_FONTS ... # END_KITTY_FONTS
ls -l "$KITTY_CONFIG_DIRECTORY/"             # kitty.conf.bak present iff there was prior content
#    The parent kitty (its $KITTY_PID) receives SIGUSR1 and live-reloads.

# 5) Prove "remembered across restarts": close and relaunch WITHOUT --config NONE,
#    keeping the SAME $KITTY_CONFIG_DIRECTORY. A normal startup reads
#    $KITTY_CONFIG_DIRECTORY/kitty.conf, so the persisted font is now in effect.
#    (Do NOT use --config NONE here: it loads no config file and would ignore the
#     persisted kitty.conf  ->  kitty/cli.py:L187-L188, kitty/conf/utils.py:L322-L329.)
kill %1 2>/dev/null; kitty/launcher/kitty &   # normal startup -> loads the persisted kitty.conf

# 6) CONTRAST: the 's' path writes to STDOUT only and leaves kitty.conf unchanged
#    (re-run the kitten, reach the final screen, press 's' -> four font_* lines on STDOUT)

# 7) Teardown: remove the throwaway dir; verify the SOURCE TREE is unchanged
rm -rf "$KITTY_CONFIG_DIRECTORY"
git status --porcelain        # MUST be empty in the kitty source tree
```

**Expected, code-grounded observations:**

- **(a)** `kitty.conf` now contains the `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` block with the four `font_*` settings — the exact shape shown in [§4.5 Case A](#45-output-evidence--algorithm-accurate-simulation-of-patcherpatch) `[api.go:L328-L340]`, and captured live in [Appendix C.2](#c2-ab-enter-path--real-patcherpatch).
- **(b)** A `kitty.conf.bak` exists **iff** the file had prior content `[api.go:L343-L345]` — see [§4.5 Case B](#45-output-evidence--algorithm-accurate-simulation-of-patcherpatch) and the live capture in [Appendix C.2](#c2-ab-enter-path--real-patcherpatch).
- **(c)** The parent received `SIGUSR1` because `--reload-in` defaults to `parent` `[main.go:L86-L95][api.go:L353-L357]`.
- **(d)** After a **normal** relaunch (no `--config NONE`) under the same `$KITTY_CONFIG_DIRECTORY`, the chosen font is active → **persisted across restarts**. Startup reads `$KITTY_CONFIG_DIRECTORY/kitty.conf` via the Python resolver `[kitty/constants.py:L131-L133][kitty/cli.py:L1064-L1069][kitty/conf/utils.py:L322-L329]`; a launch with `--config NONE` would suppress this `[kitty/cli.py:L187-L188]`.
- **(e)** The `s` path leaves `kitty.conf` unchanged `[final.go:L101-L111][main.go:L64-L66]`.

> **Note on evidence in this report:** [§4.5](#45-output-evidence--algorithm-accurate-simulation-of-patcherpatch) is an **algorithm-accurate simulation** that traces the `Patcher.Patch` algorithm step by step. The **live runtime capture** — produced by executing the *real* `tools/config.Patcher.Patch`, the *real* `config.ReloadConfigInKitty` (`SIGUSR1` delivery), and kitty's *real* Python startup loader against an isolated, throwaway `KITTY_CONFIG_DIRECTORY` — is collected separately in [Appendix C](#appendix-c--live-runtime-capture-real-production-code). The §2.4 / §2.5 listings are the **real** `kitten choose-fonts --help` / `kitten --help` output. All temporary artifacts were deleted afterward, leaving `git status --porcelain` empty in the source tree.

---

## Appendix C — Live runtime capture (real production code)

Unlike the algorithm-accurate simulation in [§4.5](#45-output-evidence--algorithm-accurate-simulation-of-patcherpatch), **every block in this appendix was produced by executing the real code from this commit**: the built `kitten` binary, the real `tools/config.Patcher.Patch`, the real `config.ReloadConfigInKitty`, and kitty's real Python startup loader. All of it ran inside a throwaway `KITTY_CONFIG_DIRECTORY` that was deleted afterward, leaving the source tree byte-for-byte unchanged (`git status --porcelain` empty). The selected family in this run was `JetBrains Mono`; the Go harness invoked the production functions exactly as the kitten's final pane does `[final.go:L78-L96]`.

### C.1 The only declared option is `--reload-in` (real `--help`)

Real output of `./kitty/launcher/kitten choose-fonts --help` from the built binary at this commit:

```text
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

This is decisive, runtime confirmation of the version-drift note in [§6.2](#62-version-drift-findings-code-is-truth): at this commit the kitten declares **only** `--reload-in` — there is **no** `--config-file-name`.

### C.2 (a)(b) ENTER path — real `Patcher.Patch`

Driving the **real** `config.Patcher{Write_backup: true}.Patch(<dir>/kitty.conf, "KITTY_FONTS", serialized(), "font_family", "bold_font", "italic_font", "bold_italic_font")` — the exact call from `[final.go:L80-L82]` — against an isolated config directory (`utils.ConfigDir()` resolved the throwaway `KITTY_CONFIG_DIRECTORY` `[paths.go:L88-L90]`).

**Fresh `kitty.conf` (file did not exist).** `Patch` returned `updated=true`; **no `.bak`** was written (there was no prior content, per `[api.go:L343-L345]`). The resulting `kitty.conf`:

```text
# BEGIN_KITTY_FONTS
font_family      JetBrains Mono
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
```

**Pre-existing `kitty.conf` (old font lines + other settings).** `Patch` returned `updated=true` and a `kitty.conf.bak` **was** created. The prior `font_family`/`bold_font` lines were commented out, the unrelated `background`/`font_size` lines were preserved, and the sentinel block was appended:

```text
# my config
# font_family     OldFamily
# bold_font       OldBold
background      #1e1e2e
font_size       12.0


# BEGIN_KITTY_FONTS
font_family      JetBrains Mono
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
```

The `kitty.conf.bak` held the **exact original** content (proving the backup is a faithful copy taken before the edit `[api.go:L343-L345]`):

```text
# my config
font_family     OldFamily
bold_font       OldBold
background      #1e1e2e
font_size       12.0
```

**Idempotent re-run (same selection again).** `Patch` returned `updated=false`; **no `.bak`** and **no reload** — there was nothing to change `[api.go:L349]`:

```text
Patch returned updated=false   ->  no backup written; Enter branch skips reload
```

### C.3 (c) The `s` path — STDOUT only, `kitty.conf` unchanged

The `s` branch sets `output_on_exit = self.settings.serialized() + "\n"` and quits without ever calling `Patch` `[final.go:L104-L110]`. The real four-line `serialized()` payload `[final.go:L62-L69]` written to STDOUT:

```text
font_family      JetBrains Mono
bold_font        auto
italic_font      auto
bold_italic_font auto
```

A byte-for-byte comparison of `kitty.conf` immediately before and after the `s` path confirmed it was **unchanged** (`byte-identical before/after: true`) — because the `s` branch contains no write to disk.

### C.4 (d) Reload signal — real `ReloadConfigInKitty` delivers `SIGUSR1`

Calling the **real** `config.ReloadConfigInKitty(true)` with `KITTY_PID` set to a controlled process whose `argv[0]` basename is `kitty` (so it passes `is_kitty_gui_cmdline` `[api.go:L282-L302]`). The target installed a `SIGUSR1` trap; the trap fired, proving delivery `[api.go:L353-L357]`:

```text
config.ReloadConfigInKitty(true)  ->  returned err=<nil>
target process (KITTY_PID=95841, argv[0] basename "kitty") received SIGUSR1
   SIGUSR1_RECEIVED_AT=1782510915.187686379
```

### C.5 (e) The next launch reads the persisted selection

Using kitty's **real** Python startup path — `default_config_paths` → `resolve_config` → `load_config` `[kitty/cli.py:L1064-L1069][kitty/conf/utils.py:L322-L329][kitty/config.py:L163-L167]` — against a `KITTY_CONFIG_DIRECTORY` holding the persisted `kitty.conf` from §C.2:

```text
KITTY_CONFIG_DIRECTORY  = /tmp/cf_read_h8dFo3
kitty.constants.defconf = /tmp/cf_read_h8dFo3/kitty.conf
SYSTEM_CONF             = /etc/xdg/kitty/kitty.conf

# NORMAL startup (no --config on the command line):
config paths   : ('/etc/xdg/kitty/kitty.conf', '/tmp/cf_read_h8dFo3/kitty.conf')
opts.font_family = FontSpec(..., system='JetBrains Mono', ..., created_from_string='JetBrains Mono')

# CONTRAST — launching with --config NONE:
config paths   : ()                      # no config file loaded at all
opts.font_family = FontSpec(..., system='monospace', ..., created_from_string='')   # builtin default
```

This is the **decisive proof of the core answer (R5)**: on a normal next launch the persisted selection is read back (`JetBrains Mono`), so the font choice **is remembered across restarts**. The `--config NONE` contrast simultaneously substantiates [Appendix B step 5](#appendix-b--reproducible-runtime-experiment-isolated-self-cleaning) and [§5.2](#52-the-reasoning-chain-grounded-in-code) — `--config NONE` resolves to *zero* config files and therefore falls back to the builtin `monospace` default, which is exactly why a persistence-verification relaunch must omit it.

---

*End of document. This file is the sole persistent artifact of the investigation; no file inside the kitty source repository was created, modified, or deleted.*
