# `choose-fonts` Kitten — End-to-End Behavior & Persistence (Runtime-Verified Onboarding Q&A)

> **Repository:** `kovidgoyal/kitty`
> **Branch / commit anchored:** `kitty_815df1e210e0` @ `815df1e21`
> **Method:** Every behavioral claim below is backed by **unedited output** captured from the **real** `kitten choose-fonts` entry point, run inside the project's canonical build. Every factual claim carries a `file:line` citation naming the concrete function/struct. Nothing here is derived from a remote-control shortcut, a debug hook, importing internals, or calling `Patcher.Patch` directly.

---

## TL;DR — Is a font selection remembered across restarts?

**Yes.** When you press **Enter** at the final confirmation screen, the kitten **patches the persistent `kitty.conf` file on disk** — it inserts a sentinel-delimited managed font block (and writes a `.bak` backup **when an existing, non-empty `kitty.conf` is replaced**) — so the choice is **durable and survives restarts**. It is *not* a session-only change. The decisive code is the Enter branch of `final_pane.on_key_event`, which builds a `config.Patcher{Write_backup: true}`, resolves `path := filepath.Join(utils.ConfigDir(), "kitty.conf")`, and calls `patcher.Patch(...)` [kittens/choose_fonts/final.go:L78-L97], delegating to `Patcher.Patch` [tools/config/api.go:L310-L350].

The kitten's single option, `--reload-in`, is a **separate concern**: it only controls whether *already-running* kitty instances are signalled (`SIGUSR1`) to live-reload their config. It does **not** gate the on-disk write — even `--reload-in none` still persists the choice to `kitty.conf`.

Direct proof (fresh processes, same temporary config dir; full transcript in **Q4**):

```
# A brand-new kitty reading the PERSISTED config resolves the chosen family:
$ KITTY_CONFIG_DIRECTORY="$CFG" xvfb-run -a kitty/launcher/kitty --debug-font-fallback bash -c true
[0.173]   Normal: FiraCodeRoman-Regular: /root/.local/share/fonts/FiraCode-VF.ttf:131072

# The SAME binary with default settings (--config NONE) does NOT get Fira Code:
$ xvfb-run -a kitty/launcher/kitty --config NONE --debug-font-fallback bash -c true
[0.161]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
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

All temporary artifacts (the harness, temporary `KITTY_CONFIG_DIRECTORY` dirs, generated `kitty.conf`/`.bak`) live outside the repository tree and are removed on completion; the working tree is left byte-for-byte unchanged apart from this document. This is demonstrated with commands and output in the **Cleanup & repository integrity** section at the end of this document.

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

A from-scratch build in the container compiles the C extension and links the Go tools, then prints the launcher path. Complete unedited output of the command (122 compile steps + 5 link steps, ending in the success banner):

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
Build successful. Run kitty as: kitty/launcher/kitty
```

> Note: `--ignore-compiler-warnings` is required in this environment's newer C toolchain; the build otherwise treats warnings as errors (the flag suppresses the warning-as-error promotion — no warning or error lines appear in the output above). The produced binaries `kitty/launcher/kitty` and `kitty/launcher/kitten` are git-ignored (they never appear in `git status`). An incremental re-run of the same command on an already-built tree emits only `Build successful. Run kitty as: kitty/launcher/kitty`.

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

The kitten spawns its Python enumeration backend by executing the `kitty` binary, which it locates via `KittyExe` (a `sync.OnceValue`) [tools/utils/paths.go:L69-L86]: it first tries the exe of the process identified by `$KITTY_PID` (the parent kitty instance the kitten runs inside), accepted only if that path is absolute and its basename is `kitty` [tools/utils/paths.go:L70-L73]; then the sibling `kitty` next to the running executable (`kitty/launcher/kitty`) [tools/utils/paths.go:L79-L81]; and finally the `$KITTY_PATH_TO_KITTY_EXE` fallback [tools/utils/paths.go:L85]. When driving the kitten outside a real kitty window, exporting `KITTY_PATH_TO_KITTY_EXE=<abs>/kitty/launcher/kitty` guarantees resolution.

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

