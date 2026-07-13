# `choose-fonts` kitten — end‑to‑end behavior and font‑persistence, verified at runtime

**Repository:** `kovidgoyal/kitty`  **Branch:** `kitty_815df1e210e0`  **Commit:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`

This document answers an onboarding question about the `choose-fonts` kitten:

1. How do I build kitty and start a single default instance?
2. How do I invoke the `choose-fonts` kitten from that instance?
3. What does the kitten do end‑to‑end — how is it registered, how are its options parsed, how do those values flow through the program, and what happens when the selection is finalized?
4. **When I press Enter at the final confirmation step, does kitty remember that font choice next time it opens (persistent), or does it only change the current session (ephemeral)?**

Every behavioral claim below was produced by **building and running the real code path** (`kitten choose-fonts`) and capturing the actual, unedited output. Each claim is grounded in a `file:line` citation at commit `815df1e2` and labeled **[observed]** (captured at runtime) or **[inferred]** (derived from reading the code).

---

## TL;DR — the direct answer

**Pressing `Enter` at the final confirmation pane PERSISTS the font choice across restarts. [observed]**

Pressing `Enter` runs `final_pane.on_key_event` (`kittens/choose_fonts/final.go:78-97`), which calls `config.Patcher.Patch` (`tools/config/api.go:310`) to **write the four font keys** (`font_family`, `bold_font`, `italic_font`, `bold_italic_font`) into `kitty.conf` **on disk**, wrapped in a `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` sentinel block, via an atomic file update (`tools/config/api.go:347`). Because the choice lives in `kitty.conf`, a brand‑new kitty process reads it back — proven here two ways: a fresh `load_config` returns the saved family, and re‑opening the kitten on a new instance pre‑selects the saved family.

The `SIGUSR1` live‑reload that `Enter` also triggers (`config.ReloadConfigInKitty`, `tools/config/api.go:352-371`) is only a **convenience** that applies the change to already‑running instances immediately; it is **not** the persistence mechanism. The persistence comes entirely from the on‑disk `kitty.conf` write.

The **session‑only alternative** is the `s` / `S` key (`kittens/choose_fonts/final.go:101-111`): it prints the same four lines to **STDOUT only** and leaves `kitty.conf` untouched. **[observed]**

---

## 0. Method, environment, and conventions

### 0.1 How this was verified (canonical, real entry point)

- All building and running was performed in the user‑provided canonical container (image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`), which ships the kitty repository at commit `815df1e2` under `/app` and the toolchain (Go, C compiler, Python, system libraries).
- The kitten was always invoked through its **real entry point** — the shell command `kitten choose-fonts` typed inside a running kitty window — never through a remote‑control hook, debug bypass, or synthetic stand‑in.
- Because `choose-fonts` is an interactive terminal UI, a headless X server (`Xvfb :99`) hosted one real kitty GUI window, and key events (family filter text, `Enter`, `Esc`, `s`, `Ctrl+c`) were delivered as **real keystrokes** via `xdotool`. kitty remote control (`kitten @ get-text`) was used **only to read the screen** for capture — it never drove or replaced the entry point.
- Configuration was **isolated**: before every launch a pristine temporary directory was created and exported as `KITTY_CONFIG_DIRECTORY`, so `kitty.conf` started empty and before/after diffs are clean. This works because `ConfigDirForName` honors `KITTY_CONFIG_DIRECTORY` first (`tools/utils/paths.go:88-91`) and `ConfigDir` is memoized once per process (`tools/utils/paths.go:132-134`), so the variable must be set **before** each `kitty`/`kitten` process starts.

### 0.2 Observed toolchain

```
### CMD: git rev-parse --abbrev-ref HEAD
HEAD
### CMD: git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
### CMD: go version
go version go1.23.4 linux/amd64
### CMD: python3 --version
Python 3.12.3
### CMD: gcc --version
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
```

The repository declares `go 1.22` (`go.mod:3`) and `requires-python = ">=3.8"` (`pyproject.toml:2`); the container satisfies both with Go 1.23.4 and Python 3.12.3. **[observed]**

### 0.3 Legend

- **[observed]** — the claim is backed by captured runtime output shown in this document.
- **[inferred]** — the claim is derived from reading the source at commit `815df1e2`; where a runtime confirmation exists it is noted.
- Absent documents are not cited. `docs/kittens/choose-fonts.rst` does **not** exist at this commit, and `kittens/choose_fonts/__init__.py` / `kittens/choose_fonts/main.py` are 0‑byte package markers with no logic. Official upstream documentation is referenced by URL only (see §7).

---

## R1 — Build kitty and launch a single default instance

### R1.1 Canonical build command

The canonical from‑source build is **`./dev.sh build`** — `dev.sh:9` is `exec go run bypy/devenv.go "$@"`, and the build docs state a C compiler and the go compiler are required, with `./dev.sh build` as the build command (`docs/build.rst:14-19`; dependency list `docs/build.rst:83-92`). **[observed / inferred]** (command grounded in `dev.sh:9`; prerequisites in `docs/build.rst`.)

Running the bare canonical command in this container **failed** because of environmental dependency drift — it must be reported exactly as observed. (In the two build transcripts below, a bare `...` on its own line denotes omitted **repetitive per‑file compile‑progress lines** of the form `[N/122] Compiling …`; every claim‑bearing line — the errors, the `Build successful` line, the `choose_fonts` target, the exit code, and the duration — is shown verbatim.)

