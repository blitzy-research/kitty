# choose-fonts: Does pressing Enter persist the font choice across restarts?

> **Subject:** the `choose-fonts` kitten in [`kovidgoyal/kitty`](https://github.com/kovidgoyal/kitty)
> **Pinned commit:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (source branch label `kitty_815df1e210e0`)
> **Question:** When a user selects a font through the kitten's interactive UI and presses **Enter** at the final confirmation, is that choice *remembered across restarts* (durable, on-disk) or *applied only to the current session*?

---

## 1. Summary / TL;DR verdict

**Pressing `Enter` at the final confirmation screen PERSISTS the font choice across restarts.** It is durable: the kitten writes the four font settings into a sentinel-delimited managed block (`# BEGIN_KITTY_FONTS … # END_KITTY_FONTS`) inside `kitty.conf` **on disk**, so every subsequent launch of kitty reads it back. This is proven below across **two restarts** with the on-disk file and the kitten's own preselection both confirming the choice survived.

By contrast, the **`s` key does NOT persist** the choice — it writes the same four settings only to `STDOUT` and leaves `kitty.conf` byte-for-byte unchanged (session/pipe output only).

The live config *reload* after the write (via the `SIGUSR1` signal) is a **separate, orthogonal** concern controlled by the `--reload-in` option: even `--reload-in none` still writes the durable file to disk; it merely skips hot-reloading the running instance.

All findings below come from a **real, default build** of kitty at the pinned commit, running the **real `kitten choose-fonts` entry point**, with every runtime claim shown next to the exact command and verbatim output that produced it, and every structural claim carrying an exact `file:line` citation.

---

## 2. Environment & method

### 2.1 Build/run environment

The build and all runtime verification were performed inside the task's canonical Docker image (`andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`).

Toolchain actually present (captured verbatim):

```console
$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
$ git rev-parse --abbrev-ref HEAD
blitzy-fd24190a-a270-4bc1-aeea-b24bd574c3ef
$ go version
go version go1.22.12 linux/amd64
$ python3 --version
Python 3.13.7
$ cc --version | head -1
cc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
```

- `HEAD` is exactly the pinned commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`, so all `file:line` citations in this document match this checkout.
- The checked-out git branch on disk is `blitzy-fd24190a-a270-4bc1-aeea-b24bd574c3ef` (the working branch). The **source branch label is `kitty_815df1e210e0`**, which is why this document's filename is `kitty_815df1e210e0.md`.
- **Observed vs. expected (reported as observed):** the running Go is `go1.22.12` (satisfying the pinned `go 1.22` — `go.mod:L3`), and Python is `3.13.7`. The task's planning notes anticipated an environment with only Python `3.12.3` and no Go/C compiler; the actual container has a complete toolchain (Go 1.22.12, gcc 15.2.0, Python 3.13.7), so the canonical build ran here directly.

### 2.2 Methodology

1. **Read-only source.** No existing repository file was modified, added, or deleted. The only artifact created is this document. All temporary scripts and throwaway config directories lived under `/tmp` and were removed afterward. The final `git status --porcelain` (see §12.1) shows only this new file.
2. **Investigate by running first.** kitty was built with the canonical `./dev.sh build`, then the **real** `kitten choose-fonts` command was driven end-to-end through a PTY, and its actual output/on-disk effects were captured *before* any prose was written.
3. **One claim → one evidence line.** Every behavioral statement below sits next to the exact command and verbatim output that produced it. Statements derived purely from reading the source (not observed at runtime) are explicitly labeled **(inferred)**.
4. **Ephemeral, isolated config.** All persistence experiments pointed `KITTY_CONFIG_DIRECTORY` at a fresh `mktemp -d` directory so the experiment writes to a throwaway `kitty.conf` and never touches the real user config or the repository. This isolation hook works because `ConfigDirForName` honors `KITTY_CONFIG_DIRECTORY` **first**, before any XDG lookup — `tools/utils/paths.go:L88-L90`:

   ```go
   func ConfigDirForName(name string) (config_dir string) {
       if kcd := os.Getenv("KITTY_CONFIG_DIRECTORY"); kcd != "" {
           return Abspath(Expanduser(kcd))
   ```

### 2.3 Headless GPU note (and how the real path was still exercised)

kitty is a GPU-based terminal and `choose-fonts` is an interactive TUI. This container has no GPU and no display, so two things were done — **neither substitutes a synthetic stand-in** for the kitten:

- The **full kitty GUI** (needed only for Q1's "single default instance" in §3.3 and for the live-reload demonstration in §10.1) was run headless under `xvfb-run` with software OpenGL (`LIBGL_ALWAYS_SOFTWARE=1`, `GALLIUM_DRIVER=llvmpipe`).
- The **`choose-fonts` kitten** (a terminal program, not a GPU app) was driven through a small PTY harness (`/tmp/blitzy_harness/cf_pty.py`, a temporary observation script, since removed) that `exec`s the real `kitty/launcher/kitten choose-fonts` and emulates a real terminal so the genuine code path runs end-to-end. The exact invocation form was `python3 cf_pty.py --cfgdir DIR --filter "Fira Code" --final enter|s [--reload-in parent|all|none] --kitty <abs path to kitty>`.

Two real-terminal behaviors had to be emulated, **each verified by observation**:

1. **The kitten queries the terminal** for color/DPI via kitty's DCS `+q` protocol — driven from `ui.go:L80`, `lp.QueryTerminal("font_size", "dpi_x", "dpi_y", "foreground", "background")`. A bare PTY that never answers those queries makes *this build* abort — observed verbatim:

   ```text
   Panicked with error: runtime error: integer divide by zero
   ```

   (font size / DPI come back as `0`, so a later cell-metric division is by zero). Note this is the **actual** observed failure at this commit — not a color-parse error. The harness answers the real `kitty-query-*` DCS requests (`font_size=11.0`, `dpi_x=96.0`, `dpi_y=96.0`, `foreground=#ffffff`, `background=#000000`), after which the **real kitten** runs with no error.

2. **The kitten enables the kitty keyboard protocol** (`CSI > <flags> u`, `tools/tui/loop/terminal-state.go:L140`), under which `Enter` arrives as `CSI 13 u` (CSI number `13` → functional `ENTER`, `tools/tui/loop/key-encoding.go:L17,L187-L189`). A bare carriage return is therefore *not* seen as Enter — observed at the family-listing pane after typing a filter:

   ```text
   [bare-CR] after sending Enter=b'\r':      faces pane reached? False
   [CSI-13u] after sending Enter=b'\x1b[13u': faces pane reached? True
   ```

   The harness therefore delivers `Enter` as `\x1b[13u` (and `s` at the final pane as `\x1b[115;1;115u`, per §8.1).

With those two details handled, the **entire** `choose-fonts` flow — scanning → family listing → faces → final confirmation → **Enter** patching `kitty.conf`, the **`s`** contrast (§9.4), and the **`--reload-in`** live-reload (§10.1) — was exercised through the genuine kitten via its real entry point (`kitten choose-fonts`). **Nothing was replaced by a synthetic stand-in**: even the post-write `SIGUSR1` reload was triggered by the real kitten's own `--reload-in` finalization against a live kitty GUI (§10.1). The only difference from a normal user's machine is that the GUI ran headless under Xvfb with software GL.

---

## 3. Q1 — Build from source & run a default instance

### 3.1 What the canonical docs prescribe

The build documentation states the only prerequisites are a C compiler and the Go compiler, and gives the canonical clone-and-build sequence. Quoted verbatim — `docs/build.rst:L14-L22`:

```rst
|kitty| is designed to run from source, for easy hack-ability. All you need to
get started is a C compiler and the `go compiler
<https://go.dev/doc/install>`__. After installing those, run the following commands::

    git clone https://github.com/kovidgoyal/kitty.git && cd kitty
    ./dev.sh build

That's it, kitty will be built from source, magically. You can run it as
:file:`kitty/launcher/kitty`.
```

The canonical commands are therefore:

```bash
git clone https://github.com/kovidgoyal/kitty.git && cd kitty
./dev.sh build
```

`./dev.sh` simply execs the Go-based dev environment builder — `dev.sh:L9`:

```bash
exec go run bypy/devenv.go "$@"
```

**Source-directory caveat (must be stated) — `docs/build.rst:L35-L37`:** the built kitty executable expects to find its source in whatever directory `./dev.sh build` was first run in; if that directory is moved or renamed, you must run `make clean && ./dev.sh build` again.

Pinned/tested runtime versions (the supported envelope — not modified by this task):
- **Go `1.22`** — `go.mod:L3` (`go 1.22`).
- **Python `>=3.8`** — `pyproject.toml:L2` (`requires-python = ">=3.8"`); CI exercises a Python version matrix — `3.8` (`.github/workflows/ci.yml:L26`), `3.10` (`.github/workflows/ci.yml:L30`), and `3.9` (`.github/workflows/ci.yml:L34`) in the main job, plus a bundle-test job on `3.11` (`.github/workflows/ci.yml:L85`) and `3.10` (`.github/workflows/ci.yml:L170`) — with Go pinned via `go-version-file: go.mod` (`.github/workflows/ci.yml:L60`, `L90`, `L150`, `L175`).

### 3.2 The build, observed

`./dev.sh build` runs the Go-based dev builder (`dev.sh:L9`), which downloads kitty's major dependencies as pre-built binaries for the platform (per `docs/build.rst:L24-L26`) and compiles kitty against them. In this container the dependency bundle was already present, so the run below is an **incremental rebuild** — the canonical terminal line `Build successful. Run kitty as: kitty/launcher/kitty` is the same for a clean or incremental build. Captured verbatim:

```console
$ ./dev.sh build
[1/1] Compiling kitty/data-types.c ...
 done
[1/1] Linking kitty/fast_data_types ...
 done
kitty/tools/cmd
Build successful. Run kitty as: kitty/launcher/kitty
```

The build produced the runnable binaries exactly where the docs say (`ls -l` sorts the two arguments alphabetically, so `kitten` prints before `kitty`):

```console
$ ls -l kitty/launcher/kitty kitty/launcher/kitten
-rwxr-xr-x 1 root root 15765764 Jul  3 00:12 kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 Jul  2 22:42 kitty/launcher/kitty
$ kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
$ kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
```

### 3.3 Running a single default instance under an isolated config dir

A fresh, empty config directory was created so that **built-in defaults apply** (no pre-existing `kitty.conf`). kitty is a GPU terminal and this container has no GPU, so the instance is launched headless under `xvfb-run` with software OpenGL (`LIBGL_ALWAYS_SOFTWARE=1`, `GALLIUM_DRIVER=llvmpipe`; see §2.3). The **exact launch command** and a **deterministic proof** that a single default instance really started *under that config directory* were captured verbatim — a tiny program is run as kitty's window process and writes kitty's own runtime identity to a file:

```console
$ export KITTY_CONFIG_DIRECTORY="$(mktemp -d /tmp/kitty-cfg-XXXXXX)"
$ echo "$KITTY_CONFIG_DIRECTORY"
/tmp/kitty-cfg-z9WrGf
$ test -f "$KITTY_CONFIG_DIRECTORY/kitty.conf" && echo EXISTS || echo ABSENT
ABSENT
$ xvfb-run -a -s "-screen 0 1280x800x24" kitty/launcher/kitty \
      sh -c 'printf "INSTANCE_ALIVE\nKITTY_PID=%s\nCFGDIR=%s\nKITTY_WINDOW_ID=%s\nTERM=%s\n" \
             "$KITTY_PID" "$KITTY_CONFIG_DIRECTORY" "$KITTY_WINDOW_ID" "$TERM" > /tmp/proof.txt' 2> /tmp/kitty.stderr
$ cat /tmp/proof.txt      # written by the program running INSIDE the instance
INSTANCE_ALIVE
KITTY_PID=86043
CFGDIR=/tmp/kitty-cfg-z9WrGf
KITTY_WINDOW_ID=1
TERM=xterm-kitty
$ cat /tmp/kitty.stderr   # the kitty GUI process's own startup diagnostic
[0.150] Failed to open systemd user bus with error: Connection refused
```

Each line above is a concrete proof point that a **single default kitty instance** started (a normal-user launch):
- **The GUI process really started** — kitty assigned `KITTY_PID=86043` and created a window (`KITTY_WINDOW_ID=1`), and its own process emitted the (harmless) startup line `[0.150] Failed to open systemd user bus with error: Connection refused`.
- **It is running under the isolated config dir** — the program spawned inside it reports `CFGDIR=/tmp/kitty-cfg-z9WrGf` (the same throwaway dir) and `TERM=xterm-kitty` (kitty's own terminal type, set only by a real kitty instance).
- **Default config applies** — the dir is empty (`kitty.conf` ABSENT above), so no user settings are loaded.

Because `KITTY_CONFIG_DIRECTORY` is honored first (`tools/utils/paths.go:L88-L90`, shown in §2.2), this same throwaway directory is the config root that the kitten later patches. That config-resolution is proven **decisively** in §9.2, where a kitten restarted from *this* directory preselects the family previously written to its `kitty.conf`. (The interactive `choose-fonts` TUI itself is driven through the PTY harness described in §2.3.)


---

## 4. Q2 — Invoking `choose-fonts`

From inside a running kitty instance, the kitten is invoked as:

```bash
kitten choose-fonts
# equivalently:
kitty +kitten choose-fonts
```

The official docs recommend exactly this — running the `kitten choose-fonts` command to select fonts through its UI.

### 4.1 The real command, observed

Running the real entry point's help confirms the command exists, its short description, and (critically) its **single** option — captured verbatim:

```console
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

Note there is **no `--config-file-name` option** in this output (see §6.3 for the full reconciliation).

### 4.2 The captured interactive screens

These screens are **rendered from the real `kitten choose-fonts`**, driven through the PTY harness of §2.3. The exact producing command and its run metadata (verbatim `HARNESS META` header) are shown first; each screen below is a verbatim `===== SCREEN: … =====` block emitted by that same run. `argv` in the header proves the child process is `kitty/launcher/kitten choose-fonts`.

```console
$ python3 /tmp/blitzy_harness/cf_pty.py \
      --cfgdir "$(mktemp -d /tmp/kitty-cf-XXXXXX)" \
      --filter "Fira Code" --final enter \
      --kitty "$PWD/kitty/launcher/kitty"
=== HARNESS META ===
argv: kitty/launcher/kitten choose-fonts
cfgdir: /tmp/kitty-cf-CbbYkv
answered_query: True   final_stage: done_wait   exit_status: 0
raw_bytes_captured: 20437
```

**Scanning screen** (`ui.go:L161`, shown while the Python backend enumerates monospaced families):

```text
===== SCREEN: scanning =====
Scanning system for fonts, please wait...
```

**Family listing pane** — a two-pane view (family list `║` preview); filterable by typing. With **no `kitty.conf`** present, the built-in default family `DejaVu Sans Mono` is preselected (marked with `>`):

```text
===== SCREEN: listing_initial =====
 Cascadia Code         ║                              DejaVu Sans Mono
 Cascadia Code NF      ║
 Cascadia Code PL      ║ Styles: Bold, Bold Oblique, Book, Oblique
 Cascadia Mono         ║
 Cascadia Mono NF      ║ Press the Enter key to choose this family
 Cascadia Mono PL      ║
 Comfy Code            ║ ───────────────────────────────── preview ─────────────────────────────────
>DejaVu Sans Mono      ║ Gt=f,I=7891231;L3Jvb3QvLmNhY2hlL2tpdHR5L2tpdHRlbi1jaG9vc2Ut…
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
Family:
```

> The `Gt=f,I=…;…` line is kitty's **graphics-protocol payload** (the inline font-preview image) that the text emulator renders as raw bytes; a real kitty window shows an actual rendered image there. It is truncated above with `…` for readability (the payload also embeds a per-run temp path).

**Family listing pane, after typing the filter `Fira Code`** — the list narrows to the match, `>Fira Code` is selected, and the preview pane updates (the search line now reads `Family: Fira Code`):

```text
===== SCREEN: listing_filtered =====
>Fira Code ║                                        Fira Code
           ║
           ║ Weight: Bold, Light, Medium, Regular, SemiBold
           ║
           ║ This font is variable allowing for finer style control
           ║ Press the Enter key to choose this family
           ║
           ║ ─────────────────────────────────────── preview ───────────────────────────────────────
           ║ Gt=f,I=7891231;L3Jvb3QvLmNhY2hlL2tpdHR5L2tpdHRlbi1jaG9vc2Ut…
Family: Fira Code
```

**Faces pane** — after pressing `Enter` on `Fira Code`, previews of the regular/bold/italic/bold-italic faces are shown; the highlighted keys fine-tune individual styles. The resolved face names are real (`FiraCodeRoman-Regular`, `FiraCodeRoman-SemiBold`):

```text
===== SCREEN: faces =====
                                             Fira Code

Press Enter to select this font, Esc to go back to the font list or any of the highlighted keys
below to fine-tune the appearance of the individual font styles.

Regular: FiraCodeRoman-Regular
Gt=f,I=7891231;L3Jvb3QvLmNhY2hlL2tpdHR5L2tpdHRlbi1jaG9vc2Ut…
Bold: FiraCodeRoman-SemiBold
Gt=f,I=7891232;L3Jvb3QvLmNhY2hlL2tpdHR5L2tpdHRlbi1jaG9vc2Ut…
Italic: FiraCodeRoman-Regular
Gt=f,I=7891233;L3Jvb3QvLmNhY2hlL2tpdHR5L2tpdHRlbi1jaG9vc2Ut…
Bold-Italic: FiraCodeRoman-SemiBold
Gt=f,I=7891234;L3Jvb3QvLmNhY2hlL2tpdHR5L2tpdHRlbi1jaG9vc2Ut…
```

**Final confirmation pane** — after a second `Enter`; its four action lines are quoted verbatim in §8.1:

```text
===== SCREEN: final =====
You have chosen the Fira Code family

What would you like to do?

Enter to modify kitty.conf and use the new fonts

Esc to abort and return to font selection

s to write the new font settings to STDOUT

Ctrl+c to quit
```

### 4.3 Architecture: Go frontend + Python backend

The kitten is a **Go TUI frontend paired with a Python backend**. The Go side spawns the Python backend by re-executing the kitty binary with `+runpy` — `kittens/choose_fonts/backend.go:L41`:

```go
k.cmd = exec.Command(exe, "+runpy", "from kittens.choose_fonts.backend import main; main()")
```

The backend executable is discovered via `utils.KittyExe()` (`backend.go:L33`), falling back to `utils.Which("kitty")` (`backend.go:L35`); if neither resolves, it errors with a hint to set `KITTY_PATH_TO_KITTY_EXE` (`backend.go:L38`).

The Python backend's dispatch is in `kittens/choose_fonts/backend.py` — `def main()` at `backend.py:L150`, handling `list_monospaced_fonts` (`L156`), `read_variable_data` (`L159`), and `render_family_samples` (`L164`), raising `Unknown action` otherwise (`L168`). It reuses kitty's own family enumeration — `backend.py:L25` `from kitty.fonts.list import create_family_groups`. The Go frontend issues the initial `list_monospaced_fonts` query (`ui.go:L95`) and stores the current resolved faces in `h.listing.resolved_faces_from_kitty_conf` (`ui.go:L97`); it also creates a scratch temp dir under the cache dir for backend I/O (`ui.go:L88` `os.MkdirTemp(utils.CacheDir(), "kitten-choose-fonts-*")`).

> _Aside (footprint):_ outside `kittens/choose_fonts/`, the strings `choose_fonts`/`choose-fonts` appear in `kitty/fonts/list.py`, `tools/cmd/tool/main.go`, and the **generated** completion file `tools/cmd/completion/kitty_generated.go`. (The AAP anticipated two such files; the generated completion file is the third — reported here as observed.)

---

## 5. Q3a — Subcommand registration

The `choose-fonts` subcommand is registered into the kitten command tree at `tools/cmd/tool/main.go:L82`:

```go
choose_fonts.EntryPoint(root)
```

`EntryPoint` adds the subcommand and wires up its `Run` closure — `kittens/choose_fonts/main.go:L74-L85`:

```go
func EntryPoint(root *cli.Command) {
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

So invoking `kitten choose-fonts` runs this closure, which parses the CLI options into an `Options` value via `cmd.GetOptionValues(&opts)` (`main.go:L80`) and then calls `main(&opts)` (`main.go:L83`).

### 5.1 The `choose_fonts` clone is NOT hidden (nuance)

Immediately after, `EntryPoint` registers an underscore-named clone `choose_fonts` and explicitly sets it **visible** — `kittens/choose_fonts/main.go:L96-L98`:

```go
clone := root.AddClone(ans.Group, ans)
clone.Hidden = false
clone.Name = "choose_fonts"
```

`clone.Hidden = false` (`main.go:L97`) means the underscore alias is **not** hidden at this commit. This was confirmed at runtime — both spellings appear in the top-level kitten listing and the alias resolves:

```console
$ kitty/launcher/kitten --help | grep -iE "choose[-_]fonts"
    choose-fonts
    choose_fonts
$ kitty/launcher/kitten choose_fonts --help | head -1
Usage: kitten choose_fonts
```

(Older descriptions that call this a "hidden clone" do not match the pinned source, which sets `Hidden = false`.)


---

## 6. Q3b — Option parsing

### 6.1 The single option and its `Options` struct

The kitten defines exactly **one** option field — `kittens/choose_fonts/main.go:L70-L72`:

```go
type Options struct {
	Reload_in string
}
```

The option specification (parsed by `cmd.GetOptionValues(&opts)` at `main.go:L80`, per §5) is the single `--reload-in` — `kittens/choose_fonts/main.go:L86-L95`:

```go
ans.Add(cli.OptionSpec{
	Name:    "--reload-in",
	Dest:    "Reload_in",
	Type:    "choices",
	Choices: "parent, all, none",
	Default: "parent",
	Help:    "By default, this kitten will signal only the parent kitty instance it is running in to reload its config, after making changes. Use this option to instead either not reload the config at all or in all running kitty instances.",
})
```

The exact literals, all confirmed by the runtime `--help` in §4.1:
- **Name:** `--reload-in`
- **Dest:** `Reload_in` (the field name in `Options`)
- **Type:** `choices`
- **Choices:** `parent, all, none`
- **Default:** `parent`

### 6.2 There is no other option at this commit

The one-field `Options` struct and the single `ans.Add(...)` call are the entirety of option parsing. The runtime `--help` (§4.1) shows only `--reload-in` and the auto-generated `--help, -h`.

### 6.3 `--config-file-name` is ABSENT here — version reconciliation

At this commit there is **no `--config-file-name` option** for `choose-fonts`. A search of the kitten's directory finds nothing:

```console
$ grep -rn "config-file-name" kittens/choose_fonts/
$   # (no output — confirmed absent)
```

That flag belongs to the **unrelated `themes` kitten**, not `choose-fonts`: it is declared at `kittens/themes/main.py:L40`, and consumed by `tools/themes/collection.go:L589` (`func (self *Theme) SaveInConf(config_dir, reload_in, config_file_name string)`).

For `choose-fonts` at this commit, the target config file is **hardcoded** to `kitty.conf` — `kittens/choose_fonts/final.go:L81`:

```go
path := filepath.Join(utils.ConfigDir(), "kitty.conf")
```

**Reconciliation with public man pages:** some newer public man pages for `kitten-choose-fonts` (e.g. ManKier) list a second option, `--config-file-name [=kitty.conf]`. That reflects a **later release** of kitty. At the pinned commit `815df1e210e0`, the option does not exist for `choose-fonts` — as proven both by the source grep above and by the runtime `--help` in §4.1 — and the destination file name is fixed to `kitty.conf`. It must therefore **not** be documented as present in this build.

---

## 7. Q3c — Option-value flow to the final step

The `Options` value flows from CLI parsing to a single consumption point at finalization. The chain, step by step:

1. **Parse → `main(opts)`.** The `Run` closure builds `opts` and calls `main(&opts)` (`main.go:L83`). `main` is declared at `kittens/choose_fonts/main.go:L16`:

   ```go
   func main(opts *Options) (rc int, err error) {
   ```

2. **`main` → `handler{opts}`.** `main` constructs the TUI handler, injecting `opts` — `kittens/choose_fonts/main.go:L35`:

   ```go
   h := &handler{lp: lp, opts: opts}
   ```

3. **The handler holds `opts` and the panes.** The handler struct carries `opts *Options`, the loop, shared state, and the panes — `kittens/choose_fonts/ui.go:L42-L62`. The pane set and order are established at `ui.go:L81`:

   ```go
   h.panes = []pane{&h.listing, &h.faces, &h.face_pane, &h.final_pane}
   ```

4. **Pane progression: listing → faces → final.**
   - The **listing** pane preselects the family currently configured in `kitty.conf` — `kittens/choose_fonts/list.go:L170`:

     ```go
     self.family_list.SelectFamily(self.resolved_faces_from_kitty_conf.Font_family.Family)
     ```

     (Observed: with no `kitty.conf`, `>DejaVu Sans Mono` is preselected; after a durable write, the previously chosen family is preselected — see §9.2.)
   - Pressing `Enter` on a family advances to the **faces** pane — `kittens/choose_fonts/list.go:L250`:

     ```go
     self.handler.faces.on_enter(family)
     ```

   - The faces pane holds the four working settings — `kittens/choose_fonts/faces.go:L14-L15`:

     ```go
     type faces_settings struct {
     	font_family, bold_font, italic_font, bold_italic_font string
     ```

     and its fine-tune keys map to each of the four faces: `r`/`R` → `font_family`, `b`/`B` → `bold_font`, `i`/`I` → `italic_font`, `o`/`O` → `bold_italic_font` (`faces.go:L129`, `L131`, `L133`, `L135`).
   - Pressing `Enter` in the faces pane advances to the **final** pane, carrying the family and settings — `kittens/choose_fonts/faces.go:L118-L120`:

     ```go
     self.handler.final_pane.on_enter(self.family, self.settings)
     ```

5. **`opts` is consumed only at finalization.** The single place `opts` is read is the reload switch, after the config file has been written — `kittens/choose_fonts/final.go:L87`:

   ```go
   switch self.handler.opts.Reload_in {
   ```

In other words, `--reload-in` has **no effect on the selection flow itself**; it is inspected only once, at the very end, to decide whether/where to trigger a live config reload after the durable write (see §8 and §10.1).


---

## 8. Q3d — What finalization does

### 8.1 The four final-screen actions (verbatim)

The final pane draws four action lines (`kittens/choose_fonts/final.go:L33-L45`). Quoted verbatim from the source (with the exact wording confirmed on the captured screen in §4.2):

- **`Enter`** → "modify `kitty.conf` and use the new fonts" — `final.go:L38`
- **`Esc`** → "abort and return to font selection" — `final.go:L40`
- **`s`** → "write the new font settings to `STDOUT`" — `final.go:L42`
- **`Ctrl+c`** → "quit" — `final.go:L44`

### 8.2 `serialized()` — the exact four keys and spacing

The settings are serialized to exactly four lines, each key padded so the values align to column 17 — `kittens/choose_fonts/final.go:L63-L70`:

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

The literal padding is: `font_family` + 6 spaces, `bold_font` + 8 spaces, `italic_font` + 6 spaces, `bold_italic_font` + 1 space.

### 8.3 The `Enter` path — writing `kitty.conf` on disk

Pressing `Enter` runs the finalization block — `kittens/choose_fonts/final.go:L78-L95`:

```go
patcher := config.Patcher{Write_backup: true}                                   // L80
path := filepath.Join(utils.ConfigDir(), "kitty.conf")                          // L81
updated, err := patcher.Patch(path, "KITTY_FONTS", self.settings.serialized(),  // L82
	"font_family", "bold_font", "italic_font", "bold_italic_font")
...
if updated {                                                                    // L86
	switch self.handler.opts.Reload_in {                                        // L87
	case "parent":
		config.ReloadConfigInKitty(true)                                        // L89
	case "all":
		config.ReloadConfigInKitty(false)                                       // L91
	}
}
self.lp.Quit(0)                                                                 // L94
```

So `Enter`:
1. Constructs a `config.Patcher` with `Write_backup: true` (`L80`),
2. Computes the **on-disk** path `<config-dir>/kitty.conf` (`L81`),
3. Calls `patcher.Patch(...)` to write the `KITTY_FONTS` managed block with the four serialized keys (`L82`),
4. If the file was updated, triggers a live reload according to `--reload-in` (`L86-L92`; see §10.1),
5. Quits (`L94`).

Because step 3 **writes to `kitty.conf` on disk**, the choice is durable — this is the crux of the PERSISTS verdict, proven in §9.

### 8.4 Observed `Enter` finalization (real kitten)

Driven through the real kitten on a fresh ephemeral config dir (`Fira Code` selected, then `Enter` at the final pane) — the same run whose screens appear in §4.2:

```console
$ python3 /tmp/blitzy_harness/cf_pty.py --cfgdir "$KITTY_CONFIG_DIRECTORY" \
      --filter "Fira Code" --final enter --kitty "$PWD/kitty/launcher/kitty"
=== HARNESS META ===
argv: kitty/launcher/kitten choose-fonts
answered_query: True   final_stage: done_wait   exit_status: 0
```

**Before** — no `kitty.conf` exists:

```console
$ ls -la "$KITTY_CONFIG_DIRECTORY"     # KITTY_CONFIG_DIRECTORY=/tmp/kitty-clean-8M1rCX
total 8
drwx--S---  2 root root 4096 ... .
drwxrwsrwx 15 root root 4096 ... ..
$ test -f "$KITTY_CONFIG_DIRECTORY/kitty.conf" && echo EXISTS || echo ABSENT
ABSENT
```

**After** pressing `Enter` at the final pane — `kitty.conf` is created on disk (139 bytes, mode `-rw-r--r--` = `0644`), containing the managed block with the four serialized keys at their exact alignment:

```console
$ ls -la "$KITTY_CONFIG_DIRECTORY"
total 12
drwx--S---  2 root root 4096 ... .
drwxrwsrwx 15 root root 4096 ... ..
-rw-r--r--  1 root root  139 ... kitty.conf
$ cat -A "$KITTY_CONFIG_DIRECTORY/kitty.conf"
# BEGIN_KITTY_FONTS$
font_family      family="Fira Code"$
bold_font        auto$
italic_font      auto$
bold_italic_font auto$
# END_KITTY_FONTS
```

The `cat -A` output (with `$` marking line ends) confirms the exact column-17 alignment from `serialized()` (§8.2). The mode `0644` matches the `Patcher` default (see §9.3, `tools/config/api.go:L311-L313`).

> _Why `font_family` is `family="Fira Code"` while the others are `auto`:_ when a family is selected fresh, `font_family` is written as a `family=` spec and the bold/italic/bold-italic variants are left as `auto`, which (per the official docs) means kitty auto-derives those variants from the chosen family. The `serialized()` function emits whatever `faces_settings` currently holds; §9.4 shows a different, resolved-face example produced by the `s` key.


---

## 9. Q4/Q5 — Persistence verdict with proof

**Verdict: pressing `Enter` PERSISTS the font choice across restarts.** The proof has four parts: (1) the durable write (already shown in §8.4), (2) the restart durability across ≥2 runs, (3) the `Patcher` algorithm that makes the write durable, and (4) the `s`-key contrast that does **not** persist.

### 9.1 The durable write (recap)

As shown in §8.4, `Enter` creates `kitty.conf` on disk with the `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` block. Since kitty reads `kitty.conf` at startup, an on-disk write is by definition durable across restarts. The remaining subsections prove it empirically and explain the mechanism.

### 9.2 Durability across restarts (≥2 runs — stable)

This was proven empirically on a single `KITTY_CONFIG_DIRECTORY`: press `Enter` once (writes `Fira Code`), then **restart** the kitten from the *same* directory twice. A restarted kitten preselects the family read from the on-disk `kitty.conf` via `list.go:L170` (`resolved_faces_from_kitty_conf`) — so if the choice persisted, both restarts must preselect `>Fira Code` (vs. `>DejaVu Sans Mono` on a config-less dir). Each restart also re-checks the on-disk md5. The `--stop-at listing` flag captures the preselection screen and quits with `Ctrl+c` **without** modifying the file. Full transcript:

```console
$ CFG="$(mktemp -d /tmp/kitty-cf-XXXXXX)"; echo "$CFG"
/tmp/kitty-cf-MyfvDA
$ test -f "$CFG/kitty.conf" && echo EXISTS || echo ABSENT
ABSENT

# ---- STEP 1: choose Fira Code and press Enter (durable write) ----
$ python3 /tmp/blitzy_harness/cf_pty.py --cfgdir "$CFG" --filter "Fira Code" \
      --final enter --kitty "$PWD/kitty/launcher/kitty"  >/dev/null
$ md5sum "$CFG/kitty.conf"
91ab8f1ca4764f412d80012820301aa4  /tmp/kitty-cf-MyfvDA/kitty.conf

# ---- STEP 2 (RESTART #1): re-invoke the kitten from the SAME dir ----
$ python3 /tmp/blitzy_harness/cf_pty.py --cfgdir "$CFG" --stop-at listing \
      --final esc --kitty "$PWD/kitty/launcher/kitty" | sed -n '/SCREEN: listing_initial/,/^ Hack/p'
===== SCREEN: listing_initial =====
 Cascadia Code         ║                                  Fira Code
 Cascadia Code NF      ║
 Cascadia Code PL      ║ Weight: Bold, Light, Medium, Regular, SemiBold
 Cascadia Mono         ║
 Cascadia Mono NF      ║ This font is variable allowing for finer style control
 Cascadia Mono PL      ║ Press the Enter key to choose this family
 Comfy Code            ║
 DejaVu Sans Mono      ║ ───────────────────────────────── preview ─────────────────────────────────
 Fantasque Sans Mono   ║ Gt=f,I=7891231;L3Jvb3QvLmNhY2hlL2tpdHR5L2tpdHRlbi1jaG9vc2Ut…
>Fira Code             ║
 Hack                  ║
$ md5sum "$CFG/kitty.conf"     # unchanged after restart #1
91ab8f1ca4764f412d80012820301aa4  /tmp/kitty-cf-MyfvDA/kitty.conf

# ---- STEP 3 (RESTART #2): re-invoke again from the SAME dir ----
$ python3 /tmp/blitzy_harness/cf_pty.py --cfgdir "$CFG" --stop-at listing \
      --final esc --kitty "$PWD/kitty/launcher/kitty" | grep -E '^>'
>Fira Code             ║
$ md5sum "$CFG/kitty.conf"     # unchanged after restart #2
91ab8f1ca4764f412d80012820301aa4  /tmp/kitty-cf-MyfvDA/kitty.conf
```

Both restarts preselect `>Fira Code` (the previewed family on the right is also `Fira Code`), and the on-disk md5 is byte-identical across the write and **both** restarts (`91ab8f1ca4764f412d80012820301aa4`). The result is therefore **stable across ≥2 runs** — the empirical proof of persistence. (Contrast: the config-less first run in §4.2 preselected `>DejaVu Sans Mono`.)

### 9.3 The `Patcher` algorithm (why the write is durable and safe)

The durable write is performed by `config.Patcher.Patch` — `tools/config/api.go:L305-L350`. Key mechanics, each cited:

- **Struct & default mode** — `api.go:L305-L307` (`Write_backup bool`, `Mode fs.FileMode`); the mode defaults to `0o644` when unset — `api.go:L311-L313`:

  ```go
  if self.Mode == 0 {
      self.Mode = 0o644
  }
  ```

  (This matches the observed `-rw-r--r--` in §8.4.)
- **Read existing** — `api.go:L318` reads the current file; a missing file becomes empty bytes (`api.go:L322-L324`).
- **Comment out prior matching keys** — `api.go:L325-L326` uses a regex `(?m)^\s*(font_family|bold_font|italic_font|bold_italic_font)\b` to prefix any pre-existing bare font settings with `# `, so they no longer take effect.
- **Replace or append the managed block** — `api.go:L328` matches `(?ms)^# BEGIN_KITTY_FONTS.+?# END_KITTY_FONTS`; the new block (`api.go:L330`) is `# BEGIN_KITTY_FONTS\n<serialized>\n# END_KITTY_FONTS`. An existing block is **replaced** (`api.go:L331-L334`); otherwise the block is **appended** (`api.go:L335-L340`), separated by `"\n\n"` when the prior file was non-empty.
- **Backup + atomic write** — `api.go:L343-L347`: a `<path>.bak` backup is written **only when the prior file was non-empty and `Write_backup` is set**, then the file is replaced atomically at the configured mode:

  ```go
  if len(raw) > 0 && self.Write_backup {
      os.WriteFile(backup_path+".bak", raw, self.Mode)
  }
  ...
  utils.AtomicUpdateFile(path, nraw, self.Mode)
  ```

**Observed `.bak` + block-replace behavior.** On the first write (§8.4) the prior file was absent, so **no `.bak`** was created (`len(raw) > 0` is false):

```console
$ ls "$KITTY_CONFIG_DIRECTORY"/*.bak 2>&1
ls: cannot access '/tmp/kitty-clean-8M1rCX/*.bak': No such file or directory
(no .bak - correct, prior file empty)
```

A **second** finalization (`JetBrains Mono`) over the now non-empty file **replaced** the block (not duplicated — exactly one block remains) and **created** `kitty.conf.bak` holding the prior `Fira Code` content:

```console
### SECOND finalize (JetBrains Mono) over the now non-empty kitty.conf ###
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"          # block REPLACED, single block
# BEGIN_KITTY_FONTS
font_family      family="JetBrains Mono"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf.bak"      # .bak now created, holds prior Fira Code
# BEGIN_KITTY_FONTS
font_family      family="Fira Code"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
$ grep -c BEGIN_KITTY_FONTS "$KITTY_CONFIG_DIRECTORY/kitty.conf"   # exactly 1 = no duplication
1
```

### 9.4 The `s`-key contrast — NOT persisted (session/STDOUT only)

The `s` (and `S`) key writes the same four serialized lines only to `STDOUT` and never touches `kitty.conf`. In the handler — `kittens/choose_fonts/final.go:L101-L106`:

```go
case "s", "S":                                       // L104
	output_on_exit = self.settings.serialized() + "\n"  // L105
	self.lp.Quit(0)                                   // L106
```

`case "s", "S"` (`final.go:L104`) matches **both** lowercase and uppercase. The string is written to `STDOUT` only on exit — `kittens/choose_fonts/main.go:L14` (`var output_on_exit string`) and `main.go:L64-L66`:

```go
if output_on_exit != "" {
	os.Stdout.WriteString(output_on_exit)
}
```

**Observed `s`-key behavior (real kitten).** Two runs prove it. The `s` key is delivered to the real kitten as `\x1b[115;1;115u` (per §8.1); the four serialized lines it prints appear on `STDOUT` after the TUI tears down, and the harness extracts them (`STDOUT-ON-EXIT`).

*(1) On a fresh, empty config dir — `s` writes STDOUT and never creates `kitty.conf`:*

```console
$ CFG="$(mktemp -d /tmp/kitty-cf-XXXXXX)"
$ test -f "$CFG/kitty.conf" && echo EXISTS || echo ABSENT
ABSENT
$ python3 /tmp/blitzy_harness/cf_pty.py --cfgdir "$CFG" --filter "Fira Code" \
      --final s --kitty "$PWD/kitty/launcher/kitty"
...
===== STDOUT-ON-EXIT (plain text after alt-screen teardown) =====
font_family      family="Fira Code"
bold_font        auto
italic_font      auto
bold_italic_font auto
$ test -f "$CFG/kitty.conf" && echo EXISTS || echo ABSENT   # still no file written
ABSENT
```

*(2) Over an existing `kitty.conf` — choosing a **different** family (`JetBrains Mono`) and pressing `s` leaves the on-disk file byte-identical (`diff -u` empty), still holding `Fira Code`:*

```console
$ # seed: choose Fira Code + Enter, then snapshot the file as before.conf
$ python3 /tmp/blitzy_harness/cf_pty.py --cfgdir "$CFG" --filter "Fira Code" \
      --final enter --kitty "$PWD/kitty/launcher/kitty" >/dev/null
$ cp "$CFG/kitty.conf" /tmp/blitzy_harness/before.conf
$ md5sum /tmp/blitzy_harness/before.conf
91ab8f1ca4764f412d80012820301aa4  /tmp/blitzy_harness/before.conf

$ # choose a DIFFERENT family (JetBrains Mono) and press s
$ python3 /tmp/blitzy_harness/cf_pty.py --cfgdir "$CFG" --filter "JetBrains Mono" \
      --final s --kitty "$PWD/kitty/launcher/kitty"
...
===== STDOUT-ON-EXIT (plain text after alt-screen teardown) =====
font_family      family="JetBrains Mono"
bold_font        auto
italic_font      auto
bold_italic_font auto

$ cp "$CFG/kitty.conf" /tmp/blitzy_harness/after.conf
$ md5sum /tmp/blitzy_harness/before.conf /tmp/blitzy_harness/after.conf
91ab8f1ca4764f412d80012820301aa4  /tmp/blitzy_harness/before.conf
91ab8f1ca4764f412d80012820301aa4  /tmp/blitzy_harness/after.conf
$ diff -u /tmp/blitzy_harness/before.conf /tmp/blitzy_harness/after.conf
$ echo "diff exit=$?"     # 0 and no output => byte-identical
diff exit=0
$ cat "$CFG/kitty.conf"   # on-disk file STILL holds Fira Code, not JetBrains Mono
# BEGIN_KITTY_FONTS
font_family      family="Fira Code"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
```

`STDOUT` shows the family just chosen (`Fira Code`, then `JetBrains Mono`), but the on-disk md5 is unchanged (`91ab8f1ca4764f412d80012820301aa4` before and after) and `diff -u` is empty — so the `s` key does **not** mutate `kitty.conf`. It is therefore **not persisted** (session/pipe output only). This is the clean negative case that confirms the `Enter` path's persistence is specifically due to the on-disk write.

### 9.5 Decisive answer

- **`Enter` → persists.** It writes the four font settings into `kitty.conf` on disk (`final.go:L80-L82` → `Patcher.Patch`), which every restart reads back. Proven durable across two restarts with a stable on-disk file (§9.2).
- **`s` → does not persist.** It writes only to `STDOUT` (`final.go:L105` → `main.go:L64-L66`), leaving `kitty.conf` unchanged (§9.4).


---

## 10. Enumerations (exact literals + cause → effect)

### 10.1 All three `--reload-in` choices

The reload switch reads `opts.Reload_in` after the durable write — `kittens/choose_fonts/final.go:L87-L92`. The **default is `parent`** (`main.go:L91`). Each choice, with its exact literal and cause → effect:

| `--reload-in` | Source branch | Effect (cause → effect) |
|---|---|---|
| **`parent`** (default) | `final.go:L88-L89` → `config.ReloadConfigInKitty(true)` | Signals **only the parent** kitty instance (the one identified by `$KITTY_PID`) to reload — `tools/config/api.go:L353-L357` sends `unix.SIGUSR1` to that PID. |
| **`all`** | `final.go:L90-L91` → `config.ReloadConfigInKitty(false)` | Iterates **all** processes and signals every kitty GUI process — `tools/config/api.go:L363-L366` sends `unix.SIGUSR1` to each. |
| **`none`** | `final.go:L87-L92` (no matching `case`) | **No reload branch executes** — the `switch` has only `parent`/`all` cases, so `none` falls through with no signal. The config is still written to disk (persists); it is simply not hot-reloaded into any running instance. **Observed** in the demo below: finalizing with `--reload-in none` produced **no** new reload in the live kitty (marker count unchanged). |

The reload signal is `SIGUSR1` in both branches — `tools/config/api.go:L357` (parent) and `L366` (all):

```go
// parent (in_parent_only == true), api.go:L353-L357
if in_parent_only {
	...                              // read $KITTY_PID, verify it is a kitty GUI cmdline
	p.SendSignal(unix.SIGUSR1)       // L357
	...
}
// all (in_parent_only == false), api.go:L363-L366
for _, p := range processes {
	...                              // for each kitty GUI process
	p.SendSignal(unix.SIGUSR1)       // L366
}
```

**Observed against a live kitty, via the real kitten `--reload-in` finalization.** All three choices were exercised end-to-end: a real kitty GUI was launched headless under `xvfb-run` (its `argv[0]` basename is `kitty`, satisfying `is_kitty_gui_cmdline`, `api.go:L282`), and the real `kitten choose-fonts --reload-in <choice>` was driven to `Enter` with `KITTY_PID` set to that live instance. To make each reload *observable*, the live kitty's `kitty.conf` contained an unknown key `blitzy_reload_probe`, which kitty logs as `Ignoring unknown config key: blitzy_reload_probe` on **every** config parse (`kitty/conf/utils.py:L250`). Counting that log line before/after each finalize shows whether a reload (config re-read) occurred. The marker stays in `kitty.conf` throughout, so the count reflects reloads — not file mutation:

```console
STARTUP: marker logged 1x (kitty read it once at startup)

===== --reload-in none  (finalize JetBrains Mono) =====
marker count before=1 after=1 ; delta=0   (0 => NO reload)

===== --reload-in parent (finalize Fira Code) =====
marker count before=1 after=2 ; delta=1   (>=1 => reload)

===== --reload-in all    (finalize Cascadia Code) =====
marker count before=2 after=3 ; delta=1   (>=1 => reload)

# kitty's own stderr timeline (each new line = one config re-read):
[0.051]  Ignoring unknown config key: blitzy_reload_probe    # startup
[17.677] Ignoring unknown config key: blitzy_reload_probe    # after --reload-in parent
[23.500] Ignoring unknown config key: blitzy_reload_probe    # after --reload-in all
```

The live kitty **survives** `SIGUSR1` (its default disposition would terminate it), and re-reads its config exactly when `parent`/`all` finalize — never for `none`. This traces the full mechanism through the real entry point: `final.go:L86-L92` (switch on `Reload_in`) → `config.ReloadConfigInKitty(true|false)` → `unix.SIGUSR1` (`api.go:L357`/`L366`) → kitty's signal handler sets `reload_config` (`kitty/child-monitor.c:L1373-L1374`) → `call_boss(load_config_file, "")` (`kitty/child-monitor.c:L535`) → config re-read (`kitty/boss.py:L2691`). **Cause → effect summary:** `parent` → `SIGUSR1` to `$KITTY_PID` only (reload observed, delta +1); `all` → `SIGUSR1` to every kitty GUI process (reload observed, delta +1); `none` → no signal, no reload (delta 0), though the on-disk write still persists.

> **Persistence is orthogonal to reload.** All three choices still perform the durable `kitty.conf` write (that happens at `final.go:L82`, *before* the reload switch at `L87`). `--reload-in` only controls whether the *already-persisted* change is hot-applied to a running instance.

### 10.2 All four font keys written by `serialized()`

The four keys are emitted by `faces_settings.serialized()` — `kittens/choose_fonts/final.go:L63-L70` — from the `faces_settings` struct (`kittens/choose_fonts/faces.go:L14-L15`) and correspond to the resolved-face JSON fields (`kittens/choose_fonts/types.go:L78-L83`). Confirmed on-disk in §8.4:

| Key (exact literal) | Struct field (`faces.go:L14-L15`) | JSON field (`types.go:L78-L83`) | Observed value (§8.4) |
|---|---|---|---|
| `font_family` | `font_family` | `json:"font_family"` | `family="Fira Code"` |
| `bold_font` | `bold_font` | `json:"bold_font"` | `auto` |
| `italic_font` | `italic_font` | `json:"italic_font"` | `auto` |
| `bold_italic_font` | `bold_italic_font` | `json:"bold_italic_font"` | `auto` |

These same four keys are also the ones the `Patcher` comments out if pre-existing (§9.3, `api.go:L325-L326`) and the four arguments passed to `patcher.Patch(...)` at `final.go:L82`.

### 10.3 Every final-screen key

| Key (exact literal) | Effect | Source |
|---|---|---|
| **`Enter`** | Patch `kitty.conf` on disk (durable) + optional reload per `--reload-in`, then quit. **Persists.** | `final.go:L38` (label), `L78-L94` (handler) |
| **`Esc`** | Abort finalization and return to the **faces** pane. No write. | `final.go:L40` (label); `final.go:L73-L76` → `self.handler.current_pane = &self.handler.faces` |
| **`s`** / **`S`** | Write the four serialized lines to `STDOUT` only, then quit. **Does not persist.** | `final.go:L42` (label); `final.go:L104-L106`; STDOUT via `main.go:L64-L66` |
| **`Ctrl+c`** | Quit the kitten. No write. | `final.go:L44` (label) |

Note `s` matches **both** `s` and `S` (`final.go:L104` `case "s", "S":`).


---

## 11. Web-validation & version reconciliation

The observed behavior was cross-checked against the official kitty documentation (`sw.kovidgoyal.net/kitty/conf/` and `sw.kovidgoyal.net/kitty/kittens/choose-fonts/`). All confirmations described in my own words, with only minimal quotation:

- **UI flow.** The official choose-fonts page describes the same sequence observed in §4.2: filter the family list by typing, press `Enter` to select a family, view previews of the regular/bold/italic faces, and fine-tune the regular face with the `R` key (which then adjusts the other styles). This matches the captured listing → faces flow and the `r`/`R` fine-tune key (`faces.go:L129`).
- **The four font keys.** The docs list the same four selection keys — font_family, bold_font, italic_font, bold_italic_font — matching `serialized()` (§8.2) and the enumeration in §10.2.
- **The `auto` keyword.** The docs explain that `auto` lets kitty automatically choose the bold/italic variants once `font_family` is set. This explains the observed `Enter` write in §8.4 where `font_family` is a `family=` spec and the other three are `auto`.
- **The `family="…"` syntax.** An official man-page example shows `font_family family="Fira Code"` — exactly the form observed on disk in §8.4.
- **`SIGUSR1` reload convention.** The docs confirm the config can be reloaded manually by pressing the reload shortcut (`ctrl+shift+f5`) or by sending `SIGUSR1` (documented as `kill -SIGUSR1 $KITTY_PID`), and that automatic reload is governed by `auto_reload_config`. This matches the `unix.SIGUSR1` sends at `tools/config/api.go:L357`/`L366` and the demonstration in §10.1.
- **Config location & override.** The docs state the config normally lives at `~/.config/kitty/kitty.conf`, overridable via `--config` or the `KITTY_CONFIG_DIRECTORY` environment variable; the kitty `--config` help further notes that when `KITTY_CONFIG_DIRECTORY` is set, "that directory is always used" and the normal search is skipped. This matches the honored-first behavior at `tools/utils/paths.go:L88-L90` that underpins the isolation method (§2.2).
- **Invocation.** The docs recommend running `kitten choose-fonts` — matching Q2 (§4).

**`--config-file-name` reconciliation (in my own words).** Some newer public man pages for `kitten-choose-fonts` list a second option, `--config-file-name [=kitty.conf]`, in addition to `--reload-in`. That reflects a **later release** of kitty. At the pinned commit `815df1e210e0`, `choose-fonts` implements **only** `--reload-in` — proven both by the source (`main.go:L86-L95`; a directory grep for `config-file-name` returns nothing) and by the runtime `--help` (§4.1) — and the destination file is **hardcoded** to `kitty.conf` (`final.go:L81`). The `--config-file-name` flag exists here only for the unrelated `themes` kitten (`kittens/themes/main.py:L40`). It must therefore not be treated as present for `choose-fonts` in this build.

---

## 12. Coverage pass

Every named item from the question is addressed and cross-referenced below. Items proven by observed runtime output are marked **[observed]**; items established purely from source reading are marked **[inferred]**.

- **[x] Q1 — Build from source & run a default instance** — §3. `./dev.sh build` → `Build successful.` **[observed]**; binaries at `kitty/launcher/{kitty,kitten}` v0.35.2 **[observed]**; `docs/build.rst:L14-L22` + caveat `L35-L37`; `dev.sh:L9`; `go.mod:L3`; `pyproject.toml:L2`; launch under ephemeral `KITTY_CONFIG_DIRECTORY` **[observed]**.
- **[x] Q2 — Invoking `choose-fonts`** — §4. `kitten choose-fonts` / `kitty +kitten choose-fonts`; captured scanning/listing/faces/final screens **[observed]**; Go-frontend + Python-backend split (`backend.go:L41` `+runpy`; `backend.py:L150-L168`; `kitty/fonts/list.py`).
- **[x] Q3a — Subcommand registration** — §5. `tools/cmd/tool/main.go:L82`; `EntryPoint`/`AddSubCommand` (`main.go:L74-L85`); `Run` closure → `GetOptionValues` → `main(&opts)`.
- **[x] Q3b — Option parsing** — §6. Single `--reload-in` (`main.go:L86-L95`); one-field `Options` (`main.go:L70-L72`); confirmed via `--help` **[observed]**.
- **[x] Q3c — Option-value flow** — §7. `opts` → `main` (`main.go:L16`) → `handler{opts}` (`main.go:L35`, `ui.go:L42-L62`) → panes (`ui.go:L81`) → consumed only at `final.go:L87`; preselect `list.go:L170` **[observed]**; transitions `list.go:L250`, `faces.go:L120`.
- **[x] Q3d — What finalization does** — §8. Four action labels (`final.go:L38/L40/L42/L44`); `serialized()` spacing (`final.go:L63-L70`); Enter path (`final.go:L78-L82`); verbatim on-disk `# BEGIN_KITTY_FONTS` block **[observed]**.
- **[x] Q4 — Remembered across restarts?** — §9. **Yes.** Restart #1 and #2 both preselect `>Fira Code`; on-disk md5 stable `91ab8f1ca4764f412d80012820301aa4` **[observed]**.
- **[x] Q5 — Or only current session?** — §9.4. The **`s` key** is the session/STDOUT-only path; on-disk `kitty.conf` md5 unchanged `91ab8f1ca4764f412d80012820301aa4` before/after, empty `diff -u`, and no file created on a fresh dir **[observed]**. `Enter` is the durable path, not session-only.
- **[x] All three `--reload-in` choices** — §10.1, all exercised via the real kitten `--reload-in` against a live kitty (marker-count deltas). `parent` → SIGUSR1 to `$KITTY_PID`, reload observed delta +1 (`final.go:L88-L89`, `api.go:L353-L357`) **[observed]**; `all` → SIGUSR1 to all kitty GUI procs, reload observed delta +1 (`final.go:L90-L91`, `api.go:L363-L366`) **[observed]**; `none` → no reload branch, no reload observed delta 0 (`final.go:L87-L92`) **[observed]**; `Default: "parent"` (`main.go:L91`).
- **[x] All four font keys** — §10.2. `font_family`, `bold_font`, `italic_font`, `bold_italic_font` (`final.go:L63-L70`; `faces.go:L14-L15`; `types.go:L78-L83`) **[observed on disk]**.
- **[x] Every final-screen key** — §10.3. `Enter` (patch), `Esc` (return to faces, `final.go:L73-L76`), `s`/`S` (STDOUT, `final.go:L104`), `Ctrl+c` (quit).
- **[x] `clone.Hidden = false` nuance** — §5.1. `main.go:L97`; both `choose-fonts` and `choose_fonts` visible at runtime **[observed]**.
- **[x] `--config-file-name` absence** — §6.3, §11. Absent for `choose-fonts` (grep empty; `--help` empty **[observed]**); themes-only (`kittens/themes/main.py:L40`); target hardcoded to `kitty.conf` (`final.go:L81`).
- **[x] `.bak` / atomic / `0o644` details** — §9.3. `.bak` only when prior file non-empty & `Write_backup` (`api.go:L343-L344`) **[observed: none first, created on 2nd write]**; atomic replace `api.go:L347`; mode default `0o644` (`api.go:L311-L313`) **[observed `-rw-r--r--`]**.
- **[x] `KITTY_CONFIG_DIRECTORY` isolation** — §2.2. Honored first (`tools/utils/paths.go:L88-L90`); every experiment used a throwaway dir **[observed]**.
- **[x] Backend `KITTY_PATH_TO_KITTY_EXE` hint** — §4.3. `backend.go:L33-L41`.
- **[x] Headless GPU constraint stated** — §2.3. Real kitten driven via PTY harness answering real kitty terminal queries; the reload was **also** exercised through the real kitten `--reload-in` against a live kitty GUI (§10.1) — nothing was substituted by a synthetic stand-in.

### 12.1 Repository cleanliness

All temporary scripts, ephemeral config directories, and harness files were created under `/tmp` (e.g. `/tmp/blitzy_harness/`, `/tmp/kitty-cf-*`) — never inside the repository — and removed after evidence capture. Any empty authoring-leftover directories (`blitzy/screen_recordings`, `blitzy/screenshots`) were also removed, so `-uall` lists exactly one path. In the authoring state (before this document is committed), `git status --porcelain -uall` names **only this new document** — verbatim:

```console
$ git status --porcelain -uall
?? blitzy/documentation/kitty_815df1e210e0.md
```

That single line is the whole of the change: no existing repository file was created, modified, or deleted; the only addition is `blitzy/documentation/kitty_815df1e210e0.md`. The unmodified source tree was additionally confirmed against the pinned commit:

```console
$ git diff --stat 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 -- . ':(exclude)blitzy/**'
$        # (empty — no source file differs from the pinned commit)
```

(Once the document is committed on the working branch, `git status --porcelain` is empty — a clean tree — because the sole change has been recorded.)

---

### Appendix — key `file:line` reference index (HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`)

| Concern | Reference |
|---|---|
| Kitten registration into command tree | `tools/cmd/tool/main.go:L82` |
| `EntryPoint` / `AddSubCommand` / `Run` closure | `kittens/choose_fonts/main.go:L74-L85` |
| Visible underscore clone `choose_fonts` | `kittens/choose_fonts/main.go:L96-L98` |
| `Options` struct (one field) | `kittens/choose_fonts/main.go:L70-L72` |
| `--reload-in` option spec | `kittens/choose_fonts/main.go:L86-L95` |
| `main(opts)` / `handler{opts}` | `kittens/choose_fonts/main.go:L16`, `L35` |
| Handler struct / pane set | `kittens/choose_fonts/ui.go:L42-L62`, `L81` |
| Family preselect / Enter→faces | `kittens/choose_fonts/list.go:L170`, `L250` |
| `faces_settings` / Enter→final / fine-tune keys | `kittens/choose_fonts/faces.go:L14-L15`, `L118-L120`, `L129-L135` |
| Final action labels | `kittens/choose_fonts/final.go:L38/L40/L42/L44` |
| `serialized()` (four keys) | `kittens/choose_fonts/final.go:L63-L70` |
| Esc → faces | `kittens/choose_fonts/final.go:L73-L76` |
| Enter → Patch + reload switch + quit | `kittens/choose_fonts/final.go:L78-L94` |
| `s`/`S` → STDOUT | `kittens/choose_fonts/final.go:L104-L106`; `main.go:L14`, `L64-L66` |
| `Patcher.Patch` algorithm | `tools/config/api.go:L305-L350` |
| `ReloadConfigInKitty` / `SIGUSR1` | `tools/config/api.go:L352-L366` (signals `L357`, `L366`) |
| `KITTY_CONFIG_DIRECTORY` honored first | `tools/utils/paths.go:L88-L90` |
| Backend spawn via `+runpy` | `kittens/choose_fonts/backend.go:L41` |
| Backend dispatch | `kittens/choose_fonts/backend.py:L150-L168` |
| Resolved-face JSON fields | `kittens/choose_fonts/types.go:L78-L83` |
| Build instructions / caveat | `docs/build.rst:L14-L22`, `L35-L37` |
| Build entry point | `dev.sh:L9` |
| Pinned Go / Python | `go.mod:L3`, `pyproject.toml:L2` |

