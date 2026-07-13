# How `kitten @ ls` Works — An End‑to‑End, Runtime‑Grounded Trace of kitty's Remote‑Control System

> **Scope & method.** This document answers, with **observed runtime evidence**, how kitty's remote‑control (RC) round trip works, using your exact example **`kitten @ ls`**. Every system claim carries a `file:line` reference and names the function/struct that does the work. Claims are tagged **OBSERVED** (reproduced at runtime, with the exact command that produced the output) or **INFERRED** (read from code but not directly instrumented). Nothing in the kitty source tree was modified to produce this document; all instrumentation was done with throwaway scripts created outside the repository and deleted afterward.
>
> **This document is explanatory only. It does not add a new RC command** — but §8 distills the exact pattern so you can add one later.

- **Checkout:** git HEAD `815df1e21` ("Wire up applying of font config"); build revision `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.
- **Build/version:** `kitty 0.35.2` (see §2).
- **Transports exercised:** (1) configured UNIX socket, and (2) shell‑integration / RC‑over‑TTY. Both are demonstrated with before/after state.

---

## 1. TL;DR — Direct answers to your four questions

**Q1 — How does `kitten @ ls` reach a running kitty, and how does it discover *where* to send? "Is it a socket? A pipe? Something else?"**
It is **a socket or terminal (DCS) escape sequences — never an anonymous pipe.** The Go `kitten` client decides at dispatch time: if it knows a target address it dials a UNIX/TCP **socket** (`do_socket_io` → `net.Dial`); if it doesn't, it writes a **DCS escape sequence to the controlling PTY** (`do_tty_io`). The single line that chooses is `tools/cmd/at/main.go:280`:
`response, err = get_response(utils.IfElse(global_options.to_network == "", do_tty_io, do_socket_io), io_data)`. The address itself defaults to the `KITTY_LISTEN_ON` environment variable when `--to` is omitted (`tools/cmd/at/main.go:371-372`). **OBSERVED** below in §3–§6.

**Q2 — Why does poking around `/tmp` for the socket fail / not match the docs?**
Three compounding reasons, all **OBSERVED** in §4:
1. **Templated path** — a `listen_on` value taken from the **config file** gets an automatic `-<PID>` suffix (`kitty/main.py:329-331`), so `unix:/tmp/mykitty` becomes `/tmp/mykitty-36692` on disk. A value passed on the **command line** (`--listen-on`) is used **verbatim** (no suffix). This asymmetry is the single most likely cause of your failed search.
2. **Abstract sockets** — `unix:@name` creates a Linux abstract‑namespace socket (`kitty/utils.py:513-514`) that has **no filesystem entry at all**; `ls /tmp` can never find it.
3. **Default‑off gating** — no socket is created unless **both** `allow_remote_control` (default `no`, `kitty/options/definition.py:2969`) **and** `listen_on` (default `none`, `:3000`) are set (`kitty/boss.py:364-366`).

**Q3 — Shell integration lets you send commands "without explicit configuration." Same socket, or "TTY magic through the terminal's own pty"?**
**Both — depending on whether a socket exists — and shell integration is *not* the enabler.** When you run `kitten @ ls` inside a kitty window, kitty has exported `KITTY_LISTEN_ON` (and `KITTY_PID`, `KITTY_PUBLIC_KEY`) into the child's environment (`kitty/child.py:244-249`). If that points at a socket, the client dials the **same socket**. If kitty is listening on **no** socket, the client falls back to writing the `@kitty-cmd` DCS escape **through the window's own PTY**, and kitty parses it out of the terminal stream (`kitty/vt-parser.c:603`) — that is the "TTY magic." The shell‑integration scripts only **consume** env vars; `grep -rn KITTY_LISTEN_ON shell-integration/` returns **nothing** (§6, **OBSERVED**). The producer is `kitty/child.py`, not the shell.

**Q4 — Show the real socket path, the wire bytes, where Python parses/routes to `ls`, the returned JSON, and the logging behavior.**
- **Real socket path:** `/tmp/mykitty` (CLI, verbatim) and `/tmp/cfgkitty-36692` (config, PID‑suffixed) — §4.
- **On the wire:** a symmetric DCS envelope `<ESC>P@kitty-cmd<JSON><ESC>\` (`<ESC>` = `0x1b`). Response first 14 bytes `1b 50 40 6b 69 74 74 79 2d 63 6d 64 7b 22`, last 2 bytes `1b 5c` — §5.2.
- **Parse/route (OBSERVED):** `parse_cmd` (`kitty/remote_control.py:56`) → `handle_cmd` (`:213`) → `command_for_name('ls')` (`kitty/rc/base.py:449`) → `LS.response_from_kitty` (`kitty/rc/ls.py:48`) → `boss.list_os_windows` (`:57`) → `json.dumps(..., indent=2, sort_keys=True)` (`:76`), wrapped as `{'ok': True, 'data': <json-string>}` (`kitty/remote_control.py:258-260`) — §5.3.
- **Returned JSON:** the full OS‑window/tab/window tree — §5.1.
- **Logging:** **error‑only.** A successful `ls` logs **nothing**; a malformed command logs `Failed to parse JSON payload of remote command, ignoring it` (`kitty/remote_control.py:62`) — §5.4.

---

## 2. Environment & canonical build

The live investigation ran inside the prepared Docker container (image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`, name `kitty-setup`), because kitty is a GPU/OpenGL GUI terminal that needs a display and (software) GL. The repository is bind‑mounted at its real path `/tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f`, which is also the repo root.

**Canonical build (R4).** The build command is `python3 setup.py`. On an already‑built tree it does nothing and prints nothing (exit 0). Run with `--verbose` it shows the compiler detection and the Go build of the `kitten` client:

```console
$ python3 setup.py --verbose
CC: ['gcc'] (13, 0)
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
Copyright (C) 2023 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Detected: CompilerType.gcc
Updating Go generated files...
/usr/local/go/bin/go build -v -ldflags '-X kitty.VCSRevision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 -s -w' -o kitty/launcher/kitten /tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f/tools/cmd
```

The build produces the launcher at **`kitty/launcher/kitty`** and the client at **`kitty/launcher/kitten`** (build logic `setup.py:1230` `build_launcher`; `launcher_dir = 'kitty/launcher'` `setup.py:2099`). **OBSERVED:**