```
### CMD: ./dev.sh build
Downloading linux-64.tar.xz
Dependencies downloaded. Now build kitty with: ./dev.sh build
[1/24] Generating wayland-xdg-shell-client-protocol.h ...
[2/24] Generating wayland-xdg-shell-client-protocol.c ...
[3/24] Generating wayland-viewporter-client-protocol.h ...
...
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
The following build command failed: /app/dependencies/linux-amd64/bin/python setup.py develop
exit status 1
### BUILD EXIT CODE: 1
### BUILD DURATION SECONDS: 13
```

**Cause [observed]:** `./dev.sh build` freshly downloaded a *newer* `linux-64.tar.xz` dependency bundle whose Wayland `wayland-protocols` now generates an `xdg-shell` header with new `XDG_TOPLEVEL_STATE_CONSTRAINED_*` enum values. The pinned `glfw/wl_window.c` at commit `815df1e2` does not handle them in its `switch` (line 668), and kitty's default build compiles C with `-pedantic-errors -Werror` (`setup.py:491`, `setup.py:1231`), so the unhandled‑enum warning is promoted to a fatal error. This is **environment drift, not a defect at this commit** — the container's own pre‑built `kitty`/`kitten` 0.35.2 binaries were produced from this very commit and work. Per the read‑only rule, no repository source was modified.

### R1.2 Completing the build without modifying any source

kitty's build system provides a supported flag that relaxes `-Werror` (it sets `werror=''`, `setup.py:491`/`setup.py:1231`; the CLI switch is declared at `setup.py:2003-2004` and forwarded to `setup.py develop` by `bypy/devenv.go:374`). Using it changes **only** the warning‑as‑error policy — no source, and no kitty runtime configuration:

```
### CMD: ./dev.sh build --ignore-compiler-warnings
[1/122] Compiling kitty/screen.c ...
[2/122] Compiling kitty/unicode-data.c ...
[3/122] Compiling [wayland] glfw/wl_window.c ...
[4/122] Compiling [x11] glfw/x11_window.c ...
[5/122] Compiling kitty/glfw.c ...
...
kitty/kittens/choose_fonts
kitty/kittens/transfer
kitty/tools/cmd/pytest
kitty/kittens/diff
kitty/tools/cmd/tool
kitty/tools/cmd/completion
kitty/tools/cmd
Build successful. Run kitty as: kitty/launcher/kitty
### BUILD EXIT CODE: 0
### BUILD DURATION SECONDS: 27
```

The build compiled all 122 C translation units and linked the Go tools, including **`kitty/kittens/choose_fonts`** (the kitten under investigation) and `kitty/tools/cmd/tool` (the kitten CLI root), finishing with `Build successful. Run kitty as: kitty/launcher/kitty`. **[observed]**

### R1.3 Produced launcher binaries

```
### CMD: ls -l kitty/launcher/kitty kitty/launcher/kitten
-rwxr-xr-x 1 root 1001 15945988 Jul 13 17:14 kitty/launcher/kitten
-rwxr-xr-x 1 root 1001    36224 Jul 13 17:14 kitty/launcher/kitty
### CMD: kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
### CMD: kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
```

A single default instance is launched from **`kitty/launcher/kitty`** (no custom configuration). The companion **`kitty/launcher/kitten`** binary provides the `kitten` command used to invoke the kitten in R2. **[observed]**

---

## R2 — Invoke the `choose-fonts` kitten

From inside the running default instance, the kitten is invoked with the shell command **`kitten choose-fonts`**. Typing it (via real keystrokes) into a kitty window backed by a pristine `KITTY_CONFIG_DIRECTORY` produced the kitten's family‑list UI:

```
CFG=/tmp/cf_kittyconf.rVAcVE SOCK=/tmp/cf_r2.sock
WID=2097164
### CMD (inside the running kitty instance): kitten choose-fonts
======== get-text: choose-fonts family-list UI ========
>DejaVu Sans Mono       ║               DejaVu Sans Mono
 Symbols Nerd Font Mono ║
                        ║ Styles: Bold, Bold Oblique, Book, Oblique
                        ║
                        ║ Press the Enter key to choose this family
                        ║
                        ║ ────────────────── preview ──────────────────
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

The UI appeared, so the subcommand is registered and runs. The `>` marks the currently selected family; the two monospaced families come from the Python backend's font enumeration (§R3.4); `Press the Enter key to choose this family` is the list→faces hand‑off prompt (§R3.3). **[observed]** This is the `FontList` pane defined in `kittens/choose_fonts/list.go`.

---

## R3 — End‑to‑end behavior with captured output

### R3.1 (a) Subcommand registration and option parsing

**Registration [observed / inferred].** The kitten is registered on the kitten‑CLI root by `tools/cmd/tool/main.go:82`, which calls `choose_fonts.EntryPoint(root)` (import at `tools/cmd/tool/main.go:9`). `EntryPoint` is defined at `kittens/choose_fonts/main.go:74-99`; it adds a `cli.Command{Name: "choose-fonts", …}` (`kittens/choose_fonts/main.go:76`) whose `Run` closure builds `opts := Options{}` (`:79`), calls `cmd.GetOptionValues(&opts)` (`:80`), and then calls `main(&opts)` (`:83`). The option struct is `type Options struct { Reload_in string }` (`kittens/choose_fonts/main.go:70-72`).

**Option parsing [observed].** The only option is `--reload-in`, declared as an `OptionSpec` at `kittens/choose_fonts/main.go:86-95` (`Dest:"Reload_in"` `:88`; `Type:"choices"` `:89`; `Choices:"parent, all, none"` `:90`; `Default:"parent"` `:91`). The `--help` output confirms the choices and default exactly:

```
### CMD: kitten choose-fonts --help
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