…while both valid variants parse cleanly. Each prints the complete, identical help block (the `--reload-in` value is accepted and does not change the help text), shown unedited below:

```
$ kitty/launcher/kitten choose-fonts --reload-in all --help
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

```
$ kitty/launcher/kitten choose-fonts --reload-in none --help
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
```

The underscore alias resolves to the same kitten (identical help body, differing only in the program name echoed in the `Usage:` and trailing banner lines). Both names also appear in the **complete** top-level `kitten --help` listing (reproduced unedited and in full below — `choose-fonts` and `choose_fonts` are the two adjacent entries near the end of the `Commands:` block):

```
$ kitty/launcher/kitten --help
Usage: kitten command [command options] [command args]

kitten serves as a launcher for running individual kittens. Each kitten can be
run as kitten command. The list of available kittens is given below.

Commands:
   @
    Control kitty remotely
   update-self
    Update this kitten binary
   edit-in-kitty
    Edit a file in a kitty overlay window
   clipboard
    Copy/paste with the system clipboard, even over SSH
   icat
    Display images in the terminal
   ssh
    Truly convenient SSH
   transfer
    Transfer files easily over the TTY device
   unicode-input
    Browse and select unicode characters by name
   show-key
    Show the codes generated by the terminal for key presses in various keyboard
    modes
   mouse-demo
    Demo the mouse handling kitty implements for terminal programs
   hyperlinked-grep
    Add hyperlinks to the output of ripgrep
   ask
    Ask the user for input
   hints
    Select text from screen with keyboard
   diff
    Pretty, side-by-side diffing of files and images
   themes
    Manage kitty color schemes easily
   run-shell
    Run the user's shell with shell integration enabled
   choose-fonts
    Choose the fonts used in kitty
   choose_fonts
    Choose the fonts used in kitty
   query-terminal
    Query the terminal for various capabilities

Get help for an individual command by running:
    kitten command -h

Options:
  --version
    The current kitten version.

  --help, -h
    Show help for this command

kitten 0.35.2 created by Kovid Goyal
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

The four `font_*` lines are byte-for-byte what `serialized()` emits, wrapped in the `# BEGIN_KITTY_FONTS` / `# END_KITTY_FONTS` sentinels from `Patch` [api.go:L330]. **No `.bak` exists here** because the pre-image was absent (empty) — exactly the `len(raw) > 0` guard [api.go:L343].

**The `.bak` case — a complete non-empty pre-image run (with `diff`).** To exercise the backup branch, the same temporary config dir is driven through Enter **twice**: Run 1 (empty pre-image) writes the DejaVu block and, as expected, creates **no** `.bak`; Run 2 (non-empty pre-image — the DejaVu block from Run 1) selects a different family (Fira Code), which replaces the block **in place** and this time **does** write `kitty.conf.bak`. Both runs are the real Enter finalize path; every command and its complete unedited output are shown:

```
$ echo "$KITTY_CONFIG_DIRECTORY"
/tmp/cf_cfg.yFCYok
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"
cat: /tmp/cf_cfg.yFCYok/kitty.conf: No such file or directory (os error 2)
$ python3 /tmp/cf_work/cf_harness.py --final-key enter -- kitty/launcher/kitten choose-fonts
FINAL: You have chosen the DejaVu Sans Mono family
final_key: enter
exit_code: 0
$ ls -la "$KITTY_CONFIG_DIRECTORY"
total 12
drwx--S---  2 root root 4096 Jul  8 05:43 .
drwxrwsrwx 12 root root 4096 Jul  8 05:43 ..
-rw-r--r--  1 root root  190 Jul  8 05:43 kitty.conf
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"
# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTS
$ grep -c "^# BEGIN_KITTY_FONTS" "$KITTY_CONFIG_DIRECTORY/kitty.conf"
1
```

After Run 1 the directory holds **only** `kitty.conf` (190 bytes) and **no** `.bak` — the `len(raw) > 0` guard [api.go:L343] suppresses the backup because the pre-image was empty. Now Run 2 finalizes Fira Code against that non-empty pre-image:

