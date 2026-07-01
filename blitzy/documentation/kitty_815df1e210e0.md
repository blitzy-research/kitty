# Does pressing **Enter** in the `choose-fonts` kitten persist the font choice across restarts? — An Evidence-Backed Investigation

**Repository:** `kovidgoyal/kitty`  **Commit under investigation:** `815df1e21` ("Wire up applying of font config")  **Branch:** `kitty_815df1e210e0`

---

## TL;DR — The verdict (proven by runtime observation, not inference)

**Pressing `Enter` at the `choose-fonts` kitten's final confirmation step PERSISTS the font selection across restarts.** It does *two* distinct things:

1. **Persist to disk** — it writes the four font settings (`font_family`, `bold_font`, `italic_font`, `bold_italic_font`) into `kitty.conf`, the file kitty reads at startup. This is what makes the choice survive a restart. Proven below by quitting kitty and relaunching: the new instance loads the written `font_family` on startup.
2. **Apply to the running session** — it then sends `SIGUSR1` to the running kitty (`KITTY_PID`), which triggers an in-place config reload. Proven below by tracing the kitten's `kill(<KITTY_PID>, SIGUSR1)` syscall.

The other three keys on the final pane are **negative controls** that do **not** write `kitty.conf`: `s`/`S` writes the settings to **STDOUT only**, `Esc` returns to face selection, and `Ctrl+c` aborts ("canceled by user").

> **Methodology (per the governing rules):** every behavioral claim below was produced by *building and running* the code first, then quoting the *actual observed output* verbatim next to the exact `file:line` that produces it. Where the runtime disagreed with the plan's assumptions, the runtime is reported faithfully and the source code at commit `815df1e21` is treated as ground truth (see the **"Reported-as-observed" callouts**).

---

## 0. How the investigation was run (environment & isolation)

All evidence was gathered inside the user-specified container, on branch `blitzy-6a4af589-7e36-4e62-b3ac-bc9a794d3169` at HEAD `815df1e21`, with a **single** kitty instance launched under **default settings** in an **isolated, disposable** config directory so that any write is observable and reversible, and so no real user `kitty.conf` is touched.

Driving a live terminal UI headlessly was done with a virtual display (**Xvfb**) plus kitty's own **remote control** (`kitty @ send-key` / `kitty @ get-text`); `get-text --extent screen` returns the *verbatim rendered text* of the kitten's screens, which is the authoritative evidence for a TUI.

```console
$ export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe        # headless GL via Xvfb + llvmpipe
$ export KITTY_CONFIG_DIRECTORY=$(mktemp -d /tmp/cf_investigation/kitty_conf.XXXXXX)   # isolated, disposable
$ echo "$KITTY_CONFIG_DIRECTORY"
/tmp/cf_investigation/kitty_conf.L5Nn9o
$ ls -la "$KITTY_CONFIG_DIRECTORY"                                          # BEFORE baseline: empty, no kitty.conf
total 8
drwx------ 2 root root 4096 .
drwxr-xr-x 3 root root 4096 ..
```

`KITTY_CONFIG_DIRECTORY` is honored *first* when the kitten resolves its config directory, so the kitten's `utils.ConfigDir()` resolves to the temp dir:

```go
// tools/utils/paths.go:88-90
func ConfigDirForName(name string) (config_dir string) {
	if kcd := os.Getenv("KITTY_CONFIG_DIRECTORY"); kcd != "" {
		return Abspath(Expanduser(kcd))
```

---

## 1. Build & default-settings launch

### 1.1 Toolchain (observed, verbatim)

The repository is a mixed **C + Python + Go** monorepo. The build prerequisites are Go ≥ 1.22, a C compiler, and native libraries (harfbuzz, zlib, libpng, liblcms2, freetype, fontconfig) per `docs/build.rst`. The versions actually present:

```console
$ go version
go version go1.22.12 linux/amd64
$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
$ python3 --version
Python 3.13.7
```

- Go `1.22.12` satisfies the module's Go requirement: `go.mod:3` declares `go 1.22`.
- CPython satisfies `pyproject.toml:2` `requires-python = ">=3.8"` (the build uses a bundled `Python 3.14.6`; the system `python3` is `3.13.7`).

### 1.2 Build (observed)

kitty's build system is `python3 setup.py`; the developer wrapper `dev.sh` delegates to it via the bootstrap tool:

```sh
# dev.sh:9
exec go run bypy/devenv.go "$@"
```

The build was run for this checkout with kitty's own supported flag `--ignore-compiler-warnings` (required on Ubuntu 25.10, where a newer `wayland-protocols` adds enum values the vendored `glfw` switch does not yet handle while `setup.py` bakes in `-Werror`; no source is modified). The exact command and its **complete** captured output follow — this run was incremental, so only the one changed C translation unit was recompiled and relinked before the success line (the trailing `...` after `Compiling` is kitty's own progress text, not an elision):

```console
$ export PATH=$PATH:/usr/local/go/bin GOPATH=$HOME/go CI=true
$ ./dev.sh build --ignore-compiler-warnings
[1/1] Compiling kitty/data-types.c ...
 done
[1/1] Linking kitty/fast_data_types ...
 done
kitty/tools/cmd
Build successful. Run kitty as: kitty/launcher/kitty
$ echo "exit=$?"
exit=0
```

The build exited `0` with the final line **`Build successful. Run kitty as: kitty/launcher/kitty`**, producing the two runnable executables `kitty/launcher/kitty` and `kitty/launcher/kitten`. Validation that the resulting launcher runs:

```console
$ kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
```

### 1.3 Launch a single default-settings instance in isolation

