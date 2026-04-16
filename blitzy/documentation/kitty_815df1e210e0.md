# `choose-fonts` kitten — onboarding answers

> Source branch: `kitty_815df1e210e0` (commit `815df1e210e0`, message "Wire up applying of font config")
> Target repository: `kovidgoyal/kitty`
>
> This document answers a specific set of behavioural questions about the
> `choose-fonts` kitten. Every claim is backed by explicit source-file paths
> and line numbers so that each answer is traceable to the code as the
> ground truth. Short rationale is included after each factual claim so
> the reader understands not only *what* happens but *why* the design
> works that way.

---

## 1. Build and launch

### 1.1 System dependencies

`setup.py` compiles C extensions, a C launcher, and several Go binaries. To
build from source on a Debian/Ubuntu host you need:

| Package | Why it is needed |
|---|---|
| `gcc` / `g++` | C/C++ compiler for the extension modules and the launcher. `setup.py` picks `gcc` first if available (`setup.py`, lines 306–311). |
| `pkg-config` | Used to discover flags for fontconfig, harfbuzz, libxxhash (for the transfer kitten's rsync extension — `setup.py:986`). |
| `libfontconfig1-dev` | Required by kitty's font backend (`kitty/fonts/fontconfig.py` on Linux — pulled in via `kitty/fonts/list.py:14`). |
| `libharfbuzz-dev` | Text-shaping engine used by the C renderer. |
| `libfreetype6-dev` | Font rasterization. |
| `libgl-dev`, `libegl-dev` | OpenGL / EGL context used by the renderer. |
| `libx11-dev`, `libxkbcommon-dev` | X11 windowing backend. |
| `libwayland-dev`, `wayland-protocols` | Wayland windowing backend. |
| `libdbus-1-dev` | D-Bus integration. |
| `librsync-dev` / `libxxhash-dev` | Transfer kitten's rsync extension (`setup.py:986`). |
| Go toolchain ≥ 1.22 | Required by `go.mod:3` (`go 1.22`). Used to build the `kitten` binary and the `kitty-tool` Go sub-binaries. |
| Python ≥ 3.8 | Required by `pyproject.toml:2` (`requires-python = ">=3.8"`). |

### 1.2 Build

The canonical command is:

```bash
python3 setup.py build --ignore-compiler-warnings
```

Rationale: `Makefile`'s default `all` target is just `python3 setup.py $(VVAL)` (`Makefile`, lines 12–13), i.e. `setup.py` is the single orchestrator. It compiles the C extension (`compile_c_extension`, `setup.py:856`), GLFW sources (`compile_glfw`, `setup.py:932`), the kittens C helpers (`compile_kittens`, `setup.py:967`), builds Go static binaries via `build_static_binaries` (`setup.py:1195`), and finally assembles the launcher (`build_launcher`, `setup.py:1230`). `--ignore-compiler-warnings` is handy in containers where the system compiler emits warnings that would otherwise fail a strict build.

`dev.sh` is a one-liner that delegates to the Go-based developer shell:

```sh
exec go run bypy/devenv.go "$@"
```

It is a convenience wrapper used during day-to-day development; for a one-shot build the `setup.py build` invocation above is enough.

### 1.3 Launch (single instance, default settings)

After a successful build, the launcher binary is `./kitty/launcher/kitty`:

```bash
./kitty/launcher/kitty
```

This is the C launcher compiled by `build_launcher()` (`setup.py:1230`). It
sets up Python, then either dispatches to a wrapped kitten or starts the
kitty GUI. With no arguments, `is_kitty_gui_cmdline()` in
`tools/config/api.go:282–303` returns `true` (`len(cmd) == 1`), so it's a
plain kitty GUI launch.

### 1.4 Headless verification

kitty is a GUI terminal emulator, so a display is required. In a
headless environment a virtual X server is enough:

```bash
Xvfb :99 -screen 0 1280x800x24 &
DISPLAY=:99 ./kitty/launcher/kitty
```

This allows confirming that the build works without a real display.

---

## 2. Invoking the `choose-fonts` kitten

### 2.1 Primary invocation

From inside a running kitty window:

```bash
kitten choose-fonts
```

### 2.2 Alternative paths

* `kitty +kitten choose-fonts` — uses the `+kitten` entry-point (see `kitty/entry_points.py:164`, `namespaced_entry_points['kitten'] = run_kitten`; `run_kitten` at lines 118–127 forwards to `kittens.runner.run_kitten`).
* `kitty +list-fonts` / `kitty +list-fonts --psnames` — the legacy entry
  point. `kitty/entry_points.py:15–17` binds `list-fonts` to
  `kitty.fonts.list:main`. That function (`kitty/fonts/list.py:34–42`)
  unconditionally drops the `--psnames` flag and `execlp`s the kitten
  binary with `choose-fonts`:

  ```python
  def main(argv: Sequence[str]) -> None:
      ...
      if '--psnames' in argv:
          argv.remove('--psnames')
      os.environ['KITTY_PATH_TO_KITTY_EXE'] = kitty_exe()
      os.execlp(kitten_exe(), 'kitten', 'choose-fonts')
  ```

  So `kitty +list-fonts` is effectively a hard alias for `kitten choose-fonts`.

### 2.3 Subcommand registration

The Go `kitten` binary is rooted at `tools/cmd/tool/main.go`. Its
`KittyToolEntryPoints()` function registers every kitten subcommand by
calling each kitten package's `EntryPoint(root)`. The `choose-fonts`
subcommand is registered at line 82:

```go
// tools/cmd/tool/main.go
// choose-fonts
choose_fonts.EntryPoint(root)            // line 82
```

`kittens/choose_fonts/main.go:74–99` defines `EntryPoint(root *cli.Command)`:

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
    ans.Add(cli.OptionSpec{
        Name:    "--reload-in",
        Dest:    "Reload_in",
        Type:    "choices",
        Choices: "parent, all, none",
        Default: "parent",
        Help:    `...`,
    })
    clone := root.AddClone(ans.Group, ans)
    clone.Hidden = false
    clone.Name = "choose_fonts"
}
```

Two things to note:

1. The primary subcommand is `choose-fonts` (hyphenated). A clone is added under the name `choose_fonts` (underscored). Both invocations work.
2. The subcommand declares exactly one option: `--reload-in`, with `Dest="Reload_in"` (the struct field it will populate), type `choices` over `parent`, `all`, `none`, and default `parent`.

### 2.4 Launcher fast-path: `choose-fonts` is NOT wrapped

`kitty/launcher/main.c:353–358` defines `delegate_to_kitten_if_possible`:

```c
static void
delegate_to_kitten_if_possible(int argc, char *argv[], char* exe_dir) {
    if (argc > 1 && argv[1][0] == '@') exec_kitten(argc, argv, exe_dir);
    if (argc > 2 && strcmp(argv[1], "+kitten") == 0 && is_wrapped_kitten(argv[2])) exec_kitten(argc - 1, argv + 1, exe_dir);
    if (argc > 3 && strcmp(argv[1], "+") == 0 && strcmp(argv[2], "kitten") == 0 && is_wrapped_kitten(argv[3])) exec_kitten(argc - 2, argv + 2, exe_dir);
}
```

`is_wrapped_kitten` (lines 332–337) checks the C preprocessor macro `WRAPPED_KITTENS`, which is defined at compile time by `setup.py:1233`:

```python
cppflags = [define(f'WRAPPED_KITTENS=" {wrapped_kittens()} "')]
```

`wrapped_kittens()` (`setup.py:1074–1081`) reads the list from the shell
wrapper `shell-integration/ssh/kitty`, which on line 27 contains:

```sh
wrapped_kittens="clipboard icat hyperlinked_grep ask hints unicode_input ssh themes diff show_key transfer query_terminal"
```

`choose-fonts` is **not** in that list. Rationale: wrapped kittens are the
ones that the C launcher can fast-path exec straight to the `kitten` Go
binary without ever starting Python. `choose-fonts` needs the Python font
engine (see §4 below) and is therefore not in the wrapped set — but the
Go `kitten` binary still registers and handles the subcommand directly.
The practical consequence is that when the user types `kitty +kitten choose-fonts`, the launcher does **not** fast-path it; the full Python startup path runs first, then it dispatches to the kitten via `run_kitten_with_metadata` (see §2.5).

### 2.5 Dispatch when invoked inside a running kitty

When invoked through a mapped action (e.g. from `boss.py`) instead of a
fresh launcher invocation, dispatch goes through
`kitty/boss.py:run_kitten_with_metadata()` (lines 1889–1975). Because
`choose-fonts` is not in `wrapped_kitten_names()` (line 1902,
`is_wrapped = False`), the dispatch falls into the non-wrapped branch at
lines 1952–1954:

```python
cmd = [kitty_exe(), '+runpy', 'from kittens.runner import main; main()']
env['PYTHONWARNINGS'] = 'ignore'
```

…and `[config_dir, kitten]` is prepended to `args` at line 1914. For a
user-typed `kitten choose-fonts` at the shell, however, the user is
executing the Go `kitten` binary directly, so this Python dispatch path
is not used — only the Go `EntryPoint` from §2.3.

---

## 3. Option registration and parsing

### 3.1 Declaring `--reload-in`

Already shown in §2.3: `kittens/choose_fonts/main.go:86–95`:

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

The Options struct is tiny (`kittens/choose_fonts/main.go:70–72`):

```go
type Options struct {
    Reload_in string
}
```

### 3.2 From flags to struct: `GetOptionValues()`

Inside the `Run` closure (`kittens/choose_fonts/main.go:78–84`):

```go
Run: func(cmd *cli.Command, args []string) (rc int, err error) {
    opts := Options{}
    if err = cmd.GetOptionValues(&opts); err != nil {
        return 1, err
    }
    return main(&opts)
},
```

`tools/cli/command.go:465–522` implements `GetOptionValues`, using Go
reflection to walk every exported field of the struct:

```go
func (self *Command) GetOptionValues(pointer_to_options_struct any) error {
    val := reflect.ValueOf(pointer_to_options_struct).Elem()
    ...
    for i := 0; i < val.NumField(); i++ {
        f := val.Field(i)
        field_name := val.Type().Field(i).Name
        if utils.Capitalize(field_name) != field_name || !f.CanSet() {
            continue
        }
        opt := self.option_map[field_name]
        if opt == nil {
            return fmt.Errorf("No option with the name: %s", field_name)
        }
        switch opt.OptionType {
        ...
        case StringOption:
            ...
            v := opt.parsed_value().(string)
            f.SetString(v)
        }
    }
    return nil
}
```

The runtime dispatch is: `cmd.ExecArgs(args)` (`tools/cli/command.go:524`)
→ `root.ParseArgs(args)` (line 529) → `self.parse_args(&ctx, args[1:])`
(line 201, inside `ParseArgs`) → `cmd.Run(cmd, cmd.Args)` (line 546). The
`Run` callback then calls `GetOptionValues`, which populates
`opts.Reload_in` from the `"Reload_in"` entry of
`self.option_map` (registered through the `Dest` field of the `OptionSpec`).

The resulting value of `opts.Reload_in` is one of the three fixed
strings: `"parent"` (the default), `"all"`, or `"none"`.

---

## 4. End-to-end flow of option values

### 4.1 The options pointer

`kittens/choose_fonts/main.go:16` defines:

```go
func main(opts *Options) (rc int, err error) {
```

Line 35 stores the options in the TUI handler:

```go
h := &handler{lp: lp, opts: opts}
```

`kittens/choose_fonts/ui.go:42–62` declares the `handler` struct, which
carries the options alongside every UI-level piece of state:

```go
type handler struct {
    opts                 *Options
    lp                   *loop.Loop
    state                State
    ...
    listing    FontList
    faces      faces
    face_pane  face_panel
    final_pane final_pane
    ...
}
```

The handler holds the four panes (`listing`, `faces`, `face_pane`,
`final_pane`). Every pane has a `handler` back-pointer and therefore can
read `h.opts.Reload_in` whenever it needs to.

### 4.2 Pane state machine

`kittens/choose_fonts/ui.go:77–102` initializes the handler, creates the
temp dir under `utils.CacheDir()`, starts the Python font backend
asynchronously (line 93–99), and sets `OnKeyEvent` / `OnText` / `OnMouseEvent`
/ `OnWakeup` callbacks on the terminal loop (see `main.go:36–53`). These
callbacks are dispatched via `h.current_pane.on_*`.

The pane-switching logic, keyed off Enter / Esc events, is:

| From | Key | Transition | Source reference |
|---|---|---|---|
| `FontList` (`listing`) | Enter on a highlighted family | → `faces.on_enter(family)` | `list.go:247–254` |
| `faces` | Enter | → `final_pane.on_enter(family, settings)` | `faces.go:118–121` |
| `faces` | Esc | → `listing` | `faces.go:113–117` |
| `faces` | `R` / `B` / `I` / `O` | → `face_pane.on_enter(...)` | `faces.go:125–143` |
| `final_pane` | Esc | → `faces` | `final.go:73–77` |
| `final_pane` | Enter | Patch `kitty.conf` → Quit | `final.go:78–97` |
| `final_pane` | `s` / `S` | Write serialized block to stdout → Quit | `final.go:101–111` |

`faces.on_enter` (`faces.go:145–159`) fills in `self.settings` — the four
font-spec strings (`font_family`, `bold_font`, `italic_font`,
`bold_italic_font`), initialized from the resolved faces originally
loaded from `kitty.conf` if the family matches, otherwise sensible
defaults (`family="..."` / `auto`). The settings flow Enter-by-Enter
through `faces` → `final_pane.on_enter` (`final.go:113–118`), which
stores them on the final pane:

```go
func (self *final_pane) on_enter(family string, settings faces_settings) error {
    self.settings = settings
    self.family = family
    self.handler.current_pane = self
    return self.handler.draw_screen()
}
```

### 4.3 Where `--reload-in` finally matters

`--reload-in` is only read at one place — the terminal step of the state
machine, in `final_pane.on_key_event` when the user presses Enter
(`final.go:78–97`). See §5.

Design rationale: keeping the option untouched through the entire UI
means the UI code is oblivious to reload semantics; only the code that
actually performs the config write inspects it. The option is pure
metadata for the final action.

---

## 5. What happens when the user presses Enter at the confirmation screen

### 5.1 The `final_pane.on_key_event` logic

`kittens/choose_fonts/final.go:72–98`:

```go
func (self *final_pane) on_key_event(event *loop.KeyEvent) (err error) {
    if event.MatchesPressOrRepeat("esc") {
        event.Handled = true
        self.handler.current_pane = &self.handler.faces
        return self.handler.draw_screen()
    }
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
    return
}
```

The serialized font block is built by `faces_settings.serialized()`
(`final.go:63–70`):

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

### 5.2 `utils.ConfigDir()` — locating `kitty.conf`

`tools/utils/paths.go:88–134` implements `ConfigDirForName` and the cached
`ConfigDir`. Resolution order:

1. `KITTY_CONFIG_DIRECTORY` (line 89–91).
2. `XDG_CONFIG_HOME`, then every entry of `XDG_CONFIG_DIRS`, then `~/.config`, then (on macOS) `~/Library/Preferences` (lines 101–112). For each candidate `loc`, if `loc/kitty/<name>` exists *and* the directory is writable, that directory is returned (lines 113–123).
3. Fallback: `$XDG_CONFIG_HOME/kitty` (or `~/.config/kitty`) (lines 124–129).

`ConfigDir()` (line 132–134) is a `sync.OnceValue` of
`ConfigDirForName("kitty.conf")`.

### 5.3 `Patcher.Patch()` — the in-place, sentinel-framed config rewrite

`tools/config/api.go:305–350`:

```go
type Patcher struct {
    Write_backup bool
    Mode         fs.FileMode
}

func (self Patcher) Patch(path, sentinel, content string, settings_to_comment_out ...string) (updated bool, err error) {
    if self.Mode == 0 {
        self.Mode = 0o644
    }
    backup_path := path
    if q, err := filepath.EvalSymlinks(path); err == nil {
        path = q
    }
    raw, err := os.ReadFile(path)
    if err != nil && !errors.Is(err, fs.ErrNotExist) {
        return false, err
    }
    if raw == nil {
        raw = []byte{}
    }
    pat := utils.MustCompile(fmt.Sprintf(`(?m)^\s*(%s)\b`, strings.Join(settings_to_comment_out, "|")))
    text := pat.ReplaceAllString(utils.UnsafeBytesToString(raw), `# $1`)

    pat = utils.MustCompile(fmt.Sprintf(`(?ms)^# BEGIN_%s.+?# END_%s`, sentinel, sentinel))
    replaced := false
    addition := fmt.Sprintf("# BEGIN_%s\n%s\n# END_%s", sentinel, content, sentinel)
    ntext := pat.ReplaceAllStringFunc(text, func(string) string {
        replaced = true
        return addition
    })
    if !replaced {
        if text != "" {
            text += "\n\n"
        }
        ntext = text + addition
    }
    nraw := utils.UnsafeStringToBytes(ntext)
    if !bytes.Equal(raw, nraw) {
        if len(raw) > 0 && self.Write_backup {
            _ = os.WriteFile(backup_path+".bak", raw, self.Mode)
        }

        return true, utils.AtomicUpdateFile(path, nraw, self.Mode)
    }
    return false, nil
}
```

Step-by-step, for the concrete call from `final.go:82`:

```go
patcher.Patch(path, "KITTY_FONTS",
    /* content: */  self.settings.serialized(),
    /* settings_to_comment_out: */ "font_family", "bold_font", "italic_font", "bold_italic_font")
```

1. Read the existing file (or empty bytes if it doesn't exist yet).
2. Comment-out old settings: any line starting with `font_family`, `bold_font`, `italic_font`, or `bold_italic_font` (ignoring leading whitespace) becomes `# <same line>`. Rationale: preserving the old values as comments keeps the user's history visible and avoids silent data loss.
3. Replace (or append) the sentinel block `# BEGIN_KITTY_FONTS ... # END_KITTY_FONTS` containing the four new directives. Rationale: the sentinel block is idempotent — a second kitten run finds and replaces the block instead of appending forever.
4. If the new text differs from the old text:
   * If `Write_backup` is true and the file was non-empty, write the original bytes to `kitty.conf.bak` (line 344). This uses the **original (pre-symlink-eval) path** as `backup_path`, so even if `kitty.conf` is a symlink, the `.bak` lands next to the symlink — a good least-surprise choice.
   * Call `utils.AtomicUpdateFile(path, nraw, self.Mode)` (line 347). `tools/utils/atomic-write.go:79–89` wraps `AtomicWriteFile`, which writes to a temp file and `rename`s into place so the config is never observed in a half-written state. Rationale: crash-safety — the user's config is either the old one or the new one, never a partial one.

`updated` is `true` iff the file actually changed. Rationale: no-op writes
skip both the backup write and the SIGUSR1 dispatch.

### 5.4 `ReloadConfigInKitty()` — firing SIGUSR1

`tools/config/api.go:352–371`:

```go
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

* Parent-only: reads `$KITTY_PID` (set by the parent kitty for every child it launches), verifies the command-line really is a kitty GUI process with `is_kitty_gui_cmdline` (`tools/config/api.go:282–303`), then sends `SIGUSR1` to just that PID.
* All: enumerates every process on the system via `github.com/shirou/gopsutil/v3/process.Processes()` (`tools/config/api.go:363`), filters by the same GUI-cmdline check, and sends `SIGUSR1` to each match. Rationale: users running multiple kitty instances usually want all of them to adopt the new fonts; this mode is opt-in via `--reload-in all` because broadcasting a signal to every kitty is heavier.
* `--reload-in none` is handled by the switch in `final.go:87–93` simply not matching either case, so **no signal is sent** — the on-disk `kitty.conf` still changes; only the live reload is skipped.

### 5.5 `SIGUSR1` in the kitty main loop

`kitty/child-monitor.c:1361–1383`:

```c
static bool
handle_signal(const siginfo_t *siginfo, void *data) {
    SignalSet *ss = data;
    switch(siginfo->si_signo) {
        case SIGINT:
        case SIGTERM:
        case SIGHUP:
            ss->kill_signal = true;
            break;
        case SIGCHLD:
            ss->child_died = true;
            break;
        case SIGUSR1:
            ss->reload_config = true;
            break;
        ...
    }
    return true;
}
```

Signals are delivered through a signalfd that the I/O loop polls
(`kitty/child-monitor.c:1510–1527`). When `SIGUSR1` arrives, the local
`SignalSet` gets `reload_config = true`, which the polling thread then
promotes to the shared flag `reload_config_signal_received = true` under
the children mutex (lines 1520–1525).

On the next parse-and-dispatch iteration,
`kitty/child-monitor.c:453–536` drains the flag:

```c
bool input_read = false, reload_config_called = false;
...
if (UNLIKELY(kill_signal_received || reload_config_signal_received)) {
    if (kill_signal_received) {
        ...
    }
    else if (reload_config_signal_received) {
        reload_config_signal_received = false;
        reload_config_called = true;
    }
}
...
if (reload_config_called) {
    call_boss(load_config_file, "");
}
```

So the final step of the signal handler is to call
`Boss.load_config_file("")` via `call_boss`.

### 5.6 `Boss.load_config_file` — re-reading the file

`kitty/boss.py:2691–2708`:

```python
def load_config_file(self, *paths: str, apply_overrides: bool = True, overrides: Sequence[str] = ()) -> None:
    from .cli import default_config_paths
    from .config import load_config
    old_opts = get_options()
    prev_paths = old_opts.all_config_paths or default_config_paths(self.args.config)
    paths = paths or prev_paths
    bad_lines: List[BadLine] = []
    final_overrides = old_opts.config_overrides if apply_overrides else ()
    if overrides:
        final_overrides += tuple(overrides)
    opts = load_config(*paths, overrides=final_overrides or None, accumulate_bad_lines=bad_lines)
    if bad_lines:
        self.show_bad_config_lines(bad_lines)
    self.apply_new_options(opts)
    ...
```

`apply_new_options` (`kitty/boss.py:2646–2680`) then:

1. Installs the new `Options` object globally (line 2649).
2. Calls `kitty.fonts.render.set_font_family(opts)` (line 2656) to rebuild the font caches using the new `font_family` / `bold_font` / `italic_font` / `bold_italic_font` values — this is where the new fonts become effective *live*.
3. Resizes each tab manager (lines 2657–2660) so windows redraw with the new metrics.
4. Refreshes colors and re-uploads GPU data (lines 2677–2679).

Net effect: the moment Enter is pressed, the on-disk config is updated
*and* the running kitty is told to re-read that file. Both bases are
covered in one step.

---

## 6. Persistence across restarts

### 6.1 TL;DR

**Yes, the font choice persists across restarts** when Enter is pressed
at the confirmation screen. This is a direct, architectural consequence
of the `Patcher.Patch()` → `AtomicUpdateFile()` chain in `tools/config/api.go`,
which rewrites the user's `kitty.conf` on disk.

### 6.2 The on-disk proof

On the next kitty start, the normal kitty config-loading pipeline reads
`kitty.conf`. The sentinel block written by `Patcher.Patch()` is just a
pair of comment lines surrounding four *plain* config directives:

```
# BEGIN_KITTY_FONTS
font_family      <chosen>
bold_font        <chosen or auto>
italic_font      <chosen or auto>
bold_italic_font <chosen or auto>
# END_KITTY_FONTS
```

The `#` lines are ignored by the parser (see `tools/config/api.go:123–131`:
`if line[0] == '#' { ...CommentsHandler... continue }` — comments never
reach `LineHandler`). The four font lines are parsed exactly the same way
as if the user had typed them by hand.