**Visible `choose_fonts` alias (nuance).** In addition to the primary `choose-fonts` name, `EntryPoint` clones the command and sets `clone.Hidden = false` (`kittens/choose_fonts/main.go:97`) and `clone.Name = "choose_fonts"` (`:98`). The underscore spelling is therefore a **visible** alias — not a hidden one. Both spellings appear on the CLI root listing and both accept `--help`:

```
### CMD: kitten choose_fonts --help   (underscore alias; clone.Hidden=false)
Usage: kitten choose_fonts 

Choose the fonts used in kitty

Options:

### CMD: kitten --help  (grep both choose-fonts spellings on the CLI root listing)
   choose-fonts
    Choose the fonts used in kitty
   choose_fonts
    Choose the fonts used in kitty
```

**[observed]** — the clone at `kittens/choose_fonts/main.go:96-98` is a visible alias.

### R3.2 (b) Option / data flow

The parsed `opts` flow as follows, ending at the finalize step **[observed for the endpoints, inferred for the wiring at the cited lines]**:

1. `cmd.GetOptionValues(&opts)` fills `opts` in the `Run` closure — `kittens/choose_fonts/main.go:80`.
2. The closure calls `main(&opts)` — `kittens/choose_fonts/main.go:83`; `func main(opts *Options)` is at `kittens/choose_fonts/main.go:16`.
3. `main` builds the event loop and stores the options on the handler: `h := &handler{lp: lp, opts: opts}` — `kittens/choose_fonts/main.go:35`. The `handler` struct and its `opts *Options` field are at `kittens/choose_fonts/ui.go:42-43`.
4. At finalize, the `Enter` branch reads `self.handler.opts.Reload_in` to pick the reload scope — `kittens/choose_fonts/final.go:87-92`.

The `--help` capture above proves the option is parsed; the *effect* of `opts.Reload_in` is observed directly in §5.4 (`parent`/`all`/`none`).


### R3.3 (c) Pane hand‑off chain: `list → faces → [face] → final`

The kitten is a linear wizard. `handler.initialize` (`kittens/choose_fonts/ui.go:77`) queries the terminal (`QueryTerminal("font_size","dpi_x","dpi_y","foreground","background")`, `ui.go:80`), registers the four panes `[]pane{&listing, &faces, &face_pane, &final_pane}` (`ui.go:81`), makes a temp working dir under `utils.CacheDir()` (with the comment "dont use /tmp as it may be mounted in RAM", `ui.go:87-88`), and spawns a goroutine that asks the backend for the font list (`query("list_monospaced_fonts", …)`, `ui.go:95`). `handler.finalize` (`ui.go:104`) removes that temp dir (`ui.go:106`).

**Step 1 — `FontList` pane** (`kittens/choose_fonts/list.go`). The initial screen from R2 (`>DejaVu Sans Mono`, `Symbols Nerd Font Mono`, styles, preview, `Family:` prompt). Pressing `Enter` runs `FontList.on_key_event` (`list.go:246`); on Enter (`:247`) it takes `CurrentFamily()` (`:249`) and calls `self.handler.faces.on_enter(family)` (`list.go:250`).

**Step 2 — `faces` pane** (`kittens/choose_fonts/faces.go`). Reached by that hand‑off:

```
======== R3c step 2: Enter at FontList (list.go:247-250 faces.on_enter) -> FACES pane ========
                           DejaVu Sans Mono
Press Enter to select this font, Esc to go back to the font list or any
of the highlighted keys below to fine-tune the appearance of the
individual font styles.
Regular: DejaVuSansMono
Bold: DejaVuSansMono-Bold
Italic: DejaVuSansMono-Oblique
Bold-Italic: DejaVuSansMono-BoldOblique
```

The four rows are the fields of `faces_settings` (`faces.go:14-16`: `font_family, bold_font, italic_font, bold_italic_font string`). `faces.on_key_event` (`faces.go:112`): `Esc` returns to the `listing` pane (`faces.go:113-116`); `Enter` calls `self.handler.final_pane.on_enter(self.family, self.settings)` (`faces.go:118-120`). The "highlighted keys" are handled in `faces.on_text` (`faces.go:125`): `r/R`→`font_family` (`:129-130`), `b/B`→`bold_font`, `i/I`→`italic_font`, `o/O`→`bold_italic_font`, each calling `self.handler.face_pane.on_enter(self.family, which, self.settings)` (`faces.go:139`) to open the optional fine‑tune pane. **[observed]** (the pane rendered; the key handlers are cited from source.)

**Step 2b — `face_panel` (optional fine‑tune).** `kittens/choose_fonts/face.go` implements variable‑axis/style adjustment; it is reached only when one of `r/b/i/o` is pressed on the faces pane. It is not on the default Enter→Enter happy path and was not entered for the persistence proof.

**Step 3 — `final_pane`** (`kittens/choose_fonts/final.go`). Pressing `Enter` on the faces pane opens the final confirmation pane, whose `draw_screen` (`final.go:29-47`) prints the key legend:

```
======== R3c step 3: Enter at FACES (faces.go:118-120 final_pane.on_enter) -> FINAL pane legend (final.go:29-47) ========
You have chosen the DejaVu Sans Mono family
What would you like to do?
Enter to modify kitty.conf and use the new fonts
Esc to abort and return to font selection
s to write the new font settings to STDOUT
Ctrl+c to quit
```

The four legend lines map to `final.go:38` (`Enter`), `:40` (`Esc`), `:42` (`s`), `:44` (`Ctrl+c`). **[observed]**

### R3.4 (d) Go frontend ↔ Python backend bridge