`kitty --config NONE` starts kitty **without loading any config file** — this is what "default settings" means here. The special value is documented in the CLI help text:

```python
# kitty/cli.py:187-188
in sequence, which are merged. Use the special value :code:`NONE` to not load
any config file.
```

Because the host is headless, the Graphics-Protocol preview is backed by a virtual display (**Xvfb**). Xvfb was started and confirmed running *before* launching kitty:

```console
$ nohup Xvfb :99 -screen 0 1920x1080x24 -ac >/tmp/cf_investigation/xvfb.log 2>&1 &
$ ps -eo pid,args | grep '[X]vfb :99'
 106134 Xvfb :99 -screen 0 1920x1080x24 -ac
$ DISPLAY=:99 xdpyinfo | head -3
name of display:    :99
version number:    11.0
vendor string:    The X.Org Foundation
```

The `ps` line proves the Xvfb process (pid `106134`) is running on display `:99`, and `xdpyinfo` proves that display answers. The single kitty instance was then launched with `--config NONE` (default settings), remote control enabled so the kitten can be driven, and the isolated config dir exported:

```console
$ kitty --config NONE -o allow_remote_control=yes \
        --listen-on unix:/tmp/cf_investigation/kitty.sock \
        /bin/bash --norc --noprofile &
```

Confirmation that **exactly one** OS window/instance is up — the `kitty @ ls` reply is a JSON array whose length is the OS-window count:

```console
$ kitty @ --to unix:/tmp/cf_investigation/kitty.sock ls | python3 -c 'import sys,json; print(len(json.load(sys.stdin)))'
1
```

The window's environment — read verbatim from that same `kitty @ ls` reply — carries the isolated config dir and the instance's process id (the reload-signal target used later):

```console
$ kitty @ --to unix:/tmp/cf_investigation/kitty.sock ls \
    | python3 -c 'import sys,json; e=json.load(sys.stdin)[0]["tabs"][0]["windows"][0]["env"]; [print(f"{k} = {e[k]}") for k in ("KITTY_PID","KITTY_CONFIG_DIRECTORY","KITTY_WINDOW_ID")]'
KITTY_PID = 106407
KITTY_CONFIG_DIRECTORY = /tmp/cf_investigation/kitty_conf.L5Nn9o
KITTY_WINDOW_ID = 1
```

- **`KITTY_PID = 106407`** — the process id of this default instance; it is the target of the live-reload signal in §7.
- **`KITTY_CONFIG_DIRECTORY = /tmp/cf_investigation/kitty_conf.L5Nn9o`** — the isolated dir is inherited by the window, so any `kitty.conf` written by the kitten lands here.

---

## 2. Kitten invocation (from inside the running instance)

### 2.1 The working invocation: `kitten choose-fonts`

From inside the running kitty, the functional way to invoke the kitten at this commit is `kitten choose-fonts` — the Go binary directly. It is **not** equivalent to `kitty +kitten choose-fonts`: at commit `815df1e21` that `+kitten` form is a **no-op** (it produces no output and does not launch the kitten, because `kittens/choose_fonts/main.py` is empty and `choose_fonts` is not a wrapped kitten). That contrast is proven with its own runtime evidence in §2.2; this section documents the working `kitten choose-fonts` path.

**Claim: invoking `kitten choose-fonts` renders the family-listing screen.** The command was sent to the window, and after the font scan the rendered screen was read back with `get-text --extent screen`. The full screen (verbatim, no lines omitted; trailing spaces trimmed for readability):

```console
$ kitty @ --to unix:/tmp/cf_investigation/kitty.sock send-text --match id:1 'kitten choose-fonts\r'
$ kitty @ --to unix:/tmp/cf_investigation/kitty.sock get-text --match id:1 --extent screen
```

```text
 Cascadia Code         ║                DejaVu Sans Mono
 Cascadia Code NF      ║
 Cascadia Code PL      ║ Styles: Bold, Bold Oblique, Book, Oblique
 Cascadia Mono         ║
 Cascadia Mono NF      ║ Press the Enter key to choose this family
 Cascadia Mono PL      ║
 Comfy Code            ║ ────────────────── preview ──────────────────
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
Family:
```

The left column is the searchable family list (`>DejaVu Sans Mono` marks the current selection); the right pane previews the highlighted family and shows `Press the Enter key to choose this family`. The bottom `Family:` line is the incremental filter input.

**Claim: inside kitty, `kitten` resolves to the Go executable in kitty's own launcher directory.** Confirmed with `command -v` and `file`, both run inside the kitty window (verbatim, full paths):

```console
$ command -v kitten
/tmp/blitzy/kitty/blitzy-6a4af589-7e36-4e62-b3ac-bc9a794d3169_d6793b/kitty/launcher/kitten
$ file "$(command -v kitten)"
/tmp/blitzy/kitty/blitzy-6a4af589-7e36-4e62-b3ac-bc9a794d3169_d6793b/kitty/launcher/kitten: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, Go BuildID=-Bc_NsnRpPBojX_KrqzM/vheVIM4VUNPG7Ag6j7Ye/9MDtLjeqzjE9kPqmpEBi/DBbZSZMkPfE4zSBUXjVI, stripped
```

The `file` output — `ELF 64-bit LSB executable … Go BuildID=…` — confirms `choose-fonts` is a **Go kitten** compiled into the `kitten` binary, not a Python kitten. This is corroborated by the kitten's Python entry files being **empty**:

```console
$ wc -c kittens/choose_fonts/main.py kittens/choose_fonts/__init__.py
0 kittens/choose_fonts/main.py
0 kittens/choose_fonts/__init__.py
0 total
```