The defaults that the kitten diverges from are in `kitty/options/types.py`:

```python
bold_font:        FontSpec = FontSpec(..., system='auto',      axes=())   # line 491
bold_italic_font: FontSpec = FontSpec(..., system='auto',      axes=())   # line 492
font_family:      FontSpec = FontSpec(..., system='monospace', axes=())   # line 523
italic_font:      FontSpec = FontSpec(..., system='auto',      axes=())   # line 537
```

Schema reference: `kitty/options/definition.py:35–57`:

```python
opt('font_family', 'monospace', option_type='parse_font_spec', ...)
opt('bold_font',        'auto', option_type='parse_font_spec')
opt('italic_font',      'auto', option_type='parse_font_spec')
opt('bold_italic_font', 'auto', option_type='parse_font_spec')
```

When the rewritten `kitty.conf` is parsed on the next startup, the four
`font_*` directives go through `parse_font_spec` and overwrite the
defaults, so the chosen family sticks.

### 6.3 Old lines won't come back

`Patcher.Patch()` comments out any pre-existing `font_family` / `bold_font`
/ `italic_font` / `bold_italic_font` lines outside the sentinel block
(`tools/config/api.go:325–326`). Rationale: if the user had set
`font_family Fira Code` before, the sentinel block's `font_family
JetBrains Mono` would otherwise be **silently overridden** by the later
plain line — kitty's parser is last-wins for duplicate keys. Commenting
out the old lines converts them to no-ops, so the sentinel block is
authoritative.

### 6.4 The `s` key: the intentionally non-persistent path

`final.go:101–111`:

```go
func (self *final_pane) on_text(text string, from_key_event bool, in_bracketed_paste bool) (err error) {
    if from_key_event {
        switch text {
        case "s", "S":
            output_on_exit = self.settings.serialized() + "\n"
            self.lp.Quit(0)
            return
        }
    }
    return
}
```

and `main.go:64–66`:

```go
if output_on_exit != "" {
    os.Stdout.WriteString(output_on_exit)
}
```

Pressing `s` bypasses `Patcher.Patch()` entirely — nothing is written to
`kitty.conf`, nothing is signaled, and the serialized block is merely
printed to the kitten's stdout for the user to copy. This path does
**not** persist. It is useful e.g. when the user wants to paste the
block into a separate config repository.

---

## 7. Runtime evidence — a simulated `Patcher.Patch()` run

To verify the on-disk transformation without modifying the real repository
or the user's actual `kitty.conf`, the same regex logic was re-implemented
in Python and run against a throw-away temp file. The Python code mirrors
`tools/config/api.go:310–350` line-for-line.

Input `kitty.conf`:

```
# Kitty config
font_family      Fira Code
bold_font        Fira Code Bold
italic_font      Fira Code Italic
bold_italic_font Fira Code Bold Italic

