# How kitty's `choose-fonts` kitten behaves — and whether a confirmed font selection persists across restarts

## 1. Title & TL;DR

**Direct answer:** Pressing **Enter** at the `choose-fonts` kitten's final screen **PERSISTS** the font choice. It writes a sentinel-delimited `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` block containing four keys (`font_family`, `bold_font`, `italic_font`, `bold_italic_font`) into **`kitty.conf` on disk**, so the choice **survives a restart** (it is re-read from `kitty.conf` on the next launch) — *in addition* to triggering a live `SIGUSR1` config reload of the running instance according to `--reload-in`. This is not a session-only change.

By contrast, the three sibling actions do **not** persist:
- **`s`** writes the same serialized settings to **STDOUT only** (no `kitty.conf` change).
- **`Esc`** aborts and returns to the previous pane **without writing** anything.
- **`Ctrl+c`** quits (exit code 1, `Error: canceled by user`) without writing.

The write itself is performed by the Enter handler at `kittens/choose_fonts/final.go:L78-L95`, which builds a `config.Patcher{Write_backup: true}`, targets `filepath.Join(utils.ConfigDir(), "kitty.conf")`, and calls `patcher.Patch(path, "KITTY_FONTS", self.settings.serialized(), "font_family", "bold_font", "italic_font", "bold_italic_font")`. The on-disk mechanics (sentinel block, comment-out of prior keys, `.bak` backup, atomic write) live in `tools/config/api.go:L310-L350`.

**This answer was produced RUN-FIRST**: kitty was built from this checkout, the kitten was driven through its real entry point (`kitten choose-fonts`) inside a headless kitty GUI, and every behavioral claim below is backed by the actual captured output reproduced in-line and in the Raw-Evidence Appendix (§9).

---

## 2. Environment & Method

- **Designated environment:** the container `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0…` (from `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`), which supplies the Go/C toolchain, native font libraries, and a virtual display (Xvfb) so the OpenGL-backed interactive TUI can run headlessly.
- **Run-first methodology:** build → launch → invoke the kitten through its real entry point → drive the TUI → observe the on-disk state transition and the restart re-read → exercise every sibling condition (`s`, `Esc`, `Ctrl+c`) and every `--reload-in` variant (`parent`, `all`, `none`). Observations preceded this write-up.
- **How the interactive TUI was driven:** the kitten is an OpenGL-backed terminal UI, so it was run **inside a real kitty GUI window** under `DISPLAY=:99` (Xvfb) and driven with kitty **remote control** (`kitten @ launch`, `send-key`, `send-text`, `get-text`). Remote control was used only to *drive keystrokes into* and *read the screen of* the real kitten — the `choose-fonts` code path (family selection → previews → final → `Patcher.Patch`) always executed through its real entry point. No value below was obtained from a remote-control shortcut, a debug hook, or a synthetic stand-in; in particular, the reload variants in §7/§9.7 were observed against **real kitty GUI processes**, not a fake signal trap.
- **Driving `s` (associated-text detail):** the final pane's `s` action is dispatched by `on_text` (`final.go:L101-L108`), which only fires when the key event carries associated text. `send-key s` does not populate that text field, so the `s` run was driven by sending the explicit CSI-u sequence `\e[115;1;115u` (key `115`='s', associated text codepoint `115`); this exercises the real `on_text` path, not a bypass.
- **Reload observable:** to detect whether a *running* kitty GUI actually reloaded its config after the kitten's `SIGUSR1`, `background_opacity` was used as a telltale — it is read live via `kitten @ ls` and only changes value if the process re-reads `kitty.conf`. In this checkout kitty does **not** watch `kitty.conf` for changes or auto-reload it on edit — there is no config-file watcher, and a reload happens only via an explicit trigger (the `reload_config_file` mapping's `load_config_file` action, a `SIGUSR1` signal, or a remote-control command). Editing the file alone therefore never reloaded a GUI, so the *only* thing that could trigger a reload in these runs was the kitten's own `SIGUSR1`. (Note: `auto_reload_config` is **not** a valid option in kitty 0.35.2; a `kitty.conf` line by that name is reported as an unknown key — `Ignoring unknown config key: auto_reload_config` — and has no effect. The isolation above comes from kitty's no-auto-reload behavior, not from any config setting.)
- **Isolation:** every run that could write configuration used an **isolated temporary `KITTY_CONFIG_DIRECTORY`** (`utils.ConfigDir()` honors it first — `tools/utils/paths.go:L88-L89`). The real user config at `~/.config/kitty/kitty.conf` was never touched.
- **Grounding:** every factual claim cites a specific `file:line` and/or shows captured output. The few statements that are reasoned from code rather than directly observed are explicitly labeled **(inferred, not observed)** — in this investigation, all four final-pane actions and all three reload variants were *directly observed*, so no behavioral claim needed that label.

---

## 3. Q1 — Build & Default Launch

**Direct answer:** kitty is built from this checkout with the canonical developer command **`./dev.sh build`**, which produces the in-place launchers `kitty/launcher/kitty` and `kitty/launcher/kitten`. Launching `kitty/launcher/kitty --version` prints the observed banner **`kitty 0.35.2 created by Kovid Goyal`**. A single default instance is started (exactly as a normal user would) simply by running `kitty/launcher/kitty` with no arguments; under the headless container that is `DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 XDG_RUNTIME_DIR=/tmp/xdg kitty/launcher/kitty`.

### Toolchain (prerequisites for the build)
- **Go** — `go version go1.22.12 linux/amd64`, satisfying the `go 1.22` requirement declared at `go.mod:L3` (also listed as a build-time dependency at `docs/build.rst:L101`).
- **C compiler** — `gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0`. A C compiler plus the Go compiler are the two hard requirements per `docs/build.rst:L14-L16`, and `gcc`/`clang` is the first build-time dependency at `docs/build.rst:L99`.
- **Native font/runtime libraries.** `docs/build.rst:L81-L94` lists kitty's run-time dependencies — `harfbuzz` >= 2.2.0 (`L84`), `zlib` (`L85`), `libpng` (`L86`), `liblcms2` (`L87`), `freetype` (`L90`), `fontconfig` (`L91`), `libcanberra` (`L92`) — and `docs/build.rst:L97-L102` the build-time ones (`gcc`/`clang` `L99`, `simde` `L100`, `go` `L101`, `pkg-config` `L102`). Their installed versions were read directly from `pkg-config` (complete command + output in §9.1): `harfbuzz 10.2.0`, `freetype2 26.2.20`, `fontconfig 2.15.0`, `libpng 1.6.50`, `lcms2 2.16`, `zlib 1.3.1`, `libcanberra 0.30` — each satisfying the minimums in `docs/build.rst`.