### 2.2 The kitty dispatch mechanism (`kitty/boss.py`) — and a reported-as-observed correction

When kitty itself dispatches a kitten (e.g. from a key mapping), it uses `run_kitten_with_metadata`, which decides between a **wrapped** (Go) path and a **non-wrapped** (Python-runner) path:

```python
# kitty/boss.py:1889
def run_kitten_with_metadata(
# kitty/boss.py:1902
        is_wrapped = kitten in wrapped_kitten_names()
# kitty/boss.py:1948-1951  (wrapped branch)
            if is_wrapped:
                cmd = [kitten_exe(), kitten]
                env['KITTEN_RUNNING_AS_UI'] = '1'
                env['KITTY_CONFIG_DIRECTORY'] = config_dir
# kitty/boss.py:1953  (non-wrapped branch)
                cmd = [kitty_exe(), '+runpy', 'from kittens.runner import main; main()']
```

The set of wrapped kittens comes from a compiled-in list:

```python
# kitty/constants.py:303-305
def wrapped_kitten_names() -> FrozenSet[str]:
    import kitty.fast_data_types as f
    return frozenset(f.wrapped_kitten_names())
```

> **⚠️ Reported-as-observed correction #A — `choose_fonts` is NOT a wrapped kitten at this commit.** The investigation plan assumed `is_wrapped == True` for `choose_fonts`. The runtime says otherwise:
>
> ```console
> $ kitty +runpy 'from kitty.constants import wrapped_kitten_names; print("choose_fonts" in wrapped_kitten_names())'
> False
> $ kitty +runpy 'from kitty.constants import wrapped_kitten_names; print(sorted(wrapped_kitten_names()))'
> ['ask', 'clipboard', 'diff', 'hints', 'hyperlinked_grep', 'icat', 'query_terminal', 'show_key', 'ssh', 'themes', 'transfer', 'unicode_input']
> ```
>
> `choose_fonts` is absent from the set. The set is a compile-time macro `WRAPPED_KITTENS` (`kitty/data-types.c:252`), sourced by `setup.py:1075-1081` (and `gen/go_code.py:429-435`) from a single line in the shell wrapper:
>
> ```console
> $ grep -n 'wrapped_kittens=' shell-integration/ssh/kitty
> 27:    wrapped_kittens="clipboard icat hyperlinked_grep ask hints unicode_input ssh themes diff show_key transfer query_terminal"
> ```
>
> `choose_fonts` does not appear there. Consequences observed at commit `815df1e21` (a *work-in-progress* commit, per its title "Wire up applying of font config"):
>
> - The Python runner path (`kitty +kitten choose-fonts`) is a **no-op** because the module it runs is empty:
>   ```console
>   $ kitty +kitten choose-fonts        # produces no output
>   $ echo $?
>   0
>   ```
>   `run_kitten` runs `runpy.run_module('kittens.choose_fonts.main')` when the name is a builtin kitten (`kittens/runner.py:115-116`), and `kittens/choose_fonts/main.py` is 0 bytes → nothing happens.
> - Therefore the **functional** way to invoke the kitten from inside kitty at this commit is the **Go binary directly**: `kitten choose-fonts` (demonstrated in §2.1). This is what the rest of this document uses.

For reference, the shell `+kitten` entry point and the Python runner it delegates to (the *contrast* path that `choose_fonts` does **not** use functionally here):

```python
# kitty/entry_points.py:118, 126-127
def run_kitten(args: List[str]) -> None:
    ...
    from kittens.runner import run_kitten as rk
    rk(kitten)
# kitty/entry_points.py:164
namespaced_entry_points['kitten'] = run_kitten
```

---

## 3. Subcommand registration with the CLI framework

The `choose-fonts` subcommand is registered into the Go `kitten` binary's CLI tree from the tool's `main`:

```go
// tools/cmd/tool/main.go:82
	choose_fonts.EntryPoint(root)
```

`EntryPoint` adds the subcommand and (at the end) a visible clone alias:

```go
// kittens/choose_fonts/main.go:74-77
func EntryPoint(root *cli.Command) {
	ans := root.AddSubCommand(&cli.Command{
		Name:             "choose-fonts",
		ShortDescription: "Choose the fonts used in kitty",
// kittens/choose_fonts/main.go:96-98
	clone := root.AddClone(ans.Group, ans)
	clone.Hidden = false
	clone.Name = "choose_fonts"
```

Runtime confirmation that the subcommand is registered and dispatchable — `kitten choose-fonts --help` returns the subcommand's own usage header and description. These are the exact opening lines of the output; the **complete** help (the full option surface) is reproduced verbatim in §4:

```console
$ kitten choose-fonts --help
Usage: kitten choose-fonts 

Choose the fonts used in kitty
```

> **⚠️ Reported-as-observed nuance #2 — the clone alias is NOT hidden.** `kittens/choose_fonts/main.go:97` sets `clone.Hidden = false`, so the `choose_fonts` alias is **visible**, not a "hidden alias".

---

## 4. Option parsing — the `--reload-in` option

The kitten defines exactly one option. Its destination struct:

```go
// kittens/choose_fonts/main.go:70-72
type Options struct {
	Reload_in string
}
```

The option declaration, with its choices and default:

```go
// kittens/choose_fonts/main.go:86-95
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

The parsed values are populated into the struct inside the `Run` closure, then handed to `main`:

```go
// kittens/choose_fonts/main.go:78-84
		Run: func(cmd *cli.Command, args []string) (rc int, err error) {
			opts := Options{}
			if err = cmd.GetOptionValues(&opts); err != nil {
				return 1, err
			}
			return main(&opts)
		},