```console
$ ls -la kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so
-rwxr-xr-x 1 root root    36224 Jul 13 16:08 kitty/launcher/kitty
-rwxr-xr-x 1 root root 15962372 Jul 13 16:35 kitty/launcher/kitten
-rwxr-xr-x 1 root root  1213072 Jul 13 16:09 kitty/fast_data_types.so
```

**Version banner (R4), verbatim:**

```console
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
$ ./kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
```

**Toolchain** (matches the project floors — Python `>=3.8` per `pyproject.toml:2`, Go `1.22` per `go.mod:3`):

```console
$ python3 --version ; go version ; gcc --version | head -1
Python 3.12.3
go version go1.23.4 linux/amd64
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
```

**Display used for the GUI runs:** a headless X server `Xvfb :99 -screen 0 1280x800x24 -ac +extension GLX +render -noreset`, with `DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe`. GL is provided by Mesa's software rasterizer, which exceeds kitty's OpenGL 3.3 requirement:

```console
$ glxinfo -B | grep -iE "OpenGL (version|renderer|core profile version)"
OpenGL renderer string: llvmpipe (LLVM 20.1.2, 256 bits)
OpenGL core profile version string: 4.5 (Core Profile) Mesa 25.2.8-0ubuntu0.24.04.2
OpenGL version string: 4.5 (Compatibility Profile) Mesa 25.2.8-0ubuntu0.24.04.2
```

> **Note on startup log noise.** In this container kitty always prints one benign, non‑RC line at startup: `[0.162] Failed to open systemd user bus with error: No medium found`. It is a D‑Bus artifact of the headless container and is unrelated to remote control. It is called out here so the "empty log on success" claim in §5.4 is precise.

---

## 3. OBJ‑1 — Transport discovery: socket vs. TTY (never a pipe)

**Direct answer.** `kitten @ ls` uses **one of exactly two transports**: a UNIX/TCP **socket**, or **DCS terminal escape sequences over the controlling PTY**. It is *never* an anonymous pipe. The client picks the branch based on whether it resolved a target network/address.

**The decision (Go client).** `tools/cmd/at/main.go:280`:

```go
response, err = get_response(utils.IfElse(global_options.to_network == "", do_tty_io, do_socket_io), io_data)
```

- If `to_network == ""` → `do_tty_io` (write DCS to the PTY).
- Otherwise → `do_socket_io` (dial the socket).

**How the address is discovered.** When `--to` is not given, the client defaults the target to the `KITTY_LISTEN_ON` environment variable, then parses it into a network+address (`tools/cmd/at/main.go:371-372`):

```go
if rc_global_opts.To == "" {
    rc_global_opts.To = os.Getenv("KITTY_LISTEN_ON")
    global_options.to_address_is_from_env_var = true
}
```

`utils.ParseSocketAddress(rc_global_opts.To)` then sets `to_network` (e.g. `unix`/`tcp`), which drives the `IfElse` above.

**The framing constants (Go).** `tools/cmd/at/socket_io.go:82-83`:

```go
const cmd_escape_code_prefix = "\x1bP@kitty-cmd"
const cmd_escape_code_suffix = "\x1b\\"
```

The socket dial is `net.Dial(global_options.to_network, global_options.to_address)` (`tools/cmd/at/socket_io.go:177`); the PTY writer queues the same prefix/suffix around the payload (`tools/cmd/at/tty_io.go:79-82`).

**The Python mirror.** kitty's built‑in Python client makes the identical choice in one line — `kitty/remote_control.py:383`:

```python
io: Union[SocketIO, RCIO] = SocketIO(to) if to else RCIO()
```

`SocketIO` is the socket transport; `RCIO` is the terminal/DCS transport.

**Which branch each run took (OBSERVED, see later sections):**

| Invocation | Condition | Branch taken | Evidence |
|---|---|---|---|
| `kitten @ --to unix:/tmp/mykitty ls` | `--to` present → `to_network="unix"` | `do_socket_io` → `net.Dial` | §4/§5 (socket dialed, wire bytes captured) |
| `kitten @ ls` in a window **with** a socket | `KITTY_LISTEN_ON=unix:/tmp/winkitty` | `do_socket_io` (same socket) | §6 Case 4 |
| `kitten @ ls` in a window **without** a socket | `KITTY_LISTEN_ON` empty → `to_network=""` | `do_tty_io` (DCS over PTY) | §6 Case 5 |

---

## 4. OBJ‑2 — The socket‑path mystery: why `/tmp` came up empty

**Direct answer.** Your `/tmp` search failed for three compounding reasons, each demonstrated live below: (a) config‑file paths get a **`-<PID>` suffix**; (b) `unix:@…` **abstract** sockets have **no file** at all; (c) with defaults, **no socket exists** because remote control is off. The **real** paths observed here are `/tmp/mykitty` (CLI, verbatim) and `/tmp/cfgkitty-36692` (config, suffixed).

### 4.1 Reason (a) — CLI value is verbatim; config‑file value gets `-<PID>`

The asymmetry lives in `expand_listen_on` (`kitty/main.py:325-343`), specifically `:329-331`:

```python
if '{kitty_pid}' not in listen_on and from_config_file and listen_on.startswith('unix:'):
    listen_on += '-{kitty_pid}'
listen_on = listen_on.replace('{kitty_pid}', str(os.getpid()))
```

The `from_config_file` flag is decided in `setup_environment` (`kitty/main.py:403-409`): if `--listen-on` was passed on the command line, `from_config_file` stays `False` (used **verbatim**); if the value came from the config file's `listen_on`, `from_config_file` becomes `True` (gets the `-<PID>` suffix). Relative UNIX paths are additionally resolved under `tempfile.gettempdir()` (`kitty/main.py:339`).

**CLI value → verbatim (OBSERVED).** Before/after with `--listen-on unix:/tmp/mykitty`:

```console
$ ls -l /tmp/mykitty*
ls: cannot access '/tmp/mykitty*': No such file or directory

$ ./kitty/launcher/kitty -o allow_remote_control=yes --listen-on unix:/tmp/mykitty bash -c "sleep 600" &
$ ls -l /tmp/mykitty*
srwxr-xr-x 1 root root 0 Jul 13 16:38 /tmp/mykitty

$ stat -c '%n type=%F mode=%A' /tmp/mykitty
/tmp/mykitty type=socket mode=srwxr-xr-x
```

The live socket, confirmed with both `lsof` and `ss` (owning process is kitty pid **36331**, fd 6):