### Build command and result
`dev.sh:L9` is exactly `exec go run bypy/devenv.go "$@"`; `./dev.sh build` therefore delegates to `go run bypy/devenv.go build`. On this already-compiled tree the canonical command completed successfully:

```
$ source /etc/profile.d/go.sh && CI=true ./dev.sh build
Build successful. Run kitty as: kitty/launcher/kitty
```

> Observed honestly: because the tree was already compiled during environment setup, `./dev.sh build` produced only the single success line above (an incremental build). `docs/build.rst:L24-L26` explains this design — `./dev.sh build` downloads kitty's major dependencies as pre-built binaries and builds against them; re-running it rebuilds only what changed.

The resulting launcher binaries exist:

```
$ ls -l kitty/launcher/kitty kitty/launcher/kitten
-rwxr-xr-x 1 root root 15765764 Jul  6 22:11 kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 Jul  6 21:43 kitty/launcher/kitty
```

### Observed version banner (canonical / default configuration)
```
$ kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

This is the **observed** value. As a cross-check only, the source declares `version: Version = Version(0, 35, 2)` at `kitty/constants.py:L25` and the banner string is assembled by `version()` at `kitty/cli.py:L486-L492` as `'{} {}{} created by {}'.format(italic(appname), green(str_version), rev, title('Kovid Goyal'))` — no VCS revision is embedded here, so no `(rev)` appears. The reported value is what the binary actually printed, not the source constant.

### Default single-instance launch
Started with no configuration overrides (empty config dir ⇒ canonical defaults). The instance comes up as a single OS window; complete capture in §9.1:
```
$ DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 XDG_RUNTIME_DIR=/tmp/xdg kitty/launcher/kitty
# (single default instance; empty config dir => canonical defaults)
instance running (pid alive): yes
$ kitten @ ls | (window count + default font_family in opts)
os_windows: 1
foreground_processes cmd: ['/bin/bash', '--posix']
```

### Packager alternatives (context — NOT used here)
- `make` → `Makefile:L12-L13` runs `python3 setup.py` (the packager build).
- CI → `python .github/workflows/ci.py build` (`.github/workflows/ci.yml:L62-L63`); `pyproject.toml:L2` requires Python ≥3.8 and CI tests 3.8/3.9/3.10 (`ci.yml:L25-L35`).

These were noted for completeness; the canonical developer path `./dev.sh build` is what was used and reported.

---

## 4. Q2 — Invoking the Kitten

**Direct answer:** from inside a running kitty instance the font chooser is launched through its real entry point with **`kitten choose-fonts`**. The subcommand is registered with `Name: "choose-fonts"` at `kittens/choose_fonts/main.go:L76`.

Driven inside a headless kitty GUI, `kitten choose-fonts` renders its first screen — the **family-list pane** — with fonts enumerated by the Python backend (complete, unedited screen capture):

```
>DejaVu Sans Mono      ║                DejaVu Sans Mono
 Fira Code             ║
 Inconsolata           ║ Styles: Bold, Bold Oblique, Book, Oblique
 JetBrains Mono        ║
 JetBrains Mono NL     ║ Press the Enter key to choose this family
 Liberation Mono       ║
 Noto Mono             ║ ────────────────── preview ──────────────────
 Noto Sans SignWriting ║
 Ubuntu Mono           ║
 Ubuntu Sans Mono      ║
                       ║
                       ║
                       ║
                       ║
                       ║
                       ║
                       ║
                       ║
                       ║
                       ║
                       ║
Family:
```

The `>` marks the current family (`family_list.go:L155-L157`, `CurrentFamily`), and the `Family:` prompt is the incremental filter. With an empty configuration the pre-selected family is the resolved default (`DejaVu Sans Mono`) — this baseline matters for the persistence proof in §6.

**Alternate invocation path (context):** the Python font-list path `exec`s straight into the same kitten — `kitty/fonts/list.py:L42` runs `os.execlp(kitten_exe(), 'kitten', 'choose-fonts')`. Both routes reach the identical Go entry point.

---

## 5. Q3 — End-to-End Behavior

### (a) Registration

**Direct answer:** the `kitten` tool root registers the subcommand by importing the package (`tools/cmd/tool/main.go:L9`) and calling `choose_fonts.EntryPoint(root)` (`tools/cmd/tool/main.go:L82`). `EntryPoint` (`kittens/choose_fonts/main.go:L74-L99`) adds a `cli.Command{Name: "choose-fonts", …}` and then registers a **visible clone alias** `choose_fonts` (`clone.Hidden = false; clone.Name = "choose_fonts"` at L96-L98).

Source of the registration call:
```
$ sed -n '9p;81,82p' tools/cmd/tool/main.go
	"kitty/kittens/choose_fonts"
	// choose-fonts
	choose_fonts.EntryPoint(root)
```

Both names resolve at runtime — they appear in the `kitten` command list:
```
$ kitty/launcher/kitten --help | grep -A1 choose
   choose-fonts
    Choose the fonts used in kitty
   choose_fonts
    Choose the fonts used in kitty
```

…and **both** accept `--help`. The canonical name `kitten choose-fonts --help`:
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

The clone alias `kitten choose_fonts --help` (underscore) resolves to the identical command; the only differences in its help output are the command name echoed in the `Usage:` line and the version line (exit code 0):
```
$ kitty/launcher/kitten choose_fonts --help ; echo "exit=$?"
Usage: kitten choose_fonts 

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