```

Runtime confirmation of the parsed option surface — the `--help` output shows the option name, its `[=parent]` default marker, and the exact `Choices: parent, all, none`:

```console
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

> **Source-is-ground-truth note — the `--config-file` option is ABSENT here.** Newer upstream docs mention a `--config-file` option; at commit `815df1e21` the kitten declares **only** `--reload-in`. Verified against the live help output:
>
> ```console
> $ kitten choose-fonts --help | grep -- '--config-file'   # (no match)
> $ echo "matched? $?"
> matched? 1
> ```
>
> The only options present are `--reload-in [=parent]` and `--help, -h`. The code is authoritative.

---

## 5. Value flow — from family selection to the finalization pane

### 5.1 The kitten is a state machine of panes

```go
// kittens/choose_fonts/ui.go:17-23
type State int

const (
	SCANNING_FAMILIES State = iota
	LISTING_FAMILIES
	CHOOSING_FACES
)
```

The `handler` holds the parsed options and the four panes; the panes are wired in a slice:

```go
// kittens/choose_fonts/ui.go:42-58
type handler struct {
	opts                 *Options
	...
	listing    FontList
	faces      faces
	face_pane  face_panel
	final_pane final_pane
// kittens/choose_fonts/ui.go:81
	h.panes = []pane{&h.listing, &h.faces, &h.face_pane, &h.final_pane}
```

### 5.2 Family list → faces

Pressing `Enter` in the family list advances to the faces pane (`FontList.on_key_event`):

```go
// kittens/choose_fonts/list.go:246-250
func (self *FontList) on_key_event(event *loop.KeyEvent) (err error) {
	if event.MatchesPressOrRepeat("enter") {
		...
			return self.handler.faces.on_enter(family)
```

`faces.on_enter` **seeds** the `faces_settings` struct — this is the value that will ultimately be written. Its four fields, and their observed defaults, are set at `kittens/choose_fonts/faces.go:152-155`:

```go
// kittens/choose_fonts/faces.go:14-16
type faces_settings struct {
	font_family, bold_font, italic_font, bold_italic_font string
}
// kittens/choose_fonts/faces.go:152-155
	d(r.Font_family, &self.settings.font_family, fmt.Sprintf(`family="%s"`, family))
	d(r.Bold_font, &self.settings.bold_font, "auto")
	d(r.Italic_font, &self.settings.italic_font, "auto")
	d(r.Bold_italic_font, &self.settings.bold_italic_font, "auto")
```

Observed: for family `Hack`, `font_family` becomes `family="Hack"` and the other three default to `auto` (matches the written file in §6). The faces pane can be fine-tuned per style with `r/R`, `b/B`, `i/I`, `o/O`, which open the per-face panel:

```go
// kittens/choose_fonts/faces.go:129-136
		case "r", "R":
			which = "font_family"
		case "b", "B":
			which = "bold_font"
		case "i", "I":
			which = "italic_font"
		case "o", "O":
			which = "bold_italic_font"
```

### 5.3 Faces → final pane

Pressing `Enter` in the faces pane forwards the family and the seeded settings to the final pane:

```go
// kittens/choose_fonts/faces.go:118-121
	if event.MatchesPressOrRepeat("enter") {
		event.Handled = true
		return self.handler.final_pane.on_enter(self.family, self.settings)
	}
```

Observed transition — after choosing `Hack` and pressing `Enter`, the **faces pane** rendered the resolved faces:

```text
                                 Hack
Press Enter to select this font, Esc to go back to the font list or any
of the highlighted keys below to fine-tune the appearance of the
individual font styles.
Regular: Hack-Regular
Bold: Hack-Bold
Italic: Hack-Italic
Bold-Italic: Hack-BoldItalic
```

### 5.4 Where the family data comes from (Python backend)

The family list and preview samples are produced by a Python back-end that the Go front-end spawns over a pipe:

```go
// kittens/choose_fonts/backend.go:41
	k.cmd = exec.Command(exe, "+runpy", "from kittens.choose_fonts.backend import main; main()")
```

That engine is `kittens/choose_fonts/backend.py` (uses `kitty.fonts.*` and `parse_font_spec` to list monospaced families and render samples). Supporting UI files traced during the investigation: `kittens/choose_fonts/list.go`/`kittens/choose_fonts/family_list.go` (browse & filter), `kittens/choose_fonts/face.go` (per-face fine-tuning panel), `kittens/choose_fonts/graphics.go` (Graphics-Protocol live preview), and `kittens/choose_fonts/styles.go`/`kittens/choose_fonts/types.go` (style helpers & shared IPC types). This IPC path produces the family data but is orthogonal to the persistence write.

---

## 6. Finalization behavior (with captured output)

### 6.1 The final confirmation pane and its legend (primary evidence)

Pressing `Enter` in the faces pane advances to the **final confirmation pane**. Its action legend is itself primary evidence of what each key does. The rendered screen (verbatim `get-text`, family `Hack`):

```text
You have chosen the Hack family

What would you like to do?

Enter to modify kitty.conf and use the new fonts
Esc to abort and return to font selection
s to write the new font settings to STDOUT
Ctrl+c to quit
```

Those four legend lines are produced here:

```go
// kittens/choose_fonts/final.go:33-45
	self.render_lines(0,
		fmt.Sprintf("You have chosen the %s family", s(current_val_style, self.family)),
		"",
		"What would you like to do?",
		"",
		fmt.Sprintf("%s to modify %s and use the new fonts", h("Enter"), s("italic", `kitty.conf`)),
		"",
		fmt.Sprintf("%s to abort and return to font selection", h("Esc")),
		"",
		fmt.Sprintf("%s to write the new font settings to %s", h("s"), s("italic", `STDOUT`)),
		"",
		fmt.Sprintf("%s to quit", h("Ctrl+c")),
	)
```