```console
$ lsof -U | grep mykitty
kitty   36331 root    6u  unix 0x0000000000000000      0t0 726289071 /tmp/mykitty type=STREAM (LISTEN)

$ ss -xlp | grep mykitty
u_str LISTEN 0      128           /tmp/mykitty 726289071            * 0    users:(("kitty",pid=36331,fd=6))
```

So the CLI value `/tmp/mykitty` is on disk **verbatim** — no PID suffix.

**Config‑file value → `-<PID>` suffix (OBSERVED).** Using a throwaway config file (outside the repo) containing `allow_remote_control yes` and `listen_on unix:/tmp/cfgkitty`, launched with `--config` (no `--listen-on` on the command line):

```console
$ cat /tmp/kitty_obs/test.conf
allow_remote_control yes
listen_on unix:/tmp/cfgkitty

$ ls -l /tmp/cfgkitty*
ls: cannot access '/tmp/cfgkitty*': No such file or directory

$ ./kitty/launcher/kitty --config /tmp/kitty_obs/test.conf bash -c "sleep 300" &
$ ls -l /tmp/cfgkitty*
srwxr-xr-x 1 root root 0 Jul 13 16:42 /tmp/cfgkitty-36692

$ ss -xlp | grep cfgkitty
u_str LISTEN 0      128    /tmp/cfgkitty-36692 726564890            * 0    users:(("kitty",pid=36692,fd=6))
```

The configured name `unix:/tmp/cfgkitty` became **`/tmp/cfgkitty-36692`** on disk, and the suffix (`36692`) equals the owning kitty PID (`36692`) — exactly what `main.py:329-331` predicts. It still works via the suffixed path:

```console
$ ./kitty/launcher/kitten @ --to unix:/tmp/cfgkitty-36692 ls | jq -c '[.[].tabs[].windows[]|{id,cmdline}]'
[{"id":1,"cmdline":["sleep","300"]}]
```

> **This is almost certainly what happened to you:** you put `listen_on unix:/tmp/something` in `kitty.conf`, kitty created `/tmp/something-<PID>`, and a search for the literal name found nothing. (The kitty docs note this hyphen‑PID append at `kitty/options/definition.py:3000-3016`.)

### 4.2 Reason (b) — abstract sockets leave no filesystem entry

`parse_address_spec` (`kitty/utils.py:502-523`) turns a leading `@` into a NUL‑prefixed Linux **abstract‑namespace** address (`:513-514`):

```python
if address.startswith('@') and len(address) > 1:
    address = '\0' + address[1:]
```

**OBSERVED** with `--listen-on unix:@mykitty`: nothing appears in the filesystem, yet the endpoint is live and usable:

```console
$ ls -l /tmp/@mykitty*
ls: cannot access '/tmp/@mykitty*': No such file or directory

$ lsof -U | grep mykitty
kitty   36331 root    6u  unix 0x0000000000000000      0t0 726289071 /tmp/mykitty type=STREAM (LISTEN)
kitty   36558 root    6u  unix 0x0000000000000000      0t0 726458861 @mykitty type=STREAM (LISTEN)

$ ss -xl | grep mykitty
u_str LISTEN 0      128           /tmp/mykitty 726289071            * 0
u_str LISTEN 0      128               @mykitty 726458861            * 0

$ ./kitty/launcher/kitten @ --to unix:@mykitty ls | jq -c '[.[].tabs[].windows[]|{id,title,cmdline}]'
[{"id":1,"title":"bash","cmdline":["sleep","300"]}]
```

The `@mykitty` endpoint has the leading `@` that marks the abstract namespace and **no path on disk** — a `/tmp` search can never find it.

### 4.3 Reason (c) — default‑off gating: usually there is no socket at all

Remote control is off by default: `allow_remote_control` defaults to `no` (`kitty/options/definition.py:2969`) and `listen_on` defaults to `none` (`:3000`). The socket is only created when **both** permit — `kitty/boss.py:364-366`:

```python
if args.listen_on and self.allow_remote_control in ('y', 'socket', 'socket-only', 'password'):
    try:
        listen_fd, self.listening_on = listen_on(args.listen_on)
```

The bind/listen itself is `kitty/boss.py:177-188` (`listen_on()`): `socket.socket(family)` → `atexit.register(remove_socket_file, …)` → `bind` → `listen`.

**OBSERVED.** Case A — a pure‑default kitty creates no socket, and a window inside it has an **empty** `KITTY_LISTEN_ON`:

```console
$ ./kitty/launcher/kitty bash -c 'echo "LISTEN_ON=[$KITTY_LISTEN_ON]" > /tmp/kitty_obs/gating_child.txt; sleep 60' &
$ ss -xlp | grep kitty | grep -v cfgkitty
u_str LISTEN 0      128           /tmp/mykitty 726289071            * 0    users:(("kitty",pid=36331,fd=6))
$ cat /tmp/kitty_obs/gating_child.txt
LISTEN_ON=[]
```

(The only listening socket is the pre‑existing `/tmp/mykitty` from §4.1; the default instance added none.) Case B — even `--listen-on` is ignored when `allow_remote_control` is left at its default `no`:

```console
$ ./kitty/launcher/kitty --listen-on unix:/tmp/gatekitty bash -c "sleep 60" &
$ ls -l /tmp/gatekitty*
ls: cannot access '/tmp/gatekitty*': No such file or directory
$ ss -xlp | grep gatekitty
(no gatekitty socket — gating confirmed)
```

> **A note on finding the live PID.** `pgrep -x kitty | head -1` is unreliable here: repeated launch/kill cycles leave `[kitty] <defunct>` zombies (reparented to PID 1) that share the process name. The authoritative owner of a socket is what `lsof -U`/`ss -xlp` report (e.g. pid `36331` for `/tmp/mykitty`).


---

## 5. OBJ‑4 — End‑to‑end over the socket: wire bytes, parse/route, JSON, logging