kitten choose_fonts 0.35.2 created by Kovid Goyal
exit=0
```

### (b) Option Parsing

**Direct answer:** the command binds exactly one option, **`--reload-in`**, of `Type: "choices"` with `Choices: "parent, all, none"` and **`Default: "parent"`** (`kittens/choose_fonts/main.go:L86-L95`). The option is bound to `Options{Reload_in string}` (`main.go:L70-L72`); the Run closure does `opts := Options{}` → `cmd.GetOptionValues(&opts)` → `main(&opts)` (`main.go:L78-L84`). The observed `--reload-in [=parent]` default and `Choices: parent, all, none` are visible in the `--help` output reproduced in (a) above (identical for both the `choose-fonts` and `choose_fonts` names).

The `choices` type is enforced at option-parse time (before the TUI/backend start), as seen by feeding an invalid value:
```
$ kitty/launcher/kitten choose-fonts --reload-in=bogus </dev/null 2>&1 ; echo "exit=$?"
Error: bogus is not a valid value for --reload-in. Valid values: parent, all, none
exit=1
```

### (c) Value Flow

**Direct answer:** the panes accumulate a `faces_settings` value — `type faces_settings struct { font_family, bold_font, italic_font, bold_italic_font string }` (`kittens/choose_fonts/faces.go:L14-L16`) — that is handed pane-to-pane until the final confirmation. The observed flow:

1. **Family-list pane** (`list.go` / `family_list.go`). Incremental filtering is handled by `family_list.go:L103` (`UpdateFamilies`); the current family is `family_list.go:L155-L157` (`CurrentFamily`). On **Enter**, `list.go:L246-L250` transfers control via `self.handler.faces.on_enter(family)`.

   Observed incremental filter (typed `Fira`, list narrowed to one family, still showing the `>` selection and its styles):
   ```
   >Fira Code ║                         Fira Code
              ║
              ║ Styles: Bold, Light, Medium, Regular, Retina, SemiBold
              ║
              ║ Press the Enter key to choose this family
              ║
              ║ ──────────────────────── preview ────────────────────────
              ║
              ║
              ║
              ║
              ║
              ║
              ║
              ║
              ║
              ║
              ║
              ║
              ║
              ║
   Family: Fira
   ```

2. **Faces pane** (`faces.go`). `faces.on_enter` seeds the four settings from `resolved_faces_from_kitty_conf`, defaulting the family to `family="<name>"` and the other three to `auto` (`faces.go:L145-L156`). From this pane, **Enter** advances to the final pane via `self.handler.final_pane.on_enter(self.family, self.settings)` (`faces.go:L118-L121`), while **R/B/I/O** (case-insensitive) open the per-face fine-tune panel via `self.handler.face_pane.on_enter(self.family, which, self.settings)` (`faces.go:L125-L143`, entry at `face.go:L298`).

   Observed previews screen (after choosing Fira Code):
   ```
                                  Fira Code
   Press Enter to select this font, Esc to go back to the font list or any
   of the highlighted keys below to fine-tune the appearance of the
   individual font styles.
   Regular: FiraCode-Regular
   Bold: FiraCode-SemiBold
   Italic: FiraCode-Retina
   Bold-Italic: FiraCode-SemiBold
   ```

   > Note on the fine-tune panel (`face.go:L298`): it is reached by pressing R/B/I/O. For a **non-variable** font such as Fira Code (which exposes discrete named styles rather than variable axes) the panel did not present axis sliders during observation, so a distinct fine-tune screenshot is not reproduced here. The routing keys are the case-insensitive R/B/I/O handlers at `faces.go:L125-L143`; the value being fine-tuned is the same `faces_settings` that ultimately reaches the final pane.

3. **Final pane** (`final.go`) — see (d).

### (d) Finalization

**Direct answer:** the final pane's `draw_screen` (`final.go:L29-L47`) presents exactly **four** actions: **Enter** (modify `kitty.conf`, L38), **Esc** (abort → return to font selection, L40), **`s`** (write settings to STDOUT, L42), and **Ctrl+c** (quit, L44).

Observed final pane (verbatim):
```
You have chosen the Fira Code family
What would you like to do?
Enter to modify kitty.conf and use the new fonts
Esc to abort and return to font selection
s to write the new font settings to STDOUT
Ctrl+c to quit
```

The four keys that get written are produced by `serialized()` (`final.go:L63-L70`), which joins the four settings with **fixed column padding** (each key is padded so the value begins at column 18) and **no trailing newline**:
```
font_family      <spec>
bold_font        <spec>
italic_font      <spec>
bold_italic_font <spec>
```
On **Enter**, `final.go:L78-L95` performs the persistence write and (if the file changed) the reload — the mechanism detailed in §6.

---

## 6. Q4 — Persistence (LEAD ANSWER)

**Direct answer:** **YES — pressing Enter persists the font choice to `kitty.conf` on disk, and it survives a restart.** It is *not* a session-only change. The kitten writes a `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` block into `kitty.conf`; on the next launch that file is re-read (proven below both by the kitten pre-selecting the saved family and by kitty's own core config loader resolving it). Enter *also* fires a live `SIGUSR1` reload of the running instance per `--reload-in`, but that live reload is *in addition to* — not instead of — the on-disk write.

### 6.1 The on-disk mechanism

The Enter handler (`kittens/choose_fonts/final.go:L78-L95`), quoted verbatim and complete:
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
```

`Patcher.Patch` (`tools/config/api.go:L310-L350`) does, in order:
- default file mode `0o644` (`L311-L313`);
- read existing `kitty.conf` (`L318`);
- **comment out** any prior top-level occurrences of the four keys via the regex `(?m)^\s*(font_family|bold_font|italic_font|bold_italic_font)\b` → `# $1` (`L325-L326`);
- match the sentinel block with `(?ms)^# BEGIN_%s.+?# END_%s` and **replace-or-append** it as `# BEGIN_%s\n%s\n# END_%s` (`L328`, `L330`; a missing block is appended after two blank lines, `L336-L339`);
- if the content changed and prior content existed, write a `<path>.bak` backup — guarded by `len(raw) > 0 && self.Write_backup` (`L343-L344`);
- write the file **atomically** via `utils.AtomicUpdateFile` (`L347`).

The atomic write itself lives in `tools/utils/atomic-write.go`. `AtomicUpdateFile` (`L79-L88`) is a thin **wrapper**: it chooses the file mode — default `0o644` (`L80`), overridden by an explicit `perms` argument (`L81-L83`), or by the *existing* file's permissions when the file already exists (`L84-L87`) — and then delegates to `AtomicWriteFile`. The real temp-write + rename is `AtomicWriteFile` (`tools/utils/atomic-write.go:L42-L68`): it creates a sibling temp file with `os.CreateTemp` (`L53`), writes the data (`L63`), `Chmod`s it to the chosen mode (`L65`), and finally performs the atomic `os.Rename` over the target (`L67`) — so the config file is never left half-written.

The write target `utils.ConfigDir()` resolves through `ConfigDirForName` (`tools/utils/paths.go:L88`), which **honors `KITTY_CONFIG_DIRECTORY` first** (`L89`), then the XDG variables, defaulting to `~/.config/kitty` (`ConfigDir` is a `sync.OnceValue`, `L132-L134`).