```
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"
# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTS
$ python3 /tmp/cf_work/cf_harness.py --search "Fira Code" --final-key enter -- kitty/launcher/kitten choose-fonts
FINAL: You have chosen the Fira Code family
final_key: enter
exit_code: 0
$ ls -la "$KITTY_CONFIG_DIRECTORY"
total 16
drwx--S---  2 root root 4096 Jul  8 05:43 .
drwxrwsrwx 12 root root 4096 Jul  8 05:43 ..
-rw-r--r--  1 root root  139 Jul  8 05:43 kitty.conf
-rw-r--r--  1 root root  190 Jul  8 05:43 kitty.conf.bak
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"
# BEGIN_KITTY_FONTS
font_family      family="Fira Code"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf.bak"
# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTS
$ diff "$KITTY_CONFIG_DIRECTORY/kitty.conf.bak" "$KITTY_CONFIG_DIRECTORY/kitty.conf"
2,5c2,5
< font_family      DejaVuSansMono
< bold_font        DejaVuSansMono-Bold
< italic_font      DejaVuSansMono-Oblique
< bold_italic_font DejaVuSansMono-BoldOblique
---
> font_family      family="Fira Code"
> bold_font        auto
> italic_font      auto
> bold_italic_font auto
$ grep -c "^# BEGIN_KITTY_FONTS" "$KITTY_CONFIG_DIRECTORY/kitty.conf"
1
```

This is the complete before/after/`diff`/backup evidence for finalization: the pre-image is the Run 1 DejaVu block; after Enter, `kitty.conf.bak` (190 bytes) holds that **exact prior content** while `kitty.conf` (139 bytes) holds the new Fira Code block; the `diff` shows precisely the four `font_*` lines changing (`.bak` = old on the `<` side, current on the `>` side); and `grep -c` is `1`, proving the block was **replaced in place, not duplicated** [tools/config/api.go:L328, L331-L334]. The `.bak` therefore appears exactly when the guard `if len(raw) > 0 && self.Write_backup` is satisfied [api.go:L343-L345] — from Run 2 onward. (Q4 below independently repeats the idempotency/backup behavior across further runs and demonstrates persistence across a real process restart.)


---

## Q4 — Is the font choice remembered across restarts? (the runtime example)

**Direct answer: YES.** Pressing Enter writes durable on-disk state to `kitty.conf` [kittens/choose_fonts/final.go:L78-L97 → tools/config/api.go:L310-L350]; it is not a session-only change. A fresh kitty process launched later against the same config directory loads the persisted fonts. Everything below is captured from real runs.

### Why a temporary config dir makes this hermetic and canonical

Setting `KITTY_CONFIG_DIRECTORY` to a throwaway directory redirects the write target away from the developer's real `~/.config/kitty`, because `ConfigDir()` [tools/utils/paths.go:L132-L134] resolves through `ConfigDirForName`, which honors that env var [tools/utils/paths.go:L88-L90]. The experiment therefore observes the *real* code path while remaining fully isolated.

### The before / after / restart experiment

The states below are one **continuous** experiment against a **single** temporary config dir (`$KITTY_CONFIG_DIRECTORY` = `/tmp/cf_cfg.RaNQBS` for this run), so the before → after → restart → idempotency progression all refers to the same file on disk.

**State 1 — before** (config dir empty; no managed block):

```
$ echo "$KITTY_CONFIG_DIRECTORY"
/tmp/cf_cfg.RaNQBS
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"
cat: /tmp/cf_cfg.RaNQBS/kitty.conf: No such file or directory (os error 2)
```

**State 2 — immediately after Enter** (drove `kitten choose-fonts`, searched `Fira Code`, Enter through faces, Enter at final). A distinctive family (Fira Code) is chosen deliberately so the restart contrast is unambiguous:

```
$ python3 /tmp/cf_work/cf_harness.py --search "Fira Code" --final-key enter -- kitty/launcher/kitten choose-fonts
FINAL: You have chosen the Fira Code family
final_key: enter
exit_code: 0
$ ls -la "$KITTY_CONFIG_DIRECTORY"
total 12
drwx--S---  2 root root 4096 Jul  8 06:03 .
drwxrwsrwx 12 root root 4096 Jul  8 06:03 ..
-rw-r--r--  1 root root  139 Jul  8 06:03 kitty.conf
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"
# BEGIN_KITTY_FONTS
font_family      family="Fira Code"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
```

The `family="Fira Code"` + three `auto` values are exactly the defaults built by `faces.on_enter` for a family not already configured [kittens/choose_fonts/faces.go:L152-L155] — confirming the real value-flow path produced this file.

**State 3 — after a fresh relaunch** (new processes, same `KITTY_CONFIG_DIRECTORY`). The GUI launch uses `xvfb-run -a` because the container is headless (a fresh kitty needs an X server and software GL); `xvfb-run` is only a display wrapper and does not affect config resolution. The `[…] Failed to open systemd user bus` line is an unrelated, harmless container-environment warning emitted before the font report. The environment variable `CFG` was set to the temporary config dir (`CFG=$KITTY_CONFIG_DIRECTORY`).

First, the persisted file survived the finalize process's death (a fresh `cat` after that process exited):

```
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"
# BEGIN_KITTY_FONTS
font_family      family="Fira Code"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
```

Next, a **brand-new** kitty process resolves its fonts from that persisted config (`--debug-font-fallback` makes startup print the font files it actually loaded) — the restart loads **Fira Code**:

```
$ KITTY_CONFIG_DIRECTORY="$CFG" xvfb-run -a kitty/launcher/kitty --debug-font-fallback bash -c true
[0.168] Failed to open systemd user bus with error: Connection refused
[0.173] Text fonts:
[0.173]   Normal: FiraCodeRoman-Regular: /root/.local/share/fonts/FiraCode-VF.ttf:131072
[0.173]   Bold: FiraCodeRoman-SemiBold: /root/.local/share/fonts/FiraCode-VF.ttf:262144
[0.173]   Italic: FiraCodeRoman-Regular: /root/.local/share/fonts/FiraCode-VF.ttf:131072
[0.173]   Bold-Italic: FiraCodeRoman-SemiBold: /root/.local/share/fonts/FiraCode-VF.ttf:262144
```

For contrast, the **same binary** launched with **default settings** (`--config NONE`, no persisted config) resolves **DejaVuSansMono**, not Fira Code:

```
$ xvfb-run -a kitty/launcher/kitty --config NONE --debug-font-fallback bash -c true
[0.158] Failed to open systemd user bus with error: Connection refused
[0.161] Text fonts:
[0.161]   Normal: DejaVuSansMono: /usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf:0
[0.161]   Bold: DejaVuSansMono-Bold: /root/.local/share/fonts/DejaVuSansMono-Bold.ttf:0
[0.161]   Italic: DejaVuSansMono-Oblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-Oblique.ttf:0
[0.161]   Bold-Italic: DejaVuSansMono-BoldOblique: /usr/share/fonts/truetype/dejavu/DejaVuSansMono-BoldOblique.ttf:0
```

The only thing that differs between the persisted launch and the `--config NONE` launch is the persisted `kitty.conf`. A restart that reads the file gets **FiraCode-VF.ttf**; a restart with default settings gets **DejaVuSansMono**. **The choice is remembered across restarts.**

> Why `--debug-font-fallback`: there is no `--debug-config` flag in this build; `--debug-font-fallback` makes a genuinely fresh kitty process print the font files it resolved at startup, which is the canonical (non-bypassing) way to observe that a new process loaded the persisted config.

### Idempotency — the managed block is replaced in place, never duplicated

Running finalize a second (and third) time against the **same** file (continuing the `/tmp/cf_cfg.RaNQBS` experiment above, whose pre-image is now the Fira Code block from State 2) shows the `(?ms)` in-place replacement [tools/config/api.go:L328, L331-L334]. Run 2 re-selects Fira Code — which is now **already configured** — so `faces.on_enter` resolves it to the concrete variable-axis/style form (`variable_name=FiraCodeRoman style=FiraCodeRoman-Light`) rather than the bare `auto` placeholders; this is why the block grows from 139 to 397 bytes while remaining a single region. Because the pre-image is now non-empty, this run also writes `kitty.conf.bak`:

```
$ python3 /tmp/cf_work/cf_harness.py --search "Fira Code" --final-key enter -- kitty/launcher/kitten choose-fonts
FINAL: You have chosen the Fira Code family
final_key: enter
exit_code: 0
$ ls -la "$KITTY_CONFIG_DIRECTORY"
total 16
drwx--S---  2 root root 4096 Jul  8 06:04 .
drwxrwsrwx 12 root root 4096 Jul  8 06:04 ..
-rw-r--r--  1 root root  397 Jul  8 06:04 kitty.conf
-rw-r--r--  1 root root  139 Jul  8 06:04 kitty.conf.bak
$ grep -c '^# BEGIN_KITTY_FONTS' "$KITTY_CONFIG_DIRECTORY/kitty.conf"
1
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf.bak"
# BEGIN_KITTY_FONTS
font_family      family="Fira Code"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
$ diff "$KITTY_CONFIG_DIRECTORY/kitty.conf.bak" "$KITTY_CONFIG_DIRECTORY/kitty.conf"
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

The `.bak` (139 bytes) holds the **exact** State-2 pre-image, `kitty.conf` (397 bytes) holds the resolved Fira Code block, `grep -c` is `1` (single region — replaced in place, not duplicated), and the `diff` shows precisely the four `font_*` lines changing from the `auto` form to the resolved form. A third run choosing a *different* family (DejaVu Sans Mono) still leaves exactly one region:

```
$ python3 /tmp/cf_work/cf_harness.py --search "DejaVu Sans Mono" --final-key enter -- kitty/launcher/kitten choose-fonts
FINAL: You have chosen the DejaVu Sans Mono family
final_key: enter
exit_code: 0
$ grep -c '^# BEGIN_KITTY_FONTS' "$KITTY_CONFIG_DIRECTORY/kitty.conf"
1
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"
# BEGIN_KITTY_FONTS
font_family      family="DejaVu Sans Mono"
bold_font        auto
italic_font      auto
bold_italic_font auto
# END_KITTY_FONTS
```

This demonstrates the `.bak` guard (`len(raw) > 0` [api.go:L343]) — the backup appears only from Run 2 onward, once a non-empty pre-image exists.

### `--reload-in` is orthogonal to persistence

`--reload-in` only decides whether *already-running* instances receive `SIGUSR1` to live-reload, via `ReloadConfigInKitty` [tools/config/api.go:L352-L371]: `parent` signals only the `$KITTY_PID` process (`SendSignal(unix.SIGUSR1)` [api.go:L357]); `all` signals every kitty GUI process (`SIGUSR1` [api.go:L366]). Crucially there is **no `none` case** in the finalize switch [kittens/choose_fonts/final.go:L87-L92], so `--reload-in none` **skips live reload but still writes to disk**.

**Observed vs. inferred (precise labeling).** What is **directly observed** at runtime below is that the on-disk `Patcher.Patch` write to `kitty.conf` happens for **`parent` (the default, shown throughout Q3c/Q4), `all`, and `none`** — i.e. persistence is independent of `--reload-in`. What is **`inferred` from source** (not directly observed here) is the actual `SIGUSR1` *delivery*: the harness drives the kitten with no live kitty GUI instance present to receive the signal, so `ReloadConfigInKitty`'s send path [tools/config/api.go:L352-L371] is read from code, not witnessed. The persistence claim (the subject of Q4) rests entirely on the observed on-disk `Patch`, which is shown for every variant.

**`--reload-in none` — Patch still occurs (live reload skipped):**

```
$ echo "$KITTY_CONFIG_DIRECTORY"
/tmp/cf_cfg.i7U2rw
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"
cat: /tmp/cf_cfg.i7U2rw/kitty.conf: No such file or directory (os error 2)
$ python3 /tmp/cf_work/cf_harness.py --final-key enter -- kitty/launcher/kitten choose-fonts --reload-in none
FINAL: You have chosen the DejaVu Sans Mono family
final_key: enter
exit_code: 0
$ ls -la "$KITTY_CONFIG_DIRECTORY"
total 12
drwx--S---  2 root root 4096 Jul  8 05:43 .
drwxrwsrwx 12 root root 4096 Jul  8 05:42 ..
-rw-r--r--  1 root root  190 Jul  8 05:43 kitty.conf
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"
# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTS
```

**`--reload-in all` — Patch still occurs (would signal every GUI instance):** the same real Enter finalize path, run with `--reload-in all`, writes the identical managed block to disk. Before/after and the exact command:

```
$ echo "$KITTY_CONFIG_DIRECTORY"
/tmp/cf_cfg.u92x9B
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"
cat: /tmp/cf_cfg.u92x9B/kitty.conf: No such file or directory (os error 2)
$ python3 /tmp/cf_work/cf_harness.py --final-key enter -- kitty/launcher/kitten choose-fonts --reload-in all
FINAL: You have chosen the DejaVu Sans Mono family
final_key: enter
exit_code: 0
$ ls -la "$KITTY_CONFIG_DIRECTORY"
total 12
drwx--S---  2 root root 4096 Jul  8 05:42 .
drwxrwsrwx 12 root root 4096 Jul  8 05:42 ..
-rw-r--r--  1 root root  190 Jul  8 05:42 kitty.conf
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"
# BEGIN_KITTY_FONTS
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
# END_KITTY_FONTS
```

Across `parent` (default), `none`, and `all`, the on-disk block is written identically (190 bytes, same four `font_*` lines) — **the `Patch` happened regardless of `--reload-in`**, confirming the option is orthogonal to persistence.

### The non-persisting final actions (contrast — each exercised, each shown)

**`s` / `S`** — `on_text` [kittens/choose_fonts/final.go:L101-L111] sets `output_on_exit = self.settings.serialized() + "\n"` [final.go:L105] and quits; `main()` prints it to STDOUT [kittens/choose_fonts/main.go:L64-L66]. **STDOUT only — nothing written to disk:**

The `s` key was pressed at the final pane; the four `font_*` lines below are what the kitten wrote to its own STDOUT (`output_on_exit`), and the config directory is left **empty** (no `kitty.conf`, no `.bak`):

```
$ export KITTY_CONFIG_DIRECTORY=/tmp/cf_cfg.ckz04G
$ python3 /tmp/cf_work/cf_harness.py --final-key s -- kitty/launcher/kitten choose-fonts
FINAL: You have chosen the DejaVu Sans Mono family
final_key: s
exit_code: 0
# --- serialized settings the kitten wrote to its STDOUT (output_on_exit), extracted from the raw PTY stream ---
font_family      DejaVuSansMono
bold_font        DejaVuSansMono-Bold
italic_font      DejaVuSansMono-Oblique
bold_italic_font DejaVuSansMono-BoldOblique
$ ls -la "$KITTY_CONFIG_DIRECTORY"
total 8
drwx--S---  2 root root 4096 Jul  8 06:01 .
drwxrwsrwx 12 root root 4096 Jul  8 06:01 ..
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"
cat: /tmp/cf_cfg.ckz04G/kitty.conf: No such file or directory (os error 2)
```

**Esc** — `on_key_event` [kittens/choose_fonts/final.go:L73-L77] sets `current_pane = &self.handler.faces` and redraws — it returns to the faces pane, **no write.** The harness pressed Esc at the final pane and then captured the resulting pane (the complete faces screen it returned to), followed by a directory listing showing nothing was written:

```
$ export KITTY_CONFIG_DIRECTORY=/tmp/cf_cfg.cctXCl
$ python3 /tmp/cf_work/cf_harness.py --final-key esc --capture-after-final faces -- kitty/launcher/kitten choose-fonts
FINAL: You have chosen the DejaVu Sans Mono family
final_key: esc
exit_code: None
---AFTER-KEY SNAPSHOT BEGIN---
                                                    DejaVu Sans Mono