Font enumeration and sample rendering are delegated to a Python backend that the Go frontend spawns. `kitty_font_backend_type.start()` (`kittens/choose_fonts/backend.go:32`) runs `exec.Command(exe, "+runpy", "from kittens.choose_fonts.backend import main; main()")` (`kittens/choose_fonts/backend.go:41`) and exchanges JSON over stdio (decoder created at `backend.go:52`, process started at `:53`). The Python side is `main()` at `kittens/choose_fonts/backend.py:150`, which reads JSON lines from stdin (`:153-154`) and dispatches the actions `list_monospaced_fonts` (`:156-158`), `read_variable_data` (`:159-163`), and `render_family_samples` (`:164-166`), raising on an unknown action (`:167-168`).

While the kitten was live, the process tree showed the Go frontend parenting the Python backend:

```
### CMD (inside instance): kitten choose-fonts
======== R3d: Go frontend + Python backend processes (backend.go:41 +runpy → backend.py:150) ========
### CMD: ps -ef | grep -E "choose.fonts|runpy" (excluding grep)
root       30351   30318  0 17:20 pts/0    00:00:00 kitten choose-fonts
root       30369   30351  1 17:20 pts/0    00:00:00 /app/kitty/launcher/kitty +runpy from kittens.choose_fonts.backend import main; main()
```

PID 30369 (the `+runpy` Python backend) has PPID 30351 (the `kitten choose-fonts` Go frontend), confirming the spawn relationship in `backend.go:41` → `backend.py:150`. **[observed]**


---

## R4 — Persistence: does pressing Enter remember the choice next time? (CORE)

**Answer: YES — pressing `Enter` writes the choice to `kitty.conf`, and a fresh kitty reads it back. [observed]**

### R4.1 The mechanism

`final_pane.on_key_event` handles `Enter` at `kittens/choose_fonts/final.go:78-97`:

- `patcher := config.Patcher{Write_backup: true}` — `final.go:80`.
- `path := filepath.Join(utils.ConfigDir(), "kitty.conf")` — `final.go:81`.
- `updated, err := patcher.Patch(path, "KITTY_FONTS", self.settings.serialized(), "font_family", "bold_font", "italic_font", "bold_italic_font")` — `final.go:82`.
- if `updated`, `switch self.handler.opts.Reload_in { case "parent": config.ReloadConfigInKitty(true); case "all": config.ReloadConfigInKitty(false) }` — `final.go:86-93`.
- `self.lp.Quit(0)` — `final.go:94`.

`faces_settings.serialized()` (`kittens/choose_fonts/final.go:63-70`) emits exactly the four lines, each key left‑padded to 17 characters, joined by `\n`. `Patcher.Patch` (`tools/config/api.go:310`) comments out any pre‑existing font lines (regex `(?m)^\s*(font_family|…)\b` → `# $1`, `api.go:325-326`), wraps the content in a `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` sentinel block (`api.go:328-330`), replaces an existing block or appends a new one (`api.go:331-340`), optionally writes a `.bak` backup (`api.go:343-344`), and updates the file atomically via `utils.AtomicUpdateFile` (`api.go:347`). Persistence is the on‑disk write; the optional `SIGUSR1` reload (`config.ReloadConfigInKitty`, `api.go:352-371`) only pushes the change into already‑running instances.

### R4.2 Before → during → after (default `--reload-in parent`)

**BEFORE — pristine, no `kitty.conf`:**

```
############ R4 BEFORE (pristine, --reload-in default=parent) ############
### CMD: ls -la $KITTY_CONFIG_DIRECTORY
total 8
drwx------ 2 root root 4096 Jul 13 17:23 .
drwxrwxrwt 1 root root 4096 Jul 13 17:23 ..
### CMD: cat $KITTY_CONFIG_DIRECTORY/kitty.conf
cat: /tmp/cf_kittyconf.Rg84X9/kitty.conf: No such file or directory
(kitty.conf does not exist yet)
```

**DURING — launch, invoke, drive `list → faces → final`, press `Enter`:**

```
############ launch + invoke kitten choose-fonts (real entry point) ############
WID=2097164
### family list reached (selected family marked with >):
>DejaVu Sans Mono       ║               DejaVu Sans Mono
 Symbols Nerd Font Mono ║
                        ║ Styles: Bold, Bold Oblique, Book, Oblique
### Enter: list->faces (list.go:250)
                           DejaVu Sans Mono
Press Enter to select this font, Esc to go back to the font list or any
### Enter: faces->final (faces.go:120)
### final pane:
You have chosen the DejaVu Sans Mono family
What would you like to do?

############ R4 DURING: press ENTER at final pane (final.go:78-97 Patcher.Patch) ############
```

**AFTER — `kitty.conf` created (190 bytes), no `.bak` on the first write:**

```
############ R4 AFTER (first write) ############
### CMD: ls -la $KITTY_CONFIG_DIRECTORY
total 12
drwx------ 2 root root 4096 Jul 13 17:23 .
drwxrwxrwt 1 root root 4096 Jul 13 17:23 ..
-rw-r--r-- 1 root root  190 Jul 13 17:23 kitty.conf
### CMD: cat $KITTY_CONFIG_DIRECTORY/kitty.conf
-----BEGIN kitty.conf-----
# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTS-----END kitty.conf-----
```

The block written is exactly `serialized()` (`final.go:63-70`, 17‑char padding) inside the sentinel (`api.go:328-330`). No `kitty.conf.bak` was produced because the backup is written only when the file was non‑empty before the write (`len(raw) > 0 && self.Write_backup`, `tools/config/api.go:343`) — here the file did not exist. **[observed]**

### R4.3 The decisive proof — restart and read back