The four action lines are at `kittens/choose_fonts/final.go:38` (`Enter`), `:40` (`Esc`), `:42` (`s`), and `:44` (`Ctrl+c`); the `s` and `Enter` lines render `kitty.conf` and `STDOUT` via the `s("italic", …)` styler, which is why the rendered screen shows the plain words `kitty.conf` and `STDOUT`.

The `Enter` legend — **"Enter to modify kitty.conf and use the new fonts"** — states the two effects proven in §7: *modify kitty.conf* (persist) and *use the new fonts* (apply to session).

### 6.2 What `Enter` does — the finalization mechanics

The `Enter` handler builds a config **patcher**, targets `kitty.conf` in the (isolated) config directory, and patches in the serialized font settings:

```go
// kittens/choose_fonts/final.go:78-82
	if event.MatchesPressOrRepeat("enter") {
		event.Handled = true
		patcher := config.Patcher{Write_backup: true}
		path := filepath.Join(utils.ConfigDir(), "kitty.conf")
		updated, err := patcher.Patch(path, "KITTY_FONTS", self.settings.serialized(), "font_family", "bold_font", "italic_font", "bold_italic_font")
```

The payload comes from `serialized()`, which emits exactly four lines:

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

`Patcher.Patch` writes the block wrapped in delimiters, at mode `0o644`, atomically:

```go
// tools/config/api.go:310-313
func (self Patcher) Patch(path, sentinel, content string, settings_to_comment_out ...string) (updated bool, err error) {
	if self.Mode == 0 {
		self.Mode = 0o644
	}
// tools/config/api.go:330  (block insertion format)
	addition := fmt.Sprintf("# BEGIN_%s\n%s\n# END_%s", sentinel, content, sentinel)
// tools/config/api.go:347  (atomic commit)
	return true, utils.AtomicUpdateFile(path, nraw, self.Mode)
```

### 6.3 The written block (captured verbatim)

Immediately **before** the decisive `Enter`, `kitty.conf` did not exist:

```console
$ ls -la "$KITTY_CONFIG_DIRECTORY"
total 8
drwx------ 2 root root 4096 .
drwxr-xr-x 3 root root 4096 ..
$ test -e "$KITTY_CONFIG_DIRECTORY/kitty.conf" && echo EXISTS || echo "kitty.conf ABSENT (before)"
kitty.conf ABSENT (before)
```

**After** pressing `Enter`, kitty created a 134-byte `kitty.conf` at mode `0o644` (matching `tools/config/api.go:311-313`):

```console
$ ls -la "$KITTY_CONFIG_DIRECTORY"
-rw-r--r-- 1 root root  134 kitty.conf
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"
# BEGIN_KITTY_FONTS
font_family      family="Hack"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
```

