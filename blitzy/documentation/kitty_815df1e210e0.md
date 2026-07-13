# How `kitten @ ls` Works — An End‑to‑End, Runtime‑Grounded Trace of kitty's Remote‑Control System

> **Scope & method.** This document answers, with **observed runtime evidence**, how kitty's remote‑control (RC) round trip works, using your exact example **`kitten @ ls`**. Every system claim carries a `file:line` reference and names the function/struct that does the work. Claims are tagged with one of three precise labels:
>
> - **OBSERVED** — reproduced at runtime in the container, shown with the exact command that produced it and its unedited output.
> - **SOURCE‑VERIFIED** — read directly from the code, or produced by a static command such as `grep`/`sed`; the `file:line` is cited, but that specific branch was not separately exercised end‑to‑end at runtime.
> - **INFERRED** — derived from the protocol spec or from reading code, and *not* exercised here; called out explicitly wherever it appears.
>
> Nothing in the kitty source tree was modified to produce this document. All instrumentation used throwaway scripts created **outside** the repository under a `mktemp -d` working directory, launched kitty in the background with explicit PID capture (`$!`), waited on readiness, and cleaned up via an `EXIT` trap; every script was deleted afterward and `git status --porcelain` was confirmed empty (§4, §5.5, and the cleanup note at the end).
>
> **This document is explanatory only. It does not add a new RC command** — but §8 distills the exact pattern so you can add one later.