Press Enter to select this font, Esc to go back to the font list or any of the highlighted keys below to fine-tune the
appearance of the individual font styles.

Regular: DejaVuSansMono


Bold: DejaVuSansMono-Bold


Italic: DejaVuSansMono-Oblique


Bold-Italic: DejaVuSansMono-BoldOblique
---AFTER-KEY SNAPSHOT END---
$ ls -la "$KITTY_CONFIG_DIRECTORY"
total 8
drwx--S---  2 root root 4096 Jul  8 06:00 .
drwxrwsrwx 12 root root 4096 Jul  8 06:00 ..
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"
cat: /tmp/cf_cfg.cctXCl/kitty.conf: No such file or directory (os error 2)
```

(`exit_code: None` because after Esc the kitten did not exit — it returned to the faces pane and was still running when the harness snapshotted it and tore down the PTY.)

**Ctrl+c** — there is **no explicit `ctrl+c` handler** in `final.go` (`on_key_event` matches only `esc`/`enter`; `on_text` only `s`/`S`). It therefore falls through to the event loop's default quit (**this default-quit routing is `inferred` from the absence of a handler**; the **no-write result is observed** — exit code 1, empty config dir):

```
$ export KITTY_CONFIG_DIRECTORY=/tmp/cf_cfg.hxpes2
$ python3 /tmp/cf_work/cf_harness.py --final-key ctrlc -- kitty/launcher/kitten choose-fonts
FINAL: You have chosen the DejaVu Sans Mono family
final_key: ctrlc
exit_code: 1
$ ls -la "$KITTY_CONFIG_DIRECTORY"
total 8
drwx--S---  2 root root 4096 Jul  8 06:00 .
drwxrwsrwx 12 root root 4096 Jul  8 06:00 ..
$ cat "$KITTY_CONFIG_DIRECTORY/kitty.conf"
cat: /tmp/cf_cfg.hxpes2/kitty.conf: No such file or directory (os error 2)
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
- **`--reload-in` (every value):** `OptionSpec` [main.go:L86-L95]; `parent` (default) → `ReloadConfigInKitty(true)` [final.go:L88-L89]; `all` → `ReloadConfigInKitty(false)` [final.go:L90-L91]; `none` → no case, still persists [final.go:L87-L92]. Evidence: `--help`, variant `--help`s (`all`/`none`), invalid-value rejection; **runtime finalize transcripts show the on-disk `Patch` occurs for `parent` (default, Q3c/Q4), `none`, and `all`** — the `SIGUSR1` *delivery* itself is **inferred** from source (no live GUI present to receive it). Reason: signal scope for live reload of running instances; persistence is orthogonal.
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
- **Managed block + `.bak`:** Evidence: Q3c non-empty pre-image run (before/after `kitty.conf`, `ls -la` showing `.bak`, `cat` of `.bak`, and before-vs-after `diff`); independently repeated by Q4 Runs 2 & 3.