**Restart proof #1 (programmatic).** A brand‑new kitty process, pointed at the same config dir, loads it with kitty's real loader `kitty.config.load_config` (`kitty/config.py:163`); `kitty.constants.config_dir` honors `KITTY_CONFIG_DIRECTORY` (`kitty/constants.py:87-89`) and `defconf` is `<dir>/kitty.conf` (`kitty/constants.py:131-133`):

```
############ R4 RESTART (decisive): a FRESH kitty process reads the saved kitty.conf ############
### CMD: KITTY_CONFIG_DIRECTORY=$CFG kitty +runpy <load_config read-back>
config_dir = /tmp/cf_kittyconf.Rg84X9
defconf = /tmp/cf_kittyconf.Rg84X9/kitty.conf exists= True
font_family = DejaVuSansMono
bold_font = DejaVuSansMono-Bold
italic_font = DejaVuSansMono-Oblique
bold_italic_font = DejaVuSansMono-BoldOblique
```

A fresh process reads back exactly what the kitten wrote. **[observed]**

**Restart proof #2 (behavioral).** After a second finalize that saved a *different* family (`Symbols Nerd Font Mono`, see §5.5), a brand‑new kitty instance re‑opening `kitten choose-fonts` against the saved dir pre‑selects that saved family (note the `>` on `Symbols Nerd Font Mono`, whereas the very first launch on an empty config had `>` on `DejaVu Sans Mono`):

```
### saved kitty.conf that a FRESH kitty will read (after 2nd finalize = Symbols):
# BEGIN_KITTY_FONTS
font_family      family="Symbols Nerd Font Mono"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
############ RESTART (behavioral): brand-new kitty instance opens choose-fonts against saved dir ############
fresh WID=2097164
### get-text: does the re-opened kitten reflect the SAVED font (Symbols Nerd Font Mono)?
 DejaVu Sans Mono       ║ Symbols Nerd Foo
>Symbols Nerd Font Mono ║
                        ║ Styles: Regular
                        ║
                        ║ Press the Enter
                        ║  key to choose
                        ║ this family
                        ║
```

This pre‑selection is produced because the kitten's backend seeds resolved faces from the loaded user config — `kittens/choose_fonts/backend.py:11` imports `create_default_opts` from `kitty.cli`, and `backend.py:157` calls it. `create_default_opts` (`kitty/cli.py:1089-1093`) is a misnomer: it actually runs `load_config(*default_config_paths(()))`, i.e. it loads the **user** `kitty.conf`. **[observed]** Both proofs confirm persistence across restarts; the conclusion is therefore **[observed]**, not merely inferred.


---

## 5. Secondary and edge paths (every condition)

### 5.1 `Esc` at the final pane → back to the faces pane; `kitty.conf` unchanged

`final_pane.on_key_event` handles `Esc` at `kittens/choose_fonts/final.go:73-76`, setting `self.handler.current_pane = &self.handler.faces` (`final.go:75`). (The legend text at `final.go:40` says "abort and return to font selection"; the actual handler returns to the **faces** pane.)

```
WID=2097164
############ EDGE: Esc at final pane (final.go:73-76 -> returns to FACES) ############
### at final pane:
You have chosen the DejaVu Sans Mono family
### press Esc:
### screen after Esc (expect FACES pane, per final.go:75 current_pane=&handler.faces):
                           DejaVu Sans Mono
Press Enter to select this font, Esc to go back to the font list or any
of the highlighted keys below to fine-tune the appearance of the
individual font styles.
### kitty.conf after Esc (expect ABSENT/unchanged):
total 8
drwx------ 2 root root 4096 Jul 13 17:29 .
drwxrwxrwt 1 root root 4096 Jul 13 17:29 ..
cat: /tmp/cf_kittyconf.7qzug6/kitty.conf: No such file or directory
(kitty.conf does not exist — unchanged)
```

**[observed]** — `Esc` returns to the faces pane and does not write `kitty.conf`.

### 5.2 `s` / `S` at the final pane → serialized settings to STDOUT only; `kitty.conf` unchanged

`final_pane.on_text` handles `s`/`S` at `kittens/choose_fonts/final.go:101-111`: it sets `output_on_exit = self.settings.serialized() + "\n"` (`final.go:105`) and calls `self.lp.Quit(0)` (`final.go:106`). On exit, `main` prints `output_on_exit` to `os.Stdout` (`kittens/choose_fonts/main.go:64-66`). Nothing is written to `kitty.conf` — this is the **session‑only / non‑persisting** path.

```
############ EDGE: s at final pane (final.go:101-111 output_on_exit=serialized(); main.go:64-66 -> STDOUT) ############
### currently at faces; Enter -> final
You have chosen the DejaVu Sans Mono family
### press s (writes serialized settings to STDOUT then Quit(0)):
### screen after s (kitten quit; the 4 serialized lines printed to STDOUT, then shell prompt):
root@8c8049ac071a:/app# kitten choose-fonts
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
root@8c8049ac071a:/app#
### kitty.conf after s (expect ABSENT/unchanged — non-persisting path):
total 8
drwx------ 2 root root 4096 Jul 13 17:29 .
drwxrwxrwt 1 root root 4096 Jul 13 17:29 ..
cat: /tmp/cf_kittyconf.7qzug6/kitty.conf: No such file or directory
(kitty.conf does not exist — unchanged)
```

**[observed]** — `s` prints exactly the four `serialized()` lines to STDOUT and leaves `kitty.conf` absent.

### 5.3 `Ctrl+c` at the final pane → `Error: canceled by user`; `kitty.conf` unchanged

There is **no explicit `Ctrl+c` handler** in `final.go` (only the legend text at `final.go:44`). The observed behavior is that the kitten aborts with `Error: canceled by user`:

```
############ EDGE: Ctrl+c at final pane (NUANCE — no explicit handler in final.go; OBSERVE) ############
### at final pane:
You have chosen the DejaVu Sans Mono family
### press Ctrl+c:
### screen after Ctrl+c (OBSERVED behavior):
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
root@8c8049ac071a:/app# kitten choose-fonts
Error: canceled by user
root@8c8049ac071a:/app#
```

`kitty.conf` was unchanged (still absent). This corrects a plausible‑but‑wrong guess: the kitten does **not** print "Killed by signal" here. The event loop returns an error, which `main` reports on the `err != nil` path (`kittens/choose_fonts/main.go:54-57`) — **not** the death‑signal path (`main.go:58-62`, which formats `"Killed by signal: "` from `DeathSignalName`). This is exactly why observation is required over inference. **[observed]**

### 5.4 `--reload-in` = `parent` / `all` / `none`

All three modes **write the file**; they differ only in whether/where a `SIGUSR1` live‑reload is sent. `config.ReloadConfigInKitty` (`tools/config/api.go:352-371`) sends `SIGUSR1` to `$KITTY_PID` when `in_parent_only == true` (`api.go:353-361`), or to every kitty‑GUI process when `false` (`api.go:363-369`, matched by `is_kitty_gui_cmdline`, `api.go:282-303`). The `final.go` switch has cases only for `"parent"` and `"all"` (`final.go:86-93`); `"none"` is absent, so with `--reload-in none` the file is written but no reload is signaled.

On this kernel, Go's runtime delivers the signal via `pidfd_open` + `pidfd_send_signal` (not `kill`/`tgkill`); the trace filter had to include those syscalls to see it. For **`--reload-in parent`** the parent `$KITTY_PID` was `33172`, and the trace shows one `SIGUSR1` sent to a pidfd opened on exactly that PID:

```
parent KITTY_PID=33172
### kitty.conf written? YES
### ALL signal-send syscalls that are NOT SIGURG (the real reload signal, if any):
33329 pidfd_open(33319, 0)              = 3
33329 pidfd_send_signal(3, 0, NULL, 0)  = 0
33329 pidfd_open(33172, 0)              = 3
33319 pidfd_open(33172, 0)              = 17
33319 pidfd_open(33172, 0)              = 18
33319 pidfd_send_signal(18, SIGUSR1, NULL, 0) = 0
### grep for the parent PID 33172 and for USR1 anywhere in the trace:
33329 pidfd_open(33172, 0)              = 3
33319 pidfd_open(33172, 0)              = 17
33319 pidfd_open(33172, 0)              = 18
33319 pidfd_send_signal(18, SIGUSR1, NULL, 0) = 0
### total non-SIGURG signal lines: 6
```

Note `pidfd_send_signal(3, 0, NULL, 0)` uses signal `0` — a liveness/existence check, distinct from the actual `SIGUSR1` reload `pidfd_send_signal(18, SIGUSR1, NULL, 0)`. For **`none`** and **`all`**:

```
=== mode=none : parent KITTY_PID=33388 ===
### kitty.conf written? YES
### SIGUSR1 sends via pidfd_send_signal (the reload signal):
### SIGUSR1 send count = 0  (expect 0 for none, >=1 for parent/all)
====================================================
=== mode=all : parent KITTY_PID=33591 ===
### kitty.conf written? YES
### SIGUSR1 sends via pidfd_send_signal (the reload signal):
33755 pidfd_send_signal(282, SIGUSR1, NULL, 0) = 0
### SIGUSR1 send count = 1  (expect 0 for none, >=1 for parent/all)
```

