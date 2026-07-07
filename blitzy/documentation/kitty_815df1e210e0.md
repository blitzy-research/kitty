# How kitty's `choose-fonts` kitten works, and whether a font selection persists

> **Investigation methodology (RUN-FIRST).** Every behavioral claim below was produced by *building kitty from this checkout, running it, driving the real `kitten choose-fonts` entry point under a PTY, and capturing the actual output.* Each claim is grounded in a specific `file:line` reference **and** the captured output that demonstrates it. Values that were reasoned-about-but-not-directly-observed are labelled **[inferred]**; values obtained through anything other than the real entry point are labelled **[non-canonical]**. Observed HEAD commit: `815df1e21 "Wire up applying of font config"`; source branch `kitty_815df1e210e0`.

---

## Bottom line up front (the direct answer)

**Pressing `Enter` on the final screen of `choose-fonts` PERSISTS the font choice to disk. It is *not* a session-only change.** The kitten writes a sentinel-delimited `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` block containing four keys (`font_family`, `bold_font`, `italic_font`, `bold_italic_font`) into `kitty.conf` in your kitty config directory, written atomically (and, when a non-empty `kitty.conf` already existed, backed up to `kitty.conf.bak` first), and then signals the running kitty to reload. Because the setting lives in `kitty.conf`, a **freshly launched kitty re-reads it on startup**, so the choice **survives a full restart**.

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

This document was produced against kitty/kitten **version `0.35.2`** (observed banner, see Q1). The canonical developer build is `./dev.sh build`; on this container's toolchain the *unadorned* command fails at a C `-Werror=switch` warning in `glfw/wl_window.c` (unrelated to `choose-fonts`), so the project's own sanctioned `--ignore-compiler-warnings` accommodation (`setup.py:2003`) was used to produce the launchers. The complete output of **both** commands — the canonical failure and the accommodated success — is shown in Q1 and Appendix A.

---

## Q1 — Build kitty from this checkout and launch a default instance

**Direct answer.** The canonical developer build is `./dev.sh build`, which produces the in-place launchers `kitty/launcher/kitty` and `kitty/launcher/kitten`. On this container's toolchain the *unadorned* `./dev.sh build` **fails** on a C `-Werror=switch` warning (`glfw/wl_window.c:668`, a newer `wayland-protocols` enum, unrelated to `choose-fonts`); the build therefore used kitty's *own* sanctioned `--ignore-compiler-warnings` flag (`setup.py:2003`), which succeeds and produces the same launchers. Launch a default instance simply by running `kitty/launcher/kitty`. The observed version banner is **`kitty 0.35.2 created by Kovid Goyal`**. (Both build commands' complete output is in Appendix A.)

### Build

`dev.sh` delegates to the Go dev-env bootstrap, which downloads kitty's major dependencies as prebuilt binaries and builds in place:

```sh
# dev.sh:9
exec go run bypy/devenv.go "$@"
```

Two build commands were run, in order.

**1) The canonical, unadorned command `./dev.sh build` — it FAILS on this toolchain.** It compiles all 122 C translation units, then errors out at `glfw/wl_window.c:668` because the container's newer `wayland-protocols` introduces `XDG_TOPLEVEL_STATE_CONSTRAINED_*` enum values not handled in a `switch`, and kitty's default C flags treat warnings as errors (`-Werror`). This is unrelated to `choose-fonts`. The failing tail (the complete log is in Appendix A.1):

```
$ ./dev.sh build
[122/122] Compiling kitty/gl-wrapper.c ...
glfw/wl_window.c: In function ‘xdgToplevelHandleConfigure’:
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_LEFT’ not handled in switch [-Werror=switch]
  668 |         switch (*state) {
      |         ^~~~~~
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_RIGHT’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_TOP’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_BOTTOM’ not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
 done
Compiling [wayland] glfw/wl_window.c ...
gcc -MMD -DNDEBUG -D_GLFW_WAYLAND -D_GLFW_BUILD_DLL -DHAS_MEMFD_CREATE ... -Werror ... -c glfw/wl_window.c -o build/glfw-wayland-glfw-wl_window.c.o
The following build command failed: /tmp/blitzy/kitty/blitzy-fa84ef14-7004-4983-831b-c229f20c11d9_5a2778/dependencies/linux-amd64/bin/python setup.py develop
exit status 1
```

(The `gcc ... -c glfw/wl_window.c ...` line above is shown abbreviated with `...` in the *body* for readability; the byte-for-byte complete line and the full log appear unedited in Appendix A.1.)

**2) The project's own sanctioned accommodation `./dev.sh build --ignore-compiler-warnings` — it SUCCEEDS.** `--ignore-compiler-warnings` is a real kitty build option (`setup.py:2003`, help text: "Ignore any warnings from the compiler while building"). The succeeding tail (the complete log is in Appendix A.2):

```
$ ./dev.sh build --ignore-compiler-warnings
[122/122] Compiling kitty/gl-wrapper.c ...
 done
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
kitty/tools/cmd
Build successful. Run kitty as: kitty/launcher/kitty
$ echo "EXIT_CODE=$?"
EXIT_CODE=0
$ ls -l kitty/launcher/kitty kitty/launcher/kitten
-rwxr-xr-x 1 root root 15765764 Jul  6 23:09 kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 Jul  6 23:08 kitty/launcher/kitty
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
- **Seeding the settings.** `faces.on_enter(family)` seeds the four fields from the *existing* `kitty.conf` (`resolved_faces_from_kitty_conf`) or, absent a match, from defaults (`family="…"`, `auto`): `kittens/choose_fonts/faces.go:145-159`. *(Observed: seeding a config with a prior `font_family` line and running `Enter` comments out that prior line and replaces the sentinel block — see the `.bak` demo in Q4.)*
- **Faces → final.** On `Enter` in the faces pane, control advances to the final pane with the accumulated settings:
  `kittens/choose_fonts/faces.go:118-121` — `return self.handler.final_pane.on_enter(self.family, self.settings)` (call at `:120`).
- **Faces → fine-tune (optional).** The keys `r/R`, `b/B`, `i/I`, `o/O` open the per-face fine-tuning panel for `font_family`/`bold_font`/`italic_font`/`bold_italic_font` respectively:
  `kittens/choose_fonts/faces.go:125-141` (the `on_text` handler; `r/R` → `font_family` at `:129-130`, the hand-off call at `:139`) → `kittens/choose_fonts/face.go:298` `func (self *face_panel) on_enter(family, which string, settings faces_settings) error`. **This branch was exercised at runtime — pressing `R` at the faces pane opened the regular-face fine-tune panel; the captured screen is shown below and in Appendix E.**
- **Faces → back.** `Esc` returns to the listing pane: `kittens/choose_fonts/faces.go:112-117`.

The Go↔Python split (enumeration/rendering only, no persistence):

- `kittens/choose_fonts/backend.go:41` — `k.cmd = exec.Command(exe, "+runpy", "from kittens.choose_fonts.backend import main; main()")`
- `kittens/choose_fonts/backend.go:44-52` — `os.Pipe()` pipes + `json.NewDecoder(k.from)`
- `kittens/choose_fonts/backend.py:11-27` — imports `kitty.cli` / `kitty.fonts.*` and performs font enumeration/sample rendering only.

Observed value flow — driving the **real** `kitten choose-fonts` entry point under a PTY (the observation harness is reproduced verbatim in Appendix H; it runs `kitten choose-fonts`, answers kitty's `kitty-query-*` terminfo handshake with the default-kitty values `foreground=#dddddd background=#000000 font_size=11 dpi_x=dpi_y=96`, and sends kitty keyboard-protocol CSI-u key events). The step sequence for this capture was:

```
# harness step list (see Appendix H for the driver):
#   wait "Press the Enter key to choose this family" -> send Enter   (family -> faces)
#   wait "select this font"                          -> send r       (faces -> fine-tune)
#   wait "Regular face"                              -> send Esc     (fine-tune -> faces)
#   wait "select this font"                          -> send Enter   (faces -> final)
#   wait "What would you like to do"                 -> send Ctrl+c  (final -> quit, no write)
$ python3 driver.py   # KITTY_CONFIG_DIRECTORY=<isolated tmp>, TERM=xterm-kitty
```

The captures below are the **rendered screens** replayed from the captured byte stream (cursor-positioning/color control sequences and the APC kitty-graphics image payloads are omitted — they carry no font text; the visible text is exactly as displayed). The complete raw byte stream (9674 bytes, all redraw frames + binary graphics payloads) was captured to a file during the run; every frame, including the family-list pane, appears in Appendix E.

**faces pane** (after selecting the family and pressing `Enter`) — the four accumulated `faces_settings` fields are visible:

```
                                                    DejaVu Sans Mono
Press Enter to select this font, Esc to go back to the font list or any of the highlighted keys below to fine-tune the
appearance of the individual font styles.
Regular: DejaVuSansMono
Bold: DejaVuSansMono-Bold
Italic: DejaVuSansMono-Oblique
Bold-Italic: DejaVuSansMono-BoldOblique
```

**regular-face fine-tune panel** (after pressing `R` at the faces pane — the `kittens/choose_fonts/faces.go:125-141` → `kittens/choose_fonts/face.go:298` branch; `kittens/choose_fonts/face.go:146-166` renders this screen):

```
                                             DejaVu Sans Mono: Regular face
Press Enter to accept any changes or Esc to cancel. Click on a style name below to switch to it.
Current setting: DejaVuSansMono
Styles: Bold, Bold Oblique, Book, Oblique
─────────────────────────────────────────────────────── preview ───────────────────────────────────────────────────────
```

**final pane** (after pressing `Enter` at the faces pane):

```
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
- **`s` / `S`** — set `output_on_exit = self.settings.serialized() + "\n"` and quit; the string is emitted to STDOUT at process exit. `kittens/choose_fonts/final.go:101-111` (case at `:104-106`) and `kittens/choose_fonts/main.go:64-66` (`if output_on_exit != "" { os.Stdout.WriteString(output_on_exit) }`). Runtime capture in Appendix C.
- **`Ctrl+c`** — quit with **no write**. `Ctrl+c` never reaches the final pane's key handler at all: it is intercepted one level up, in the top-level `handler.on_key_event`, which returns an error *before* it dispatches to `current_pane.on_key_event`:

  ```go
  // kittens/choose_fonts/ui.go:195-204
  func (h *handler) on_key_event(event *loop.KeyEvent) (err error) {
  	if event.MatchesPressOrRepeat("ctrl+c") {
  		event.Handled = true
  		return fmt.Errorf("canceled by user")
  	}
  	if h.current_pane != nil {
  		err = h.current_pane.on_key_event(event)
  	}
  	return
  }
  ```

  Because a non-nil error unwinds the event loop, the final pane's `Enter`→`Patch` path (`kittens/choose_fonts/final.go:78-95`) is never entered, so nothing is written. Observed at runtime — driving the real `kitten choose-fonts` to the final pane and sending `Ctrl+c` (`ESC [ 99 ; 5 u`) in an isolated `KITTY_CONFIG_DIRECTORY` — the **complete** raw tail after the alt-screen teardown is the error line, and the config directory is left empty:

  ```
  # complete unedited raw tail (single repr, from the alt-screen restore ESC[?1049l to EOF):
  b'\x1b[?1049l\x1b[?r\x1b[#Q\x1b8\x1b[1;91mError\x1b[221;39m: canceled by user\r\n'
  # os.listdir(KITTY_CONFIG_DIRECTORY) afterwards:
  []   # empty -> Ctrl+c did not write kitty.conf
  ```

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

### Observed: BEFORE → AFTER → restart (first run, no prior `kitty.conf`)

The transcript below is a single reproducible run: the observation harness (Appendix H) drives the **real** `kitten choose-fonts`, and `reread.py` (listed in full in Appendix G) is executed in a **fresh** `kitty +runpy` process afterwards. **BEFORE**, the config directory has no `kitty.conf`; the kitten is driven to the final pane and `Enter` is pressed; **AFTER**, `kitty.conf` exists with the sentinel block; the fresh process (a restart) then re-reads it:

```
$ export KITTY_CONFIG_DIRECTORY=/tmp/blitzy_obs/pA_b0mlt5ao
$ ls -la $KITTY_CONFIG_DIRECTORY                 # BEFORE
total 8
drwx------  2 root root 4096 Jul  6 23:38 .
drwxr-xr-x 13 root root 4096 Jul  6 23:38 ..
$ cat $KITTY_CONFIG_DIRECTORY/kitty.conf         # BEFORE
cat: /tmp/blitzy_obs/pA_b0mlt5ao/kitty.conf: No such file or directory (os error 2)
# (drove kitten choose-fonts -> family -> final -> Enter; process exit 0)
$ ls -la $KITTY_CONFIG_DIRECTORY                 # AFTER
total 12
drwx------  2 root root 4096 Jul  6 23:38 .
drwxr-xr-x 13 root root 4096 Jul  6 23:38 ..
-rw-r--r--  1 root root  190 Jul  6 23:38 kitty.conf
$ cat $KITTY_CONFIG_DIRECTORY/kitty.conf         # AFTER
# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTS
$ kitty/launcher/kitty +runpy "$(cat reread.py)"   # fresh process (restart), same dir
config_dir       = /tmp/blitzy_obs/pA_b0mlt5ao
defconf          = /tmp/blitzy_obs/pA_b0mlt5ao/kitty.conf
config_paths     = ('/tmp/blitzy_obs/pA_b0mlt5ao/kitty.conf',)
font_family      = FontSpec(family='', style='', postscript_name='', full_name='', system='DejaVuSansMono', axes=(), variable_name='', created_from_string='DejaVuSansMono')
bold_font        = FontSpec(family='', style='', postscript_name='', full_name='', system='DejaVuSansMono-Bold', axes=(), variable_name='', created_from_string='DejaVuSansMono-Bold')
italic_font      = FontSpec(family='', style='', postscript_name='', full_name='', system='DejaVuSansMono-Oblique', axes=(), variable_name='', created_from_string='DejaVuSansMono-Oblique')
bold_italic_font = FontSpec(family='', style='', postscript_name='', full_name='', system='DejaVuSansMono-BoldOblique', axes=(), variable_name='', created_from_string='DejaVuSansMono-BoldOblique')
```

`config_paths` shows the fresh process **read the written file**, and the four `FontSpec` values (`system='DejaVuSansMono'`, `-Bold`, `-Oblique`, `-BoldOblique`) are precisely the keys the kitten wrote. The choice therefore survives a restart ⇒ **on-disk persistence, not a session-only change.** *(Reproduced across independent runs.)*

For contrast, a fresh config directory with **no** `kitty.conf` yields the built-in defaults, and `config_paths = ()` confirms nothing was read from disk:

```
$ export KITTY_CONFIG_DIRECTORY=/tmp/blitzy_obs/empty_i4pz   # empty, no kitty.conf
$ kitty/launcher/kitty +runpy "$(cat reread.py)"
config_dir       = /tmp/blitzy_obs/empty_i4pz
defconf          = /tmp/blitzy_obs/empty_i4pz/kitty.conf
config_paths     = ()
font_family      = FontSpec(family='', style='', postscript_name='', full_name='', system='monospace', axes=(), variable_name='', created_from_string='')
bold_font        = FontSpec(family='', style='', postscript_name='', full_name='', system='auto', axes=(), variable_name='', created_from_string='')
italic_font      = FontSpec(family='', style='', postscript_name='', full_name='', system='auto', axes=(), variable_name='', created_from_string='')
bold_italic_font = FontSpec(family='', style='', postscript_name='', full_name='', system='auto', axes=(), variable_name='', created_from_string='')
```

so the transition `system='monospace' → system='DejaVuSansMono'` is a genuine, observed state change.

### Observed: the `.bak` backup and prior-key handling

The `.bak` backup is written **only when a non-empty file already existed** and its content actually changed — the guard is `len(raw) > 0 && self.Write_backup` (`tools/config/api.go:342-344`), which is exactly why the first-run case above produced **no** `.bak`. To exercise the backup path, a config directory was seeded with a *prior* `kitty.conf` (a top-level `font_family Fira Code` plus an existing `# BEGIN_KITTY_FONTS` block), then `Enter` was pressed:

```
$ cat $KITTY_CONFIG_DIRECTORY/kitty.conf         # BEFORE (seeded prior file)
# my prior config
font_family      Fira Code
bold_font        Fira Code Bold

# BEGIN_KITTY_FONTS
font_family      OldFamily
bold_font        OldFamily-Bold
italic_font      OldFamily-Italic
bold_italic_font OldFamily-BoldItalic
# END_KITTY_FONTS
# (drove kitten choose-fonts -> family -> final -> Enter; process exit 0)
$ ls -l $KITTY_CONFIG_DIRECTORY                  # AFTER (note kitty.conf.bak)
total 8
-rw-r--r-- 1 root root 248 Jul  6 23:38 kitty.conf
-rw-r--r-- 1 root root 247 Jul  6 23:38 kitty.conf.bak
$ diff $KITTY_CONFIG_DIRECTORY/kitty.conf.bak $KITTY_CONFIG_DIRECTORY/kitty.conf
2,3c2,3
< font_family      Fira Code
< bold_font        Fira Code Bold
---
> # font_family      Fira Code
> # bold_font        Fira Code Bold
6,9c6,9
< font_family      OldFamily
< bold_font        OldFamily-Bold
< italic_font      OldFamily-Italic
< bold_italic_font OldFamily-BoldItalic
---
> font_family      DejaVuSansMono
> bold_font        DejaVuSansMono
> italic_font      DejaVuSansMono
> bold_italic_font DejaVuSansMono
```

Two behaviors are visible: (1) the prior top-level `font_family`/`bold_font` lines are **commented out** (`# font_family …`) by the `(?m)^\s*(keys)\b → # $1` substitution (`tools/config/api.go:325-326`), and (2) the existing `# BEGIN_KITTY_FONTS` block is **replaced in place** (`OldFamily* → DejaVuSansMono`) (`tools/config/api.go:328-340`); `kitty.conf.bak` (247 bytes) preserves the original byte-for-byte. *(Observed nuance, reported exactly as seen and reproduced across two runs: in this seeded run the four faces serialized to the bare family name `DejaVuSansMono` rather than the `-Bold`/`-Oblique` PostScript names seen in the first-run capture above; this does not affect the backup / comment-out / replace behavior demonstrated here.)*

### Condition matrix (every final-screen action + every `--reload-in` value — all observed)

| Trigger | On-disk write to `kitty.conf`? | Reload signal | Persists across restart? | Evidence |
|---|---|---|---|---|
| **`Enter`** (`--reload-in parent`, default) | **Yes** — `# BEGIN_KITTY_FONTS` block, atomic write (`.bak` only if a non-empty file pre-existed) | `SIGUSR1` to parent (`KITTY_PID`) | **Yes** | BEFORE/AFTER + reread above; `kittens/choose_fonts/final.go:78-95` |
| **`s` / `S`** | **No** | none | **No** | serialized block on STDOUT; `kitty.conf` absent (Appendix C); `kittens/choose_fonts/final.go:101-111`, `kittens/choose_fonts/main.go:64-66` |
| **`Esc`** | **No** | none | **No** | returns to faces pane; `kitty.conf` absent (Appendix D); `kittens/choose_fonts/final.go:72-77` |
| **`Ctrl+c`** | **No** | none | **No** | intercepted in `kittens/choose_fonts/ui.go:196-199` before the final pane's handler → `"canceled by user"`; `kitty.conf` absent (Q3d) |
| **`--reload-in parent`** + Enter | **Yes** | `SIGUSR1` to parent only | **Yes** | real kitty GUI (PID 127748): reload log 2→3 (Appendix F); `tools/config/api.go:352-361` |
| **`--reload-in all`** + Enter | **Yes** | `SIGUSR1` to **all** kitty GUIs | **Yes** | real kitty GUI: reload log 3→4 (Appendix F); `tools/config/api.go:363-368` |
| **`--reload-in none`** + Enter | **Yes** | **none** (no `switch` case) | **Yes** | real kitty GUI: no reload (log 4→4), file still written (Appendix F); `kittens/choose_fonts/final.go:86-93` |

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
3. **Comment out prior keys** — any prior top-level lines matching the given keys (`font_family`, `bold_font`, `italic_font`, `bold_italic_font`) are prefixed with `# ` via a `(?m)^\s*(keys)\b` substitution (`:325-326`). *(Observed in the `.bak` diff in Q4: `font_family      Fira Code` → `# font_family      Fira Code`.)*
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

This `SIGUSR1` reload mechanism was **observed directly** against a running kitty GUI (Appendix F) and is grounded in the source: a delivered `SIGUSR1` sets the child-monitor's `reload_config` flag (`kitty/child-monitor.c:1373-1374`), which then calls `load_config_file` (`kitty/child-monitor.c:534-535`) — kitty's normal config-reload path. *(Note: this checkout, v0.35.2, has no `auto_reload_config` option — `grep -rn auto_reload_config` over the tree returns nothing — so `SIGUSR1` is the reload trigger exercised here.)*

---

## Appendix — evidence, with the commands that produced it

Each appendix below states exactly what kind of listing it is: a **complete, raw/unedited capture** (build logs, command output, and byte-stream tails — nothing removed, abbreviated, or de-duplicated) *or* a **rendered screen** faithfully replayed from the captured raw byte stream (the interactive-TUI frame captures in Appendices D and E, where cursor-positioning/color control sequences and binary graphics payloads are omitted because they carry no font text). The two are labeled distinctly and never conflated.

### Appendix A — Build (canonical failure + sanctioned accommodation) and version banners

Both build logs below are the **complete, unedited** captured output (`2>&1`-redirected); no line between the first command line and the last output line has been removed, abbreviated, or de-duplicated. Lines beginning with `$` are the commands. The trailing `EXIT_CODE` annotation from the raw capture is shown here as an explicit `echo "$?"` exchange.

**A.1 — `./dev.sh build` (the canonical, unadorned command) — FAILS on this toolchain.** Complete output (135 lines: all 122 C compile steps, then the `-Werror=switch` failure at `glfw/wl_window.c:668`, then `exit status 1`):

```
$ ./dev.sh build
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
[4/122] Compiling [x11] glfw/x11_window.c ...
[5/122] Compiling kitty/glfw.c ...
[6/122] Compiling kitty/graphics.c ...
[7/122] Compiling kitty/child-monitor.c ...
[8/122] Compiling kitty/fonts.c ...
[9/122] Compiling kitty/shaders.c ...
[10/122] Compiling kitty/vt-parser.c ...
[11/122] Compiling kitty/vt-parser.c ...
[12/122] Compiling kitty/state.c ...
[13/122] Compiling [x11] glfw/input.c ...
[14/122] Compiling [wayland] glfw/input.c ...
[15/122] Compiling kitty/mouse.c ...
[16/122] Compiling [x11] glfw/xkb_glfw.c ...
[17/122] Compiling [wayland] glfw/xkb_glfw.c ...
[18/122] Compiling kitty/freetype.c ...
[19/122] Compiling [wayland] glfw/wl_client_side_decorations.c ...
[20/122] Compiling [x11] glfw/window.c ...
[21/122] Compiling [wayland] glfw/window.c ...
[22/122] Compiling kitty/line.c ...
[23/122] Compiling kitty/glfw-wrapper.c ...
[24/122] Compiling kittens/transfer/algorithm.c ...
[25/122] Compiling [wayland] glfw/wl_init.c ...
[26/122] Compiling [x11] glfw/x11_init.c ...
[27/122] Compiling kitty/freetype_render_ui_text.c ...
[28/122] Compiling [x11] glfw/egl_context.c ...
[29/122] Compiling [wayland] glfw/egl_context.c ...
[30/122] Compiling kitty/disk-cache.c ...
[31/122] Compiling [x11] glfw/glx_context.c ...
[32/122] Compiling kitty/line-buf.c ...
[33/122] Compiling kitty/data-types.c ...
[34/122] Compiling kitty/colors.c ...
[35/122] Compiling kitty/history.c ...
[36/122] Compiling kitty/keys.c ...
[37/122] Compiling [x11] glfw/x11_monitor.c ...
[38/122] Compiling kitty/fontconfig.c ...
[39/122] Compiling [x11] glfw/context.c ...
[40/122] Compiling [wayland] glfw/context.c ...
[41/122] Compiling kitty/crypto.c ...
[42/122] Compiling [x11] glfw/ibus_glfw.c ...
[43/122] Compiling [wayland] glfw/ibus_glfw.c ...
[44/122] Compiling kitty/key_encoding.c ...
[45/122] Compiling kitty/launcher/main.c ...
[46/122] Compiling [x11] glfw/monitor.c ...
[47/122] Compiling [wayland] glfw/monitor.c ...
[48/122] Compiling kitty/font-names.c ...
[49/122] Compiling [x11] glfw/backend_utils.c ...
[50/122] Compiling [wayland] glfw/backend_utils.c ...
[51/122] Compiling kitty/charsets.c ...
[52/122] Compiling [x11] glfw/linux_joystick.c ...
[53/122] Compiling [wayland] glfw/linux_joystick.c ...
[54/122] Compiling [x11] glfw/init.c ...
[55/122] Compiling [wayland] glfw/init.c ...
[56/122] Compiling [x11] glfw/dbus_glfw.c ...
[57/122] Compiling [wayland] glfw/dbus_glfw.c ...
[58/122] Compiling kitty/gl.c ...
[59/122] Compiling [x11] glfw/vulkan.c ...
[60/122] Compiling [wayland] glfw/vulkan.c ...
[61/122] Compiling [x11] glfw/osmesa_context.c ...
[62/122] Compiling [wayland] glfw/osmesa_context.c ...
[63/122] Compiling kitty/cursor.c ...
[64/122] Compiling kitty/launcher/single-instance.c ...
[65/122] Compiling kitty/desktop.c ...
[66/122] Compiling kitty/loop-utils.c ...
[67/122] Compiling 3rdparty/ringbuf/ringbuf.c ...
[68/122] Compiling kitty/simd-string.c ...
[69/122] Compiling kitty/systemd.c ...
[70/122] Compiling kitty/shlex.c ...
[71/122] Compiling [wayland] glfw/wayland-tablet-unstable-v2-client-protocol.c ...
[72/122] Compiling kitty/child.c ...
[73/122] Compiling [wayland] glfw/linux_desktop_settings.c ...
[74/122] Compiling [wayland] glfw/wl_text_input.c ...
[75/122] Compiling [wayland] glfw/wl_monitor.c ...
[76/122] Compiling kitty/kittens.c ...
[77/122] Compiling 3rdparty/base64/lib/codec_choose.c ...
[78/122] Compiling kitty/png-reader.c ...
[79/122] Compiling [wayland] glfw/wayland-xdg-shell-client-protocol.c ...
[80/122] Compiling [x11] glfw/linux_notify.c ...
[81/122] Compiling [wayland] glfw/linux_notify.c ...
[82/122] Compiling kitty/rowcolumn-diacritics.c ...
[83/122] Compiling kitty/hyperlink.c ...
[84/122] Compiling [wayland] glfw/wayland-primary-selection-unstable-v1-client-protocol.c ...
[85/122] Compiling kitty/wcswidth.c ...
[86/122] Compiling [wayland] glfw/wayland-pointer-constraints-unstable-v1-client-protocol.c ...
[87/122] Compiling kitty/fast-file-copy.c ...
[88/122] Compiling [wayland] glfw/wayland-text-input-unstable-v3-client-protocol.c ...
[89/122] Compiling [wayland] glfw/wayland-wlr-layer-shell-unstable-v1-client-protocol.c ...
[90/122] Compiling 3rdparty/base64/lib/lib.c ...
[91/122] Compiling [x11] glfw/posix_thread.c ...
[92/122] Compiling [wayland] glfw/posix_thread.c ...
[93/122] Compiling kitty/window_logo.c ...
[94/122] Compiling [wayland] glfw/wayland-xdg-activation-v1-client-protocol.c ...
[95/122] Compiling [wayland] glfw/wayland-xdg-decoration-unstable-v1-client-protocol.c ...
[96/122] Compiling [wayland] glfw/wayland-relative-pointer-unstable-v1-client-protocol.c ...
[97/122] Compiling [wayland] glfw/wayland-cursor-shape-v1-client-protocol.c ...
[98/122] Compiling [wayland] glfw/wayland-fractional-scale-v1-client-protocol.c ...
[99/122] Compiling kitty/glyph-cache.c ...
[100/122] Compiling [wayland] glfw/wayland-viewporter-client-protocol.c ...
[101/122] Compiling kitty/logging.c ...
[102/122] Compiling 3rdparty/base64/lib/arch/neon64/codec.c ...
[103/122] Compiling [wayland] glfw/wayland-single-pixel-buffer-v1-client-protocol.c ...
[104/122] Compiling 3rdparty/base64/lib/tables/tables.c ...
[105/122] Compiling [wayland] glfw/wl_cursors.c ...
[106/122] Compiling 3rdparty/base64/lib/arch/neon32/codec.c ...
[107/122] Compiling [wayland] glfw/wayland-kwin-blur-v1-client-protocol.c ...
[108/122] Compiling 3rdparty/base64/lib/arch/avx/codec.c ...
[109/122] Compiling 3rdparty/base64/lib/arch/ssse3/codec.c ...
[110/122] Compiling 3rdparty/base64/lib/arch/sse42/codec.c ...
[111/122] Compiling 3rdparty/base64/lib/arch/sse41/codec.c ...
[112/122] Compiling 3rdparty/base64/lib/arch/avx2/codec.c ...
[113/122] Compiling kitty/utmp.c ...
[114/122] Compiling 3rdparty/base64/lib/arch/avx512/codec.c ...
[115/122] Compiling 3rdparty/base64/lib/arch/generic/codec.c ...
[116/122] Compiling kitty/cleanup.c ...
[117/122] Compiling [x11] glfw/monotonic.c ...
[118/122] Compiling [wayland] glfw/monotonic.c ...
[119/122] Compiling kitty/monotonic.c ...
[120/122] Compiling kitty/simd-string-128.c ...
[121/122] Compiling kitty/simd-string-256.c ...
[122/122] Compiling kitty/gl-wrapper.c ...
glfw/wl_window.c: In function ‘xdgToplevelHandleConfigure’:
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_LEFT’ not handled in switch [-Werror=switch]
  668 |         switch (*state) {
      |         ^~~~~~
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_RIGHT’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_TOP’ not handled in switch [-Werror=switch]
glfw/wl_window.c:668:9: error: enumeration value ‘XDG_TOPLEVEL_STATE_CONSTRAINED_BOTTOM’ not handled in switch [-Werror=switch]
cc1: all warnings being treated as errors
 done
Compiling [wayland] glfw/wl_window.c ...
gcc -MMD -DNDEBUG -D_GLFW_WAYLAND -D_GLFW_BUILD_DLL -DHAS_MEMFD_CREATE -I/tmp/blitzy/kitty/blitzy-fa84ef14-7004-4983-831b-c229f20c11d9_5a2778/dependencies/linux-amd64/include -Wextra -Wfloat-conversion -Wno-missing-field-initializers -Wall -Wstrict-prototypes -std=c11 -pedantic-errors -Werror -O3 -fwrapv -fstack-protector-strong -pipe -fvisibility=hidden -fno-plt -fPIC -D_FORTIFY_SOURCE=2 -flto -fcf-protection=full -march=native -mtune=native -fPIC -pthread -I/tmp/blitzy/kitty/blitzy-fa84ef14-7004-4983-831b-c229f20c11d9_5a2778/dependencies/linux-amd64/include -I/tmp/blitzy/kitty/blitzy-fa84ef14-7004-4983-831b-c229f20c11d9_5a2778/dependencies/linux-amd64/include -I/tmp/blitzy/kitty/blitzy-fa84ef14-7004-4983-831b-c229f20c11d9_5a2778/dependencies/linux-amd64/include -I/usr/include/dbus-1.0 -I/usr/lib/x86_64-linux-gnu/dbus-1.0/include -c glfw/wl_window.c -o build/glfw-wayland-glfw-wl_window.c.o
The following build command failed: /tmp/blitzy/kitty/blitzy-fa84ef14-7004-4983-831b-c229f20c11d9_5a2778/dependencies/linux-amd64/bin/python setup.py develop
exit status 1
$ echo "EXIT_CODE=$?"
EXIT_CODE=1
```

**A.2 — `./dev.sh build --ignore-compiler-warnings` (kitty's own sanctioned flag, `setup.py:2003`) — SUCCEEDS.** Complete output (131 lines: the 122 C compile steps, the 5 link steps, the Go `kitty/tools/cmd` build, then `Build successful`), followed by the produced launchers and the observed version banners:

```
$ ./dev.sh build --ignore-compiler-warnings
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
[4/122] Compiling [x11] glfw/x11_window.c ...
[5/122] Compiling kitty/glfw.c ...
[6/122] Compiling kitty/graphics.c ...
[7/122] Compiling kitty/child-monitor.c ...
[8/122] Compiling kitty/fonts.c ...
[9/122] Compiling kitty/shaders.c ...
[10/122] Compiling kitty/vt-parser.c ...
[11/122] Compiling kitty/vt-parser.c ...
[12/122] Compiling kitty/state.c ...
[13/122] Compiling [x11] glfw/input.c ...
[14/122] Compiling [wayland] glfw/input.c ...
[15/122] Compiling kitty/mouse.c ...
[16/122] Compiling [x11] glfw/xkb_glfw.c ...
[17/122] Compiling [wayland] glfw/xkb_glfw.c ...
[18/122] Compiling kitty/freetype.c ...
[19/122] Compiling [wayland] glfw/wl_client_side_decorations.c ...
[20/122] Compiling [x11] glfw/window.c ...
[21/122] Compiling [wayland] glfw/window.c ...
[22/122] Compiling kitty/line.c ...
[23/122] Compiling kitty/glfw-wrapper.c ...
[24/122] Compiling kittens/transfer/algorithm.c ...
[25/122] Compiling [wayland] glfw/wl_init.c ...
[26/122] Compiling [x11] glfw/x11_init.c ...
[27/122] Compiling kitty/freetype_render_ui_text.c ...
[28/122] Compiling [x11] glfw/egl_context.c ...
[29/122] Compiling [wayland] glfw/egl_context.c ...
[30/122] Compiling kitty/disk-cache.c ...
[31/122] Compiling [x11] glfw/glx_context.c ...
[32/122] Compiling kitty/line-buf.c ...
[33/122] Compiling kitty/data-types.c ...
[34/122] Compiling kitty/colors.c ...
[35/122] Compiling kitty/history.c ...
[36/122] Compiling kitty/keys.c ...
[37/122] Compiling [x11] glfw/x11_monitor.c ...
[38/122] Compiling kitty/fontconfig.c ...
[39/122] Compiling [x11] glfw/context.c ...
[40/122] Compiling [wayland] glfw/context.c ...
[41/122] Compiling kitty/crypto.c ...
[42/122] Compiling [x11] glfw/ibus_glfw.c ...
[43/122] Compiling [wayland] glfw/ibus_glfw.c ...
[44/122] Compiling kitty/key_encoding.c ...
[45/122] Compiling kitty/launcher/main.c ...
[46/122] Compiling [x11] glfw/monitor.c ...
[47/122] Compiling [wayland] glfw/monitor.c ...
[48/122] Compiling kitty/font-names.c ...
[49/122] Compiling [x11] glfw/backend_utils.c ...
[50/122] Compiling [wayland] glfw/backend_utils.c ...
[51/122] Compiling kitty/charsets.c ...
[52/122] Compiling [x11] glfw/linux_joystick.c ...
[53/122] Compiling [wayland] glfw/linux_joystick.c ...
[54/122] Compiling [x11] glfw/init.c ...
[55/122] Compiling [wayland] glfw/init.c ...
[56/122] Compiling [x11] glfw/dbus_glfw.c ...
[57/122] Compiling [wayland] glfw/dbus_glfw.c ...
[58/122] Compiling kitty/gl.c ...
[59/122] Compiling [x11] glfw/vulkan.c ...
[60/122] Compiling [wayland] glfw/vulkan.c ...
[61/122] Compiling [x11] glfw/osmesa_context.c ...
[62/122] Compiling [wayland] glfw/osmesa_context.c ...
[63/122] Compiling kitty/cursor.c ...
[64/122] Compiling kitty/launcher/single-instance.c ...
[65/122] Compiling kitty/desktop.c ...
[66/122] Compiling kitty/loop-utils.c ...
[67/122] Compiling 3rdparty/ringbuf/ringbuf.c ...
[68/122] Compiling kitty/simd-string.c ...
[69/122] Compiling kitty/systemd.c ...
[70/122] Compiling kitty/shlex.c ...
[71/122] Compiling [wayland] glfw/wayland-tablet-unstable-v2-client-protocol.c ...
[72/122] Compiling kitty/child.c ...
[73/122] Compiling [wayland] glfw/linux_desktop_settings.c ...
[74/122] Compiling [wayland] glfw/wl_text_input.c ...
[75/122] Compiling [wayland] glfw/wl_monitor.c ...
[76/122] Compiling kitty/kittens.c ...
[77/122] Compiling 3rdparty/base64/lib/codec_choose.c ...
[78/122] Compiling kitty/png-reader.c ...
[79/122] Compiling [wayland] glfw/wayland-xdg-shell-client-protocol.c ...
[80/122] Compiling [x11] glfw/linux_notify.c ...
[81/122] Compiling [wayland] glfw/linux_notify.c ...
[82/122] Compiling kitty/rowcolumn-diacritics.c ...
[83/122] Compiling kitty/hyperlink.c ...
[84/122] Compiling [wayland] glfw/wayland-primary-selection-unstable-v1-client-protocol.c ...
[85/122] Compiling kitty/wcswidth.c ...
[86/122] Compiling [wayland] glfw/wayland-pointer-constraints-unstable-v1-client-protocol.c ...
[87/122] Compiling kitty/fast-file-copy.c ...
[88/122] Compiling [wayland] glfw/wayland-text-input-unstable-v3-client-protocol.c ...
[89/122] Compiling [wayland] glfw/wayland-wlr-layer-shell-unstable-v1-client-protocol.c ...
[90/122] Compiling 3rdparty/base64/lib/lib.c ...
[91/122] Compiling [x11] glfw/posix_thread.c ...
[92/122] Compiling [wayland] glfw/posix_thread.c ...
[93/122] Compiling kitty/window_logo.c ...
[94/122] Compiling [wayland] glfw/wayland-xdg-activation-v1-client-protocol.c ...
[95/122] Compiling [wayland] glfw/wayland-xdg-decoration-unstable-v1-client-protocol.c ...
[96/122] Compiling [wayland] glfw/wayland-relative-pointer-unstable-v1-client-protocol.c ...
[97/122] Compiling [wayland] glfw/wayland-cursor-shape-v1-client-protocol.c ...
[98/122] Compiling [wayland] glfw/wayland-fractional-scale-v1-client-protocol.c ...
[99/122] Compiling kitty/glyph-cache.c ...
[100/122] Compiling [wayland] glfw/wayland-viewporter-client-protocol.c ...
[101/122] Compiling kitty/logging.c ...
[102/122] Compiling 3rdparty/base64/lib/arch/neon64/codec.c ...
[103/122] Compiling [wayland] glfw/wayland-single-pixel-buffer-v1-client-protocol.c ...
[104/122] Compiling 3rdparty/base64/lib/tables/tables.c ...
[105/122] Compiling [wayland] glfw/wl_cursors.c ...
[106/122] Compiling 3rdparty/base64/lib/arch/neon32/codec.c ...
[107/122] Compiling [wayland] glfw/wayland-kwin-blur-v1-client-protocol.c ...
[108/122] Compiling 3rdparty/base64/lib/arch/avx/codec.c ...
[109/122] Compiling 3rdparty/base64/lib/arch/ssse3/codec.c ...
[110/122] Compiling 3rdparty/base64/lib/arch/sse42/codec.c ...
[111/122] Compiling 3rdparty/base64/lib/arch/sse41/codec.c ...
[112/122] Compiling 3rdparty/base64/lib/arch/avx2/codec.c ...
[113/122] Compiling kitty/utmp.c ...
[114/122] Compiling 3rdparty/base64/lib/arch/avx512/codec.c ...
[115/122] Compiling 3rdparty/base64/lib/arch/generic/codec.c ...
[116/122] Compiling kitty/cleanup.c ...
[117/122] Compiling [x11] glfw/monotonic.c ...
[118/122] Compiling [wayland] glfw/monotonic.c ...
[119/122] Compiling kitty/monotonic.c ...
[120/122] Compiling kitty/simd-string-128.c ...
[121/122] Compiling kitty/simd-string-256.c ...
[122/122] Compiling kitty/gl-wrapper.c ...
 done
[1/5] Linking kitty/fast_data_types ...
[2/5] Linking [x11] kitty/glfw-x11 ...
[3/5] Linking [wayland] kitty/glfw-wayland ...
[4/5] Linking kittens/transfer/rsync ...
[5/5] Linking launcher ...
 done
kitty/tools/cmd
Build successful. Run kitty as: kitty/launcher/kitty
$ echo "EXIT_CODE=$?"
EXIT_CODE=0
$ ls -l kitty/launcher/kitty kitty/launcher/kitten
-rwxr-xr-x 1 root root 15765764 Jul  6 23:09 kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 Jul  6 23:08 kitty/launcher/kitty
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
   query-terminal
```

*(Complete, unedited output. The trailing `query-terminal` line is the `-A1` trailing-context line — it is the first line of the **next** command entry printed after the `choose_fonts` match, not part of `choose-fonts`. Both the canonical `choose-fonts` command and its hidden `choose_fonts` alias appear because `EntryPoint` registers both — `kittens/choose_fonts/main.go:74-98`.)*

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

### Appendix C — `s` writes the four-key block to STDOUT only (not persisted)

**Producing command.** The harness (Appendix H) drove the real `kitten choose-fonts` to the final pane and sent `s`, delivered as the kitty keyboard-protocol CSI-u sequence `ESC [ 115 ; ; 115 u`. Each run used its own isolated `KITTY_CONFIG_DIRECTORY` created with `tempfile.mkdtemp`:

```python
# steps executed by the driver (Appendix H), TERM=xterm-kitty:
steps = [
    ("Press the Enter key to choose this family", "enter", 1.8),  # family -> faces
    ("select this font",                          "enter", 1.5),  # faces  -> final
    ("What would you like to do",                 "s",     1.2),  # final  -> emit to STDOUT, quit
]
raw_s, status, _, _ = driver.run(steps, cfgdir)
# -> raw bytes: 7689   exit: 0
```

On `s`/`S` the handler stores `output_on_exit = self.settings.serialized() + "\n"` (`kittens/choose_fonts/final.go:101-111`, case at `:104-106`) and quits; `kittens/choose_fonts/main.go:64-66` writes that string to STDOUT at process exit. The **complete** unedited raw tail — a single `repr()` of the byte slice from the alt-screen restore `ESC [ ? 1049 l` to end-of-stream, nothing elided — shows the teardown immediately followed by the four-key block on STDOUT:

```
b'\x1b[?1049l\x1b[?r\x1b[#Q\x1b8font_family      DejaVuSansMono\r\nbold_font        DejaVuSansMono-Bold\r\nitalic_font      DejaVuSansMono-Oblique\r\nbold_italic_font DejaVuSansMono-BoldOblique\r\n'
```

Decoded, the STDOUT block is exactly the four keys (this is `serialized()` output, **not** written to `kitty.conf`):

```
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
```

And the isolated config directory was left with **no** `kitty.conf` (the `s` path does not persist):

```
>>> sorted(os.listdir(KITTY_CONFIG_DIRECTORY))
[]      # empty: 's' emitted to STDOUT and did NOT write kitty.conf
```

### Appendix D — `Esc` aborts (no write)

**Producing command.** The harness drove `kitten choose-fonts` to the final pane, sent `Esc` (`ESC [ 27 u`), then sent `Ctrl+c` to quit — so the capture shows the pane state *after* `Esc`:

```python
# steps executed by the driver (Appendix H), isolated KITTY_CONFIG_DIRECTORY:
steps = [
    ("Press the Enter key to choose this family", "enter",  1.8),  # family -> faces
    ("select this font",                          "enter",  1.5),  # faces  -> final
    ("What would you like to do",                 "esc",    1.5),  # final  -> Esc -> back to faces
    ("select this font",                          "ctrl+c", 1.0),  # confirm we're on faces; quit
]
raw_e, status, snaps, _ = driver.run(steps, cfgdir)
# -> raw bytes: 8346   exit: 256
```

On `Esc` the final pane sets `current_pane = &faces` and redraws (`kittens/choose_fonts/final.go:72-77`) — **no write**. The rendered screen captured immediately after `Esc` (a rendered screen replayed from the byte stream, positioning/color control sequences omitted — the same rendering used in Appendix E, **not** the raw byte stream) is the **faces pane**, proving control returned there rather than proceeding to any write:

```
                                                    DejaVu Sans Mono
Press Enter to select this font, Esc to go back to the font list or any of the highlighted keys below to fine-tune the
appearance of the individual font styles.
Regular: DejaVuSansMono
Bold: DejaVuSansMono-Bold
Italic: DejaVuSansMono-Oblique
Bold-Italic: DejaVuSansMono-BoldOblique
```

Corroborating marker counts over the full captured stream — the final-pane prompt appears exactly once (we entered it once), and the faces-pane prompt appears both before entering the final pane *and* again after `Esc` returned to it:

```
>>> driver.readable(raw_e).count("What would you like to do")   # final-pane prompt
1
>>> driver.readable(raw_e).count("select this font")            # faces-pane prompt (before + after Esc)
3
```

And the isolated config directory was left with **no** `kitty.conf` (`Esc` aborts without writing):

```
>>> sorted(os.listdir(KITTY_CONFIG_DIRECTORY))
[]      # empty: Esc returned to faces and did NOT write kitty.conf
```

### Appendix E — Value-flow capture (family → faces → fine-tune → final)

**Producing command:** the observation harness in Appendix H, run against an isolated `KITTY_CONFIG_DIRECTORY`, with the step list shown in Q3c. It drives the **real** `kitten choose-fonts` entry point under a PTY. The complete raw capture was **9674 bytes** (all redraw frames and the binary APC kitty-graphics image payloads). Each block below is a **rendered screen** replayed from that byte stream via the small VT replayer in Appendix H — i.e. the terminal cursor-movement/color control sequences and the APC graphics image payloads are not shown (they contain no font-selection text); the visible text is exactly what the screen displayed. These are complete rendered frames, **not** de-duplicated text and **not** the raw byte stream.

**Frame 1 — family-list pane** (the family list on the left, the live preview panel on the right; `>` marks the pre-highlighted family):

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
                       ║
                       ║
                       ║
                       ║
Family:
```

**Frame 2 — faces pane** (regular/bold/italic/bold-italic previews; the four `faces_settings` fields):

```
                                                    DejaVu Sans Mono
Press Enter to select this font, Esc to go back to the font list or any of the highlighted keys below to fine-tune the
appearance of the individual font styles.
Regular: DejaVuSansMono
Bold: DejaVuSansMono-Bold
Italic: DejaVuSansMono-Oblique
Bold-Italic: DejaVuSansMono-BoldOblique
```

**Frame 3 — regular-face fine-tune panel** (after pressing `R`):

```
                                             DejaVu Sans Mono: Regular face
Press Enter to accept any changes or Esc to cancel. Click on a style name below to switch to it.
Current setting: DejaVuSansMono
Styles: Bold, Bold Oblique, Book, Oblique
─────────────────────────────────────────────────────── preview ───────────────────────────────────────────────────────
```

**Frame 4 — final pane** (after pressing `Enter` at the faces pane):

```
You have chosen the DejaVu Sans Mono family
What would you like to do?
Enter to modify kitty.conf and use the new fonts
Esc to abort and return to font selection
s to write the new font settings to STDOUT
Ctrl+c to quit
```

After the final pane, `Ctrl+c` was sent; the isolated config directory contained **no** `kitty.conf` afterwards (Ctrl+c quits without writing — see Q3d and Appendix D).

### Appendix F — `--reload-in` variants: SIGUSR1 vs on-disk write (real kitty GUI target)

The `--reload-in` variants were exercised against a **real kitty GUI process**, not a synthetic stand-in. A genuine kitty GUI was launched under Xvfb and its PID captured; its command line satisfies `is_kitty_gui_cmdline` (`tools/config/api.go:282-303` — argv[0] basename `kitty`, and `argv[1]` is `-o`, i.e. not a `@`/`+` subcommand):

```
$ DISPLAY=:99 kitty/launcher/kitty -o allow_remote_control=yes \
      --listen-on unix:/tmp/blitzy_obs/kitty_gui.sock sh -c 'sleep 600' &
$ K=127748    # its PID
$ tr '\0' ' ' < /proc/$K/cmdline
/tmp/.../kitty/launcher/kitty -o allow_remote_control=yes --listen-on unix:/tmp/blitzy_obs/kitty_gui.sock sh -c sleep 600
```

**Making the reload observable.** The GUI's `kitty.conf` was seeded with a deliberately unknown key so that *every* config (re)parse logs a line to the GUI's stderr. Because `SIGUSR1` makes the C child-monitor set `reload_config` (`kitty/child-monitor.c:1373-1374`) and call `load_config_file` (`kitty/child-monitor.c:534-535`), each delivered `SIGUSR1` re-parses the seeded config and appends another "Ignoring unknown config key" line — so an **increment in that count directly observes that the real GUI received `SIGUSR1` and reloaded**. A manual `SIGUSR1` confirms the method:

```
$ cat $KITTY_CONFIG_DIRECTORY/kitty.conf                 # seed
totally_bogus_option_xyz 123
font_family monospace
$ grep -c "Ignoring unknown config key" kitty_gui.log    # after startup
1
$ kill -USR1 $K ; sleep 3                                 # manual SIGUSR1 forces a reload
$ grep -c "Ignoring unknown config key" kitty_gui.log
2
```

Then, for each variant, the config was reset to the seed (no font block, so the kitten's write is always a real change → `updated == true`), and the **real** `kitten choose-fonts --reload-in <variant>` was driven to `Enter` with `KITTY_PID` pointed at the real GUI:

```
=== --reload-in parent ===
  full invocation : KITTY_PID=127748 KITTY_CONFIG_DIRECTORY=/tmp/blitzy_obs/guiCfg_9s8e kitten choose-fonts --reload-in parent  (then drive Enter)
  kitten exit     : 0
  bad-key reload log count  before=2  after=3  delta=1
  => SIGUSR1 delivered to real kitty GUI (reload observed)?  YES
  => kitty.conf written with # BEGIN_KITTY_FONTS block?      YES

=== --reload-in all ===
  full invocation : KITTY_PID=127748 KITTY_CONFIG_DIRECTORY=/tmp/blitzy_obs/guiCfg_9s8e kitten choose-fonts --reload-in all  (then drive Enter)
  kitten exit     : 0
  bad-key reload log count  before=3  after=4  delta=1
  => SIGUSR1 delivered to real kitty GUI (reload observed)?  YES
  => kitty.conf written with # BEGIN_KITTY_FONTS block?      YES

=== --reload-in none ===
  full invocation : KITTY_PID=127748 KITTY_CONFIG_DIRECTORY=/tmp/blitzy_obs/guiCfg_9s8e kitten choose-fonts --reload-in none  (then drive Enter)
  kitten exit     : 0
  bad-key reload log count  before=4  after=4  delta=0
  => SIGUSR1 delivered to real kitty GUI (reload observed)?  NO
  => kitty.conf written with # BEGIN_KITTY_FONTS block?      YES
```

The corresponding GUI-side log (timestamps are seconds since GUI start; `[16.455]` is the manual sanity check, `[86.633]` is `parent`, `[97.285]` is `all`; `none` adds no line):

```
[0.053] Ignoring unknown config key: totally_bogus_option_xyz
[0.155] Failed to open systemd user bus with error: Connection refused
[16.455] Ignoring unknown config key: totally_bogus_option_xyz
[86.633] Ignoring unknown config key: totally_bogus_option_xyz
[97.285] Ignoring unknown config key: totally_bogus_option_xyz
```

**Conclusion:** `parent` and `all` each delivered a real `SIGUSR1` to the real kitty GUI (observed reload); `none` delivered none. In **all three** cases the `# BEGIN_KITTY_FONTS` block was written to `kitty.conf` — confirming that persistence is independent of the reload signal.

### Appendix G — Persistence: `reread.py` and the restart re-read

**`reread.py`** — the exact script used for the restart proof. It resolves the config path the same way kitty does at startup (`kitty.constants.defconf`, which honors `KITTY_CONFIG_DIRECTORY`) and re-reads it with kitty's own loader (`kitty.config.load_config`):

```python
# Run inside a FRESH kitty process:  kitty +runpy "$(cat reread.py)"
# Mimics a real restart: resolves the config path exactly as kitty does at
# startup (kitty.constants.defconf, which honors KITTY_CONFIG_DIRECTORY) and
# re-reads it with kitty's own loader (kitty.config.load_config).
import os
from kitty.constants import config_dir, defconf
from kitty.config import load_config
opts = load_config(defconf)
print("config_dir       =", config_dir)
print("defconf          =", defconf)
print("config_paths     =", opts.config_paths)
print("font_family      =", opts.font_family)
print("bold_font        =", opts.bold_font)
print("italic_font      =", opts.italic_font)
print("bold_italic_font =", opts.bold_italic_font)
```

**Restart re-read against the persisted config** (fresh process, same `KITTY_CONFIG_DIRECTORY` as the `Enter` run in Q4 §"BEFORE → AFTER → restart"):

```
$ kitty/launcher/kitty +runpy "$(cat reread.py)"
config_dir       = /tmp/blitzy_obs/pA_b0mlt5ao
defconf          = /tmp/blitzy_obs/pA_b0mlt5ao/kitty.conf
config_paths     = ('/tmp/blitzy_obs/pA_b0mlt5ao/kitty.conf',)
font_family      = FontSpec(family='', style='', postscript_name='', full_name='', system='DejaVuSansMono', axes=(), variable_name='', created_from_string='DejaVuSansMono')
bold_font        = FontSpec(family='', style='', postscript_name='', full_name='', system='DejaVuSansMono-Bold', axes=(), variable_name='', created_from_string='DejaVuSansMono-Bold')
italic_font      = FontSpec(family='', style='', postscript_name='', full_name='', system='DejaVuSansMono-Oblique', axes=(), variable_name='', created_from_string='DejaVuSansMono-Oblique')
bold_italic_font = FontSpec(family='', style='', postscript_name='', full_name='', system='DejaVuSansMono-BoldOblique', axes=(), variable_name='', created_from_string='DejaVuSansMono-BoldOblique')
```

The full BEFORE/AFTER transcript (config-dir listings, `cat kitty.conf`, and the seeded-prior-file `.bak`/`diff` run) is shown inline in **Q4 §"Observed: BEFORE → AFTER → restart"** and **Q4 §"Observed: the `.bak` backup and prior-key handling"**; the real config-directory paths (`/tmp/blitzy_obs/pA_b0mlt5ao`, `/tmp/blitzy_obs/pB_…`) and complete `FontSpec` values appear there verbatim.

---

### Appendix H — The observation harness (reproducible)

Every runtime capture in this document was produced by driving the **real** `kitten choose-fonts` entry point under a pseudo-terminal (PTY) with two small, self-contained Python scripts. They are reproduced here in full so every result above is reproducible from this document alone. (Per the read-only scope, these scripts were created only for observation and were removed from the working tree afterwards; their text is preserved here.)

`driver.py` runs `kitten choose-fonts` under a PTY with `TERM=xterm-kitty`, answers kitty's `kitty-query-*` terminfo handshake (`kitty/terminfo.py:520-560`) with the **byte-identical default-kitty values** (`foreground=#dddddd`, `background=#000000` per `kitty/options/definition.py:1459,1464`; `font_size=11`; logical `dpi_x=dpi_y=96` per `kittens/query_terminal/main.py`), sends kitty keyboard-protocol CSI-u key events, synchronizes on on-screen text markers, and captures the complete raw byte stream:

```python
"""Reusable PTY driver for `kitten choose-fonts` (the REAL entry point).

Runs the kitten under a PTY with TERM=xterm-kitty, answers kitty's
`kitty-query-*` terminfo handshake with the byte-identical values a REAL
default kitty terminal returns, sends kitty keyboard-protocol CSI-u key
events, synchronizes on on-screen text markers, and captures the COMPLETE
raw byte stream produced by the kitten.
"""
import os, sys, pty, select, time, fcntl, termios, struct, re
sys.path.insert(0, '/tmp/blitzy_obs')
import term_render

REPO = "/tmp/blitzy/kitty/blitzy-fa84ef14-7004-4983-831b-c229f20c11d9_5a2778"
KITTEN = REPO + "/kitty/launcher/kitten"
EXTRA_ARGS = []

# kitty keyboard-protocol CSI-u encodings (kitten enables report-all-keys via CSI>29u)
KEYS = {
    "enter":  b"\x1b[13u",
    "esc":    b"\x1b[27u",
    "s":      b"\x1b[115;;115u",
    "S":      b"\x1b[83;1;83u",
    "r":      b"\x1b[114;;114u",
    "b":      b"\x1b[98;;98u",
    "i":      b"\x1b[105;;105u",
    "o":      b"\x1b[111;;111u",
    "ctrl+c": b"\x1b[99;5u",
}

# Canonical terminal query responses — byte-identical to what a REAL default
# kitty terminal returns: foreground '#dddddd' / background '#000000'
# (kitty/options/definition.py:1459,1464), font_size 11, logical DPI 96
# (kittens/query_terminal/main.py get_result). These are the standard
# terminal-side answers to kitty's kitty-query-* handshake (kitty/terminfo.py:520-560).
QUERY_VALUES = {
    "font_size": "11", "dpi_x": "96", "dpi_y": "96",
    "foreground": "#dddddd", "background": "#000000",
}
_QUERY_RE = re.compile(rb"\x1bP\+q([0-9a-fA-F;]*?)(?:\x07|\x1b\\)")

def _query_response(payload_hex):
    out = b""
    for enc in payload_hex.split(b";"):
        try:
            name = bytes.fromhex(enc.decode("ascii")).decode("utf-8")
        except Exception:
            continue
        if name.startswith("kitty-query-"):
            key = name[len("kitty-query-"):]
            val = QUERY_VALUES.get(key)
            if val is not None:
                out += b"\x1bP1+r" + enc + b"=" + val.encode("utf-8").hex().encode("ascii") + b"\x1b\\"
            else:
                out += b"\x1bP0+r" + enc + b"\x1b\\"
    return out

def set_winsize(fd, rows, cols):
    fcntl.ioctl(fd, termios.TIOCSWINSZ, struct.pack("HHHH", rows, cols, 0, 0))

def readable(raw):
    txt = raw.decode("utf-8", "replace")
    txt = re.sub(r'\x1b_G[^\x1b]*\x1b\\', '', txt)            # APC kitty graphics
    txt = re.sub(r'\x1bP[^\x1b]*\x1b\\', '', txt)             # DCS
    txt = re.sub(r'\x1b\][^\x07\x1b]*(\x07|\x1b\\)', '', txt) # OSC
    txt = re.sub(r'\x1b\[[0-9;?>=!]*[A-Za-z]', '', txt)       # CSI
    txt = re.sub(r'\x1b[=>()#][0-9A-Za-z]?', '', txt)
    txt = re.sub(r'\x1b.', '', txt)
    txt = txt.replace('\x0f','').replace('\x0e','').replace('\r','')
    return txt

def run(steps, cfgdir, rows=40, cols=120, extra_env=None, capture_path=None, drain=3.0):
    """steps: list of (wait_marker_or_None, key_or_None, settle_seconds)."""
    os.makedirs(cfgdir, exist_ok=True)
    pid, fd = pty.fork()
    if pid == 0:
        os.environ["TERM"] = "xterm-kitty"
        os.environ["KITTY_CONFIG_DIRECTORY"] = cfgdir
        os.environ["TMPDIR"] = "/tmp/kitty_tmp"
        if extra_env:
            os.environ.update(extra_env)
        os.execv(KITTEN, [KITTEN, "choose-fonts"] + EXTRA_ARGS)
        os._exit(1)
    set_winsize(fd, rows, cols)
    captured = bytearray()
    answered = 0  # bytes already scanned for queries
    snapshots = []

    def read_once(t=0.2):
        nonlocal answered
        r,_,_ = select.select([fd], [], [], t)
        if r:
            try:
                d = os.read(fd, 65536)
            except OSError:
                return False
            if not d:
                return False
            captured.extend(d)
            # answer any query in the newly-arrived + small overlap window
            start = max(0, answered - 8)
            for m in _QUERY_RE.finditer(bytes(captured[start:])):
                os.write(fd, _query_response(m.group(1)))
            answered = len(captured)
        return True

    def pump(sec):
        end = time.time() + sec
        while time.time() < end:
            if read_once(0.1) is False:
                return

    def wait_marker(marker, maxwait=20):
        end = time.time() + maxwait
        while time.time() < end:
            if read_once(0.2) is False:
                return False
            if marker in readable(bytes(captured)):
                return True
        return False

    for marker, key, settle in steps:
        if marker is not None:
            if not wait_marker(marker):
                sys.stderr.write("MARKER NOT FOUND: %r\n" % marker)
        pump(settle)
        snapshots.append((marker, key, term_render.render(bytes(captured), rows, cols)))
        if key is not None:
            os.write(fd, KEYS[key])
        pump(0.3)

    end = time.time() + drain
    while time.time() < end:
        if read_once(0.2) is False:
            break
    try: os.close(fd)
    except OSError: pass
    status = None
    try:
        _, status = os.waitpid(pid, 0)
    except OSError:
        pass
    raw = bytes(captured)
    final_frame = term_render.render(raw, rows, cols)
    if capture_path:
        with open(capture_path, "wb") as f:
            f.write(raw)
    return raw, status, snapshots, final_frame
```

`term_render.py` is a minimal VT replayer used only to turn a captured raw byte stream into the final rendered screen text shown in the frame captures (it does not touch the kitten; it only replays already-captured bytes):

```python
"""Minimal VT screen renderer: replays a raw terminal byte stream into a
rows x cols cell grid and returns the final rendered screen as text. Handles
CUP/HVP cursor positioning, ED/EL erases, cursor moves, CR/LF, and printable
text; strips SGR/OSC/DCS/APC. Good enough to reproduce the kitten's redrawn
screens (the loop framework redraws with absolute cursor moves)."""
import re, sys

class Screen:
    def __init__(self, rows=40, cols=120):
        self.rows, self.cols = rows, cols
        self.grid = [[" "]*cols for _ in range(rows)]
        self.r = self.c = 0
    def _clamp(self):
        self.r = max(0, min(self.rows-1, self.r))
        self.c = max(0, min(self.cols-1, self.c))
    def put(self, ch):
        if self.c >= self.cols:
            return
        self.grid[self.r][self.c] = ch
        self.c += 1
    def text(self):
        return "\n".join("".join(row).rstrip() for row in self.grid)

def render(raw, rows=40, cols=120):
    s = Screen(rows, cols)
    i, n = 0, len(raw)
    txt = raw.decode("utf-8", "replace")
    n = len(txt)
    while i < n:
        ch = txt[i]
        if ch == "\x1b":
            if i+1 < n and txt[i+1] == "[":
                m = re.match(r"\x1b\[([0-9;?>=!]*)[ -/]*([A-Za-z@])", txt[i:])
                if m:
                    params, cmd = m.group(1), m.group(2)
                    nums = [int(x) for x in params.split(";") if x.isdigit()] if params and not params.startswith("?") else []
                    if cmd in ("H", "f"):
                        r = (nums[0]-1) if len(nums) >= 1 else 0
                        c = (nums[1]-1) if len(nums) >= 2 else 0
                        s.r, s.c = r, c; s._clamp()
                    elif cmd == "A": s.r -= (nums[0] if nums else 1); s._clamp()
                    elif cmd == "B": s.r += (nums[0] if nums else 1); s._clamp()
                    elif cmd == "C": s.c += (nums[0] if nums else 1); s._clamp()
                    elif cmd == "D": s.c -= (nums[0] if nums else 1); s._clamp()
                    elif cmd == "G": s.c = (nums[0]-1) if nums else 0; s._clamp()
                    elif cmd == "d": s.r = (nums[0]-1) if nums else 0; s._clamp()
                    elif cmd == "J":
                        mode = nums[0] if nums else 0
                        if mode == 2 or mode == 3:
                            s.grid = [[" "]*s.cols for _ in range(s.rows)]
                        elif mode == 0:
                            for c in range(s.c, s.cols): s.grid[s.r][c] = " "
                            for r in range(s.r+1, s.rows): s.grid[r] = [" "]*s.cols
                    elif cmd == "K":
                        mode = nums[0] if nums else 0
                        if mode == 0:
                            for c in range(s.c, s.cols): s.grid[s.r][c] = " "
                        elif mode == 1:
                            for c in range(0, s.c+1): s.grid[s.r][c] = " "
                        else:
                            s.grid[s.r] = [" "]*s.cols
                    i += m.end(); continue
                else:
                    i += 2; continue
            elif i+1 < n and txt[i+1] == "]":  # OSC ... BEL or ST
                m = re.match(r"\x1b\][^\x07\x1b]*(\x07|\x1b\\)", txt[i:])
                if m: i += m.end(); continue
                i += 2; continue
            elif i+1 < n and txt[i+1] in "P_^":  # DCS/APC/PM ... ST
                m = re.match(r"\x1b[P_^].*?\x1b\\", txt[i:], re.S)
                if m: i += m.end(); continue
                i += 2; continue
            else:
                i += 2; continue
        elif ch == "\r":
            s.c = 0; i += 1; continue
        elif ch == "\n":
            s.r += 1; s._clamp(); i += 1; continue
        elif ch in ("\x0f", "\x0e", "\x07", "\x08"):
            i += 1; continue
        else:
            if ch >= " ":
                s.put(ch)
            i += 1; continue
    return s.text()

if __name__ == "__main__":
    raw = open(sys.argv[1], "rb").read()
    print(render(raw))
```

Typical invocation (used, with different step lists, throughout):

```python
import sys; sys.path.insert(0, ".")
import driver, tempfile
cfg = tempfile.mkdtemp(prefix="obs_")
steps = [
    ("Press the Enter key to choose this family", "enter", 2.0),
    ("select this font", "enter", 1.5),
    ("What would you like to do", "enter", 1.0),   # Enter -> persist (or "s"/"esc"/"ctrl+c")
]
raw, status, snapshots, final_frame = driver.run(steps, cfg, capture_path="cap.bin")
```

## Coverage checklist

- **Q1** — build (`./dev.sh build`) + default launch (`kitty/launcher/kitty`) + observed banner `0.35.2` ✓ (Q1, Appendix A)
- **Q2** — real entry point `kitten choose-fonts` + `--help`; alternate `kitty/fonts/list.py:41-42`; remote-control values flagged **[non-canonical]** ✓ (Q2, Appendix B)
- **Q3a** — registration (`tools/cmd/tool/main.go:82`, `kittens/choose_fonts/main.go:74-98`) ✓
- **Q3b** — `--reload-in`, default `parent`, choices `parent, all, none` (`kittens/choose_fonts/main.go:86-95`) ✓
- **Q3c** — value flow (`faces_settings`, pane hand-offs, Go↔Python backend) ✓ (Q3c, Appendix E)
- **Q3d** — finalization: four actions (`kittens/choose_fonts/final.go:38-44`) + `serialized()` (`kittens/choose_fonts/final.go:63-70`) ✓
- **Q4** — persistence PROVEN: BEFORE (no block) / AFTER (`# BEGIN_KITTY_FONTS`) / `.bak` / restart re-read; condition matrix for `Enter`/`s`/`Esc`/`--reload-in parent|all|none`; `--config NONE` caveat ✓
- **Mechanism** — `Patcher.Patch` sentinel-block algorithm, `ConfigDir` resolution, `SIGUSR1` reload (distinct from `load_config`) ✓