### Q4 — Persistence across restarts
- **Persistence = YES.** Enter → disk write [final.go:L78-L97 → api.go:L310-L350]. Evidence: State 1/2/3, fresh-process Fira Code vs `--config NONE` DejaVu contrast.
- **Temp-dir mechanism:** `ConfigDir` [paths.go:L132-L134] → `ConfigDirForName` honors `KITTY_CONFIG_DIRECTORY` [paths.go:L88-L90]. Evidence: writes land in the temp dir.
- **before/after/restart:** captured (States 1/2/3).
- **Idempotency:** in-place replace, one region, `.bak` holds prior [api.go:L328, L331-L334]. Evidence: Runs 2 & 3, `grep -c` = 1.
- **`--reload-in` ↔ persistence separation:** `ReloadConfigInKitty` [api.go:L352-L371]; no `none` case [final.go:L87-L92]. Evidence: both `--reload-in none` and `--reload-in all` still patch the identical block to disk (190 bytes); the SIGUSR1 delivery is inferred from source.
- **`s`/Esc/Ctrl+c non-persistence:** `s` → STDOUT via `output_on_exit` [final.go:L101-L111, L105; main.go:L64-L66]; Esc → faces [final.go:L73-L77]; Ctrl+c → default-quit (**inferred** from absence of handler), no-write **observed**. Evidence: three transcripts, empty config dirs.