font_size 12.0
cursor_shape beam
```

Call:

```
Patcher.Patch(
    path=<tmp>/kitty.conf,
    sentinel="KITTY_FONTS",
    content="font_family      JetBrains Mono\nbold_font        auto\nitalic_font      auto\nbold_italic_font auto",
    settings_to_comment_out=["font_family", "bold_font", "italic_font", "bold_italic_font"],
)
```

Resulting `kitty.conf`:

```
# Kitty config
# font_family      Fira Code
# bold_font        Fira Code Bold
# italic_font      Fira Code Italic
# bold_italic_font Fira Code Bold Italic

font_size 12.0
cursor_shape beam


# BEGIN_KITTY_FONTS
font_family      JetBrains Mono
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
```

Resulting `kitty.conf.bak` (verbatim copy of the original):

```
# Kitty config
font_family      Fira Code
bold_font        Fira Code Bold
italic_font      Fira Code Italic
bold_italic_font Fira Code Bold Italic

font_size 12.0
cursor_shape beam
```

Observations:

1. Old font lines are commented out (prefixed with `# `), not deleted.
2. A new `# BEGIN_KITTY_FONTS` / `# END_KITTY_FONTS` block is appended. A subsequent run would *replace* this block in place (via the `(?ms)^# BEGIN_KITTY_FONTS.+?# END_KITTY_FONTS` regex at `tools/config/api.go:328`).
3. A `.bak` is created alongside the original. The file is left intact for disaster recovery.
4. On the next kitty start, `kitty/config.py`'s standard parser ingests the four directives inside the sentinel block as ordinary `font_family` / `bold_font` / `italic_font` / `bold_italic_font` settings, overriding the `monospace` / `auto` defaults from `kitty/options/types.py:491–537`.