**[observed]** — `none` writes the file and sends **0** `SIGUSR1` (confirming `"none"` is absent from the reload switch); `parent` sends exactly one `SIGUSR1` to `$KITTY_PID`; `all` sends one `SIGUSR1` to a kitty‑GUI process found by iterating all processes. (For `all`, the signalled PID `282` differs from the launched instance's `$KITTY_PID` because `api.go:363-369` enumerates every process; the point is that `all` signals a GUI process whereas `none` does not.)

### 5.5 Empty vs. pre‑populated `kitty.conf` — append, replace, comment‑out, `.bak`

`Patcher.Patch` has three observable behaviors depending on the file's prior content (`tools/config/api.go:331-340`, `:343-344`):

- **Empty / no block → append.** Shown in §R4.2: the whole file becomes the sentinel block; no `.bak` (`api.go:343` gate).
- **Existing block → replace in place; `.bak` written.** A second finalize on the now‑non‑empty file, selecting `Symbols Nerd Font Mono`, replaced the block and wrote a backup equal to the prior block:

```
############ R4 AFTER second finalize ############
### CMD: ls -la $CFG
total 16
drwx------ 2 root root 4096 Jul 13 17:25 .
drwxrwxrwt 1 root root 4096 Jul 13 17:23 ..
-rw-r--r-- 1 root root  152 Jul 13 17:25 kitty.conf
-rw-r--r-- 1 root root  190 Jul 13 17:25 kitty.conf.bak
### CMD: cat $CFG/kitty.conf
-----BEGIN kitty.conf-----
# BEGIN_KITTY_FONTS
font_family      family="Symbols Nerd Font Mono"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
-----END kitty.conf-----
### CMD: cat $CFG/kitty.conf.bak
-----BEGIN kitty.conf.bak-----
# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTS
-----END kitty.conf.bak-----
```

(`Symbols Nerd Font Mono` has only a `Regular` style, so `bold_font`/`italic_font`/`bold_italic_font` resolve to `auto`, and the family is written in the `family="…"` spec form.) The block is replaced in place (`api.go:331-334`) and `.bak` is written because the file was non‑empty (`api.go:343-344`). **[observed]**

- **Pre‑populated with other content → comment‑out existing font lines + append after a blank‑line separator; `.bak` written.** Seeding `kitty.conf` with a comment, a non‑font setting, and a stray `font_family` line, then finalizing once:

```
############ EDGE: pre-populated kitty.conf (api.go:325-340 comment-out + append + .bak) ############
### BEFORE (seeded kitty.conf):
-----
# my existing kitty config
cursor_shape beam
font_family Old Existing Font
-----
total 12
drwx------ 2 root root 4096 Jul 13 17:40 .
drwxrwxrwt 1 root root 4096 Jul 13 17:40 ..
-rw-r--r-- 1 root root   75 Jul 13 17:40 kitty.conf
### AFTER ls -la:
total 16
drwx------ 2 root root 4096 Jul 13 17:40 .
drwxrwxrwt 1 root root 4096 Jul 13 17:40 ..
-rw-r--r-- 1 root root  269 Jul 13 17:40 kitty.conf
-rw-r--r-- 1 root root   75 Jul 13 17:40 kitty.conf.bak
### AFTER kitty.conf (expect: existing font_family commented out, comment+cursor_shape preserved, block appended after blank line):
-----BEGIN kitty.conf-----
# my existing kitty config
cursor_shape beam
# font_family Old Existing Font


# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTS
-----END kitty.conf-----
### kitty.conf.bak (expect = original seeded content):
-----BEGIN kitty.conf.bak-----
# my existing kitty config
cursor_shape beam
font_family Old Existing Font
-----END kitty.conf.bak-----
```

The prior `font_family Old Existing Font` was commented to `# font_family Old Existing Font` (comment‑out regex, `api.go:325-326`); the comment and `cursor_shape beam` were preserved; the sentinel block was appended after a `\n\n` separator (`api.go:337`); and `kitty.conf.bak` (the original 75 bytes) was written because the file was non‑empty (`api.go:343-344`). **[observed]**

### 5.6 Secondary invocation path

The `list-fonts` code path execs the same kitten. `kitty/fonts/list.py:41` sets `os.environ['KITTY_PATH_TO_KITTY_EXE'] = kitty_exe()`, and `kitty/fonts/list.py:42` calls `os.execlp(kitten_exe(), 'kitten', 'choose-fonts')`:

```
=== kitty/fonts/list.py lines 38-44 (secondary invocation path) ===
    38	    argv = list(argv)
    39	    if '--psnames' in argv:
    40	        argv.remove('--psnames')
    41	    os.environ['KITTY_PATH_TO_KITTY_EXE'] = kitty_exe()
    42	    os.execlp(kitten_exe(), 'kitten', 'choose-fonts')
```

**[observed via source read at commit 815df1e2]** — it re‑enters the identical `kitten choose-fonts` entry point, so all behavior above applies unchanged.


---

## 6. Coverage pass — every named item mapped to evidence

| Item | Answer / value | `file:line` (function/method/struct) | Evidence | Reasoning |
|------|----------------|--------------------------------------|----------|-----------|
| Build command | `./dev.sh build` (bare failed on deps drift; completed with `--ignore-compiler-warnings`, read‑only‑safe) | `dev.sh:9`; werror gate `setup.py:491,1231`; flag `setup.py:2003-2004`; forwarded `bypy/devenv.go:374` | §R1.1–R1.2 | Canonical dev build; `-Werror` relaxed only, no source touched |
| Launch command | `kitty/launcher/kitty` (one default instance) | produced by build; `docs/build.rst:14-19` | §R1.3 | Default launcher binary, no custom config |
| Toolchain | Go 1.23.4, Python 3.12.3, gcc 13.3.0 (declared `go 1.22`, `python>=3.8`) | `go.mod:3`, `pyproject.toml:2` | §0.2 | Container satisfies declared minimums |
| Invocation | `kitten choose-fonts` | real entry point | §R2 | Family‑list UI rendered |
| Registration | `choose_fonts.EntryPoint(root)` | `tools/cmd/tool/main.go:82` (import `:9`); `EntryPoint` `kittens/choose_fonts/main.go:74-99` | §R3.1 | Subcommand added to CLI root; UI ran |
| `--reload-in` option | choices `parent, all, none`, default `parent` | `kittens/choose_fonts/main.go:86-95`; struct `:70-72` | §R3.1 (`--help`) | `OptionSpec`; `GetOptionValues(&opts)` `:80` |
| `choose_fonts` alias | **visible** (`clone.Hidden=false`) | `kittens/choose_fonts/main.go:96-98` | §R3.1 | Both spellings on root listing + `--help` |
| Option/data flow | `opts` → `main(&opts)` → `handler.opts` → final Enter branch | `main.go:80,83,35`; `ui.go:42-43`; `final.go:87-92` | §R3.2, §5.4 | `--help` proves parse; §5.4 proves effect |
| Pane: list→faces | `faces.on_enter(family)` on Enter | `kittens/choose_fonts/list.go:246-250` | §R3.3 | Faces pane rendered after Enter |
| Pane: faces→final | `final_pane.on_enter(family, settings)` on Enter | `kittens/choose_fonts/faces.go:112-120`; settings `:14-16` | §R3.3 | Final legend rendered after Enter |
| Pane: fine‑tune | `face_pane.on_enter(...)` via `r/b/i/o` | `kittens/choose_fonts/faces.go:125-139`; `face.go` | §R3.3 | Optional; off the happy path |
| Backend bridge | Go frontend spawns Python `+runpy` backend over JSON stdio | `backend.go:32,41,52-53`; `backend.py:150-168` | §R3.4 | `ps` shows PPID relationship |
| **Enter (finalize)** | writes 4 font keys to `kitty.conf`; **persists** | `final.go:78-97` → `Patcher.Patch` `api.go:310`; atomic `api.go:347` | §R4 | Block on disk; restart reads it back |
| `serialized()` | 4 lines, keys padded to 17 chars | `kittens/choose_fonts/final.go:63-70` | §R4.2, §5.2 | Verbatim in `kitty.conf` and STDOUT |
| Sentinel block | `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` | `tools/config/api.go:328-330` | §R4.2 | Wraps the four lines |
| `.bak` backup | only when file non‑empty before write | `tools/config/api.go:343-344` | §R4.2 (none), §5.5 (written) | Empty first write → no `.bak` |
| Restart read‑back | fresh `load_config` returns saved family; re‑open pre‑selects it | `kitty/config.py:163`; `kitty/constants.py:87-89,131-133`; `kitty/cli.py:1089-1093`; `backend.py:11,157` | §R4.3 | Upgrades conclusion to **[observed]** |
| Esc | returns to faces pane; `kitty.conf` unchanged | `kittens/choose_fonts/final.go:73-76` | §5.1 | `current_pane=&handler.faces` |
| `s` / `S` | serialized → STDOUT only; `kitty.conf` unchanged | `final.go:101-111`; printed by `main.go:64-66` | §5.2 | Non‑persisting path |
| Ctrl+c | `Error: canceled by user`; `kitty.conf` unchanged | error path `kittens/choose_fonts/main.go:54-57` (not `:58-62`) | §5.3 | Observed, corrects the "Killed by signal" guess |
| `--reload-in none` | file written, **0** `SIGUSR1` | `"none"` absent from `final.go:86-93` | §5.4 | Trace shows count 0 |
| `--reload-in parent` | file written, 1 `SIGUSR1` to `$KITTY_PID` | `api.go:352-361` | §5.4 | `pidfd_send_signal(...,SIGUSR1,...)` to `$KITTY_PID` |
| `--reload-in all` | file written, 1 `SIGUSR1` to a kitty‑GUI proc | `api.go:363-369`; `is_kitty_gui_cmdline` `api.go:282-303` | §5.4 | Trace shows one send |
| Empty vs populated | append / replace / comment‑out existing font lines | `tools/config/api.go:325-340` | §R4.2, §5.5 | All three branches observed |
| Config isolation | `KITTY_CONFIG_DIRECTORY` honored first; memoized per process | `tools/utils/paths.go:88-91,132-134` | §0.1 | Set before each launch |
| Secondary invocation | `list-fonts` execs `kitten choose-fonts` | `kitty/fonts/list.py:41-42` | §5.6 | Same entry point |

---

## 7. Notes, nuances, and references

**Nuances confirmed at runtime:**

1. The underscore `choose_fonts` is a **visible** alias (`clone.Hidden = false`, `kittens/choose_fonts/main.go:97`), not a hidden clone. **[observed]**
2. `Esc` at the final pane returns to the **faces** pane (`final.go:73-76`), not the family list; the legend text at `final.go:40` is only descriptive. **[observed]**
3. `--reload-in none` writes the file but signals **no** reload — `"none"` has no case in the switch (`final.go:86-93`). **[observed]**
4. There is no explicit `Ctrl+c` handler in `final.go`; the observed abort is `Error: canceled by user` via the loop error path (`kittens/choose_fonts/main.go:54-57`), **not** the death‑signal path (`main.go:58-62`). **[observed]**
5. `kitty.conf.bak` is written only when the file was non‑empty before the write (`tools/config/api.go:343`); an initially empty/absent `kitty.conf` yields no `.bak` on the first finalize. **[observed]**
6. `ConfigDir` is memoized once per process (`tools/utils/paths.go:132-134`), so `KITTY_CONFIG_DIRECTORY` must be set before each `kitty`/`kitten` process starts. Applied throughout. **[observed]**

**On the `--reload-in` trace method [observed]:** `strace -p` (attach) failed in this container because Yama `ptrace_scope=1` and the container lacks `CAP_SYS_PTRACE`; the target was therefore run *under* `strace` (tracing a child is permitted). The reload signal is delivered by Go's runtime via `pidfd_open` + `pidfd_send_signal` rather than `kill`/`tgkill`, so the syscall filter had to include the `pidfd_*` calls to observe it.

**Absent in‑repo docs (not cited):** `docs/kittens/choose-fonts.rst` does not exist at commit `815df1e2`; `kittens/choose_fonts/__init__.py` and `kittens/choose_fonts/main.py` are 0‑byte package markers.

**Official upstream documentation (referenced by URL only):**

- `choose-fonts` kitten — https://sw.kovidgoyal.net/kitty/kittens/choose-fonts/
- `kitty.conf` reference (font keys; auto‑reload / `SIGUSR1`) — https://sw.kovidgoyal.net/kitty/conf/
- Build from source (C compiler + Go compiler) — https://sw.kovidgoyal.net/kitty/build/

**One‑line conclusion:** pressing `Enter` at the `choose-fonts` final pane writes `font_family`/`bold_font`/`italic_font`/`bold_italic_font` into `kitty.conf` (via `final_pane.on_key_event` → `config.Patcher.Patch` → `AtomicUpdateFile`), so the choice **persists across restarts**; `s` is the session‑only STDOUT alternative, and the `SIGUSR1` reload is a convenience, not the persistence mechanism. **[observed]**