**Direct answer.** The request and response are a **symmetric DCS envelope** `<ESC>P@kitty-cmd<JSON><ESC>\` (`<ESC>` = byte `0x1b`), where the JSON envelope carries `cmd`, `version`, `no_response`, `kitty_window_id`, `payload` (`docs/rc_protocol.rst:8,14-20`). On the server the bytes are decoded by **`parse_cmd`** (`kitty/remote_control.py:56`), dispatched by **`handle_cmd`** (`:213`) → **`command_for_name('ls')`** (`kitty/rc/base.py:449`) → **`LS.response_from_kitty`** (`kitty/rc/ls.py:48`), which builds the tree with `boss.list_os_windows(...)` (`:57`) and returns `json.dumps(data, indent=2, sort_keys=True)` (`:76`); the result is wrapped as `{'ok': True, 'data': <json-string>}` (`kitty/remote_control.py:258-260`) and framed back in the same DCS envelope. **Logging is error‑only**: a successful call logs nothing; a malformed one emits one `log_error` line.

All output below was produced against the live instance from §4.1 (kitty pid **38010**, socket `/tmp/mykitty`, version banner `kitty 0.35.2`).

### 5.1 The real `ls` JSON response (OBSERVED, complete & unedited)

Command and complete output (`kitten @ … ls`, `EXIT=0`):

```console
$ ./kitty/launcher/kitten @ --to unix:/tmp/mykitty ls
[
  {
    "background_opacity": 1.0,
    "id": 1,
    "is_active": true,
    "is_focused": true,
    "last_focused": true,
    "platform_window_id": 2097164,
    "tabs": [
      {
        "active_window_history": [
          1
        ],
        "enabled_layouts": [
          "fat",
          "grid",
          "horizontal",
          "splits",
          "stack",
          "tall",
          "vertical"
        ],
        "groups": [
          {
            "id": 1,
            "windows": [
              1
            ]
          }
        ],
        "id": 1,
        "is_active": true,
        "is_focused": true,
        "layout": "fat",
        "layout_opts": {
          "bias": 50,
          "full_size": 1,
          "mirrored": false
        },
        "layout_state": {
          "biased_map": {},
          "main_bias": [
            0.5,
            0.5
          ],
          "num_full_size_windows": 1
        },
        "title": "bash",
        "windows": [
          {
            "at_prompt": false,
            "cmdline": [
              "sleep",
              "600"
            ],
            "columns": 71,
            "created_at": 1783961945144781620,
            "cwd": "/tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f",
            "env": {
              "COLORTERM": "truecolor",
              "DISPLAY": ":99",
              "GALLIUM_DRIVER": "llvmpipe",
              "HOME": "/root",
              "HOSTNAME": "9af7dc1b7700",
              "KITTY_INSTALLATION_DIR": "/tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f",
              "KITTY_LISTEN_ON": "unix:/tmp/mykitty",
              "KITTY_PID": "38010",
              "KITTY_PUBLIC_KEY": "1:B-Np9SyFTzP$i$UQ!J023rz$z<P+OA-jH0jaP($2",
              "KITTY_SHELL_INTEGRATION": "enabled",
              "KITTY_WINDOW_ID": "1",
              "LC_CTYPE": "C.UTF-8",
              "LIBGL_ALWAYS_SOFTWARE": "1",
              "OLDPWD": "/tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f",
              "PATH": "/tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f/kitty/launcher:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
              "PWD": "/tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f",
              "PYTEST_ADDOPTS": "--tb=short -v --continue-on-collection-errors --reruns=3",
              "SHLVL": "0",
              "TERM": "xterm-kitty",
              "TERMINFO": "/tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f/terminfo",
              "UV_HTTP_TIMEOUT": "60",
              "WINDOWID": "2097164",
              "_": "/usr/bin/sleep"
            },
            "foreground_processes": [
              {
                "cmdline": [
                  "sleep",
                  "600"
                ],
                "cwd": "/tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f",
                "pid": 38082
              }
            ],
            "id": 1,
            "is_active": true,
            "is_focused": true,
            "is_self": false,
            "last_cmd_exit_status": 0,
            "last_reported_cmdline": "",
            "lines": 22,
            "pid": 38082,
            "title": "bash",
            "user_vars": {}
          }
        ]
      }
    ],
    "wm_class": "kitty",
    "wm_name": "kitty"
  }
]
```

This matches the tree documented at `docs/remote-control.rst:84-120`: the top level is a **list of OS windows**; each has `id` + `tabs`; each tab has `id`, `title`, `layout`, and `windows`; each window has `id`, `title`, `cwd`, `pid`, `cmdline`, `env`, and `is_self`. Here `is_self` is **`false`** because the `kitten @ ls` client ran *outside* the listed window (it dialed the socket from a separate process); contrast this with §6 where an in‑window run yields `is_self: true`.

> **Security note (non‑secret).** The `KITTY_PUBLIC_KEY` value above (prefix `1:`) is kitty's **public** X25519 key — it is exported into every child so clients can *encrypt to* kitty; it is ephemeral per‑instance and is **not** a credential. Only a *private* key would be sensitive; none appears in any captured output.

### 5.2 The raw on‑the‑wire bytes (OBSERVED)

Using the canonical `socat` recipe from `docs/rc_protocol.rst:35-42` (the request JSON here carries `version:[0,35,2]`, matching this build). The **request** is exactly `<ESC>P@kitty-cmd{…}<ESC>\`, 45 bytes:

```console
$ printf '\033P@kitty-cmd{"cmd":"ls","version":[0,35,2]}\033\\' | od -c
0000000 033   P   @   k   i   t   t   y   -   c   m   d   {   "   c   m
0000020   d   "   :   "   l   s   "   ,   "   v   e   r   s   i   o   n
0000040   "   :   [   0   ,   3   5   ,   2   ]   } 033   \
0000055
$ printf '\033P@kitty-cmd{"cmd":"ls","version":[0,35,2]}\033\\' | wc -c
45
```

`033` is octal for `0x1b` (ESC); the leading bytes are `ESC P @kitty-cmd` and the trailing bytes are `ESC \` — exactly the constants the Go client uses at `tools/cmd/at/socket_io.go:82-83` (`cmd_escape_code_prefix = "\x1bP@kitty-cmd"`, `cmd_escape_code_suffix = "\x1b\\"`) and the Python client at `kitty/remote_control.py:308-310` (`encode_send`).

The **response** is the same envelope. First 14 bytes and last 2 bytes (hex) prove the framing, and the `od -c` head shows the JSON envelope beginning `{"ok": true, "data": "[\n …`:

```console
$ printf '\033P@kitty-cmd{"cmd":"ls","version":[0,35,2]}\033\\' | socat - unix:/tmp/mykitty | head -c 14 | od -An -tx1
 1b 50 40 6b 69 74 74 79 2d 63 6d 64 7b 22
$ printf '\033P@kitty-cmd{"cmd":"ls","version":[0,35,2]}\033\\' | socat - unix:/tmp/mykitty | od -c | head -6
0000000 033   P   @   k   i   t   t   y   -   c   m   d   {   "   o   k
0000020   "   :       t   r   u   e   ,       "   d   a   t   a   "   :
0000040       "   [   \   n           {   \   n                   \   "
0000060   b   a   c   k   g   r   o   u   n   d   _   o   p   a   c   i
0000100   t   y   \   "   :       1   .   0   ,   \   n                
0000120   \   "   i   d   \   "   :       1   ,   \   n                
$ printf '\033P@kitty-cmd{"cmd":"ls","version":[0,35,2]}\033\\' | socat - unix:/tmp/mykitty | tail -c 2 | od -An -tx1
 1b 5c
```

`1b 50 40 6b 69 74 74 79 2d 63 6d 64 7b 22` = `ESC P @kitty-cmd{"` and `1b 5c` = `ESC \`. This response framing is produced by `encode_response_for_peer` (`kitty/remote_control.py:52-53`).

**Decoding the envelope with the in‑repo recipe.** `awk '{ print substr($0, 13, length($0) - 14) }'` strips the 12‑byte prefix `\x1bP@kitty-cmd` and the 2‑byte suffix `\x1b\\`, leaving the JSON envelope; `jq '.data | fromjson'` then parses the inner tree (note `data` is itself a **JSON string**, so it must be re‑parsed):

```console
$ printf '\033P@kitty-cmd{"cmd":"ls","version":[0,35,2]}\033\\' | socat - unix:/tmp/mykitty \
    | awk '{ print substr($0, 13, length($0) - 14) }' \
    | jq -c '.data | fromjson | [.[].tabs[].windows[]|{id,title,cmdline,is_self}]'
[{"id":1,"title":"bash","cmdline":["sleep","600"],"is_self":false}]
```

The **envelope shape** — outer `{ok, data}` where `data` is a string — confirms `handle_cmd`'s wrapping (`kitty/remote_control.py:258-260`) and `ls`'s `json.dumps` producing a string (`kitty/rc/ls.py:76`):

```console
$ printf '\033P@kitty-cmd{"cmd":"ls","version":[0,35,2]}\033\\' | socat - unix:/tmp/mykitty \
    | awk '{ print substr($0, 13, length($0) - 14) }' \
    | jq '{ok: .ok, data_is_json_string: (.data|type)}'
{
  "ok": true,
  "data_is_json_string": "string"
}
```


### 5.3 The Python parse/route chain — where `ls` gets handled (OBSERVED)

This is the part you asked about directly: *"where in the Python code the incoming command gets parsed and routed to the `ls` handler."* To observe (not merely read) the chain, a **temporary, deletable** instrumentation shim was placed **outside the repo** (`/tmp/kitty_obs/instrument/sitecustomize.py`, loaded via `PYTHONPATH`). It installs a `sys.meta_path` import hook that, *after* each target module loads, wraps the real functions so they print when entered and then call the originals — the **canonical code path still runs**; nothing is bypassed or hand‑crafted, and no repository file is edited. (An eager `sitecustomize` import failed because `kitty` is not yet on `sys.path` at interpreter start, and a polling thread was starved by kitty's C event loop; the import‑hook approach is what worked.)

Running the **real** `kitten @ --to unix:/tmp/instrkitty ls` against a shim‑instrumented instance produced this firing order (OBSERVED):

```text
001 meta_path hook installed at interpreter start
002 patched kitty.remote_control   (imported LAZILY on first RC use, ~6s after startup)
003 parse_cmd              [kitty/remote_control.py:56]   -> cmd='ls' version=[0, 26, 0]
004 handle_cmd             [kitty/remote_control.py:213]  cmd='ls'  (version guard @ :218)
005 command_for_name       [kitty/rc/base.py:449]         cmd_name='ls' -> import kitty.rc.ls
006 patched kitty.rc.ls    (LS imported lazily by command_for_name)
007 LS.response_from_kitty [kitty/rc/ls.py:48]            -> boss.list_os_windows [ls.py:57], then json.dumps [ls.py:76]
```

So the observed core chain is:

```
parse_cmd  ->  handle_cmd  ->  command_for_name('ls')  ->  LS.response_from_kitty
[rc.py:56]     [rc.py:213]     [rc/base.py:449]            [rc/ls.py:48]
```

Two observed nuances worth recording:

- **The real client sends RC protocol version `[0,26,0]`** — the *remote‑control protocol* version, **not** the `0.35.2` release version. It is `<=` the instance version, so it passes the guard `if tuple(v)[:2] > version[:2]:` at `kitty/remote_control.py:218`. *(INFERRED extension, not exercised: a request whose major/minor is newer than the instance would be rejected there — `docs/rc_protocol.rst:22-25`.)*
- **Dispatch is name‑based and dynamic** — `command_for_name` does `cmd_name.replace('-', '_')` (`kitty/rc/base.py:451`) then `import_module(f'kitty.rc.{cmd_name}')` (`:453`). `kitty.rc.ls` is imported **lazily**, on the first `ls` request. **This is exactly the extension point a new command plugs into** (see §8).

**How each transport reaches this core (ingress).** The parse/route core above is shared; only the ingress differs. The ingress steps below are **INFERRED from reading** (they were not individually instrumented), but the transport itself is OBSERVED end‑to‑end (§4, §6) and both transports were observed to reach the *same* instrumented `parse_cmd`:

- **Socket ingress (INFERRED‑from‑reading):** `boss.peer_message_received` (`kitty/boss.py:776`) → `_handle_remote_command` (`:590`) → `_execute_remote_command` (`:700`) → `from .remote_control import handle_cmd`.
- **TTY ingress (INFERRED‑from‑reading):** the C DCS parser dispatches `@kitty-cmd` at `kitty/vt-parser.c:603` (`dispatch("cmd{", handle_remote_cmd, 1)`, only after `starts_with("kitty-")` at `:600`) → `kitty/window.py:1279` `handle_remote_cmd` → `get_boss().handle_remote_cmd(...)` (`:1280`) → `kitty/boss.py:849` `handle_remote_cmd` → `_handle_remote_command` (`:590`).

Running the **socketless in‑window** `kitten @ ls` (TTY ingress) against the shim produced the **identical** observed core chain (`parse_cmd [:56] → handle_cmd [:213] → command_for_name [base.py:449] → LS.response_from_kitty [ls.py:48]`, `EXIT=0`), confirming both transports converge on the same parse/route core.

### 5.4 Logging behavior — error‑only (OBSERVED, both states)

kitty's RC path logs **nothing on success** and exactly one `log_error` line on a parse failure. Captured against the live pid‑38010 instance whose stderr was redirected to a file at launch:

```console
# SUCCESS — a real ls adds no new stderr line
$ BEFORE=$(wc -l < /tmp/kitty_obs/recap_stderr.log)
$ ./kitty/launcher/kitten @ --to unix:/tmp/mykitty ls >/dev/null 2>&1
$ AFTER=$(wc -l < /tmp/kitty_obs/recap_stderr.log)
$ echo "stderr_lines_before=$BEFORE stderr_lines_after=$AFTER"
stderr_lines_before=1 stderr_lines_after=1

# ERROR — malformed JSON over the socket emits one log_error line
$ printf '\033P@kitty-cmd{bad json}\033\\' | socat - unix:/tmp/mykitty >/dev/null 2>&1
$ tail -n 3 /tmp/kitty_obs/recap_stderr.log
[0.277] Failed to open systemd user bus with error: No medium found
[25.791] Failed to parse JSON payload of remote command, ignoring it
```

- The **success** run left the line count unchanged (`1 → 1`) — no RC log on success.
- The **error** run added exactly `Failed to parse JSON payload of remote command, ignoring it`, which is `log_error(...)` at `kitty/remote_control.py:62` inside `parse_cmd`.
- The only *pre‑existing* line, `[0.277] Failed to open systemd user bus with error: No medium found`, is a **benign D‑Bus startup warning from the container** — it is unrelated to remote control and is present regardless of any RC activity. That is why the "before" count is `1` rather than `0`.

Two other error lines were observed during earlier probing (recorded for completeness): `Malformatted remote control message received from peer, ignoring` — `log_error` at `kitty/boss.py:792` (`peer_message_received`, triggered by a broken frame); and the third parse‑error site `Failed to parse remote command with error: {e}` at `kitty/boss.py:605` (`_handle_remote_command`) was **not** triggered in these runs. `log_error` itself lives in `kitty/utils.py:130`.


---

## 6. OBJ‑3 — Shell integration & RC‑over‑TTY: the "no configuration" magic

**Direct answer.** Running `kitten @ ls` *inside a kitty window* with no `--to` works because **kitty exports `KITTY_LISTEN_ON` (plus `KITTY_PID` and `KITTY_PUBLIC_KEY`) into every child process's environment**, and the Go client uses `KITTY_LISTEN_ON` as its default target (`tools/cmd/at/main.go:371-372`). If a socket target is present it dials that **same socket**; if **no** socket is configured, the client writes the `@kitty-cmd` **DCS escape sequence straight to the controlling PTY**, and the running kitty parses it out of the terminal stream — that is the "TTY magic … through the terminal's own pty." **Shell integration is *not* the enabler**: the shell‑integration scripts only *consume* env vars — they never set `KITTY_LISTEN_ON`. So the resolution of your either/or is: **it's the same socket when one is configured, and it's TTY‑delivered DCS escapes when no socket exists — and either way the capability comes from kitty's env export, not from the shell scripts.**

### 6.1 The env vars you couldn't trace: producer vs consumer (OBSERVED)

The variables are **produced by kitty's C‑child setup, in Python**, not by the shell scripts. `get_final_env` at `kitty/child.py:244-249`:

```python
env['KITTY_PID'] = getpid()
env['KITTY_PUBLIC_KEY'] = boss.encryption_public_key
if self.add_listen_on_env_var and boss.listening_on:
    env['KITTY_LISTEN_ON'] = boss.listening_on
else:
    env.pop('KITTY_LISTEN_ON', None)
```

Note the condition: `KITTY_LISTEN_ON` is exported **only when `boss.listening_on` is truthy** (i.e. kitty is actually listening on a socket). The **consumer** is the Go client, `tools/cmd/at/main.go:371-373`:

```go
if rc_global_opts.To == "" {
    rc_global_opts.To = os.Getenv("KITTY_LISTEN_ON")
    global_options.to_address_is_from_env_var = true
```

`KITTY_SHELL_INTEGRATION` is a *separate* variable, produced by `kitty/shell_integration.py:223` (`env['KITTY_SHELL_INTEGRATION'] = ksi`).

**Proof the shell scripts do not set `KITTY_LISTEN_ON` (OBSERVED):** a recursive grep across the whole `shell-integration/` tree finds nothing (grep exits `1` = no match):

```console
$ grep -rn "KITTY_LISTEN_ON" shell-integration/ ; echo "grep_exit=$?"
grep_exit=1
```

The scripts only ever **read** the kitty‑set variables — e.g. `shell-integration/bash/kitty.bash:215` (`if [[ -z "$KITTY_PID" ]]; then`), `shell-integration/zsh/kitty-integration:249` (`if [[ -n "$KITTY_PID" ]]; then`), and `shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish:29` (`test -n "$KITTY_SHELL_INTEGRATION" || return 0`). They use `KITTY_PID` only for local‑vs‑SSH detection; none of them creates the RC target.

### 6.2 Case 4 — in‑window, socket configured → uses the same socket (OBSERVED)

Launch kitty with a socket, then run bare `kitten @ ls` (no `--to`) from inside the window:

```console
--- env inside window ---
ENV: PID=[38187] LISTEN_ON=[unix:/tmp/winkitty] SI=[enabled]
/tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f/kitty/launcher/kitten
--- kitten @ ls exit ---
KITTEN_EXIT=0
--- kitten @ ls (projected windows) ---
[{"id":1,"title":"bash","is_self":true,"cmdline":["bash","/tmp/kitty_obs/case4_inner.sh"]}]
--- listening sockets for THIS run ---
u_str LISTEN 0  128  /tmp/winkitty 727288485  * 0  users:(("kitty",pid=38187,fd=6))
```

`KITTY_LISTEN_ON` is `unix:/tmp/winkitty`, the target resolves from it, `is_self` is now **`true`** (the client ran inside the listed window), and a real socket exists. Branch taken: `to_network="unix"` → `do_socket_io` (`tools/cmd/at/main.go:280`).

### 6.3 Case 5 — in‑window, **no** socket → DCS‑over‑PTY (OBSERVED)

Launch kitty with `allow_remote_control=yes` but **no** `--listen-on`, then run bare `kitten @ ls`:

```console
--- env inside window ---
ENV: PID=[38284] LISTEN_ON=[] SI=[enabled]
/tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f/kitty/launcher/kitten
--- kitten @ ls exit ---
KITTEN_EXIT=0
--- kitten @ ls (projected windows) ---
[{"id":1,"title":"bash","is_self":true,"cmdline":["bash","/tmp/kitty_obs/case5_inner.sh"]}]
--- listening sockets for THIS run ---
(none besides pre-existing /tmp/mykitty)
```

Here `KITTY_LISTEN_ON` is **empty**, and there is **no listening socket** for this instance (`ss` shows only the pre‑existing `/tmp/mykitty` from another instance). Yet `kitten @ ls` still succeeds (`EXIT=0`, `is_self:true`). Because the target is empty (`To=""` → `to_network=""`), the client takes the `do_tty_io` branch (`tools/cmd/at/main.go:280`) and writes the DCS escape to the controlling PTY (`tools/cmd/at/tty_io.go:79-82`); kitty parses it from the terminal stream at `kitty/vt-parser.c:603` and routes it via `kitty/window.py:1279`. **This definitively demonstrates the socketless TTY transport** — there is no socket anywhere, and it still works.

### 6.4 Case 6 — disambiguation: `shell_integration=disabled` still works (OBSERVED, required by R5)

To prove shell integration is not the enabler, launch with shell integration **off** but remote control **on**:

```console
--- kitty options: -o shell_integration=disabled -o allow_remote_control=yes --listen-on unix:/tmp/nosikitty ---
--- env inside window ---
ENV: PID=[38380] LISTEN_ON=[unix:/tmp/nosikitty] SI=[]
/tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f/kitty/launcher/kitten
--- kitten @ ls exit ---
KITTEN_EXIT=0
--- kitten @ ls (projected windows) ---
[{"id":1,"title":"bash","is_self":true,"cmdline":["bash","/tmp/kitty_obs/case6_inner.sh"]}]
--- listening sockets for THIS run ---
u_str LISTEN 0  128  /tmp/nosikitty 727319852  * 0  users:(("kitty",pid=38380,fd=6))
```

`KITTY_SHELL_INTEGRATION` is now **empty** (shell integration genuinely off), but `KITTY_LISTEN_ON` is **still set** (`child.py` exports it regardless of shell integration), and `kitten @ ls` still works (`EXIT=0`). **Conclusion: shell integration is not what makes RC work** — the enabler is kitty's env export (`kitty/child.py:244-249`) plus the two RC transports.

> **Gotcha you may hit:** the correct value is `shell_integration=disabled`, **not** `shell_integration=no`. `no` is an invalid token that kitty silently ignores, leaving shell integration *enabled*; the default is `enabled` (`kitty/options/definition.py:3141`).

### 6.5 Resolution of your either/or

- **Same socket?** Yes — *when a socket is configured*, the in‑window client resolves `KITTY_LISTEN_ON` and dials that very socket (Case 4).
- **TTY magic through the pty?** Yes — *when no socket exists*, the client writes the `@kitty-cmd` DCS escape to the controlling PTY and kitty parses it from the terminal stream (Case 5).
- **Either way**, the target/capability originates from kitty's environment export in `kitty/child.py`, **not** from the shell‑integration scripts (Case 6 + the empty grep).


---

## 7. Encryption & authorization context (for completeness — NOT exercised, INFERRED‑from‑spec)

This path was **not** run in this investigation (no `remote_control_password` was configured), so this section is **INFERRED from the in‑repo spec and code**, included only for context; nothing here was altered.

- **When is it used?** Only when a `remote_control_password` is involved. The plaintext `kitten @ ls` runs above are **not** encrypted (the socket/TTY payloads shown in §5.2 are cleartext DCS + JSON).
- **Scheme (`docs/rc_protocol.rst:57-84`):** the command is encrypted using the recipient's **public key** taken from `KITTY_PUBLIC_KEY` (protocol prefix `1:`, `docs/rc_protocol.rst:60-61`); key agreement is **X25519** ECDH (`:64`); a **time‑based nonce** is added and commands whose `timestamp` is **more than 5 minutes** from now are **rejected** (`:66-68`); the command is then encrypted with **AES‑256‑GCM** (authenticated) (`:69`) and the fields are base85‑encoded into the envelope (`iv`, `tag`, `pubkey`, `encrypted`).
- **Authorization gating (context):** `_handle_remote_command` (`kitty/boss.py:598-600`) rejects when `allow_remote_control == 'n'` (`Remote control is disabled`) and, for `socket-only`, rejects non‑socket callers (`Remote control is allowed over a socket only`). Per‑command permission is decided by `is_cmd_allowed` / `PasswordAuthorizer` (`kitty/remote_control.py:134,177`). Altering encryption/auth is **out of scope** (AAP §0.5.2).

---

## 8. How to add a NEW remote‑control command (the pattern — NOT implemented here)

This task **does not** add a command (AAP §0.5.2); this section only *equips* you, grounded in how `ls` is built. A new command follows the exact `ls` pattern:

1. **Create `kitty/rc/<name>.py`** defining `class <Name>(RemoteCommand)` (base class in `kitty/rc/base.py`), implementing the two halves that `ls` demonstrates:
   - **client side** `message_to_kitty(self, global_opts, opts, args)` — builds the request payload. `ls` returns `{'all_env_vars', 'match', 'match_tab'}` (`kitty/rc/ls.py:45`).
   - **server side** `response_from_kitty(self, boss, window, payload_get)` — does the work and returns data. `ls` calls `boss.list_os_windows(...)` (`:57`) and returns `json.dumps(data, indent=2, sort_keys=True)` (`:76`). A JSON‑string data field is fine.
   - end the module with a singleton instance, as `ls` does: `ls = LS()` (`kitty/rc/ls.py:79`).
2. **Routing is automatic and name‑based** — no registry edit is needed. `command_for_name('<name>')` maps `cmd-name` → `cmd_name` (`kitty/rc/base.py:451`) and dynamically `import_module(f'kitty.rc.{cmd_name}')` (`:453`). Dropping the module in `kitty/rc/` is enough for `kitten @ <name>` to find it.
3. **Responses are wrapped for you** — `handle_cmd` wraps the return as `{'ok': True, 'data': …}` (`kitty/remote_control.py:258-260`) and frames it back in the DCS envelope (`:52-53`).

> **State clearly:** no such module is added by this task; the repository's `kitty/rc/` tree is unchanged.

---

## 9. Observed‑vs‑Inferred ledger

Every major claim, tagged **OBSERVED** (reproduced at runtime, with the producing command) or **INFERRED** (from reading code/spec, with `file:line`).

| # | Claim | Status | Evidence |
|---|-------|--------|----------|
| 1 | Build is `python3 setup.py`; launcher at `kitty/launcher/kitty`; banner `kitty 0.35.2` | OBSERVED | §2 — `--verbose` build tail + `--version` |
| 2 | Transport = socket **or** DCS‑over‑PTY, never a pipe | OBSERVED (both) | §4/§6 socket dial + §6.3 socketless TTY; code `main.go:280` |
| 3 | `--to` present → `do_socket_io` (`net.Dial`) | OBSERVED | §4.1 + `socket_io.go:177` |
| 4 | No `--to`, no socket → `do_tty_io` (DCS to PTY) | OBSERVED | §6.3 (no socket, still works) + `tty_io.go:79-82` |
| 5 | CLI `--listen-on` value is **verbatim** (no `-PID`) | OBSERVED | §4.1 `/tmp/mykitty` + `main.py:403-409` |
| 6 | Config‑file `listen_on` gets `-<PID>` suffix | OBSERVED | §4.1 `/tmp/cfgkitty-36692` + `main.py:329-331` |
| 7 | `unix:@name` = abstract socket, no FS entry | OBSERVED | §4.2 `@mykitty` via `ss`/`lsof` + `utils.py:513-514` |
| 8 | Default‑off: no socket unless `allow_remote_control` **and** `listen_on` set | OBSERVED | §4.3 + `boss.py:364-366`, defaults `definition.py:2969,3000` |
| 9 | Request/response framing `<ESC>P@kitty-cmd…<ESC>\` | OBSERVED | §5.2 `od`/hex + `socket_io.go:82-83`, `remote_control.py:52-53` |
| 10 | Envelope is `{ok:true, data:<json-string>}` | OBSERVED | §5.2 shape check + `remote_control.py:258-260`, `ls.py:76` |
| 11 | Parse/route: `parse_cmd → handle_cmd → command_for_name('ls') → LS.response_from_kitty` | OBSERVED | §5.3 shim firing order |
| 12 | Both transports converge on the same parse/route core | OBSERVED | §5.3 (socket + TTY identical chain) |
| 13 | Real `ls` JSON tree structure | OBSERVED | §5.1 complete output |
| 14 | Logging: empty on success, one `log_error` on malformed | OBSERVED | §5.4 line‑count + error line, `remote_control.py:62` |
| 15 | `KITTY_LISTEN_ON`/`KITTY_PID`/`KITTY_PUBLIC_KEY` produced by `kitty/child.py` | OBSERVED | §6.1 code + §5.1/§6 env dumps |
| 16 | Shell‑integration scripts do NOT set `KITTY_LISTEN_ON` | OBSERVED | §6.1 `grep` exit 1 |
| 17 | `shell_integration=disabled` → RC still works (SI not the enabler) | OBSERVED | §6.4 |
| 18 | Client sends RC protocol version `[0,26,0]` | OBSERVED | §5.3 shim |
| 19 | Socket ingress `peer_message_received → _handle_remote_command → _execute_remote_command` | INFERRED | `boss.py:776,590,700` (parse_cmd it calls IS observed) |
| 20 | TTY ingress `vt-parser.c dispatch → window.handle_remote_cmd → boss.handle_remote_cmd` | INFERRED | `vt-parser.c:603`, `window.py:1279`, `boss.py:849` |
| 21 | Response functions `encode_response_for_peer` / `send_cmd_response` | INFERRED (shape OBSERVED) | `remote_control.py:52-53`, `window.py:1386` |
| 22 | Version newer than instance is rejected | INFERRED | `remote_control.py:218`, `rc_protocol.rst:22-25` |
| 23 | X25519 + AES‑256‑GCM encryption, 5‑min nonce | INFERRED‑from‑spec | `rc_protocol.rst:64,68,69` (not exercised) |

---

## 10. Coverage pass — your four questions, answered

1. **How does `kitten @ ls` reach kitty, and how does the client discover *where* to send? Socket, pipe, or something else?**
   → **§1 (TL;DR #1) + §3 + §5.** It is a **UNIX/TCP socket** *or* **terminal DCS escape sequences over the PTY** — **never a pipe**. The client (`tools/cmd/at/main.go:280`) picks `do_socket_io` when it knows a target address and `do_tty_io` otherwise; the target defaults to `KITTY_LISTEN_ON` (`:371-372`).

2. **Why does poking around `/tmp` for the socket fail / not match the docs?**
   → **§1 (TL;DR #2) + §4.** Three compounding reasons, each shown live: config‑file paths get a **`-<PID>` suffix** (`/tmp/cfgkitty-36692`, §4.1); `unix:@…` **abstract** sockets have **no file** (§4.2); and by default there is **no socket at all** unless `allow_remote_control` + `listen_on` are both set (§4.3).

3. **Shell integration "without configuration": same socket, or TTY magic through the pty? And where are those env vars used?**
   → **§1 (TL;DR #3) + §6.** **Same socket when one is configured** (§6.2, `KITTY_LISTEN_ON`); **DCS‑over‑PTY when no socket exists** (§6.3). The env vars are **produced by `kitty/child.py:244-249`** and **consumed by `tools/cmd/at/main.go:371-372`**; the shell scripts only *read* them (grep exit 1, §6.1) — proven by `shell_integration=disabled` still working (§6.4).

4. **Show the real socket path, the on‑the‑wire messages, where Python parses/routes to `ls`, the returned JSON, and logging.**
   → **§1 (TL;DR #4) + §4/§5.** Real path `/tmp/mykitty` (§4.1); raw + decoded DCS bytes (§5.2); parse/route chain `parse_cmd → handle_cmd → command_for_name('ls') → LS.response_from_kitty` (§5.3); complete `ls` JSON (§5.1); logging empty‑on‑success vs one `log_error` on malformed (§5.4).

---

*End of document. All runtime evidence above was captured on `kitty 0.35.2` (VCS `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`) inside the prepared container, under `Xvfb :99` with the Mesa `llvmpipe` software GL renderer. Volatile values (PIDs such as `38010`/`38187`/`38284`/`38380`, socket inode numbers, timestamps, and the ephemeral public key) are reported exactly as observed and will differ on other runs.*