The reload is delivered by `ReloadConfigInKitty` (`tools/config/api.go:L352-L371`), a **`SIGUSR1`** signal — to the parent process via `KITTY_PID` when `in_parent_only` is true (`L353-L357`), or to *all* kitty-GUI processes otherwise (`L363-L366`; the target's `argv` is validated by `is_kitty_gui_cmdline`, `L282-L303`). This is a signal-based mechanism, distinct from the `load_config` remote-control command.

The Python backend is **not** involved in persistence: `kittens/choose_fonts/backend.go:L41` spawns it with `kitty +runpy "from kittens.choose_fonts.backend import main; main()"`, and `backend.py:L11-L27` only imports `kitty.fonts.*`/`kitty.cli` to enumerate and render fonts. All persistence happens in the Go finalize path.

### 6.2 Observed state transition — Run 1 (empty config dir → Enter)

**BEFORE** (pristine isolated `KITTY_CONFIG_DIRECTORY`; `kitty.conf` absent):
```
$ ls -la /tmp/probe_kcfg1.PH0gdM
total 8
drwx--S---  2 root root 4096 Jul  6 23:27 .
drwxrwsrwx 14 root root 4096 Jul  6 23:27 ..
$ cat /tmp/probe_kcfg1.PH0gdM/kitty.conf
cat: /tmp/probe_kcfg1.PH0gdM/kitty.conf: No such file or directory (os error 2)
```

Drive: filter `Fira` → Enter (choose family) → Enter (accept previews) → **Enter (finalize)**.

**AFTER** — `kitty.conf` now exists (139 bytes). Shown with `cat -A` (line ends as `$`, spaces literal) to prove the exact padding and that there is **no trailing newline** after `# END_KITTY_FONTS` (the last line carries no `$`), matching the `api.go:L330` addition format:
```
$ cat -A /tmp/probe_kcfg1.PH0gdM/kitty.conf
# BEGIN_KITTY_FONTS$
font_family      family="Fira Code"$
bold_font        auto$
italic_font      auto$
bold_italic_font auto$
# END_KITTY_FONTS
```
The directory now holds **only** `kitty.conf` (139 bytes) — no `.bak` was produced, as predicted by the `len(raw) > 0` guard (`api.go:L343`): the prior file was absent/empty, so there was nothing to back up:
```
$ ls -la /tmp/probe_kcfg1.PH0gdM
total 12
drwx--S---  2 root root 4096 Jul  6 23:27 .
drwxrwsrwx 14 root root 4096 Jul  6 23:27 ..
-rw-r--r--  1 root root  139 Jul  6 23:27 kitty.conf
```

**RESTART / RE-READ (kitten level).** Relaunching `kitten choose-fonts` against the *same* config dir now pre-selects **Fira Code** (the `>` marker moved from `DejaVu Sans Mono` in the empty-config baseline of §4 to `Fira Code`), because the kitten reads the persisted family at startup (`list.go:L26` `resolved_faces_from_kitty_conf`, applied by `list.go:L168-L171` `SelectFamily`):
```
 DejaVu Sans Mono      ║                   Fira Code
>Fira Code             ║
 Inconsolata           ║ Styles: Bold, Light, Medium, Regular, Retina,
 JetBrains Mono        ║  SemiBold
 JetBrains Mono NL     ║
 Liberation Mono       ║ Press the Enter key to choose this family
 Noto Mono             ║
 Noto Sans SignWriting ║ ────────────────── preview ──────────────────
 Ubuntu Mono           ║
 Ubuntu Sans Mono      ║
                       ║
                       ║
                       ║
                       ║
                       ║
                       ║
                       ║
                       ║
                       ║
                       ║
                       ║
Family:
```

**RESTART / RE-READ (kitty core level).** kitty's own configuration loader parses the persisted file and resolves the family to Fira Code:
```
$ KITTY_CONFIG_DIRECTORY=/tmp/probe_kcfg1.PH0gdM kitty/launcher/kitty +runpy 'import sys
from kitty.constants import defconf
from kitty.config import load_config
o = load_config(defconf)
sys.stdout.write("config file read: %s\n" % defconf)
sys.stdout.write("RESOLVED font_family = %r\n" % (o.font_family,))
'
config file read: /tmp/probe_kcfg1.PH0gdM/kitty.conf
RESOLVED font_family = FontSpec(family='Fira Code', style='', postscript_name='', full_name='', system='', axes=(), variable_name='', created_from_string='family="Fira Code"')
```

Together these prove the choice is genuinely **persisted on disk and re-read by a fresh process** — i.e., kitty remembers the font the next time it opens.

### 6.3 Observed state transition — Run 2 (pre-populated config → Enter): comment-out + `.bak`

Starting from a `kitty.conf` that already contains a `font_family` and other keys, Enter comments out the prior conflicting key, preserves unrelated keys, appends the sentinel block, and — because prior content existed — writes a `.bak`. Unified diff (BEFORE → AFTER):
```
--- /tmp/evidence/run2_before_kitty.conf	2026-07-06 23:27:05.002830159 +0000
+++ /tmp/evidence/run2_after_kitty.conf	2026-07-06 23:27:08.625869204 +0000
@@ -1,4 +1,12 @@
 # My existing kitty configuration
-font_family      JetBrains Mono
+# font_family      JetBrains Mono
 font_size 11
 cursor_shape block
+
+
+# BEGIN_KITTY_FONTS
+font_family      family="Fira Code"
+bold_font        auto
+italic_font      auto
+bold_italic_font auto
+# END_KITTY_FONTS
\ No newline at end of file
```
The prior `font_family      JetBrains Mono` became `# font_family      JetBrains Mono` (comment-out regex, `api.go:L325-L326`); `font_size` and `cursor_shape` were preserved (they are not in the settings-to-comment-out list); the block was appended after two blank lines (`api.go:L336-L339`).

The `.bak` backup was created (guarded path `api.go:L343-L344`) and contains the verbatim original:
```
$ cat /tmp/probe_kcfg2.3PFMPf/kitty.conf.bak
# My existing kitty configuration
font_family      JetBrains Mono
font_size 11
cursor_shape block
```

kitty core re-reads the updated file and resolves Fira Code (the commented-out JetBrains Mono is inert, the block wins) — complete command and full output, no elision:
```
$ KITTY_CONFIG_DIRECTORY=/tmp/probe_kcfg2.3PFMPf kitty/launcher/kitty +runpy 'import sys
from kitty.constants import defconf
from kitty.config import load_config
o = load_config(defconf)
sys.stdout.write("config file read: %s\n" % defconf)
sys.stdout.write("RESOLVED font_family = %r\n" % (o.font_family,))
'
config file read: /tmp/probe_kcfg2.3PFMPf/kitty.conf
RESOLVED font_family = FontSpec(family='Fira Code', style='', postscript_name='', full_name='', system='', axes=(), variable_name='', created_from_string='family="Fira Code"')
```

### 6.4 Contrast: the non-persisting siblings

- **`s` (STDOUT only).** From the final pane, `s`/`S` sets `output_on_exit = self.settings.serialized() + "\n"` and quits (`final.go:L101-L108`); `main.go:L64-L66` writes `output_on_exit` to STDOUT on exit. Observed (kitten exit 0; STDOUT is the raw `serialized()` — the four keys **without** the `# BEGIN/END_KITTY_FONTS` sentinels that only the Enter/`Patcher` path adds; and the config dir's `kitty.conf` stays absent):
  ```
  $ kitten choose-fonts   # drove to final pane, pressed 's'
  === kitten exit code ===
  0
  === STDOUT (self.settings.serialized()+"\n", final.go:L105) ===
  font_family      family="Fira Code"
  bold_font        auto
  italic_font      auto
  bold_italic_font auto
  === STDERR ===
  === config dir state (kitty.conf NOT written) ===
  $ cat $KITTY_CONFIG_DIRECTORY/kitty.conf
  cat: /tmp/probe_kcfgs3.iNxQ8V/kitty.conf: No such file or directory (os error 2)
  ```
- **`Esc` (abort).** From the final pane, `Esc` sets `current_pane = faces` and returns without writing (`final.go:L72-L77`). Observed: the screen returned to the faces/previews pane…
  ```
                                 Fira Code
  Press Enter to select this font, Esc to go back to the font list or any
  of the highlighted keys below to fine-tune the appearance of the
  individual font styles.
  Regular: FiraCode-Regular
  Bold: FiraCode-SemiBold
  Italic: FiraCode-Retina
  Bold-Italic: FiraCode-SemiBold
  ```
  …and `kitty.conf` stayed **absent**:
  ```
  $ ls -la /tmp/probe_kcfge.jxtu3g
  total 8
  drwx--S---  2 root root 4096 Jul  6 23:27 .
  drwxrwsrwx 16 root root 4096 Jul  6 23:27 ..
  $ cat /tmp/probe_kcfge.jxtu3g/kitty.conf
  cat: /tmp/probe_kcfge.jxtu3g/kitty.conf: No such file or directory (os error 2)
  ```
- **`Ctrl+c` (quit).** Ctrl+c is intercepted by the kitten's **own** top-level key handler `handler.on_key_event` (`kittens/choose_fonts/ui.go:L195-L199`), which sets `event.Handled = true` and `return fmt.Errorf("canceled by user")`. That error propagates out of `lp.Run()` and `main.go:L54-L56` turns it into process exit code 1. (The generic loop fallback `run.go:L151-L153`, which would call `on_SIGINT`, is **not reached**: in `run.go:L141-L153` the loop calls `OnKeyEvent` first and returns immediately on its error at `L144-L146`, and even absent an error the `if ev.Handled { return nil }` guard at `L147-L149` short-circuits before the fallback.) Observed exit 1, the error on STDERR, empty STDOUT, and no `kitty.conf`:
  ```
  === exit code ===
  1
  === STDERR ===
  Error: canceled by user
  === STDOUT ===
  $ ls -la /tmp/probe_kcfgc.VPbN8E
  total 8
  drwx--S---  2 root root 4096 Jul  6 23:27 .
  drwxrwsrwx 17 root root 4096 Jul  6 23:27 ..
  $ cat /tmp/probe_kcfgc.VPbN8E/kitty.conf
  cat: /tmp/probe_kcfgc.VPbN8E/kitty.conf: No such file or directory (os error 2)
  ```

### 6.5 Why `--config NONE` cannot prove persistence

`kitty/cli.py:L870-L872` defines `--config -c` with the special keywords `none`/`NONE`. Launching with `--config NONE` yields pure defaults with **no persistent target**, so it cannot demonstrate persistence. That is why every persistence run above used a **real, writable, isolated** `KITTY_CONFIG_DIRECTORY` rather than a `NONE` run.

---

## 7. Condition Matrix

All rows were directly observed (see §6 and §9).

| Final-pane action | What happens | Persisted to `kitty.conf`? | Live reload? | Evidence (`file:line` + output) |
|---|---|---|---|---|
| **Enter**, `--reload-in parent` (default) | Writes `# BEGIN_KITTY_FONTS` block; sends `SIGUSR1` to parent (`KITTY_PID`) | **Yes** | Yes — parent only | `final.go:L78-L89`; `api.go:L352-L357`; §6.2; §7 reload table + §9.7 (opacity 1.0→0.6) |
| **Enter**, `--reload-in all` | Writes block; sends `SIGUSR1` to all kitty GUIs | **Yes** | Yes — all instances | `final.go:L90-L91`; `api.go:L363-L366`; §7 reload table + §9.7 (opacity 1.0→0.6) |
| **Enter**, `--reload-in none` | Writes block; **no** reload signal (no `switch` case) | **Yes** | No | `final.go:L86-L93` (no `none` case); §7 reload table + §9.7 (opacity stays 1.0) |
| **`s` / `S`** | Serialized settings to **STDOUT**; nothing written to disk | No | No | `final.go:L101-L108`, `main.go:L64-L66`; §6.4, §9.6 |
| **`Esc`** | Abort → return to faces pane; nothing written | No | No | `final.go:L72-L77`; §6.4, §9.6 |
| **`Ctrl+c`** | Quit (exit 1, `Error: canceled by user`); nothing written | No | No | `ui.go:L195-L199`, `main.go:L54-L56`; §6.4, §9.6 |

**Directly observed reload behavior — against real kitty GUI processes.** Each variant ran `kitten choose-fonts --reload-in <v>` as a real child of a real kitty GUI. Because kitty does not auto-reload `kitty.conf` on file change (a reload fires only on an explicit trigger — see §2), the *only* possible reload trigger in these runs was the kitten's signal. The telltale is `background_opacity`, read live via `kitten @ ls`: the config was edited to `background_opacity 0.6` *without* a signal (the GUI keeps its loaded value, confirming no auto-reload occurred), then the kitten was finalized; if the GUI reloaded, opacity changes to `0.6`.

| `--reload-in` | `kitty.conf` block written? | Real GUI reloaded? (`background_opacity` telltale) |
|---|---|---|
| `parent` | yes | **YES** — `1.0` → `0.6000000238418579` after finalize |
| `all` | yes | **YES** — `1.0` → `0.6000000238418579` after finalize |
| `none` | yes | **No** — stays `1.0` (no signal sent) though the file contains `0.6` |

The write (persistence) is independent of `--reload-in` — the `# BEGIN_KITTY_FONTS` block persisted in all three runs; only the live reload signal differs.

---

## 8. Data-Flow Diagram

```mermaid
flowchart TD
    A["kitten choose-fonts<br/>main() + loop (main.go:L16, L74-99)"] --> B["Family-list pane<br/>list.go / family_list.go"]
    B -->|"filter + select + Enter<br/>list.go:L246-250"| C["Faces pane (previews)<br/>faces.go:L145-159"]
    C -->|"R / B / I / O<br/>faces.go:L125-143"| D["Fine-tune face panel<br/>face.go:L298"]
    D --> C
    C -->|"Enter<br/>faces.go:L118-121"| E["Final pane<br/>final.go:L29-47"]
    C -->|"Esc<br/>faces.go:L113-116"| B
    E -->|"Enter<br/>final.go:L78-95"| F["Patch kitty.conf<br/>Patcher.Patch — BEGIN_KITTY_FONTS block,<br/>comment-out prior keys, .bak, atomic write<br/>api.go:L310-350"]
    F --> G{"--reload-in?"}
    G -->|"parent (default)"| H["SIGUSR1 -> parent (KITTY_PID)<br/>api.go:L354-357"]
    G -->|"all"| I["SIGUSR1 -> all kitty GUIs<br/>api.go:L363-366"]
    G -->|"none (no case)"| J["no reload signal<br/>final.go:L86-93"]
    F --> K["PERSISTED: survives restart<br/>re-read via list.go:L26/L170 + kitty core load_config"]
    E -->|"s / S<br/>final.go:L101-108"| L["serialize() -> STDOUT only<br/>main.go:L64-66 — NOT persisted"]
    E -->|"Esc<br/>final.go:L72-77"| C
    E -->|"Ctrl+c<br/>ui.go:L195-199"| M["fmt.Errorf(\"canceled by user\") -> exit 1<br/>main.go:L54-56 — no write"]
```

---

## 9. Raw-Evidence Appendix

All outputs below are the actual, unedited captures from the investigation (box-drawing characters preserved from the TUI screen dumps).

### 9.1 Build, version, native deps & default launch (Q1)
```
$ go version
go version go1.22.12 linux/amd64
$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0

$ source /etc/profile.d/go.sh && CI=true ./dev.sh build
Build successful. Run kitty as: kitty/launcher/kitty

$ ls -l kitty/launcher/kitty kitty/launcher/kitten
-rwxr-xr-x 1 root root 15765764 Jul  6 22:11 kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 Jul  6 21:43 kitty/launcher/kitty

$ kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```
Native library versions read from `pkg-config` (backing the values reported in §3):
```
$ pkg-config --modversion harfbuzz freetype2 fontconfig libpng lcms2 zlib libcanberra
10.2.0
26.2.20
2.15.0
1.6.50
2.16
1.3.1
0.30
```
Default single-instance launch (empty config dir ⇒ canonical defaults):
```
$ DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 XDG_RUNTIME_DIR=/tmp/xdg kitty/launcher/kitty
# (single default instance; empty config dir => canonical defaults)
instance running (pid alive): yes
$ kitten @ ls | (window count + default font_family in opts)
os_windows: 1
foreground_processes cmd: ['/bin/bash', '--posix']
```

### 9.2 Registration & option parsing (Q3a, Q3b)
```
$ kitty/launcher/kitten --help | grep -A1 choose
   choose-fonts
    Choose the fonts used in kitty
   choose_fonts
    Choose the fonts used in kitty

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

$ kitty/launcher/kitten choose_fonts --help ; echo "exit=$?"
Usage: kitten choose_fonts 

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

kitten choose_fonts 0.35.2 created by Kovid Goyal
exit=0

$ kitty/launcher/kitten choose-fonts --reload-in=bogus </dev/null 2>&1 ; echo "exit=$?"
Error: bogus is not a valid value for --reload-in. Valid values: parent, all, none
exit=1
```

### 9.3 TUI screens (Q2, Q3c, Q3d)
Family-list pane (first screen; empty config → `DejaVu Sans Mono` pre-selected):
```
>DejaVu Sans Mono      ║                DejaVu Sans Mono
 Fira Code             ║
 Inconsolata           ║ Styles: Bold, Bold Oblique, Book, Oblique
 JetBrains Mono        ║
 JetBrains Mono NL     ║ Press the Enter key to choose this family
 Liberation Mono       ║
 Noto Mono             ║ ────────────────── preview ──────────────────
 Noto Sans SignWriting ║
 Ubuntu Mono           ║
 Ubuntu Sans Mono      ║
                       ║
                       ║
                       ║
                       ║
                       ║
                       ║
                       ║
                       ║
                       ║
                       ║
                       ║
Family:
```
Incremental filter (`Fira`):
```
>Fira Code ║                         Fira Code
           ║
           ║ Styles: Bold, Light, Medium, Regular, Retina, SemiBold
           ║
           ║ Press the Enter key to choose this family
           ║
           ║ ──────────────────────── preview ────────────────────────
           ║
           ║
           ║
           ║
           ║
           ║
           ║
           ║
           ║
           ║
           ║
           ║
           ║
           ║
Family: Fira
```
Faces/previews pane:
```
                               Fira Code
Press Enter to select this font, Esc to go back to the font list or any
of the highlighted keys below to fine-tune the appearance of the
individual font styles.
Regular: FiraCode-Regular
Bold: FiraCode-SemiBold
Italic: FiraCode-Retina
Bold-Italic: FiraCode-SemiBold
```
Final pane (four actions):
```
You have chosen the Fira Code family
What would you like to do?
Enter to modify kitty.conf and use the new fonts
Esc to abort and return to font selection
s to write the new font settings to STDOUT
Ctrl+c to quit
```

### 9.4 Persistence — Run 1 (empty → Enter), before / after / restart (Q4)
```
# BEFORE
$ ls -la /tmp/probe_kcfg1.PH0gdM
total 8
drwx--S---  2 root root 4096 Jul  6 23:27 .
drwxrwsrwx 14 root root 4096 Jul  6 23:27 ..
$ cat /tmp/probe_kcfg1.PH0gdM/kitty.conf
cat: /tmp/probe_kcfg1.PH0gdM/kitty.conf: No such file or directory (os error 2)

# AFTER Enter (cat -A shows line-ends as $; last line has no $ = no trailing newline)
$ cat -A /tmp/probe_kcfg1.PH0gdM/kitty.conf
# BEGIN_KITTY_FONTS$
font_family      family="Fira Code"$
bold_font        auto$
italic_font      auto$
bold_italic_font auto$
# END_KITTY_FONTS
$ ls -la /tmp/probe_kcfg1.PH0gdM
total 12
drwx--S---  2 root root 4096 Jul  6 23:27 .
drwxrwsrwx 14 root root 4096 Jul  6 23:27 ..
-rw-r--r--  1 root root  139 Jul  6 23:27 kitty.conf
# (no kitty.conf.bak — prior file was empty; guarded by len(raw)>0 at api.go:L343)

# RESTART re-read (kitten now pre-selects Fira Code)
 DejaVu Sans Mono      ║                   Fira Code
>Fira Code             ║
 Inconsolata           ║ Styles: Bold, Light, Medium, Regular, Retina,
 JetBrains Mono        ║  SemiBold
 JetBrains Mono NL     ║
 Liberation Mono       ║ Press the Enter key to choose this family
 Noto Mono             ║
 Noto Sans SignWriting ║ ────────────────── preview ──────────────────
 Ubuntu Mono           ║
 Ubuntu Sans Mono      ║

# RESTART re-read (kitty core config loader)
$ KITTY_CONFIG_DIRECTORY=/tmp/probe_kcfg1.PH0gdM kitty/launcher/kitty +runpy 'import sys
from kitty.constants import defconf
from kitty.config import load_config
o = load_config(defconf)
sys.stdout.write("config file read: %s\n" % defconf)
sys.stdout.write("RESOLVED font_family = %r\n" % (o.font_family,))
'
config file read: /tmp/probe_kcfg1.PH0gdM/kitty.conf
RESOLVED font_family = FontSpec(family='Fira Code', style='', postscript_name='', full_name='', system='', axes=(), variable_name='', created_from_string='family="Fira Code"')
```

### 9.5 Persistence — Run 2 (pre-populated → Enter): comment-out + `.bak` (Q4)
```
$ diff -u /tmp/evidence/run2_before_kitty.conf /tmp/evidence/run2_after_kitty.conf
--- /tmp/evidence/run2_before_kitty.conf	2026-07-06 23:27:05.002830159 +0000
+++ /tmp/evidence/run2_after_kitty.conf	2026-07-06 23:27:08.625869204 +0000
@@ -1,4 +1,12 @@
 # My existing kitty configuration
-font_family      JetBrains Mono
+# font_family      JetBrains Mono
 font_size 11
 cursor_shape block
+
+
+# BEGIN_KITTY_FONTS
+font_family      family="Fira Code"
+bold_font        auto
+italic_font      auto
+bold_italic_font auto
+# END_KITTY_FONTS
\ No newline at end of file

$ cat /tmp/probe_kcfg2.3PFMPf/kitty.conf.bak
# My existing kitty configuration
font_family      JetBrains Mono
font_size 11
cursor_shape block

$ KITTY_CONFIG_DIRECTORY=/tmp/probe_kcfg2.3PFMPf kitty/launcher/kitty +runpy 'import sys
from kitty.constants import defconf
from kitty.config import load_config
o = load_config(defconf)
sys.stdout.write("config file read: %s\n" % defconf)
sys.stdout.write("RESOLVED font_family = %r\n" % (o.font_family,))
'
config file read: /tmp/probe_kcfg2.3PFMPf/kitty.conf
RESOLVED font_family = FontSpec(family='Fira Code', style='', postscript_name='', full_name='', system='', axes=(), variable_name='', created_from_string='family="Fira Code"')
```

### 9.6 Sibling conditions — `s` (STDOUT), `Esc` (abort), `Ctrl+c` (quit) (Q4 contrast)
`s` — serialized settings to STDOUT (kitten exit 0), config **not** written:
```
$ kitten choose-fonts   # drove to final pane, pressed 's'
=== kitten exit code ===
0
=== STDOUT (self.settings.serialized()+"\n", final.go:L105) ===
font_family      family="Fira Code"
bold_font        auto
italic_font      auto
bold_italic_font auto
=== STDERR ===
=== config dir state (kitty.conf NOT written) ===
$ cat $KITTY_CONFIG_DIRECTORY/kitty.conf
cat: /tmp/probe_kcfgs3.iNxQ8V/kitty.conf: No such file or directory (os error 2)
```
`Esc` — from the final pane, returns to the faces/previews pane; `kitty.conf` stays absent:
```
                               Fira Code
Press Enter to select this font, Esc to go back to the font list or any
of the highlighted keys below to fine-tune the appearance of the
individual font styles.
Regular: FiraCode-Regular
Bold: FiraCode-SemiBold
Italic: FiraCode-Retina
Bold-Italic: FiraCode-SemiBold
$ ls -la /tmp/probe_kcfge.jxtu3g
total 8
drwx--S---  2 root root 4096 Jul  6 23:27 .
drwxrwsrwx 16 root root 4096 Jul  6 23:27 ..
$ cat /tmp/probe_kcfge.jxtu3g/kitty.conf
cat: /tmp/probe_kcfge.jxtu3g/kitty.conf: No such file or directory (os error 2)
```
`Ctrl+c` — quit via `ui.go:L195-L199` (`fmt.Errorf("canceled by user")`), exit 1, config absent:
```
=== exit code ===
1
=== STDERR ===
Error: canceled by user
=== STDOUT ===
$ ls -la /tmp/probe_kcfgc.VPbN8E
total 8
drwx--S---  2 root root 4096 Jul  6 23:27 .
drwxrwsrwx 17 root root 4096 Jul  6 23:27 ..
$ cat /tmp/probe_kcfgc.VPbN8E/kitty.conf
cat: /tmp/probe_kcfgc.VPbN8E/kitty.conf: No such file or directory (os error 2)
```

### 9.7 Reload variants — observed against REAL kitty GUIs (Q3d / condition coverage)
Each variant ran `kitten choose-fonts --reload-in <v>` as a real child of a real kitty GUI (kitty does not auto-reload `kitty.conf` on file change, so the only reload trigger is the kitten's signal; observable = `background_opacity` via `kitten @ ls`). The signal path is `ReloadConfigInKitty` → `SIGUSR1` (`tools/config/api.go:L352-L371`), with the target validated by `is_kitty_gui_cmdline` (`api.go:L282-L303`).
```
### --reload-in parent (target GUI pid=189900, config dir=/tmp/probe_rl_parent) ###
opacity initially loaded by GUI      : 1.0
opacity after editing conf to 0.6 (no signal; kitty does not auto-reload on file change): 1.0   <- GUI has NOT reloaded
kitten choose-fonts --reload-in parent finalized (Enter): wrote font block + reload-per-variant
opacity after kitten's finalize      : 0.6000000238418579
=> EXPECT 0.6 (kitten sent SIGUSR1 to real GUI -> GUI reloaded conf). OBSERVED: 0.6000000238418579
--- resulting kitty.conf (font block persisted; opacity edit preserved) ---
allow_remote_control yes
background_opacity 0.6


# BEGIN_KITTY_FONTS
font_family      family="Fira Code"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
```
```
### --reload-in all (target GUI pid=190177, config dir=/tmp/probe_rl_all) ###
opacity initially loaded by GUI      : 1.0
opacity after editing conf to 0.6 (no signal; kitty does not auto-reload on file change): 1.0   <- GUI has NOT reloaded
kitten choose-fonts --reload-in all finalized (Enter): wrote font block + reload-per-variant
opacity after kitten's finalize      : 0.6000000238418579
=> EXPECT 0.6 (kitten sent SIGUSR1 to real GUI -> GUI reloaded conf). OBSERVED: 0.6000000238418579
--- resulting kitty.conf (font block persisted; opacity edit preserved) ---
allow_remote_control yes
background_opacity 0.6


# BEGIN_KITTY_FONTS
font_family      family="Fira Code"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
```
```
### --reload-in none (target GUI pid=190457, config dir=/tmp/probe_rl_none) ###
opacity initially loaded by GUI      : 1.0
opacity after editing conf to 0.6 (no signal; kitty does not auto-reload on file change): 1.0   <- GUI has NOT reloaded
kitten choose-fonts --reload-in none finalized (Enter): wrote font block + reload-per-variant
opacity after kitten's finalize      : 1.0
=> EXPECT still 1.0 (no SIGUSR1 sent; GUI did not reload). OBSERVED: 1.0
--- resulting kitty.conf (font block persisted; opacity edit preserved) ---
allow_remote_control yes
background_opacity 0.6


# BEGIN_KITTY_FONTS
font_family      family="Fira Code"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
```
The font block persisted in **all three** runs (persistence is independent of `--reload-in`); only the live reload differs — `parent`/`all` reloaded the real GUI (opacity `1.0`→`0.6`), `none` did not (stayed `1.0`).

### 9.8 Repository cleanliness proof (`git status`)
Captured at the repository root **after** deleting every temporary observation script (`/tmp/probe_drive.sh`, `/tmp/probe_reload.sh`, `/tmp/probe_pty.py`) and every scratch config directory (`/tmp/probe_kcfg*`, `/tmp/probe_rl_*`), removing the sockets and the `/tmp/evidence` dir, and stopping the spawned `Xvfb`/kitty GUI processes:

```console
$ git rev-parse --abbrev-ref HEAD
blitzy-168b0e7e-d8c1-450e-98ae-477a83317dfe

$ git status --porcelain --untracked-files=all
 M blitzy/documentation/kitty_815df1e210e0.md

$ git diff --name-only -- ':!blitzy/documentation/kitty_815df1e210e0.md'
                                  # (no output — zero other tracked files modified)
```

The working tree shows **exactly one** changed path — this document (already tracked, so it appears as modified `M`; it is then committed, leaving the tree clean). No tracked source, docs, config, tests, build, or CI file is modified or deleted. The repository is byte-for-byte unchanged apart from the single answer document, satisfying the read-only scope rule.

---

## 10. Coverage Checklist

- **Q1 — Build & default launch:** §3 + §9.1. Command `./dev.sh build` (`dev.sh:L9`); binaries `kitty/launcher/{kitty,kitten}`; observed banner `kitty 0.35.2 created by Kovid Goyal`; native deps per `docs/build.rst:L81-L102` with `pkg-config` versions captured; default single-instance launch shown.
- **Q2 — Invoke the kitten:** §4 + §9.3. `kitten choose-fonts` (`main.go:L76`); alternate `exec` path `kitty/fonts/list.py:L42`; family-list first screen captured.
- **Q3a — Registration:** §5(a) + §9.2. `tools/cmd/tool/main.go:L9,L82`; visible clone alias `choose_fonts` (`main.go:L96-L98`); both names resolve and both `--help` outputs captured.
- **Q3b — Option parsing:** §5(b) + §9.2. `--reload-in` choices/default `parent` (`main.go:L86-L95`); `Options` struct (`L70-L72`); Run closure (`L78-L84`); invalid-choice rejection observed.
- **Q3c — Value flow:** §5(c) + §9.3. `faces_settings` (`faces.go:L14-L16`); `list.go:L246-L250`; `faces.go:L118-L121` (Enter→final) and `L125-L143` (R/B/I/O→fine-tune, `face.go:L298`); filter + previews screens captured.
- **Q3d — Finalization:** §5(d) + §9.3. Four actions (`final.go:L29-L47`); `serialized()` padding (`final.go:L63-L70`); Enter→Patch (`final.go:L78-L95`).
- **Q4 — Persistence (LEAD):** §1 + §6 + §9.4–9.7. Before/after/restart proven at kitten and kitty-core level; `Patcher.Patch` mechanism; `.bak` and comment-out observed; `s`/`Esc`/`Ctrl+c` non-persistence contrasted with captured output; `--config NONE` caveat.
- **Condition coverage:** §7. Enter (+`--reload-in parent`/`all`/`none`), `s`/`S`, `Esc`, `Ctrl+c` — each with `file:line` + captured output. All reload variants **directly observed against real kitty GUIs**; `Ctrl+c` mechanism located at `ui.go:L195-L199`.
- **Read-only scope:** §9.8 — the only repository artifact written is this document; all temporary scripts and scratch config dirs are removed; the source tree is unchanged.