This is exactly the `serialized()` four-line payload (`kittens/choose_fonts/final.go:63-70`) wrapped by the `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` delimiters from `tools/config/api.go:330`, with values seeded at `kittens/choose_fonts/faces.go:152-155` (`font_family family="Hack"`, the rest `auto`). After the write the kitten **exited** (the window's foreground process returned to `/bin/bash`), matching `lp.Quit(0)` at `kittens/choose_fonts/final.go:94`.

---

## 7. The persistence verdict — proven across a restart

The core question: does `Enter` **persist** the choice across restarts, or merely change the current session? The answer is **both** — it persists to disk *and* live-applies. Four independent pieces of evidence follow.

### 7.1 (a) Before/after diff of `kitty.conf`

`kitty.conf` went from **absent** (§6.3, "before") to containing the delimited `KITTY_FONTS` block (§6.3, "after"). That write to `kitty.conf` — the file kitty reads at startup — is the persistence mechanism. The before/after is unambiguous:

```console
# BEFORE: kitty.conf ABSENT (before)
# AFTER:
# BEGIN_KITTY_FONTS
font_family      family="Hack"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
```

### 7.2 (b) The `.bak` backup — reported exactly as observed (nuance #3)

The backup is **guarded** so it is only written when the pre-existing file was non-empty:

```go
// tools/config/api.go:343-344
		if len(raw) > 0 && self.Write_backup {
			_ = os.WriteFile(backup_path+".bak", raw, self.Mode)
```

> **⚠️ Reported-as-observed nuance #3 — no `.bak` on the first write.** On the **first** finalization the file was empty (`len(raw) == 0`), so **no `.bak` was produced**:
>
> ```console
> $ test -e "$KITTY_CONFIG_DIRECTORY/kitty.conf.bak" && echo ".bak EXISTS" || echo "NO .bak (first write, len(raw)==0)"
> NO .bak (first write, len(raw)==0)
> ```
>
> Finalizing a **second** time (family `Fira Code`, so the file already had content) produced the `.bak`, and its contents were byte-for-byte the previous `Hack` config:
>
> ```console
> $ ls -la "$KITTY_CONFIG_DIRECTORY"
> -rw-r--r-- 1 root root  139 kitty.conf
> -rw-r--r-- 1 root root  134 kitty.conf.bak
> $ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf.bak"
> # BEGIN_KITTY_FONTS
> font_family      family="Hack"
> bold_font        auto
> italic_font      auto
> bold_italic_font auto
> # END_KITTY_FONTS
> $ diff kitty.conf.before_2nd_write "$KITTY_CONFIG_DIRECTORY/kitty.conf.bak" && echo "IDENTICAL (backup == prior content)"
> IDENTICAL (backup == prior content)
> ```

### 7.3 (c) The `SIGUSR1` live reload (apply-to-session)

After a successful write, the kitten signals kitty to reload, based on `opts.Reload_in`:

```go
// kittens/choose_fonts/final.go:86-91
		if updated {
			switch self.handler.opts.Reload_in {
			case "parent":
				config.ReloadConfigInKitty(true)
			case "all":
				config.ReloadConfigInKitty(false)
```

> **⚠️ Reported-as-observed nuance #1 — the `Reload_in` read is at line 87.** The `self.handler.opts.Reload_in` value is read by the `switch` at `kittens/choose_fonts/final.go:87` (inside the `if updated {` block that begins at `kittens/choose_fonts/final.go:86`).

`ReloadConfigInKitty` resolves the target kitty from `KITTY_PID` and sends `SIGUSR1`:

```go
// tools/config/api.go:352-357
func ReloadConfigInKitty(in_parent_only bool) error {
	if in_parent_only {
		if pid, err := strconv.Atoi(os.Getenv("KITTY_PID")); err == nil {
			if p, err := process.NewProcess(int32(pid)); err == nil {
				if c, err := p.CmdlineSlice(); err == nil && is_kitty_gui_cmdline(c...) {
					return p.SendSignal(unix.SIGUSR1)
```

Running the finalization (family `Source Code Pro`) with the kitten under `strace -f -e trace=kill,tgkill` captured the **exact** signal-send syscall — the kitten (pid `109984`) sending `SIGUSR1` to the parent kitty PID `106407` (which is the `KITTY_PID` from §1.3):

```console
$ grep -E "SIGUSR1" /tmp/cf_investigation/kitten_sig.log
109984 kill(106407, SIGUSR1)            = 0
```

On the receiving side, kitty's child-monitor turns `SIGUSR1` into a config reload:

```c
// kitty/child-monitor.c:1373-1374
        case SIGUSR1:
            ss->reload_config = true;
```

*(Note: a manual `kill -USR1 <pid>` does not appear as a strace "--- SIGUSR1 ---" delivery line on the kitty side because kitty consumes handled signals via a signalfd/self-pipe; tracing the **sender** — the kitten — is therefore the correct proof, hence `kill(106407, SIGUSR1) = 0` above.)*

### 7.4 (d) Post-restart confirmation (the definitive proof)

Persistence means the choice survives a **restart**. The original `--config NONE` instance (`KITTY_PID 106407`) was quit, and kitty was relaunched **without** `--config NONE`, pointed at the **same** `KITTY_CONFIG_DIRECTORY` (so the written `kitty.conf` is loaded), with `--debug-font-fallback` to reveal which fonts it actually loaded on startup:

```console
$ kill 106407          # quit original instance
$ kitty -o allow_remote_control=yes --listen-on unix:/tmp/cf_investigation/kitty2.sock \
        --debug-font-fallback /bin/bash --norc --noprofile &
$ grep -iE "Text fonts:|Normal:|Bold:|Italic:|Bold-Italic:" /tmp/cf_investigation/kitty_restart.log
[0.153] Text fonts:
[0.153]   Normal: SourceCodePro-Regular: /root/.local/share/fonts/SourceCodePro-Regular.otf:0
[0.153]   Bold: SourceCodePro-Semibold: /root/.local/share/fonts/SourceCodePro-Semibold.otf:0
[0.153]   Italic: SourceCodePro-It: /root/.local/share/fonts/SourceCodePro-It.otf:0
[0.153]   Bold-Italic: SourceCodePro-SemiboldIt: /root/.local/share/fonts/SourceCodePro-SemiboldIt.otf:0
```

The freshly-started kitty loaded **Source Code Pro** — the family written by the kitten — on startup. This is the persisted choice being applied across a restart.

The config-load pipeline (`kitty/config.py` → typed `Options` in `kitty/options/types.py`) confirms it directly. **Claim: loading the persisted file yields the written `font_family`.** The full, unelided `FontSpec` returned:

```console
$ kitty +runpy "from kitty.config import load_config; print(load_config('$KITTY_CONFIG_DIRECTORY/kitty.conf').font_family)"
FontSpec(family='Source Code Pro', style='', postscript_name='', full_name='', system='', axes=(), variable_name='', created_from_string='family="Source Code Pro"')
```

**Claim: loading defaults (no file) yields the built-in `monospace`.** The full, unelided `FontSpec` returned:

```console
$ kitty +runpy "from kitty.config import load_config; print(load_config().font_family)"
FontSpec(family='', style='', postscript_name='', full_name='', system='monospace', axes=(), variable_name='', created_from_string='')
```

The contrast is decisive: the persisted file produces `FontSpec(family='Source Code Pro', …)` while the no-file default produces `FontSpec(…, system='monospace', …)` — the written value is what kitty loads on startup.

### 7.5 Verdict + reasoning

**Enter PERSISTS the font choice across restarts.** Reasoning, grounded in the observations above:

- **Persist-to-disk:** `Enter` writes the four font settings into `kitty.conf` (`kittens/choose_fonts/final.go:78-82` → `tools/config/api.go:330,347`), the file kitty reads at startup. Proven by §7.4: after quitting and relaunching, kitty loaded the written `font_family` (`Source Code Pro`) on startup, and `load_config()` returns the persisted `FontSpec` rather than the default `monospace`. This is what makes the choice survive a restart.
- **Apply-to-session:** `Enter` *additionally* sends `SIGUSR1` to the running kitty (`kittens/choose_fonts/final.go:87` → `tools/config/api.go:352-357`), which sets `reload_config = true` (`kitty/child-monitor.c:1373-1374`), live-applying the new fonts to the current session. Proven by §7.3: `109984 kill(106407, SIGUSR1) = 0`.

These are two distinct effects; `Enter` does both. The choice is therefore **not** merely a current-session change — it is persisted to disk and reloaded on every subsequent startup.

---

## 8. Non-persisting contrast paths (negative controls)

The other three legend keys are the negative controls that isolate persistence to the `Enter` path. Baseline for these tests: `kitty.conf` held `font_family family="Source Code Pro"`, and it stayed **unchanged** through all three.

> **Key-injection note (for reproducibility).** The kitten enables kitty's keyboard protocol. Sending a plain `s` via `send-text` arrives with `from_key_event == false`, and the final pane's `on_text` handler only acts when `from_key_event` is true — so a plain `s` is ignored. The working method was to craft the full kitty-keyboard-protocol CSI-u sequence *with* an associated-text section: `\x1b[115;1;115u` (keycode 115 = `s`, no mods, text `s`), which the loop parses into a text-bearing key event.

### 8.1 `s` / `S` — writes to STDOUT only (no file write)

```go
// kittens/choose_fonts/final.go:101-106
func (self *final_pane) on_text(text string, from_key_event bool, in_bracketed_paste bool) (err error) {
	if from_key_event {
		switch text {
		case "s", "S":
			output_on_exit = self.settings.serialized() + "\n"
			self.lp.Quit(0)
```

`output_on_exit` is flushed to STDOUT after the loop ends:

```go
// kittens/choose_fonts/main.go:64-65
	if output_on_exit != "" {
		os.Stdout.WriteString(output_on_exit)
```

Observed — running the kitten with STDOUT redirected to a file, selecting `JetBrains Mono NL`, reaching the final pane, and injecting the crafted `s`, produced the serialized settings on **STDOUT** (110 bytes), while `kitty.conf` was **unchanged**:

```console
$ cat /tmp/cf_investigation/s_stdout.txt
font_family      family="JetBrains Mono NL"
bold_font        auto
italic_font      auto
bold_italic_font auto
$ [ "$md5_before" = "$md5_after_s" ] && echo "UNCHANGED: $(grep font_family "$KITTY_CONFIG_DIRECTORY/kitty.conf")"
UNCHANGED: font_family      family="Source Code Pro"
```

Note the STDOUT payload is the **raw `serialized()`** output — it does **not** carry the `# BEGIN_KITTY_FONTS … # END_KITTY_FONTS` wrapper, because that wrapper is added only by `Patcher.Patch` (`tools/config/api.go:330`) on the `Enter` path. The kitten then exited via `lp.Quit(0)` (`kittens/choose_fonts/final.go:106`).

### 8.2 `Esc` — returns to face selection (no write)

```go
// kittens/choose_fonts/final.go:73-76
	if event.MatchesPressOrRepeat("esc") {
		event.Handled = true
		self.handler.current_pane = &self.handler.faces
		return self.handler.draw_screen()
```

Observed — `send-key esc` at the final pane switched the screen back to the **faces** pane, and `kitty.conf` was unchanged:

```text
                           JetBrains Mono NL
Press Enter to select this font, Esc to go back to the font list or any
of the highlighted keys below to fine-tune the appearance of the
individual font styles.
Regular: JetBrainsMonoNL-Regular
Bold: JetBrainsMonoNL-SemiBold
Italic: JetBrainsMonoNL-Italic
Bold-Italic: JetBrainsMonoNL-SemiBoldItalic
```

### 8.3 `Ctrl+c` — aborts, "canceled by user" (no write) — reported as observed

> **⚠️ Reported-as-observed correction — the source is `kittens/choose_fonts/ui.go`, not `kittens/choose_fonts/main.go`.** The plan expected `Ctrl+c` to hit a "Killed by signal" death-signal branch (`kittens/choose_fonts/main.go:58-62`). The observed output was instead `Error: canceled by user`, produced by the handler's own key event:
>
> ```go
> // kittens/choose_fonts/ui.go:195-199
> func (h *handler) on_key_event(event *loop.KeyEvent) (err error) {
> 	if event.MatchesPressOrRepeat("ctrl+c") {
> 		event.Handled = true
> 		return fmt.Errorf("canceled by user")
> 	}
> ```
>
> Observed at the final pane with `send-key ctrl+c`:
>
> ```console
> $ tail -1 /tmp/cf_investigation/ctrlc_out.txt
> Error: canceled by user
> ```
>
> The handler returns this error, which propagates out through `lp.Run()` and causes the kitten to exit **before** any patch occurs; `kitty.conf` was unchanged. (The loop's own SIGINT path is not reached because the handler's key event returns the error first.)

### 8.4 Summary of the negative controls

| Key | Action | Writes `kitty.conf`? | Evidence |
|-----|--------|----------------------|----------|
| `Enter` | modify `kitty.conf` + reload | **YES** (persist + apply) | §6, §7 |
| `s`/`S` | serialized settings → STDOUT | No | §8.1 (`kittens/choose_fonts/final.go:101-106`) |
| `Esc` | back to face selection | No | §8.2 (`kittens/choose_fonts/final.go:73-76`) |
| `Ctrl+c` | abort, "canceled by user" | No | §8.3 (`kittens/choose_fonts/ui.go:195-199`) |

Only `Enter` writes `kitty.conf`; persistence is isolated to the `Enter` path.

---

## 9. Coverage-pass checklist

Every named item from the question, mapped to the section that answers it by name with verbatim evidence and an exact `file:line`.

| # | Named item from the question | Answered in | Key evidence / citation |
|---|------------------------------|-------------|-------------------------|
| 1 | **Build the repository** | §1.1–1.2 | `go version` → `go1.22.12`; `go.mod:3` (`go 1.22`); `pyproject.toml:2`; build via `setup.py` / `dev.sh:9` |
| 2 | **Start a single instance with default settings** | §1.3 | `kitty --config NONE`; `kitty/cli.py:187-188` ("special value `NONE` to not load any config file") |
| 3 | **Isolate config** (implicit) | §0, §1.3 | `KITTY_CONFIG_DIRECTORY` via `tools/utils/paths.go:88-90`; empty-dir baseline |
| 4 | **Invoke the `choose-fonts` kitten from inside** | §2.1 | Functional invocation is `kitten choose-fonts` (Go binary; renders the family-listing screen; `file` shows a Go ELF). `kitty +kitten choose-fonts` is **not** equivalent — at this commit it is a no-op (see row 6 and §2.2) |
| 5 | **Wrapped-kitten dispatch** (`run_kitten_with_metadata`, `wrapped_kitten_names`, `KITTEN_RUNNING_AS_UI`) | §2.2 | `kitty/boss.py:1889/1902/1948-1951`; `kitty/constants.py:303-305`; **correction #A**: `choose_fonts ∉ wrapped_kitten_names()` (runtime `False`) |
| 6 | **`+kitten` shell entry / `kittens/runner.py` contrast** | §2.2 | `kitty/entry_points.py:118/126-127/164`; empty `main.py`/`__init__.py` (`wc -c` = 0); `+kitten choose-fonts` is a no-op |
| 7 | **Subcommand registration** | §3 | `tools/cmd/tool/main.go:82`; `kittens/choose_fonts/main.go:74-77`; clone alias `kittens/choose_fonts/main.go:96-98` (**nuance #2**: `clone.Hidden = false`) |
| 8 | **Option parsing** (`--reload-in`, choices `parent, all, none`, default `parent`, `GetOptionValues`) | §4 | `kittens/choose_fonts/main.go:70-72`, `86-95`, `78-84`; live `--help`; `--config-file` **ABSENT** |
| 9 | **Value flow to finalization** (`faces_settings`, `faces.on_enter`, `final_pane.on_enter`, handler/panes, state machine, fine-tune keys, backend `+runpy`) | §5 | `kittens/choose_fonts/faces.go:14-16/118-121/129-136/152-155`; `kittens/choose_fonts/ui.go:17-23/42-58/81`; `kittens/choose_fonts/list.go:246-250`; `kittens/choose_fonts/backend.go:41` |
| 10 | **Finalization behavior with output** (legend, Enter handler, `Patcher.Patch`, `serialized()`, written `KITTY_FONTS` block) | §6 | `kittens/choose_fonts/final.go:38/40/42/44`, `63-70`, `78-82`, `94`; `tools/config/api.go:310-313/330/347`; verbatim 134-byte `kitty.conf` |
| 11 | **THE persistence question — persist across restart or session-only?** | §7 (verdict §7.5) | before/after diff; **restart** loads `Source Code Pro` (`--debug-font-fallback`); `load_config()` → persisted `FontSpec` vs default `monospace` |
| 12 | **`SIGUSR1` live reload** (`ReloadConfigInKitty`, `KITTY_PID`) | §7.3 | `kittens/choose_fonts/final.go:86-91` (**nuance #1**: read at L87); `tools/config/api.go:352-357`; strace `kill(106407, SIGUSR1) = 0`; `kitty/child-monitor.c:1373-1374` |
| 13 | **`.bak` backup behavior** | §7.2 | `tools/config/api.go:343-344` (**nuance #3**: no `.bak` on first write; appears on 2nd, `diff` IDENTICAL) |
| 14 | **`s` / `S` contrast (STDOUT only)** | §8.1 | `kittens/choose_fonts/final.go:101-106`; `kittens/choose_fonts/main.go:64-65`; verbatim 110-byte STDOUT; `kitty.conf` UNCHANGED |
| 15 | **`Esc` contrast (return to selection)** | §8.2 | `kittens/choose_fonts/final.go:73-76`; faces pane re-rendered; no write |
| 16 | **`Ctrl+c` contrast (quit / canceled)** | §8.3 | **correction**: `Error: canceled by user` from `kittens/choose_fonts/ui.go:195-199` (not `kittens/choose_fonts/main.go:58-62`); no write |

**All three required nuances reported as observed:** #1 `Reload_in` read at `kittens/choose_fonts/final.go:87`; #2 `clone.Hidden = false` (`kittens/choose_fonts/main.go:97`, alias visible); #3 `.bak` only when `len(raw) > 0` (`tools/config/api.go:343`, none on first write).

**Reported-as-observed deviations from the plan (source is ground truth):** (A) `choose_fonts` is not in `wrapped_kitten_names()` at this commit, so the functional invocation is the Go `kitten choose-fonts` binary and `+kitten choose-fonts` is a no-op; (B) `Ctrl+c` yields `Error: canceled by user` from `kittens/choose_fonts/ui.go:195-199`; (C) `--debug-config` is unavailable in this build, so `--debug-font-fallback` plus `load_config()` were used to prove startup application.

**Verdict (restated):** pressing **`Enter`** at the `choose-fonts` final pane **PERSISTS** the selection across restarts by writing `kitty.conf` (survives restart) **and** live-applies it via `SIGUSR1`. It is not a session-only change.