### Explicitly labeled inferences
- The **Ctrl+c** finalize routing to the loop's default quit is **inferred** from the absence of a `ctrl+c` branch in `final.go` (`on_key_event` handles only `esc`/`enter`; `on_text` only `s`/`S`). The **outcome** (process quits, nothing written to `kitty.conf`) is **observed** (exit 1, empty config dir).
- The `parent`/`all` `SIGUSR1` *delivery* is read from `ReloadConfigInKitty` [tools/config/api.go:L352-L371] (**inferred** from source); what is **observed** at runtime is that the on-disk `Patch` occurs for `parent`, `all`, and `none` alike.


---

## Cleanup & repository integrity (read-only mandate)

The investigation used only ephemeral artifacts **outside** the repository tree: the PTY harness and helper scripts under `/tmp/cf_work` (`cf_harness.py`, `extract_s.py`, `keytest.py`, a `caps/` directory of captured transcripts, a `backup/` copy of the already-gitignored launcher binaries, and `__pycache__`), plus throwaway `KITTY_CONFIG_DIRECTORY` directories under `/tmp/cf_cfg.*`. No file inside the repository was created, edited, or deleted except this single answer document. On completion, every temporary artifact is removed and the absence of lingering processes is verified (unedited):

```
$ # 1) Inventory the temporary investigation artifacts (all OUTSIDE the repo tree)
$ ls -d /tmp/cf_work /tmp/cf_cfg.* 2>/dev/null
/tmp/cf_work
$ ls -la /tmp/cf_work
total 40
drwxr-xr-x  5 root root  4096 Jul  8 05:45 .
drwxrwsrwx 11 root root  4096 Jul  8 06:12 ..
drwxr-xr-x  2 root root  4096 Jul  8 05:37 __pycache__
drwxr-xr-x  3 root root  4096 Jul  8 05:30 backup
drwxr-xr-x  2 root root  4096 Jul  8 06:04 caps
-rw-r--r--  1 root root 10038 Jul  8 05:39 cf_harness.py
-rw-r--r--  1 root root   646 Jul  8 05:45 extract_s.py
-rw-r--r--  1 root root  1547 Jul  8 05:38 keytest.py
$ # 2) Remove ALL temporary artifacts: PTY harness, helpers, captures, binary backup, temp config dirs
$ rm -rf /tmp/cf_work /tmp/cf_cfg.*
$ # 3) Verify nothing temporary remains (ls fails => the paths are gone)
$ ls -d /tmp/cf_work /tmp/cf_cfg.* 2>/dev/null; echo "exit=$?"
exit=2
$ # 4) No lingering kitty / kitten / Xvfb processes (grep finds nothing => exit 1)
$ ps -eo pid,comm | grep -iE "kitty|kitten|xvfb" | grep -v grep; echo "exit=$?"
exit=1
```

Repository integrity, relative to the pinned baseline commit `815df1e21`:

```
$ git status --porcelain
 M blitzy/documentation/kitty_815df1e210e0.md
$ git diff --name-status 815df1e21..HEAD
A	blitzy/documentation/kitty_815df1e210e0.md
$ git ls-files --others --exclude-standard
$
```

The `git status --porcelain` output contains **exactly one** entry — this document — proving no source file, manifest, `docs/` page, build/CI file, or temporary artifact is dirty; `git diff --name-status 815df1e21..HEAD` shows the sole change since baseline is the **addition** of this one document (`A`); and `git ls-files --others --exclude-standard` prints nothing, so there are no stray untracked (non-ignored) files. Committing this single document as the final step leaves the working tree clean (a subsequent `git status --porcelain` prints nothing). The already-gitignored build outputs (`kitty/launcher/kitty`, `kitty/launcher/kitten`, `build/`, `kitty/fast_data_types.so`) are left in place and do not appear in any of the above, so the source repository is byte-for-byte unchanged apart from this answer document.