The temporary simulation scripts used for this verification were deleted after running; no files outside this document were added to the repository.

---

## Summary

| Question | Answer |
|---|---|
| How do I build kitty? | Install system dependencies (gcc, pkg-config, fontconfig/harfbuzz/freetype/GL/X11/Wayland/dbus dev libs, librsync), Go ≥ 1.22, Python ≥ 3.8; run `python3 setup.py build --ignore-compiler-warnings`. |
| How do I launch a single kitty instance? | `./kitty/launcher/kitty` (under `Xvfb :99` for headless). |
| How do I invoke `choose-fonts` from inside kitty? | `kitten choose-fonts` (or `kitty +kitten choose-fonts`, or the legacy `kitty +list-fonts`). |
| Where is the subcommand registered? | `tools/cmd/tool/main.go:82` → `kittens/choose_fonts/main.go:74–99` (`EntryPoint`). |
| How is `--reload-in` parsed? | Declared in `main.go:86–95`, populated into `Options.Reload_in` by `tools/cli/command.go:GetOptionValues` (line 465) via Go reflection. |
| How do values reach the final step? | `Options` → `handler.opts` (`ui.go:42`) → read in `final_pane.on_key_event` (`final.go:87–93`). |
| What does Enter do on the confirmation pane? | Calls `Patcher.Patch("kitty.conf", "KITTY_FONTS", ...)` (`final.go:80–85`, `tools/config/api.go:310–350`) which rewrites the file atomically with a `.bak` backup; then `ReloadConfigInKitty(...)` sends `SIGUSR1` (`tools/config/api.go:352–371`) to parent / all kitty processes, which kitty handles in `kitty/child-monitor.c:1373` / `534` by calling `Boss.load_config_file` in `kitty/boss.py:2691`. |
| Does the choice persist across restarts? | **Yes**, because `Patcher.Patch` writes a sentinel-framed block to `kitty.conf` on disk; kitty's parser reads those lines on every startup (`tools/config/api.go:123–150`). The `s` key alternative (`final.go:101–111`) writes to stdout only and does **not** persist. |