- **Branch:** `blitzy-5272820a-f82d-4851-9076-f4ffb122d22c`.
- **Code checkout:** the RC subsystem was read and run at commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` ("Wire up applying of font config"), the branch's **base** commit. This document is added on top of that base as **documentation-only commit(s)**; the first was `7da6b37726335799f63f5ea0bca9185625382c21`, and the branch has since advanced with follow-up documentation edits (each touching **only this `.md`**). Because none of those commits change any C, Go, or Python file, the C/Go/Python **source is byte‑identical to `815df1e2`** — but `setup.py` stamps the binary with the current git `HEAD`, so the **authoring-time** build embedded `VCSRevision=7da6b37726335799f63f5ea0bca9185625382c21` (OBSERVED in the `--verbose` build line, §2).
- **Build/version:** `kitty 0.35.2` (see §2).
- **Transports exercised:** (1) configured UNIX socket, and (2) shell‑integration / RC‑over‑TTY. Both are demonstrated with before/after state.

---

## 1. TL;DR — Direct answers to your four questions

**Q1 — How does `kitten @ ls` reach a running kitty, and how does it discover *where* to send? "Is it a socket? A pipe? Something else?"**
It is **a socket or terminal (DCS) escape sequences — never an anonymous pipe.** The Go `kitten` client decides at dispatch time: if it knows a target address it dials a UNIX/TCP **socket** (`do_socket_io` → `net.Dial`); if it doesn't, it writes a **DCS escape sequence to the controlling PTY** (`do_tty_io`). The single line that chooses is `tools/cmd/at/main.go:280`:
`response, err = get_response(utils.IfElse(global_options.to_network == "", do_tty_io, do_socket_io), io_data)`. The address itself defaults to the `KITTY_LISTEN_ON` environment variable when `--to` is omitted (`tools/cmd/at/main.go:371-372`). **OBSERVED** below in §3–§6.

**Q2 — Why does poking around `/tmp` for the socket fail / not match the docs?**
Three compounding reasons, all **OBSERVED** in §4:
1. **Templated path (the big one)** — a `listen_on` value taken from the **config file** gets an automatic `-<PID>` suffix appended (`kitty/main.py:329-330`), so a configured `unix:.../cfgkitty` becomes `.../cfgkitty-<PID>` on disk (observed: `.../cfgkitty` → `.../cfgkitty-40761`, §4.1). A value passed on the **command line** (`--listen-on`) is used **verbatim** — with one exception: an explicit `{kitty_pid}` token *anywhere* in the value is always substituted with the real PID (`kitty/main.py:331`, unconditional), on the command line too (observed: `.../tplkitty-{kitty_pid}` → `.../tplkitty-40680`, §4.1). Only the *automatic* suffix is config‑file‑only; the `{kitty_pid}` expansion applies to both. This asymmetry is the single most likely cause of your failed search.
2. **Abstract sockets** — `unix:@name` creates a Linux abstract‑namespace socket (`kitty/utils.py:513-514`) that has **no filesystem entry at all**; `ls /tmp` can never find it.
3. **Default‑off gating** — no socket is created unless **both** `allow_remote_control` (default `no`, `kitty/options/definition.py:2969`) **and** `listen_on` (default `none`, `:3000`) are set (`kitty/boss.py:364-366`).

**Q3 — Shell integration lets you send commands "without explicit configuration." Same socket, or "TTY magic through the terminal's own pty"?**
**Both — decided by whether the client holds a target *address string*, not by probing for a live socket — and shell integration is *not* the enabler.** When kitty spawns a child it always exports `KITTY_PID` and `KITTY_PUBLIC_KEY`, and it exports `KITTY_LISTEN_ON` **only when it is actually listening on a socket** (`kitty/child.py:244-249`: `KITTY_PID`/`KITTY_PUBLIC_KEY` are unconditional; `KITTY_LISTEN_ON` is set only `if self.add_listen_on_env_var and boss.listening_on`, otherwise it is popped from the environment). So for `kitten @ ls` (no `--to`) inside a window:
- If kitty **is** listening, `KITTY_LISTEN_ON` is populated, the client resolves a `unix:`/`tcp:` target and dials the **same socket** (OBSERVED: §6 Case 4).
- If kitty is **not** listening, `KITTY_LISTEN_ON` is empty, so the target string is empty and the client takes the TTY branch **from the outset**, writing the `@kitty-cmd` DCS escape **through the window's own PTY**, which kitty parses out of the terminal stream (`kitty/vt-parser.c:603`) — that is the "TTY magic" (OBSERVED: §6 Case 5, with the actual PTY bytes in §5.2).

Crucially, this is a decision on the **presence of the target string** (`tools/cmd/at/main.go:280`), *not* a dial‑then‑fallback: a target that is set but unreachable still takes the socket branch and **fails with no TTY fallback** (SOURCE‑VERIFIED, §3). The shell‑integration scripts only **consume** env vars — `grep -rn KITTY_LISTEN_ON shell-integration/` returns **nothing** (exit 1; §6, OBSERVED) — so the producer of `KITTY_LISTEN_ON` is `kitty/child.py`, not the shell.

**Q4 — Show the real socket path, the wire bytes, where Python parses/routes to `ls`, the returned JSON, and the logging behavior.**
- **Real socket path:** `.../mykitty` (CLI, verbatim, no PID suffix) and `.../cfgkitty-40761` (config, PID‑suffixed) — full observed paths under a `mktemp -d` dir in §4.
- **On the wire:** a symmetric DCS envelope `<ESC>P@kitty-cmd<JSON><ESC>\` (`<ESC>` = `0x1b`). The **real client** request is **58 bytes** carrying `"version":[0,26,0]` and `"payload":{}`; the response is 3655 bytes, beginning `1b 50 40 6b 69 74 74 79 2d 63 6d 64 7b 22 6f 6b` (`ESC P @kitty-cmd{"ok`) and ending `1b 5c` (`ESC \`). The **same 58‑byte frame** is what the client writes to the PTY in the socketless case — §5.2.
- **Parse/route (OBSERVED):** `parse_cmd` (`kitty/remote_control.py:56`) → `handle_cmd` (`:213`) → `command_for_name('ls')` (`kitty/rc/base.py:449`) → `LS.response_from_kitty` (`kitty/rc/ls.py:48`) → `boss.list_os_windows` (`:57`) → `json.dumps(..., indent=2, sort_keys=True)` (`:76`), wrapped as `{'ok': True, 'data': <json-string>}` (`kitty/remote_control.py:258-260`) — §5.3.
- **Returned JSON:** the full OS‑window/tab/window tree — §5.1.
- **Logging:** **error‑only.** A successful `ls` logs **nothing**; a malformed command logs `Failed to parse JSON payload of remote command, ignoring it` (`kitty/remote_control.py:62`) — §5.5.

---

## 2. Environment & canonical build

The live investigation ran inside the prepared Docker container (image `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`, name `kitty-setup`), because kitty is a GPU/OpenGL GUI terminal that needs a display and (software) GL. The repository is bind‑mounted at its real path `/tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f`, which is also the repo root.

**Canonical build (R4).** The build command is `python3 setup.py`. It is an **incremental** build, not a one‑shot: it checks each C source against its object file and the Go client against its inputs, rebuilding only what is stale. This has two observed states (contrary to a plain "does nothing"):

*(a) Already up to date* — nothing is stale, so it compiles nothing and prints nothing, exiting 0:

```console
$ python3 setup.py ; echo "exit=$?"
exit=0
```

*(b) A source is newer than its artifact* — here forced with `touch kitty/data-types.c`, which changes only the file's **mtime**, not its bytes (so the tracked file stays unmodified and `git status` remains clean). `setup.py` then recompiles and relinks the C extension, exiting 0:

```console
$ touch kitty/data-types.c && python3 setup.py ; echo "exit=$?"
[1/1] Compiling kitty/data-types.c ...
 done
[1/1] Linking kitty/fast_data_types ...
 done
exit=0
```

Run with `--verbose` it additionally prints compiler detection and the Go build of the `kitten` client. (Between `Detected:` and `Updating Go generated files...`, `--verbose` also echoes the two full gcc commands — one compile line for `data-types.c` and one link line spanning ~60 object files into `fast_data_types.so`; those two phases are exactly the `Compiling`/`Linking` steps shown complete above, so the mechanical gcc lines are not repeated here.) The Go build line is shown in full and embeds the git‑`HEAD` VCS revision:

```console
$ touch kitty/data-types.c && python3 setup.py --verbose
CC: ['gcc'] (13, 0)
gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
Copyright (C) 2023 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Detected: CompilerType.gcc
Updating Go generated files...
/usr/local/go/bin/go build -v -ldflags '-X kitty.VCSRevision=7da6b37726335799f63f5ea0bca9185625382c21 -s -w' -o kitty/launcher/kitten /tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f/tools/cmd
```

(The `VCSRevision` value captured above (`7da6b377…`) is the **authoring-time** documentation commit — the *first* commit that added this document — because `setup.py` derives the stamp from `git rev-parse HEAD`. The branch has since advanced with follow-up documentation-only commits, so a fresh `python3 setup.py` build today stamps the *current* `HEAD` instead; the stamp reflects only the checked-out commit. The compiled C/Go/Python source is identical to `815df1e2` regardless — so the binary's behavior is unchanged — as explained in the header.)

The build produces the launcher at **`kitty/launcher/kitty`** and the client at **`kitty/launcher/kitten`** (build logic `setup.py:1230` `build_launcher`; `launcher_dir = 'kitty/launcher'` `setup.py:2099`). **OBSERVED:**

```console
$ ls -la kitty/launcher/kitty kitty/launcher/kitten kitty/fast_data_types.so
-rwxr-xr-x 1 root root  1213072 Jul 13 18:04 kitty/fast_data_types.so
-rwxr-xr-x 1 root root 15962372 Jul 13 18:04 kitty/launcher/kitten
-rwxr-xr-x 1 root root    36224 Jul 13 16:08 kitty/launcher/kitty
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

> **Note on startup log noise.** In this container kitty always prints one benign, non‑RC line at startup: `Failed to open systemd user bus with error: No medium found` (the leading bracketed timestamp is a monotonic uptime and varies per run — observed as `[0.162]`, `[0.175]`, `[0.277]`, etc.). It is a D‑Bus artifact of the headless container and is unrelated to remote control. It is called out here so the "empty log on success" claim in §5.5 is precise: that one line is present regardless of any RC activity.

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
| `kitten @ --to unix:.../mykitty ls` | `--to` present → `to_network="unix"` | `do_socket_io` → `net.Dial` | §4.1 (socket dialed) / §5.2 (real wire bytes) |
| `kitten @ ls` in a window **with** a socket | `KITTY_LISTEN_ON=unix:.../winkitty` (kitty pid 41386) | `do_socket_io` (same socket) | §6 Case 4 |
| `kitten @ ls` in a window **without** a socket | `KITTY_LISTEN_ON` empty → `to_network=""` | `do_tty_io` (DCS over PTY) | §6 Case 5 |

---

## 4. OBJ‑2 — The socket‑path mystery: why `/tmp` came up empty

**Direct answer.** Your `/tmp` search failed for three compounding reasons, each demonstrated live below: (a) config‑file paths get a **`-<PID>` suffix**; (b) `unix:@…` **abstract** sockets have **no file** at all; (c) with defaults, **no socket exists** because remote control is off. The **real** paths observed here are `.../mykitty` (CLI, verbatim) and `.../cfgkitty-40761` (config, suffixed), where `...` is the `mktemp -d` working directory described just below.

> ⚠️ **Security / trust boundary — read before you enable remote control.** Every runtime block below turns remote control **on** with `allow_remote_control=yes` purely for observation. Be deliberate about this in real use. kitty's own option help warns that once remote control is enabled, "other programs can control all aspects of kitty, including sending text to kitty windows, opening new windows, closing windows, reading the content of windows, etc." and that "this even works over SSH connections" (`kitty/options/definition.py:2970-2976`). The value `yes` accepts **every** request with **no authentication** — anything that can reach the transport gains full control of your terminal. A `unix:` socket is guarded **only** by filesystem permissions on the socket path, and a `tcp:` listener is reachable by anything that can open the port. For anything beyond throwaway local testing, prefer a least‑privilege mode (`socket-only`, `socket`, or `password`) as detailed in **§7**. This investigation uses `yes` **only** because it runs disposable instances inside an isolated container.

> **Observation harness (safe, reproducible — R7/finding on scripting hygiene).** Every runtime block in §4–§5 was produced by a throwaway script created **outside** the repository. The scaffold is identical throughout: create an isolated working directory with `mktemp -d`, capture each background kitty's PID with `$!`, wait for the socket to appear before probing, and delete everything on exit via a `trap`:
>
> ```bash
> #!/usr/bin/env bash
> set -u
> KITTY=/tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f/kitty/launcher/kitty
> KITTEN=${KITTY%kitty}kitten
> WORK="$(mktemp -d /tmp/kitty_obs_work/run.XXXXXX)"   # e.g. /tmp/kitty_obs_work/run.e6tI6X — NOT in the repo
> PIDS=()
> cleanup(){ for p in "${PIDS[@]}"; do kill "$p" 2>/dev/null; done; rm -rf "$WORK"; }
> trap cleanup EXIT
> wait_sock(){ for _ in $(seq 1 50); do [ -S "$1" ] && return 0; sleep 0.1; done; return 1; }
> ```
>
> The concrete `WORK` for the §4 blocks below was `/tmp/kitty_obs_work/run.e6tI6X`.

### 4.1 Reason (a) — CLI value is verbatim (unless it contains `{kitty_pid}`); config‑file value gets `-<PID>`

The asymmetry lives in `expand_listen_on` (`kitty/main.py:325-343`), specifically `:329-331`:

```python
if '{kitty_pid}' not in listen_on and from_config_file and listen_on.startswith('unix:'):
    listen_on += '-{kitty_pid}'
listen_on = listen_on.replace('{kitty_pid}', str(os.getpid()))
```

Two distinct behaviors live on these adjacent lines and are easy to conflate:
- **`main.py:329-330`** — the *automatic* `-{kitty_pid}` suffix. It is appended **only** when the value did not already contain `{kitty_pid}`, **and** it came from the config file (`from_config_file`), **and** it is a `unix:` spec.
- **`main.py:331`** — the `{kitty_pid}` **token substitution**. It runs **unconditionally** on every value, so an explicit `{kitty_pid}` token is *always* replaced with the real PID — on the command line as well as in the config file.

The `from_config_file` flag is decided in `setup_environment` (`kitty/main.py:403-409`): if `--listen-on` was passed on the command line, `from_config_file` stays `False`; if the value came from the config file's `listen_on`, it becomes `True`. Relative UNIX paths are additionally resolved under `tempfile.gettempdir()` (`kitty/main.py:339`). Net effect: a **command‑line** value is used **verbatim** *unless it contains `{kitty_pid}`*; a **config‑file** value gets the automatic `-<PID>` suffix on top of any token substitution.

**(i) CLI value → verbatim, no suffix (OBSERVED).** Launched via the harness with `--listen-on "unix:$WORK/mykitty"`; the background kitty's PID was captured as `$!` = **40579**:

```console
$ "$KITTY" -o allow_remote_control=yes --listen-on "unix:$WORK/mykitty" bash -c 'sleep 600' & KPID=$!
$ echo "before:"; ls -l "$WORK"/mykitty* 2>&1
before:
ls: cannot access '/tmp/kitty_obs_work/run.e6tI6X/mykitty*': No such file or directory
$ wait_sock "$WORK/mykitty" && echo "after (kitty bg pid=$KPID):"; ls -l "$WORK"/mykitty*
after (kitty bg pid=40579):
srwxr-xr-x 1 root root 0 Jul 13 17:47 /tmp/kitty_obs_work/run.e6tI6X/mykitty
$ stat -c '%n type=%F mode=%A' "$WORK/mykitty"
/tmp/kitty_obs_work/run.e6tI6X/mykitty type=socket mode=srwxr-xr-x
```

The live socket, confirmed with both `lsof` and `ss` (owning process is kitty pid **40579**, fd 6 — the PID we captured):

```console
$ lsof -U | grep mykitty
kitty   40579 root    6u  unix 0x0000000000000000      0t0 728475319 /tmp/kitty_obs_work/run.e6tI6X/mykitty type=STREAM (LISTEN)
$ ss -xlp | grep mykitty
u_str LISTEN 0      128    /tmp/kitty_obs_work/run.e6tI6X/mykitty 728475319            * 0    users:(("kitty",pid=40579,fd=6))
$ "$KITTEN" @ --to "unix:$WORK/mykitty" ls | jq -c '[.[].tabs[].windows[]|{id,cmdline}]' ; echo "exit=$?"
[{"id":1,"cmdline":["sleep","600"]}]
exit=0
```

So the CLI value `.../mykitty` is on disk **verbatim** — no PID suffix.

**(ii) CLI value *with an explicit* `{kitty_pid}` token → substituted (OBSERVED — the `main.py:331` nuance).** To prove `{kitty_pid}` is expanded even on the command line, the harness requested `--listen-on "unix:$WORK/tplkitty-{kitty_pid}"` (literal token). The owning kitty was captured as PID **40680**:

```console
$ "$KITTY" -o allow_remote_control=yes --listen-on "unix:$WORK/tplkitty-{kitty_pid}" bash -c 'sleep 600' & KPID=$!
requested --listen-on = unix:/tmp/kitty_obs_work/run.e6tI6X/tplkitty-{kitty_pid}  (contains literal {kitty_pid})
$ ls -l "$WORK"/tplkitty-*
srwxr-xr-x 1 root root 0 Jul 13 17:47 /tmp/kitty_obs_work/run.e6tI6X/tplkitty-40680
$ ss -xlp | grep tplkitty
u_str LISTEN 0      128    /tmp/kitty_obs_work/run.e6tI6X/tplkitty-40680 728482090            * 0    users:(("kitty",pid=40680,fd=6))
```

The literal `{kitty_pid}` became `40680`, which equals the owning kitty PID — the unconditional replacement at `main.py:331`, on the command line. (Only the *automatic* `-<PID>` suffix in (iii) is config‑file‑only.)

**(iii) Config‑file value → automatic `-<PID>` suffix (OBSERVED).** Using a throwaway config file (inside `$WORK`, outside the repo) with `allow_remote_control yes` and `listen_on unix:$WORK/cfgkitty`, launched with `--config` (no `--listen-on` on the command line). The owning kitty was captured as PID **40761**:

```console
$ cat "$WORK/test.conf"
allow_remote_control yes
listen_on unix:/tmp/kitty_obs_work/run.e6tI6X/cfgkitty
$ echo "before:"; ls -l "$WORK"/cfgkitty* 2>&1
before:
ls: cannot access '/tmp/kitty_obs_work/run.e6tI6X/cfgkitty*': No such file or directory
$ "$KITTY" --config "$WORK/test.conf" bash -c 'sleep 300' & KPID=$!
$ ls -l "$WORK"/cfgkitty*
srwxr-xr-x 1 root root 0 Jul 13 17:47 /tmp/kitty_obs_work/run.e6tI6X/cfgkitty-40761
$ ss -xlp | grep cfgkitty
u_str LISTEN 0      128    /tmp/kitty_obs_work/run.e6tI6X/cfgkitty-40761 728347542            * 0    users:(("kitty",pid=40761,fd=6))
```

The configured name `unix:$WORK/cfgkitty` became **`.../cfgkitty-40761`** on disk, and the suffix (`40761`) equals the owning kitty PID — exactly the `main.py:329-330` auto‑suffix.

> **This is almost certainly what happened to you:** you put `listen_on unix:/tmp/something` in `kitty.conf`, kitty created `/tmp/something-<PID>`, and a search for the literal name found nothing. (The kitty docs note this hyphen‑PID append at `kitty/options/definition.py:3000-3016`.)

### 4.2 Reason (b) — abstract sockets leave no filesystem entry

`parse_address_spec` (`kitty/utils.py:502-523`) turns a leading `@` into a NUL‑prefixed Linux **abstract‑namespace** address (`:513-514`):

```python
if address.startswith('@') and len(address) > 1:
    address = '\0' + address[1:]
```

**OBSERVED** with `--listen-on unix:@obskitty` (via the same harness): nothing appears in the filesystem, yet the endpoint is live and usable:

```console
$ "$KITTY" -o allow_remote_control=yes --listen-on 'unix:@obskitty' bash -c 'sleep 600' & KPID=$!
$ echo "filesystem search for @obskitty:"; ls -l /tmp/@obskitty* 2>&1
filesystem search for @obskitty:
ls: cannot access '/tmp/@obskitty*': No such file or directory
$ ss -xl | grep obskitty
u_str LISTEN 0      128                                        @obskitty 728435570            * 0
$ "$KITTEN" @ --to 'unix:@obskitty' ls | jq -c '[.[].tabs[].windows[]|{id,cmdline}]' ; echo "exit=$?"
[{"id":1,"cmdline":["sleep","600"]}]
exit=0
```

The `@obskitty` endpoint has the leading `@` that marks the abstract namespace and **no path on disk** — a `/tmp` search can never find it, yet a client that names `unix:@obskitty` reaches it and `ls` returns cleanly (`exit=0`).

### 4.3 Reason (c) — default‑off gating: usually there is no socket at all

Remote control is off by default: `allow_remote_control` defaults to `no` (`kitty/options/definition.py:2969`) and `listen_on` defaults to `none` (`:3000`). The socket is only created when **both** permit — `kitty/boss.py:364-366`:

```python
if args.listen_on and self.allow_remote_control in ('y', 'socket', 'socket-only', 'password'):
    try:
        listen_fd, self.listening_on = listen_on(args.listen_on)
```

The bind/listen itself is `kitty/boss.py:177-188` (`listen_on()`): `socket.socket(family)` → `atexit.register(remove_socket_file, …)` → `bind` → `listen`.

**OBSERVED — gating by `allow_remote_control`.** Even when `--listen-on` **is** given, no socket is created if `allow_remote_control` is left at its default `no`. The `grep` for the socket returns **exit 1** (no match) — that non‑zero exit is the real, unfabricated evidence, not a hand‑written "not found" line:

```console
$ "$KITTY" --listen-on "unix:$WORK/gatekitty" bash -c 'sleep 60' & KPID=$!    # allow_remote_control left at default 'no'
$ echo "filesystem search:"; ls -l "$WORK"/gatekitty* 2>&1
filesystem search:
ls: cannot access '/tmp/kitty_obs_work/run.e6tI6X/gatekitty*': No such file or directory
$ ss -xl | grep gatekitty ; echo "grep_exit=$?"
grep_exit=1
```

The empty `grep` output with `grep_exit=1` means **no listening socket named `gatekitty` exists** — the `--listen-on` request was ignored because `allow_remote_control` was `no`, precisely the `kitty/boss.py:364` guard.

**OBSERVED — gating by `listen_on`.** The complementary half is shown in §6 Case 5: a kitty started with `allow_remote_control=yes` but **no** `listen_on`/`--to` creates no socket, and a window inside it therefore sees an **empty** `KITTY_LISTEN_ON` (`KITTY_LISTEN_ON=[]`) — the child‑env consequence of `kitty/child.py:246-249` popping the variable when `boss.listening_on` is falsy. Together, the two halves confirm that **both** `allow_remote_control` **and** `listen_on` must be set before any socket appears.

> **A note on finding the live PID.** `pgrep -x kitty | head -1` is unreliable here: repeated launch/kill cycles leave `[kitty] <defunct>` zombies (reparented to PID 1) that share the process name. The authoritative owner of a socket is what `lsof -U`/`ss -xlp` report — e.g. pid `40579` for `.../mykitty` in §4.1, captured independently as the launch‑time `$!`. Every PID quoted in this document is the socket's `ss -xlp` owner, cross‑checked against the captured `$!`.


---

## 5. OBJ‑4 — End‑to‑end over the socket: wire bytes, parse/route, JSON, logging

**Direct answer.** The request and response are a **symmetric DCS envelope** `<ESC>P@kitty-cmd<JSON><ESC>\` (`<ESC>` = byte `0x1b`), where the JSON envelope carries `cmd`, `version`, `no_response`, `kitty_window_id`, `payload` (`docs/rc_protocol.rst:8,14-20`). On the server the bytes are decoded by **`parse_cmd`** (`kitty/remote_control.py:56`), dispatched by **`handle_cmd`** (`:213`) → **`command_for_name('ls')`** (`kitty/rc/base.py:449`) → **`LS.response_from_kitty`** (`kitty/rc/ls.py:48`), which builds the tree with `boss.list_os_windows(...)` (`:57`) and returns `json.dumps(data, indent=2, sort_keys=True)` (`:76`); the result is wrapped as `{'ok': True, 'data': <json-string>}` (`kitty/remote_control.py:258-260`) and framed back in the same DCS envelope. **Logging is error‑only**: a successful call logs nothing; a malformed one emits one `log_error` line.

**One consolidated instance for all of §5 (OBSERVED).** So that every value below — socket path, owning PID, wire bytes, JSON, and log lines — is mutually consistent, §5.1–§5.5 were all captured against a **single** instance launched by the harness. Its background PID was captured as `$!` = **42106**, and it listened on `unix:/tmp/kitty_obs_work/run.XgZliG/kitty.sock`; the socket was **absent before** and **present after**, owned by pid 42106:

```console
$ echo "before:"; ls -l "$WORK/kitty.sock" 2>&1
before:
ls: cannot access '/tmp/kitty_obs_work/run.XgZliG/kitty.sock': No such file or directory
$ "$KITTY" -o allow_remote_control=yes --listen-on "unix:$WORK/kitty.sock" bash -c 'sleep 3600' 2>"$WORK/kitty.log" & KPID=$!
$ wait_sock "$WORK/kitty.sock" && echo "after (kitty bg pid=$KPID):"; ls -l "$WORK/kitty.sock"
after (kitty bg pid=42106):
srwxr-xr-x 1 root root 0 Jul 13 17:56 /tmp/kitty_obs_work/run.XgZliG/kitty.sock
$ lsof -U | grep kitty.sock
kitty   42106 root    6u  unix 0x0000000000000000      0t0 728690455 /tmp/kitty_obs_work/run.XgZliG/kitty.sock type=STREAM (LISTEN)
$ ss -xlp | grep kitty.sock
u_str LISTEN 0      128    /tmp/kitty_obs_work/run.XgZliG/kitty.sock 728690455            * 0    users:(("kitty",pid=42106,fd=6))
```

The window it opened runs `sleep 3600` as child pid **42182** (this is the `pid`/`cmdline` that appears in the `ls` JSON below). All §5 output uses this instance, version banner `kitty 0.35.2`.

### 5.1 The real `ls` JSON response (OBSERVED, complete & unedited)

Command and complete output (`kitten @ … ls`, `EXIT=0`):

```console
$ "$KITTEN" @ --to "unix:$WORK/kitty.sock" ls ; echo "exit=$?"
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
              "3600"
            ],
            "columns": 71,
            "created_at": 1783965378071409170,
            "cwd": "/tmp/kitty_obs_work",
            "env": {
              "COLORTERM": "truecolor",
              "DISPLAY": ":99",
              "GALLIUM_DRIVER": "llvmpipe",
              "HOME": "/root",
              "HOSTNAME": "9af7dc1b7700",
              "KITTY_INSTALLATION_DIR": "/tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f",
              "KITTY_LISTEN_ON": "unix:/tmp/kitty_obs_work/run.XgZliG/kitty.sock",
              "KITTY_PID": "42106",
              "KITTY_PUBLIC_KEY": "1:a0J8LSF$PtKFfn)oo_Iq!cdVXfKy%#=5^0JR;uk%",
              "KITTY_SHELL_INTEGRATION": "enabled",
              "KITTY_WINDOW_ID": "1",
              "LC_CTYPE": "C.UTF-8",
              "LIBGL_ALWAYS_SOFTWARE": "1",
              "OLDPWD": "/tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f",
              "PATH": "/tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f/kitty/launcher:/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
              "PWD": "/tmp/kitty_obs_work",
              "PYTEST_ADDOPTS": "--tb=short -v --continue-on-collection-errors --reruns=3",
              "SHLVL": "1",
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
                  "3600"
                ],
                "cwd": "/tmp/kitty_obs_work",
                "pid": 42182
              }
            ],
            "id": 1,
            "is_active": true,
            "is_focused": true,
            "is_self": false,
            "last_cmd_exit_status": 0,
            "last_reported_cmdline": "",
            "lines": 22,
            "pid": 42182,
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
exit=0
```

This matches the tree shape documented at `docs/remote-control.rst:84-120` and — authoritatively for the field set — the `ls` handler's own `desc` docstring (`kitty/rc/ls.py:24-32`), which explicitly documents the `environment` of the process (`:29`) and the `is_self` parameter (`:30`). The top level is a **list of OS windows**; each has `id` + `tabs`; each tab has `id`, `title`, `layout`, and `windows`; each window has `id`, `title`, `cwd` (current working directory), `pid`, `cmdline`, `env` (the process `environment`), and `is_self`. Here `is_self` is **`false`** because the `kitten @ ls` client ran *outside* the listed window (it dialed the socket from a separate process); contrast this with §6 where an in‑window run yields `is_self: true`.

> **Security note (non‑secret).** The `KITTY_PUBLIC_KEY` value above (prefix `1:`) is kitty's **public** X25519 key — it is exported into every child so clients can *encrypt to* kitty; it is ephemeral per‑instance and is **not** a credential. Only a *private* key would be sensitive; none appears in any captured output.

### 5.2 The raw on‑the‑wire bytes (OBSERVED — captured from the *real* client)

The bytes below are **the exact bytes the real `kitten @ ls` client sent and received** — not a hand‑crafted `printf`. The in‑repo `socat` recipe (`docs/rc_protocol.rst:35-42`) documents how to *decode* an RC frame, but to capture the **canonical** client's own bytes the harness inserted a transparent `tee`‑style proxy in front of the socket: the real client dialed the proxy, which forwarded to kitty and copied each direction to `request.bin`/`response.bin`. The client exited 0, so these are a genuine round trip.

> **Why this matters (finding on canonical evidence).** The real client sends the *remote‑control protocol* version `[0,26,0]` (from `var ProtocolVersion [3]int = [3]int{0, 26, 0}` at `tools/cmd/at/main.go:33`) and includes a `"payload":{}` object — **not** the release version `[0,35,2]`. A `printf` that hand‑types `[0,35,2]` also happens to work (it is still `<=` the instance version, so it passes the guard), but it is **non‑canonical**; the capture here is the actual client frame.

**The request — 58 bytes, `<ESC>P@kitty-cmd{…}<ESC>\`:**

```console
$ wc -c < request.bin
58
$ od -c request.bin
0000000 033   P   @   k   i   t   t   y   -   c   m   d   {   "   c   m
0000020   d   "   :   "   l   s   "   ,   "   v   e   r   s   i   o   n
0000040   "   :   [   0   ,   2   6   ,   0   ]   ,   "   p   a   y   l
0000060   o   a   d   "   :   {   }   } 033   \
0000072
$ od -An -tx1 request.bin
 1b 50 40 6b 69 74 74 79 2d 63 6d 64 7b 22 63 6d
 64 22 3a 22 6c 73 22 2c 22 76 65 72 73 69 6f 6e
 22 3a 5b 30 2c 32 36 2c 30 5d 2c 22 70 61 79 6c
 6f 61 64 22 3a 7b 7d 7d 1b 5c
```

Decoded, the JSON envelope is `{"cmd":"ls","version":[0,26,0],"payload":{}}`. `033` is octal for `0x1b` (ESC); the leading bytes `1b 50 40 6b 69 74 74 79 2d 63 6d 64` are `ESC P @kitty-cmd` and the trailing `1b 5c` are `ESC \` — exactly the constants the Go client uses at `tools/cmd/at/socket_io.go:82-83` (`cmd_escape_code_prefix = "\x1bP@kitty-cmd"`, `cmd_escape_code_suffix = "\x1b\\"`) and the Python client at `kitty/remote_control.py:308-310` (`encode_send`).

**The response — same envelope, 3655 bytes.** First 16 bytes and last 2 bytes prove the framing; the `od -c` head shows the envelope beginning `{"ok": true, "data": "[\n …` (note the `"` after `data:` and the escaped `\n` — `data` is itself a **JSON string**):

```console
$ wc -c < response.bin
3655
$ head -c 16 response.bin | od -An -tx1
 1b 50 40 6b 69 74 74 79 2d 63 6d 64 7b 22 6f 6b
$ head -c 40 response.bin | od -c
0000000 033   P   @   k   i   t   t   y   -   c   m   d   {   "   o   k
0000020   "   :       t   r   u   e   ,       "   d   a   t   a   "   :
0000040       "   [   \   n           {
0000050
$ tail -c 2 response.bin | od -An -tx1
 1b 5c
```

`1b 50 40 6b 69 74 74 79 2d 63 6d 64 7b 22 6f 6b` = `ESC P @kitty-cmd{"ok` and `1b 5c` = `ESC \`. This response framing is produced by `encode_response_for_peer` (`kitty/remote_control.py:52-53`). The outer envelope is `{ok, data}` where `data` is a **string** — the wrapping that `handle_cmd` applies (`kitty/remote_control.py:258-260`) around the string `ls` produces via `json.dumps(...)` (`kitty/rc/ls.py:76`). Re‑parsing that inner `data` string (e.g. `jq '.data|fromjson'`, or strip the 12‑byte prefix + 2‑byte suffix with the `docs/rc_protocol.rst:35-42` `awk` recipe) yields the tree shown **complete in §5.1** for this same instance.

**The socketless case writes the *same* frame to the PTY (OBSERVED — actual PTY bytes).** To confirm the earlier claim that the TTY transport carries the identical DCS frame, the harness ran the real client under a pseudo‑terminal (`script`) with `KITTY_LISTEN_ON` unset, and dumped exactly what the client wrote to the controlling PTY. The `@kitty-cmd` frame it emitted is **byte‑for‑byte identical** to the socket request above:

```console
$ python3 -c 'raw=open("ptycap.raw","rb").read(); i=raw.find(b"\x1bP@kitty-cmd"); j=raw.find(b"\x1b\\",i); f=raw[i:j+2]; print("len",len(f)); print(f.hex(" "))'
len 58
1b 50 40 6b 69 74 74 79 2d 63 6d 64 7b 22 63 6d 64 22 3a 22 6c 73 22 2c 22 76 65 72 73 69 6f 6e 22 3a 5b 30 2c 32 36 2c 30 5d 2c 22 70 61 79 6c 6f 61 64 22 3a 7b 7d 7d 1b 5c
```

That is the same 58‑byte `\033P@kitty-cmd{"cmd":"ls","version":[0,26,0],"payload":{}}\033\\`. So whether it travels over a socket or over the PTY, the on‑the‑wire request frame is identical; only the *carrier* differs. (Under a bare `script` PTY there is no kitty to answer, so the client writes its request frame and then exits without a response — which is why this block shows the request only; the socket capture above shows the full round trip.)

### 5.3 The Python parse/route chain — where `ls` gets handled (OBSERVED)

This is the part you asked about directly: *"where in the Python code the incoming command gets parsed and routed to the `ls` handler."* To **observe** (not merely read) the chain, a temporary, deletable instrumentation shim was placed **outside the repo** at `/tmp/kitty_obs_work/shim/sitecustomize.py` and loaded via `PYTHONPATH`. It installs a `sys.meta_path` finder that, *after* each target module imports, wraps the real functions so they append a line to a log when entered and then call the originals — **the canonical code path still runs**; nothing is bypassed or hand‑crafted, and no repository file is edited. Reproducible invocation (socket ingress):

```bash
SHIM=/tmp/kitty_obs_work/shim; LOG="$WORK/shim_socket.log"
PYTHONPATH="$SHIM" KITTY_OBS_SHIM_LOG="$LOG" \
  "$KITTY" -o allow_remote_control=yes --listen-on "unix:$WORK/instr.sock" bash -c 'sleep 300' & KPID=$!
wait_sock "$WORK/instr.sock"
"$KITTEN" @ --to "unix:$WORK/instr.sock" ls >/dev/null ; echo "kitten_exit=$?"   # -> kitten_exit=0
cat "$LOG"
```

The log it produced (OBSERVED — verbatim):

```text
### shim installed (meta_path finder) at interpreter start
### patched kitty.remote_control (parse_cmd, handle_cmd, command_for_name)
parse_cmd              [kitty/remote_control.py:56]  -> cmd='ls' version=[0, 26, 0]
handle_cmd             [kitty/remote_control.py:213] cmd='ls' version=[0, 26, 0] (guard @:218)
command_for_name       [kitty/rc/base.py:449]        cmd_name='ls' -> import_module kitty.rc.ls
### patched kitty.rc.ls (LS.response_from_kitty)
LS.response_from_kitty [kitty/rc/ls.py:48]           -> boss.list_os_windows [ls.py:57] + json.dumps [ls.py:76]
```

So the observed core chain is:

```
parse_cmd  ->  handle_cmd  ->  command_for_name('ls')  ->  LS.response_from_kitty
[rc.py:56]     [rc.py:213]     [rc/base.py:449]            [rc/ls.py:48]
```

**Both ingress paths reach the identical core (OBSERVED).** The same shim was run a second time against a **socketless in‑window** `kitten @ ls` (TTY ingress, `TTY_KITTEN_EXIT=0`). Its log (`shim_tty.log`) was **byte‑for‑byte identical** to the socket log above — the same four `parse_cmd → handle_cmd → command_for_name → LS.response_from_kitty` lines with `version=[0, 26, 0]` — confirming both transports converge on the same parse/route core.

Two observed nuances worth recording:

- **The real client sends RC protocol version `[0,26,0]`** — the *remote‑control protocol* version, **not** the `0.35.2` release version. It is `<=` the instance version, so it passes the guard `if tuple(v)[:2] > version[:2]:` at `kitty/remote_control.py:218`. *(INFERRED, not exercised: a request whose major/minor is newer than the instance would be rejected there — `docs/rc_protocol.rst:22-25`.)*
- **Dispatch is name‑based and dynamic** — `command_for_name` does `cmd_name.replace('-', '_')` (`kitty/rc/base.py:451`) then `import_module(f'kitty.rc.{cmd_name}')` (`:453`). `kitty.rc.ls` is imported **lazily**, on the first `ls` request. **This is exactly the extension point a new command plugs into** (see §8).

**How each transport reaches this core (ingress).** The parse/route core above is shared; only the ingress differs. The ingress steps below are **SOURCE‑VERIFIED** (read from the code; the individual C/boss hops were not separately instrumented), while the shared core they feed into is OBSERVED above from *both* transports:

- **Socket ingress (SOURCE‑VERIFIED):** `boss.peer_message_received` (`kitty/boss.py:776`) → `_handle_remote_command` (`:590`) → `_execute_remote_command` (`:701`) → `from .remote_control import handle_cmd`.
- **TTY ingress (SOURCE‑VERIFIED):** the C DCS parser dispatches `@kitty-cmd` at `kitty/vt-parser.c:603` (only after matching the `kitty-` marker) → `kitty/window.py:1279-1280` `handle_remote_cmd` → `get_boss().handle_remote_cmd(...)` → `kitty/boss.py:849` `handle_remote_cmd` → `_handle_remote_command` (`:590`).

### 5.4 The Go client's receive → decode → print path (the return leg)

The response DCS frame from §5.2 travels back to the `kitten` client, which decodes and either **prints the data to stdout** or **reports an error on stderr with a non‑zero exit**. This is the leg the earlier trace omitted. The receive chain is **SOURCE‑VERIFIED**:

- **Read the frame off the transport.** For a socket, `read_response_from_conn` (`tools/cmd/at/socket_io.go:53-58`) feeds bytes to an `EscapeCodeParser` whose `HandleDCS` strips the `@kitty-cmd` prefix and yields the JSON envelope. For the TTY, `OnRCResponse` (`tools/cmd/at/tty_io.go:154`) sets the serialized response from the DCS payload read off the terminal.
- **Decode.** `get_response` (`tools/cmd/at/main.go:223`) hands the serialized bytes to `json.Unmarshal` (`:247`), producing a `Response` struct with fields `Ok`, `Data`, `Error`, and `Traceback`.
- **Print or fail.** `send_rc_command` inspects `response.Ok` (`tools/cmd/at/main.go:284-297`):
  - if **`!Ok`**, it prints `response.Traceback` to **stderr** when non‑empty (`:286`) and returns `response.Error` (`:288`), giving a non‑zero exit;
  - otherwise it prints the payload with `fmt.Println(strings.TrimRight(response.Data.as_str, "\n \t"))` (`:297`).

**OBSERVED — success (`Ok=true`).** The complete JSON tree shown in **§5.1** *is* exactly this printed `response.Data.as_str`; that run exited `0`. So the stdout you see from `kitten @ ls` is the Go client echoing the decoded `data` string.

**OBSERVED — error (`Ok=false`).** Asking `ls` to match a malformed selector makes the **server** reject it and return `ok:false`; the Go client's `!Ok` branch then surfaces it on stderr with exit 1 (stdout empty):

```console
$ "$KITTEN" @ --to "unix:$WORK/kitty.sock" ls --match 'nonsense:1' ; echo "exit=$?"
Error: nonsense is not a recognized location in nonsense:1
exit=1
```

The message text originates **server‑side** — `kitty/search_query_parser.py:141` raises `"{a} is not a recognized location in {tt}"` — so this genuinely exercises the *receive‑and‑decode* path: the client unmarshalled a `Response{Ok:false, Error:"nonsense is not a recognized location in nonsense:1"}` and printed `response.Error` (the `Error:` prefix is added by the CLI framework). No `Traceback` line appeared because the server produced a *clean* error rather than an unhandled exception; the traceback branch at `main.go:286` is **SOURCE‑VERIFIED** (it fires only when the server includes a `Traceback`, which a clean validation error does not).

### 5.5 Logging behavior — error‑only (OBSERVED, both states)

kitty's RC path logs **nothing on success** and exactly one `log_error` line on a parse failure. This was captured against the **same consolidated instance** (pid **42106**), whose stderr was redirected to `$WORK/kitty.log` at launch. Crucially, the success test **first asserts the client exited 0 and returned valid JSON**, and only then compares the RC‑relevant log‑line count — otherwise a silently‑failed call that produced no output would masquerade as a "no log on success":

```console
# SUCCESS — assert a real, valid round trip FIRST, then check the log
$ before=$(grep -c 'remote command' "$WORK/kitty.log")
$ out=$("$KITTEN" @ --to "unix:$WORK/kitty.sock" ls) ; ec=$?
$ echo "$out" | jq -e . >/dev/null 2>&1 && valid=VALID || valid=INVALID
$ after=$(grep -c 'remote command' "$WORK/kitty.log")
$ echo "kitten_exit=$ec response=$valid rc_log_before=$before rc_log_after=$after"
kitten_exit=0 response=VALID rc_log_before=0 rc_log_after=0
```

Because `kitten_exit=0` **and** `response=VALID`, the unchanged RC count (`0 → 0`) is a genuine "no log on success". Now the error case — a malformed frame over the socket:

```console
# ERROR — a malformed frame emits exactly one new RC log line
$ printf '\033P@kitty-cmd{bad json}\033\\' | socat - "unix:$WORK/kitty.sock" >/dev/null 2>&1
$ grep -c 'remote command' "$WORK/kitty.log"
1
$ cat "$WORK/kitty.log"
[0.175] Failed to open systemd user bus with error: No medium found
[1.177] Failed to parse JSON payload of remote command, ignoring it
```

- **Success:** with the exit‑0 + valid‑JSON precondition satisfied, the RC‑relevant line count stayed `0 → 0` — **no log on success**.
- **Error:** the malformed frame added **exactly one** RC line, `Failed to parse JSON payload of remote command, ignoring it` — `log_error(...)` at `kitty/remote_control.py:62` inside `parse_cmd`. The other line, `Failed to open systemd user bus …`, is the pre‑existing D‑Bus startup noise from §2; it is **not** an RC line, which is precisely why the RC‑relevant count went `0 → 1` (not `1 → 2`). Counting only RC‑relevant lines (`grep -c 'remote command'`) rather than total lines avoids conflating the two.
- **Other `log_error` sites exist but were *not* triggered in these runs (SOURCE‑VERIFIED, not observed):** a structurally broken peer frame would log `Malformatted remote control message received from peer, ignoring` at `kitty/boss.py:792` (`peer_message_received`), and a command that raises while being parsed on the boss side would log `Failed to parse remote command with error: {e}` at `kitty/boss.py:605` (`_handle_remote_command`). `log_error` itself is defined at `kitty/utils.py:130`. They are listed for completeness and labeled SOURCE‑VERIFIED because they were read from the code, not reproduced here.


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

**Two of the three variables are exported unconditionally; only one is conditional (SOURCE‑VERIFIED, and corroborated by the env dumps in §5.1 and §6.2‑6.4):**

- **`KITTY_PID`** (`child.py:244`) and **`KITTY_PUBLIC_KEY`** (`child.py:245`) are set on **every** child, always — there is no guard around them.
- **`KITTY_LISTEN_ON`** (`child.py:246-249`) is exported **only when `self.add_listen_on_env_var and boss.listening_on`** is truthy (i.e. kitty is actually listening on a socket); otherwise the `else` branch **pops** it so it is guaranteed absent. This is exactly why Case 5 below shows `KITTY_LISTEN_ON=[]` (empty) while `KITTY_PID` is still populated.

The **consumer** of `KITTY_LISTEN_ON` is the Go client, `tools/cmd/at/main.go:371-373` — it becomes the default target only when `--to` was not given:

```go
if rc_global_opts.To == "" {
    rc_global_opts.To = os.Getenv("KITTY_LISTEN_ON")
    global_options.to_address_is_from_env_var = true
```

**`KITTY_PUBLIC_KEY` has its own, separate consumers (SOURCE‑VERIFIED)** and is read **only when password‑based encryption is requested** — it is *not* touched by a plaintext `kitten @ ls`. The Go client reads it in `get_pubkey` (`tools/cmd/at/main.go:72-94`): it falls back to `os.Getenv("KITTY_PUBLIC_KEY")` (`:74`), splits the `1:` version prefix (`:80`), checks the version, and base85‑decodes the key. The Python client mirrors this in `get_pubkey` (`kitty/remote_control.py:516-524`): `os.environ.get('KITTY_PUBLIC_KEY', '')` (`:517`), `split(':', 1)` (`:520`), `b85decode` (`:524`). Both raise "Password usage requested but KITTY_PUBLIC_KEY environment variable is not available" if it is missing — confirming it is consumed on the encryption path only (see §7). This closes the loop the question raised about env vars "I can't trace back to where they're actually used": `KITTY_PID` → shell‑integration local/SSH detection; `KITTY_LISTEN_ON` → the RC client's default target; `KITTY_PUBLIC_KEY` → the RC encryption layer.

`KITTY_SHELL_INTEGRATION` is a *separate* variable again, produced by `kitty/shell_integration.py:223` (`env['KITTY_SHELL_INTEGRATION'] = ksi`), and consumed **only for reading** by the shell scripts.

**Proof the shell scripts do not set `KITTY_LISTEN_ON` (OBSERVED):** a recursive grep across the whole `shell-integration/` tree finds nothing (grep exits `1` = no match):

```console
$ grep -rn "KITTY_LISTEN_ON" shell-integration/ ; echo "grep_exit=$?"
grep_exit=1
```

The scripts only ever **read** the kitty‑set variables — e.g. `shell-integration/bash/kitty.bash:215` (`if [[ -z "$KITTY_PID" ]]; then`), `shell-integration/zsh/kitty-integration:249` (`if [[ -n "$KITTY_PID" ]]; then`), and `shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish:29` (`test -n "$KITTY_SHELL_INTEGRATION" || return 0`). They use `KITTY_PID` only for local‑vs‑SSH detection; none of them creates the RC target.

> **In‑window harness (Cases 4‑6, shared `WORK=/tmp/kitty_obs_work/run.4VK8TB`).** Each case launches a kitty whose window runs a throwaway inner script (`caseN_inner.sh`, created under `WORK`, deleted on exit) that (1) dumps the four env vars **as the window's child sees them**, (2) resolves `which kitten` to prove the real launcher is used, (3) runs the **bare** `kitten @ ls` (no `--to`) and records its exit code, (4) validates the returned JSON, (5) projects the window list to `id,is_self,cmdline`, and (6) prints the listening socket **scoped to this kitty's own PID** (`ss -x` filtered to the owner) so a stray socket from another instance cannot be mistaken for this one. Output is shown complete and unedited.

### 6.2 Case 4 — in‑window, socket configured → uses the same socket (OBSERVED)

Launched with `-o allow_remote_control=yes --listen-on "unix:$WORK/winkitty"`; the window's inner script then runs bare `kitten @ ls` (no `--to`):

```console
--- env inside window (case4) ---
KITTY_PID=[41386]
KITTY_LISTEN_ON=[unix:/tmp/kitty_obs_work/run.4VK8TB/winkitty]
KITTY_SHELL_INTEGRATION=[enabled]
KITTY_PUBLIC_KEY_prefix=[1:...(len=42)]
which kitten -> /tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f/kitty/launcher/kitten
--- run REAL: kitten @ ls  (no --to) ---
KITTEN_EXIT=0
--- response validity ---
RESPONSE_JSON=VALID
--- projected windows (id,is_self,cmdline) ---
[{"id":1,"is_self":true,"cmdline":["bash","/tmp/kitty_obs_work/run.4VK8TB/case4_inner.sh"]}]
--- listener scoped to THIS kitty pid 41386 ---
u_str LISTEN 0      128    /tmp/kitty_obs_work/run.4VK8TB/winkitty 728371053            * 0    users:(("kitty",pid=41386,fd=6))
```

`KITTY_LISTEN_ON` is `unix:/tmp/kitty_obs_work/run.4VK8TB/winkitty`, the target resolves from it, the call succeeds (`KITTEN_EXIT=0`, `RESPONSE_JSON=VALID`), `is_self` is now **`true`** (the client ran inside the listed window), and a real socket owned by **this** instance (pid `41386`, fd `6`) exists. Branch taken: `to_network="unix"` → `do_socket_io` (`tools/cmd/at/main.go:280`).

### 6.3 Case 5 — in‑window, **no** socket → DCS‑over‑PTY (OBSERVED)

Launched with `-o allow_remote_control=yes` but **no** `--listen-on`; the window's inner script runs bare `kitten @ ls`:

```console
--- env inside window (case5) ---
KITTY_PID=[41484]
KITTY_LISTEN_ON=[]
KITTY_SHELL_INTEGRATION=[enabled]
KITTY_PUBLIC_KEY_prefix=[1:...(len=42)]
which kitten -> /tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f/kitty/launcher/kitten
--- run REAL: kitten @ ls  (no --to) ---
KITTEN_EXIT=0
--- response validity ---
RESPONSE_JSON=VALID
--- projected windows (id,is_self,cmdline) ---
[{"id":1,"is_self":true,"cmdline":["bash","/tmp/kitty_obs_work/run.4VK8TB/case5_inner.sh"]}]
--- listener scoped to THIS kitty pid 41484 ---
(no listening socket owned by pid 41484)
```

Here `KITTY_LISTEN_ON` is **empty** (the conditional export in `child.py:246-249` popped it because this instance is not listening), and — scoped strictly to this instance's PID — **there is no listening socket owned by pid `41484`**. Yet `kitten @ ls` still succeeds (`KITTEN_EXIT=0`, `RESPONSE_JSON=VALID`, `is_self:true`). Because the target string is empty (`To=""` → `to_network=""`), the client takes the `do_tty_io` branch (`tools/cmd/at/main.go:280`) and writes the DCS escape to the controlling PTY (`tools/cmd/at/tty_io.go:79-82`); kitty parses it out of the terminal stream at `kitty/vt-parser.c:603` and routes it via `kitty/window.py:1279`. **This demonstrates the socketless TTY transport**: with no RC socket for this instance and no `--to`, the command still reaches kitty over the window's own PTY. (The scoping to pid `41484` matters — a naive unscoped `ss` might list an RC socket belonging to *another* kitty on the same machine, which would be irrelevant to this window.)

### 6.4 Case 6 — disambiguation: `shell_integration=disabled` still works (OBSERVED, required by R5)

To prove shell integration is not the enabler, launch with shell integration **off** (`-o shell_integration=disabled`) but remote control **on** (`-o allow_remote_control=yes --listen-on "unix:$WORK/nosikitty"`):

```console
--- env inside window (case6) ---
KITTY_PID=[41580]
KITTY_LISTEN_ON=[unix:/tmp/kitty_obs_work/run.4VK8TB/nosikitty]
KITTY_SHELL_INTEGRATION=[]
KITTY_PUBLIC_KEY_prefix=[1:...(len=42)]
which kitten -> /tmp/blitzy/kitty/blitzy-5272820a-f82d-4851-9076-f4ffb122d22c_31872f/kitty/launcher/kitten
--- run REAL: kitten @ ls  (no --to) ---
KITTEN_EXIT=0
--- response validity ---
RESPONSE_JSON=VALID
--- projected windows (id,is_self,cmdline) ---
[{"id":1,"is_self":true,"cmdline":["bash","/tmp/kitty_obs_work/run.4VK8TB/case6_inner.sh"]}]
--- listener scoped to THIS kitty pid 41580 ---
u_str LISTEN 0      128    /tmp/kitty_obs_work/run.4VK8TB/nosikitty 728558053            * 0    users:(("kitty",pid=41580,fd=6))
```

`KITTY_SHELL_INTEGRATION` is now **empty** (`SI=[]`, shell integration genuinely off), yet `KITTY_PID` and `KITTY_LISTEN_ON` are **still set** (`child.py:244-249` exports them regardless of shell integration), and `kitten @ ls` still works (`KITTEN_EXIT=0`, `RESPONSE_JSON=VALID`) over the socket owned by pid `41580`. **Conclusion: shell integration is not what makes RC work** — the enabler is kitty's env export (`kitty/child.py:244-249`) plus the two RC transports.

> **Gotcha you may hit:** the correct value is `shell_integration=disabled`, **not** `shell_integration=no`. `no` is an invalid token that kitty silently ignores, leaving shell integration *enabled*; the default is `enabled` (`kitty/options/definition.py:3141`).

### 6.5 Resolution of your either/or

- **Same socket?** Yes — *when a socket is configured*, the in‑window client resolves `KITTY_LISTEN_ON` and dials that very socket (Case 4).
- **TTY magic through the pty?** Yes — *when no socket exists*, the client writes the `@kitty-cmd` DCS escape to the controlling PTY and kitty parses it from the terminal stream (Case 5).
- **Either way**, the target/capability originates from kitty's environment export in `kitty/child.py`, **not** from the shell‑integration scripts (Case 6 + the empty grep).


---

## 7. Security model — authorization modes & encryption

Remote control is a genuine security boundary: whatever can send a command gains the capabilities quoted in the §4 warning (send text, read window contents, spawn/close windows, "even over SSH"). The **authorization modes** and **gating logic** below are **SOURCE‑VERIFIED** (read from the option definition and the boss dispatcher; the `yes`/default‑`no` and socket‑vs‑TTY behaviors were also exercised in §4‑§6). The **encryption scheme** in §7.3 is **INFERRED‑from‑spec** — it was *not* run here because no `remote_control_password` was configured; it is included for completeness and nothing about it was altered (AAP §0.5.2).

### 7.1 `allow_remote_control` modes (SOURCE‑VERIFIED) — choose least privilege

`opt('allow_remote_control', 'no', choices=('password','socket-only','socket','no','n','false','yes','y','true'))` (`kitty/options/definition.py:2969`). The meaning of each accepted value, verbatim from its `long_text` (`definition.py:2977-2999`):

| Value | Behavior | Trust granted |
|-------|----------|---------------|
| `no` / `n` / `false` (**default**) | "Remote control is completely disabled." No socket is ever created (§4.3). | none |
| `yes` / `y` / `true` | "Remote control requests are always accepted." **No authentication** on either transport. | full, unauthenticated |
| `password` | "Remote control requests received over both the TTY device and the socket are confirmed based on passwords" (`remote_control_password`). | password‑gated on both |
| `socket-only` | "Requests received over a socket are accepted unconditionally. Requests received over the TTY are denied." | socket callers only |
| `socket` | "Requests received over a socket are accepted unconditionally. Requests received over the TTY are confirmed based on password." | socket unconditional; TTY password‑gated |

**Least‑privilege guidance (derived from the above):** the `yes` used throughout §4‑§6 is appropriate **only** for disposable, isolated instances like this container. For real use: prefer `socket-only` when only trusted local processes with filesystem access to the socket should drive kitty (this also **blocks** the DCS‑over‑TTY path, so a program merely printing escape codes into your terminal cannot control it); use `password`/`socket` with a `remote_control_password` when the TTY path is needed; keep the socket a `unix:` path with restrictive permissions and **avoid `tcp:`** unless the port is firewalled, because a TCP listener is reachable by anything that can connect.

### 7.2 Authorization gating in the dispatcher (SOURCE‑VERIFIED)

`_handle_remote_command` (`kitty/boss.py:594-640`) applies the mode before any command runs:

```python
if not window_has_remote_control and not is_fd_peer:
    if self.allow_remote_control == 'n':
        return {'ok': False, 'error': 'Remote control is disabled'}
    if self.allow_remote_control == 'socket-only' and not from_socket:
        return {'ok': False, 'error': 'Remote control is allowed over a socket only'}
```

It then computes `allowed_unconditionally` (`boss.py:623-628`): true when `allow_remote_control == 'y'` (`:624`), or the caller is a socket peer and the mode is `socket`/`socket-only` (`:625`), or a per‑window/per‑fd password check passes (`:626-627`). Otherwise per‑command permission falls to `is_cmd_allowed` (`boss.py:633`) and, for password mode, `PasswordAuthorizer` (`kitty/remote_control.py:134,177`). Because our runs used `allow_remote_control=yes`, the `:624` `== 'y'` branch is what admitted every `ls` above — no password path was taken.

### 7.3 Encryption scheme (INFERRED‑from‑spec — NOT exercised)

The plaintext `kitten @ ls` runs above are **not** encrypted: the real socket request and the real PTY frame captured in **§5.2** are cleartext DCS + JSON (`{"cmd":"ls","version":[0,26,0],"payload":{}}` with no `iv`/`tag`/`encrypted` fields). Encryption engages **only** when a `remote_control_password` is involved, and works as follows per the in‑repo spec (`docs/rc_protocol.rst:57-84`): the command is encrypted using the recipient's **public key** taken from `KITTY_PUBLIC_KEY` (protocol prefix `1:`, `:60-61` — the very variable whose consumers are traced in §6.1); key agreement is **X25519** ECDH (`:64`); a **time‑based nonce** is added and commands whose `timestamp` is **more than 5 minutes** from now are **rejected** (`:66-68`); the command is then sealed with **AES‑256‑GCM** (authenticated) (`:69`) and the fields (`iv`, `tag`, `pubkey`, `encrypted`) are base85‑encoded into the JSON envelope. Altering encryption/auth is **out of scope** (AAP §0.5.2).

---

## 8. How to add a NEW remote‑control command (the pattern — NOT implemented here)

This task **does not** add a command (AAP §0.5.2); this section only *equips* you, grounded in how `ls` is built. A new command follows the exact `ls` pattern — but note the **asymmetry between the two sides**: the **Python server** discovers your command automatically, whereas the **Go `kitten` client** needs a *generated* file and a **rebuild**.

**1. Author the Python command module — `kitty/rc/<name>.py`** defining `class <Name>(RemoteCommand)` (base class in `kitty/rc/base.py`), implementing the two halves that `ls` demonstrates:
   - **client‑request builder** `message_to_kitty(self, global_opts, opts, args)` — builds the request payload. `ls` returns `{'all_env_vars', 'match', 'match_tab'}` (`kitty/rc/ls.py:45`).
   - **server‑side handler** `response_from_kitty(self, boss, window, payload_get)` — does the work and returns data. `ls` calls `boss.list_os_windows(...)` (`:57`) and returns `json.dumps(data, indent=2, sort_keys=True)` (`:76`). A JSON‑string data field is fine.
   - end the module with a singleton instance, as `ls` does: `ls = LS()` (`kitty/rc/ls.py:79`).

**2. Server‑side routing is automatic and name‑based (no registry edit).** `command_for_name('<name>')` maps `cmd-name` → `cmd_name` (`kitty/rc/base.py:451`) and dynamically `import_module(f'kitty.rc.{cmd_name}')` (`:453`). Dropping the module in `kitty/rc/` is enough for the **running kitty** to parse and dispatch `@kitty-cmd{"cmd":"<name>",…}`.

**3. The Go `kitten` client is NOT automatic — it is code‑generated, then compiled in.** The `kitten @ <name>` *subcommand* does not exist until a Go file is generated for it. Each command has a `tools/cmd/at/cmd_<name>_generated.go` whose header reads `// Code generated by go_code.py; DO NOT EDIT.` (e.g. `tools/cmd/at/cmd_ls_generated.go:1`) and whose `func init()` registers the subcommand: `register_at_cmd(setup_ls)` (`cmd_ls_generated.go:148-149`), appending to the client's command table via `register_at_cmd` (`tools/cmd/at/main.go:359`). That file is produced by the generator `gen/go_code.py`, which iterates `all_command_names()` (`gen/go_code.py:667`), reads each Python command's metadata through the **same** `command_for_name(name)` (`:668`), renders Go via `go_code_for_remote_command(...)` (`:669`), and writes `tools/cmd/at/cmd_{name}_generated.go` (`:670`) only when it changed (`replace_if_needed`, `:671`). Generation is wired into the build: `update_go_generated_files` (`setup.py:1102`) runs `kitty +launch gen/go_code.py` (`setup.py:1112`), after which the Go compiler builds the refreshed sources into the `kitten` binary. **Practical consequence:** after adding `kitty/rc/<name>.py` you must **re‑run `python3 setup.py`** so the generator emits `cmd_<name>_generated.go` and the client is recompiled; only then does `kitten @ <name>` exist. (The running kitty server, by contrast, needs no regeneration — just the new module on its import path.)

**4. Responses are wrapped for you** — `handle_cmd` wraps the return as `{'ok': True, 'data': …}` (`kitty/remote_control.py:258-260`) and frames it back in the DCS envelope (`:52-53`); the Go client then decodes and prints it via the §5.4 return path.

> **State clearly:** no such module is added by this task; the repository's `kitty/rc/` tree and the generated `tools/cmd/at/cmd_*_generated.go` files are unchanged.

---

## 9. Observed‑vs‑Inferred ledger

Every major claim is tagged **OBSERVED** (reproduced at runtime, with the producing command), **SOURCE‑VERIFIED** (read directly from code/spec at `file:line`, but that specific branch was not exercised at runtime), or **INFERRED** (spec‑only, not run). Volatile values (PIDs/inodes) are the exact ones captured.

| # | Claim | Status | Evidence |
|---|-------|--------|----------|
| 1 | Build is `python3 setup.py`; launcher at `kitty/launcher/kitty`; banner `kitty 0.35.2`; binary stamps `VCSRevision` of the checked-out git HEAD (authoring-time capture `7da6b377`; a fresh build stamps the current documentation commit) | OBSERVED | §2 `--verbose` build tail + `--version` |
| 2 | Transport = socket **or** DCS‑over‑PTY, never a pipe | OBSERVED (both) | §4/§6 socket dial + §6.3 socketless TTY; `main.go:280` |
| 3 | Target string present → `do_socket_io` (`net.Dial`) | OBSERVED | §4.1/§6.2 + `socket_io.go:177` |
| 4 | Empty target string → `do_tty_io` (DCS to PTY), even with no socket for this instance | OBSERVED | §6.3 (no listener for pid 41484, still works) + `tty_io.go:79-82` |
| 5 | CLI `--listen-on` is **verbatim** unless it contains `{kitty_pid}` | OBSERVED | §4.1 `.../mykitty` (pid 40579) & `.../tplkitty-40680`; `main.py:329-331` |
| 6 | Config‑file `listen_on` gets an automatic `-<PID>` suffix | OBSERVED | §4.1 `.../cfgkitty-40761` (pid 40761) + `main.py:329-330` |
| 7 | `unix:@name` = abstract socket, no FS entry | OBSERVED | §4.2 `@obskitty` via `ss`/`lsof` + `utils.py:513-514` |
| 8 | Default‑off: no socket unless `allow_remote_control` **and** `listen_on` set | OBSERVED | §4.3 grep_exit=1 + §6.3 empty `LISTEN_ON`; `boss.py:364-366`, `definition.py:2969,3000` |
| 9 | Client request is a 58‑byte cleartext DCS `<ESC>P@kitty-cmd{…}<ESC>\`, version `[0,26,0]` | OBSERVED | §5.2 real‑client `od`/hex; `socket_io.go:82-83`, `main.go:33` |
| 10 | Response is `{ok:true, data:<json-string>}` in the same DCS envelope | OBSERVED | §5.2 response bytes + §5.1; `remote_control.py:52-53,258-260`, `ls.py:76` |
| 11 | The socketless PTY frame is **byte‑identical** to the socket request (58 B) | OBSERVED | §5.2 `ptycap.raw` vs `request.bin` |
| 12 | Parse/route: `parse_cmd → handle_cmd → command_for_name('ls') → LS.response_from_kitty`; both ingresses identical | OBSERVED | §5.3 shim firing order (socket + TTY) |
| 13 | Real `ls` JSON tree structure | OBSERVED | §5.1 complete output (pid 42106 instance) |
| 14 | Logging: empty on success (exit 0 + valid JSON asserted first), one `log_error` on malformed | OBSERVED | §5.5 line‑count + error line; `remote_control.py:62` |
| 15 | Go return leg: decode → `Ok`→print `Data`; `!Ok`→print `Error`, exit 1 | OBSERVED | §5.4 `ls` tree on stdout + `--match 'nonsense:1'`; `main.go:223,248,288,297` |
| 16 | `KITTY_PID` and `KITTY_PUBLIC_KEY` exported **unconditionally** to every child | OBSERVED + SOURCE‑VERIFIED | §5.1/§6.2‑6.4 env dumps (populated in all cases); `child.py:244-245` |
| 17 | `KITTY_LISTEN_ON` exported **conditionally** (only when listening; else popped) | OBSERVED + SOURCE‑VERIFIED | §6.3 `KITTY_LISTEN_ON=[]` while PID set; `child.py:246-249` |
| 18 | Shell‑integration scripts do **not** set `KITTY_LISTEN_ON` (they only read env) | OBSERVED | §6.1 `grep` exit 1 |
| 19 | `shell_integration=disabled` → RC still works (SI is not the enabler) | OBSERVED | §6.4 (`SI=[]`, exit 0) |
| 20 | `KITTY_PUBLIC_KEY` is consumed **only** on the password/encryption path | SOURCE‑VERIFIED (not exercised) | `get_pubkey` `main.go:72-94`, `remote_control.py:516-524` |
| 21 | `allow_remote_control` modes: `no`/`yes`/`password`/`socket-only`/`socket` & gating | SOURCE‑VERIFIED (`no`+`yes` OBSERVED) | `definition.py:2969-2999`; `boss.py:594-640` |
| 22 | New command: Python server auto‑routes; Go client needs a generated file + rebuild | SOURCE‑VERIFIED | `base.py:451-453`; `cmd_ls_generated.go:1,148-149`, `go_code.py:667-671`, `setup.py:1102,1112` |
| 23 | Socket ingress `peer_message_received → _handle_remote_command → _execute_remote_command` | SOURCE‑VERIFIED (calls the OBSERVED `parse_cmd`) | `boss.py:776,590,701` |
| 24 | TTY ingress `vt-parser.c dispatch → window.handle_remote_cmd → boss.handle_remote_cmd` | SOURCE‑VERIFIED | `vt-parser.c:603`, `window.py:1279`, `boss.py:849` |
| 25 | Response encode `encode_response_for_peer` / `send_cmd_response` | SOURCE‑VERIFIED (shape OBSERVED) | `remote_control.py:52-53`, `window.py:1386` |
| 26 | A version newer than the instance is rejected | SOURCE‑VERIFIED (guard present; not triggered) | `remote_control.py:218`, `rc_protocol.rst:22-25` |
| 27 | X25519 + AES‑256‑GCM encryption, 5‑min nonce window | INFERRED‑from‑spec (not exercised) | `rc_protocol.rst:64,68,69` |

---

## 10. Coverage pass — your four questions, answered

1. **How does `kitten @ ls` reach kitty, and how does the client discover *where* to send? Socket, pipe, or something else?**
   → **§1 (TL;DR #1) + §3 + §5.** It is a **UNIX/TCP socket** *or* **terminal DCS escape sequences over the PTY** — **never a pipe**. The client (`tools/cmd/at/main.go:280`) picks `do_socket_io` when it knows a target address and `do_tty_io` otherwise; the target defaults to `KITTY_LISTEN_ON` (`:371-372`).

2. **Why does poking around `/tmp` for the socket fail / not match the docs?**
   → **§1 (TL;DR #2) + §4.** Three compounding reasons, each shown live: config‑file paths get an automatic **`-<PID>` suffix** (`.../cfgkitty-40761`, §4.1); `unix:@…` **abstract** sockets have **no file** (`@obskitty`, §4.2); and by default there is **no socket at all** unless `allow_remote_control` + `listen_on` are both set (§4.3).

3. **Shell integration "without configuration": same socket, or TTY magic through the pty? And where are those env vars used?**
   → **§1 (TL;DR #3) + §6.** **Same socket when one is configured** (§6.2, `KITTY_LISTEN_ON`); **DCS‑over‑PTY when no socket exists** (§6.3). The env vars are **produced by `kitty/child.py:244-249`** and **consumed by `tools/cmd/at/main.go:371-372`**; the shell scripts only *read* them (grep exit 1, §6.1) — proven by `shell_integration=disabled` still working (§6.4).

4. **Show the real socket path, the on‑the‑wire messages, where Python parses/routes to `ls`, the returned JSON, and logging.**
   → **§1 (TL;DR #4) + §4/§5.** Real socket path (`.../run.e6tI6X/mykitty` verbatim, §4.1; the §5 round trip used `.../run.XgZliG/kitty.sock`, pid 42106); raw + decoded DCS bytes (§5.2); parse/route chain `parse_cmd → handle_cmd → command_for_name('ls') → LS.response_from_kitty` (§5.3); complete `ls` JSON (§5.1); logging empty‑on‑success vs one `log_error` on malformed (§5.5); and the Go client's receive→decode→print return leg (§5.4).

---

*End of document. All runtime evidence above was captured on `kitty 0.35.2` (source tree at base VCS `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`; the built binary stamps `VCSRevision` with the git HEAD checked out at build time (authoring-time capture `7da6b377`, the initial documentation commit; a fresh build stamps the current documentation commit) — see §2) inside the prepared container, under `Xvfb :99` with the Mesa `llvmpipe` software GL renderer. Volatile values (PIDs such as `40579`/`42106`/`42182` in §4‑§5 and `41386`/`41484`/`41580` in §6, socket inode numbers, timestamps, and the ephemeral public key) are reported exactly as observed and will differ on other runs.*
