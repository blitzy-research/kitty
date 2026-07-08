# How the kitty SSH kitten establishes a session — a run-first, end-to-end trace

This document answers, with **observed runtime evidence**, how the kitty **SSH kitten** builds and
transfers its shell‑integration archive, tracks per‑connection state, decides whether to reuse an
existing SSH connection, encodes its per‑shell bootstrap, runs end‑to‑end, secures credentials through
POSIX shared memory, and talks back and forth with the remote shell over the controlling TTY.

Every behavioral claim below follows the pattern **claim → exact command → unedited output →
`file:line`**. The canonical implementation exercised throughout is the **Go** `kitten` binary
(entry point `kitten ssh`); the Python module `kittens/ssh/main.py` is only the option/configuration
layer, and `kittens/ssh/utils.py` is the local terminal‑side responder invoked from
`kitty/window.py`. Anything I could only infer from reading (not run) is explicitly labelled
**(inferred)**. Instance‑specific values (the generated password `pw`, the `shm_name` suffix, the
`%C` connection hash, chosen `KITTY_PID`/`KITTY_WINDOW_ID`) vary per run and are called out as such.

---

## §0 — Methodology & Environment

### 0.1 Canonical toolchain versions (observed)

| Tool | Observed version | Required by | Role |
|------|------------------|-------------|------|
| Go | `go1.22.12 linux/amd64` | `go 1.22` (`go.mod:3`) | Compiles the canonical Go `kitten` binary (embeds the SSH kitten) |
| Python | `3.13.7` | `>=3.8` (`pyproject.toml:2`) | Drives `setup.py build`; hosts the local responder `kittens/ssh/utils.py` |
| C compiler | `gcc 15.2.0` | `setup.py build` | Compiles the kitty terminal (C sources) |
| OpenSSH | `OpenSSH_10.0p2 … OpenSSL 3.5.3` | external runtime | Real SSH transport + gates the `SSH_ASKPASS_REQUIRE` branch (§0.3) |

The four version commands, run unedited:

```
$ go version
go version go1.22.12 linux/amd64

$ python3 --version
Python 3.13.7

$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0

$ ssh -V
OpenSSH_10.0p2 Ubuntu-5ubuntu5.4, OpenSSL 3.5.3 16 Sep 2025
```

Go `1.22.x` satisfies `go 1.22` in `go.mod:3`; Python `3.13.7` satisfies the floor `>=3.8` in
`pyproject.toml:2`.

### 0.2 Canonical build and the Go `kitten` binary (observed)

The project builds through `setup.py`, which drives the C compiler for the terminal and the Go
toolchain for the `kitten` binary (which embeds the SSH kitten). The unedited output of the canonical
build command (this run was incremental — the artifacts had been built once already, so only the
changed C translation unit is recompiled; a from-scratch build compiles every C source but yields the
same two artifacts and `exit_status: 0`):

```
$ python3 setup.py build
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
[1/1] Compiling kitty/data-types.c ...
 done
[1/1] Linking kitty/fast_data_types ...
 done
kitty/tools/cmd
```

Exit status was `0` (the wayland-protocols message is a benign warning — the Wayland backend is not
needed for the SSH kitten). The build produced the two canonical artifacts `kitty/launcher/kitty`
(the C terminal) and `kitty/launcher/kitten` (the Go binary that embeds the SSH kitten).

The produced `kitten` is the **Go** binary (not a Python shim). Three independent, unedited checks
confirm this — the version banner, the ELF header with its Go BuildID, and Go's own module metadata:

```
$ ./kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal

$ file ./kitty/launcher/kitten
./kitty/launcher/kitten: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, Go BuildID=XcrnQQQOszIbq0c8_YOH/jTzMthXhrwGftKlt38DS/K7FuwbAUdFCyUsqzOl6s/x0UjjztcmvjPg4mwaNuG, stripped

$ go version -m ./kitty/launcher/kitten | head -3
./kitty/launcher/kitten: go1.22.12
	path	kitty/tools/cmd
	mod	kitty	(devel)
```

`go version -m` reports the embedding module is `kitty` built with `go1.22.12` and the entry package
is `kitty/tools/cmd` — the Go command dispatcher whose `ssh` case runs the kitten
(`tools/cmd/main.go:18-19` routes `ssh_askpass`; the `ssh` subcommand is registered from the same Go
tree). The Go BuildID is instance-specific to this build; the value above is the one for the binary
exercised throughout this document.

The SSH kitten entry point is present:

```
$ ./kitty/launcher/kitten ssh --help | head -6
Usage: kitten ssh arguments for the ssh command

The ssh kitten is a thin wrapper around the ssh command. It automatically
enables shell integration on the remote host, re-uses existing connections to
reduce latency, makes the kitty terminfo database available, etc. Its invocation
is identical to the ssh command. For details on its usage, see Truly convenient
```

That usage/description text is the `ssh.Usage` and `ssh.HelpText` set in `specialize_command()` at
`kittens/ssh/main.go:838-841`.

### 0.3 Loopback SSH endpoint (observed)

The remote host for every session in this document is `localhost`, served by a key-only `sshd`
listening on the default port `22`. The endpoint is verified three ways — the daemon is listening,
`localhost`'s host key is already trusted in `known_hosts` (so no interactive host-key prompt fires on
the happy path), and a plain non-interactive `ssh` command succeeds with key-only auth **before the
kitten is ever involved**:

```
$ pgrep -ax sshd | head -2
8246 sshd: /usr/sbin/sshd -o PermitRootLogin=prohibit-password -o PasswordAuthentication=no [listener] 0 of 10-100 startups

$ ssh-keygen -F localhost | head -2   # localhost present in known_hosts (host-key already trusted)
# Host localhost found: line 2 
localhost ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBOaO2pJA2E0qNVkPg4LqwqQGij127J7km+ThL9uWDKQEdhikOPDVKE24ouH5fgyrUTEB6541GpjgD1YVv1nlN4Y=

$ ssh -o BatchMode=yes localhost 'echo REMOTE_OK: id=$(id -un)@$(hostname)'   # key-only auth, no prompt
REMOTE_OK: id=root@reverse-code-generator-a966ddcf-cwgm9
exit=0
```

The daemon runs with `PasswordAuthentication=no` and `PermitRootLogin=prohibit-password`, so
authentication is by the root key pair in `/root/.ssh` only; `BatchMode=yes` proves no password/prompt
is required. Because `localhost` is already a known host, the canonical `kitten ssh localhost` run in
§5 connects without any host-key confirmation dialog. (The askpass **fingerprint** prompt is still
exercised directly in §7 by driving `RunSSHAskpass` with a fingerprint message.)

**Why the OpenSSH version matters (cause → effect):** OpenSSH `10.0` is ≥ `8.4`, so
`SSHVersion.SupportsAskpassRequire()` returns true — `return self.Major > 8 || (self.Major == 8 &&
self.Minor >= 4)` at `kittens/ssh/utils.go:206-208`. As a result the kitten's **default** path uses
kitty's native askpass and sets `need_to_request_data = false` (`kittens/ssh/main.go:147-169`), so in
the default configuration the **local** kitten pushes the data via a DCS rather than the remote
requesting it. To also exercise the "remote requests the data" path (`request_data=true`) I force it
with `--kitten askpass=ssh` (see §3, §5, §7).

### 0.4 Observation instrumentation (no repository edits)

All instrumentation lives **outside** the repository, under `/tmp/kssh-obs/`, and is removed at the
end (see [§Cleanup](#cleanup)). Nothing in the repository is edited; the source tree is verified clean
in §0.5 and again after cleanup.

**Evidence tiers (used consistently throughout, per the run-first mandate):**

1. **Canonical runtime observation** — the primary proof. The real built Go binary
   `./kitty/launcher/kitten ssh …` is run end-to-end against the loopback `sshd`; the kitten, the
   `ssh` client, the remote `sshd`, and the remote `bootstrap.sh`/`bootstrap.py` are **all real**.
   Only the *terminal* is emulated (see the PTY driver below), exactly as a headless kitty window
   would behave.
2. **Supplemental harness** — clearly labelled `[SUPPLEMENTAL]`. A read-only Python script that
   `import`s the repository's **own** functions (e.g. `kittens.ssh.utils.get_ssh_data`,
   `read_data_from_shared_memory`, `kitty.shm.SharedMemory`, `kittens.ssh.askpass` via the binary) to
   drive an individual guard or error path that is awkward to trigger in a full session (§6 guard
   firings). These call the real code; they never re-implement it.
3. **Static source verification** — quoting the exact source line(s) with a `file:line` anchor to
   explain *why* an observed value is what it is. The few facts that are background rather than
   observed — external‑mechanism details such as the ~104‑byte Unix‑domain‑socket path
   limit and general OpenSSH `ControlMaster` / POSIX `shm_open` semantics — are labelled
   **(inferred from source)** or **(inferred)**. (The over‑long-runtime-dir symlink branch itself is
   *observed* canonically in §3.2 by forcing a >35‑char runtime dir.)

**The PTY driver `ptydrv.py` (terminal emulation only).** It `pty.fork()`s the real kitten with a
controlling terminal (the kitten requires `KITTY_WINDOW_ID`+`KITTY_PID` and a TTY on stdin —
`kittens/ssh/main.go:826,829`) and, on the master side, plays the role a kitty window plays: it
answers the kitten's DCS control strings using **kitty's own code**. Its two handlers are byte-for-byte
the same calls the real terminal makes in `kitty/window.py`:

```python
# ptydrv.py — @kitty-ssh handler; identical call to kitty/window.py:1289-1292 handle_remote_ssh
from kittens.ssh.utils import get_ssh_data
def handle_ssh(payload):
    for line in get_ssh_data(memoryview(payload), used_request_id):   # request_id = f"{KITTY_PID}-{KITTY_WINDOW_ID}"
        os.write(master, bytes(line))                                 # window.py does self.write_to_child(line)

# ptydrv.py — @kitty-ask handler; identical to kitty/window.py:1351-1360 handle_remote_askpass
from kitty.shm import SharedMemory
def handle_ask(name):
    with SharedMemory(name=name.decode(), readonly=True) as shm:
        shm.seek(1); data = json.loads(shm.read_data_with_size())     # read question JSON (sentinel byte at [0])
    answer = (str(ans).lower() in ("1","yes","true","y")) if data.get("type")=="confirm" else str(ans)
    with SharedMemory(name=name.decode()) as shm:
        shm.seek(1); shm.write_data_with_size(json.dumps(answer))
        shm.flush(); shm.seek(0); shm.write(b"\x01")                  # flip sentinel 0->1 (askpass.go:72 poll)
```

The comparison — `kitty/window.py:1289-1292`:

```
    def handle_remote_ssh(self, msg: memoryview) -> None:
        from kittens.ssh.utils import get_ssh_data
        for line in get_ssh_data(msg, f'{os.getpid()}-{self.id}'):
            self.write_to_child(line)
```

Because the driver calls the identical `get_ssh_data` with the identical `KITTY_PID-KITTY_WINDOW_ID`
request-id, the data returned to the remote is produced by the real responder, not a stand-in. Raw
child bytes are written verbatim to a `.raw` file so DCS control sequences are visible; rendered
`cat -v` style, ESC (`0x1b`) shows as `^[` and the ST terminator `ESC \` as `^[\`.

**The `PATH` shim (`ssh`, `scp`, `sftp`) — transparent, not a stub.** A directory placed ahead of the
real binaries on `PATH`. The `ssh` shim logs the exact argv the kitten passes (both a human-readable
log and a byte-exact JSON array), then **`exec`s the real `/usr/bin/ssh`** so the full remote handshake
proceeds for real. The one exception is the `ssh -O check` probe: for that it **runs** (not `exec`s)
the real `ssh` so it can also record the exit code the kitten's `master_is_functional()` observes
(§3). The `scp`/`sftp` shims log-and-exec too, purely to prove (§1) that the kitten never invokes them
(the archive travels over the TTY). The full `ssh` shim:

```sh
#!/bin/bash
# Transparent ssh shim: log full argv (human-readable + byte-exact JSON), then run the real ssh.
# For `-O check` probes we RUN (not exec) to also log the exit code master_is_functional() observes.
LOG="${KSSH_SHIM_LOG:-/tmp/kssh-obs/shim.log}"
JSON="${KSSH_SHIM_JSON:-}"
{ echo "=== $(date +%s.%N) ssh invoked (argc=$#) ==="; i=0; for a in "$@"; do echo "ARGV[$i]=$a"; i=$((i+1)); done; } >> "$LOG" 2>&1
if [ -n "$JSON" ]; then
  KSSH_ARGS_JSON="$JSON" /usr/bin/python3 - "$@" <<'PY' 2>/dev/null || true
import json, os, sys
open(os.environ["KSSH_ARGS_JSON"], "a").write(json.dumps(sys.argv[1:]) + "\n")
PY
fi
is_check=0; for a in "$@"; do [ "$a" = "check" ] && is_check=1; done
if [ "$is_check" = "1" ]; then /usr/bin/ssh "$@"; rc=$?; echo "ACTION=RAN_O_CHECK rc=$rc" >> "$LOG"; exit $rc; fi
exec /usr/bin/ssh "$@"
```

**Variant toggling.** Interpreter via `--kitten interpreter=<sh|python3>` (§4); connection sharing via
a pre-started master vs. none (§3); `request_data` via `--kitten askpass=ssh` (forces true) vs. the
default (false on OpenSSH ≥ 8.4, §0.3); askpass sub-type via `SSH_ASKPASS_PROMPT` and the prompt text
(§7). A representative canonical invocation (the default path of §1/§2/§5) is:

```
KSSH_SHIM_LOG=… KSSH_SHIM_JSON=… PATH="/tmp/kssh-obs/shim:$PATH" \
python3 /tmp/kssh-obs/ptydrv.py --raw <raw> --log <log> --dump-tar <tar> \
  --kitty-pid <pid> --kitty-window-id 7 --send-after ']133;A' 'exit' \
  -- ./kitty/launcher/kitten ssh --kitten share_connections=no localhost
```

### 0.5 Read‑only compliance (observed)

All source files this document cites are the frozen snapshot at commit `815df1e21` (the assigned
branch's base). The branch adds **exactly one** commit — the one containing this answer document — and
nothing else. After all captures are complete and every temporary artifact under `/tmp/kssh-obs/` is
removed ([§Cleanup](#cleanup)), the working tree is clean:

```
$ git rev-parse --abbrev-ref HEAD
blitzy-7525139e-05ba-449a-8d19-b10f48711fc6

$ git rev-list --count 815df1e21..HEAD
1

$ git status --porcelain
(no output — working tree clean)

$ git diff 815df1e21..HEAD --name-status
A	blitzy/documentation/kitty_815df1e210e0.md
```

`git rev-list --count 815df1e21..HEAD` reporting `1` confirms the branch adds exactly one commit on
top of the frozen base; the empty `git status --porcelain` (explicit empty-output marker above)
confirms no source file is modified, added, or deleted; and `git diff 815df1e21..HEAD --name-status`
confirms that single commit's only net change is the addition of
`blitzy/documentation/kitty_815df1e210e0.md`. These commands are hash‑independent, so they reproduce
verbatim at HEAD. `815df1e21` ("Wire up applying of font config") is the untouched source base against
which every `file:line` citation below resolves.

---

## §1 — Archive build & TTY transfer  (Q1: "how does that archive with all the shell integration stuff get built and sent over?")

**Direct answer.** The archive is a single **gzip(BestCompression) + PAX tar**, assembled entirely
in memory by `make_tarfile()` from the compiled‑in `go:embed` filesystem (never read from disk at
runtime). It contains an env script `data.sh`, `bootstrap-utils.sh` (only for the `sh` interpreter),
the shell‑integration tree (with the `ssh/` bootstrap files and the legacy `zsh/kitty.zsh`
explicitly excluded), optional `kitty/version` + `kitty/bin/{kitty,kitten}` wrapper binaries, and the
kitty terminfo. That tar is base64‑encoded into a small JSON envelope `{tarfile, pw, hostname,
username}`, and the local responder streams the base64 **over the controlling TTY** in **254‑byte
lines** between `KITTY_DATA_START`/`OK` and `KITTY_DATA_END`. There is **no `scp`/`sftp`** — the bytes
travel over the same terminal the user is typing into, using kitty's DCS protocol.

### 1.1 gzip(BestCompression) + PAX, assembled in memory

`make_tarfile()` creates a gzip writer at best compression wrapping a tar writer:

```go
// kittens/ssh/main.go:255-259
func make_tarfile(cd *connection_data, get_local_env func(string) (string, bool)) ([]byte, error) {
	env_script, ksi := serialize_env(cd, get_local_env)
	w := bytes.Buffer{}
	w.Grow(64 * 1024)
	gw, err := gzip.NewWriterLevel(&w, gzip.BestCompression)
```

Every tar header is written in PAX format, e.g. `Format: tar.FormatPAX` at
`kittens/ssh/main.go:297` and `:310`.

**Observed (canonical runtime).** During a real `./kitty/launcher/kitten ssh … localhost` session, the
`ptydrv.py` driver answered the kitten's `@kitty-ssh` request with the repository's own
`get_ssh_data`, decoded the base64 the responder streamed, and wrote the resulting tar to disk. Its
first bytes are the gzip magic `1f 8b` and its members parse as PAX:

```
$ python3 /tmp/kssh-obs/ptydrv.py --raw env.raw --log env.log --dump-tar env.tar \
    --kitty-pid 4242 --kitty-window-id 7 --send-after ']133;A' 'exit' \
    -- ./kitty/launcher/kitten ssh --kitten share_connections=no localhost
$ python3 -c "import tarfile; t=tarfile.open('env.tar','r:gz'); \
    print('gzip magic', open('env.tar','rb').read(2).hex()); \
    print('format', {2:'PAX'}.get(t.format,t.format)); \
    print('compressed size', __import__('os').path.getsize('env.tar'))"
gzip magic 1f8b
format PAX
compressed size 23484
```

The `1f8b` magic and `gzip.BestCompression` writer are established at `kittens/ssh/main.go:259`; the
PAX format at `:297`/`:310`.

### 1.2 Exact member set (observed, 15 members)

The 15 members of the tar the responder actually streamed (sorted by name for readability; the tar's
internal member *ordering* varies run-to-run because `make_tarfile` iterates the `embed.FS` map, but
the member *set*, sizes, and modes are identical — see §1.5 stability):

```
$ python3 -c "import tarfile; \
    [print(f'0o{m.mode:03o} {m.size:>7}  {m.name}') for m in sorted(tarfile.open('env.tar','r:gz').getmembers(), key=lambda m:m.name)]"
0o644    8468  bootstrap-utils.sh
0o644     224  data.sh
0o755    2761  home/.local/share/kitty-ssh-kitten/kitty/bin/kitten
0o755    4377  home/.local/share/kitty-ssh-kitten/kitty/bin/kitty
0o644       6  home/.local/share/kitty-ssh-kitten/kitty/version
0o644   17363  home/.local/share/kitty-ssh-kitten/shell-integration/bash/kitty.bash
0o644     294  home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_completions.d/clone-in-kitty.fish
0o644     286  home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_completions.d/kitten.fish
0o644     285  home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_completions.d/kitty.fish
0o644   10409  home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish
0o644    1880  home/.local/share/kitty-ssh-kitten/shell-integration/zsh/.zshenv
0o644     280  home/.local/share/kitty-ssh-kitten/shell-integration/zsh/completions/_kitty
0o644   22557  home/.local/share/kitty-ssh-kitten/shell-integration/zsh/kitty-integration
0o644    4271  home/.terminfo/kitty.terminfo
0o644    3711  home/.terminfo/x/xterm-kitty

total members = 15
```

Mapping each member to the code that adds it:

- **`data.sh`** — the serialized environment script, added first:
  `if err = add_data(fe{"data.sh", ...}); ...` at `kittens/ssh/main.go:321`.
- **`bootstrap-utils.sh`** — added **only for the `sh` interpreter**, guarded by
  `if cd.script_type == "sh" {` at `kittens/ssh/main.go:324-325`. (For the Python interpreter this
  member is absent — see §4.)
- **shell‑integration tree** under `home/<remote_dir>/…` via
  `shell_integration.Data().FilesMatching("shell-integration/", …)` at
  `kittens/ssh/main.go:329-333`, which **excludes** two patterns, quoted verbatim from the source:

  ```go
  // kittens/ssh/main.go:332-333
  "shell-integration/ssh/.+",        // bootstrap files are sent as command line args
  "shell-integration/zsh/kitty.zsh", // backward compat file not needed by ssh kitten
  ```

  Consistent with this, the listing contains **no** `shell-integration/ssh/*` files and **no**
  `zsh/kitty.zsh`. The default remote dir is `.local/share/kitty-ssh-kitten` (see §2), which is why
  the arcnames are `home/.local/share/kitty-ssh-kitten/…`.
- **wrapper binaries** — `kitty/version` plus `kitty/bin/{kitty,kitten}` (mode `0o755`), added **only
  when** `cd.host_opts.Remote_kitty != Remote_kitty_no`:

  ```go
  // kittens/ssh/main.go:342-349
  if cd.host_opts.Remote_kitty != Remote_kitty_no {
  	arcname := path.Join("home/", rd, "/kitty")
  	err = add_data(fe{arcname + "/version", utils.UnsafeStringToBytes(kitty.VersionString)})
  	if err != nil {
  		return nil, err
  	}
  	for _, x := range []string{"kitty", "kitten"} {
  		err = add_entries(path.Join(arcname, "bin"), shell_integration.Data()[path.Join("shell-integration", "ssh", x)])
  ```
- **terminfo** — `home/.terminfo/kitty.terminfo` and `home/.terminfo/x/<DefaultTermName>`:

  ```go
  // kittens/ssh/main.go:355-358
  err = add_entries(path.Join("home", ".terminfo"), shell_integration.Data()["terminfo/kitty.terminfo"])
  if err == nil {
  	err = add_entries(path.Join("home", ".terminfo", "x"), shell_integration.Data()["terminfo/x/"+kitty.DefaultTermName])
  }
  ```

  The observed `home/.terminfo/x/xterm-kitty` confirms `kitty.DefaultTermName == "xterm-kitty"`.

The `add()` helper forces owner‑only‑writable bits (`h.Mode |= 0o600`) for regular data at
`kittens/ssh/main.go:269`; the two wrapper launchers keep their `0o755` executable bit (observed
above).

### 1.3 The embedded (`go:embed`) archive source

The shell‑integration tree and terminfo are compiled **into** the `kitten` binary:

```go
// tools/tui/shell_integration/data.go:19-20
//go:embed data_generated.bin
var embedded_data string
```

`FilesMatching(...)` (used by `make_tarfile`) is defined at `tools/tui/shell_integration/data.go:49`.
Because the source is an in‑memory `embed.FS`, the tar is assembled from RAM — which is exactly why
the member listing above is produced with no filesystem reads of the shell‑integration tree at run
time. **Observed:** `data_generated.bin` exists as the embedded blob (25574 bytes) alongside the Go
source.

### 1.4 The JSON envelope `{tarfile, pw, hostname, username}`

The tar bytes are base64‑encoded and wrapped in a JSON map:

```go
// kittens/ssh/main.go:431-442
pw, err := secrets.TokenHex()
if err != nil {
	return err
}
tfd, err := make_tarfile(cd, os.LookupEnv)
if err != nil {
	return err
}
data := map[string]string{
	"tarfile":  base64.StdEncoding.EncodeToString(tfd),
	"pw":       pw,
	"hostname": cd.hostname_for_match, "username": cd.username,
```

**Observed (canonical runtime).** During the same real `kitten ssh … localhost` session, the driver
opened the SHM object the kitten created (read-only, before `get_ssh_data` unlinks it) and decoded the
JSON envelope. The keys are exactly `hostname, pw, tarfile, username`; the values are instance-specific
(the `pw` is a per-run throwaway credential and is **redacted** here — only its 64-hex length is
reported):

```
$ grep 'envelope keys' env.log
           envelope keys=['hostname', 'pw', 'tarfile', 'username'] hostname='localhost' username='root' pw=<redacted 64-hex> tarfile_b64_len=31312
```

`hostname='localhost'` is `cd.hostname_for_match` and `username='root'` is `cd.username` for
`kitten ssh localhost` (§2); `tarfile` is the base64 of the 23484-byte gzip tar from §1.1
(`tarfile_b64_len=31312`); `pw` is the `secrets.TokenHex()` password from `kittens/ssh/main.go:431`
that also guards the SHM read (§6).

### 1.5 Streaming over the TTY in 254‑byte lines (not scp/sftp)

The **local** responder `get_ssh_data()` first yields the framing marker
`b'\nKITTY_DATA_START\n'` (`kittens/ssh/utils.py:117`), then after validating the request it
yields `OK` and streams the base64 payload split into 254-byte lines. The exact contiguous
source of the OK+streaming core is:

```python
# kittens/ssh/utils.py:138-148  (verbatim; the leading KITTY_DATA_START marker is yielded earlier at L117)
            yield b'OK\n'
            encoded_data = memoryview(env_data['tarfile'].encode('ascii'))
            # macOS has a 255 byte limit on its input queue as per man stty.
            # Not clear if that applies to canonical mode input as well, but
            # better to be safe.
            line_sz = 254
            while encoded_data:
                yield encoded_data[:line_sz]
                yield b'\n'
                encoded_data = encoded_data[line_sz:]
            yield b'KITTY_DATA_END\n'
```

**Observed** framing and line sizing, captured during the canonical `kitten ssh localhost`
session of §1.1. The PTY driver logs the exact bytes the repository's own `get_ssh_data()`
streamed back over the controlling TTY (it is the terminal side only; the kitten, `ssh`, remote
`sshd` and remote bootstrap are all the real binaries):

```
# /tmp/kssh-obs/cap/env.log
# child argv: ./kitty/launcher/kitten ssh --kitten share_connections=no localhost
framing chunk[0]=b'\nKITTY_DATA_START\n' chunk[1]=b'OK\n' last=b'KITTY_DATA_END\n'
stream: 124 data lines; distinct lengths=[70, 254]; first=254 last=70
get_ssh_data streamed 31472 bytes back to child
```

The three framing markers are exactly the `yield b'\nKITTY_DATA_START\n'` (`kittens/ssh/utils.py:117`),
`yield b'OK\n'` (`kittens/ssh/utils.py:138`) and `yield b'KITTY_DATA_END\n'` (`kittens/ssh/utils.py:148`)
statements. The base64 payload always begins with `H4sI` — the base64 of the gzip magic bytes
`1f 8b 08` — confirming the streamed body is the gzip tar of §1.1.

**Stability across three canonical runs (observed).** The 254-byte line size is invariant; only the
*last* line's length varies, because the random `pw`/`shm_name` shift the compressed size by a few
bytes. Three independent `kitten ssh localhost` sessions (two with `share_connections=no`, one
default) gave:

| capture | data lines | distinct line lengths | last line |
|---------|-----------|-----------------------|-----------|
| `cap/default.log`  | 124 | `[158, 254]` | 158 |
| `cap/env.log`      | 124 | `[70, 254]`  | 70  |
| `cap/default2.log` | 125 | `[32, 254]`  | 32  |

Every line but the last is exactly 254 bytes in all three runs — the invariant the code guarantees.
The `254` constant is fixed at `kittens/ssh/utils.py:143`.

### 1.6 The remote receives it over the TTY, decodes, and untars

On the remote side, the bootstrap reads the base64 from the TTY, decodes it, and untars into a temp
dir before atomically moving files into `$HOME`:

```sh
# shell-integration/ssh/bootstrap.sh:108-113 (verbatim, body of untar_and_read_env)
    tdir=$(command mktemp -d "$HOME/.kitty-ssh-kitten-untar-XXXXXXXXXXXX")
    [ $? = 0 ] || die "Creating temp directory failed"
    # suppress STDERR for tar as tar prints various warnings if for instance, timestamps are in the future
    old_umask=$(umask)
    umask 000
    read_base64_from_tty | base64_decode | command tar "xpzf" "-" "-C" "$tdir" 2> /dev/null
```

then `compile_terminfo` and `mv_files_and_dirs "$tdir/home" "$HOME"`
(`shell-integration/ssh/bootstrap.sh:130-131`; helpers at
`shell-integration/ssh/bootstrap-utils.sh:9,18`).

**Observed.** This exact `untar_and_read_env` body is present verbatim in the bootstrap script
the kitten transmitted as the last `ssh` argument (`ARGV[7]`), captured in
`/tmp/kssh-obs/cap/noscp.log`; and the 15 members it untars are exactly those dumped from the
streamed tar in §1.1–§1.2. The remote reaching a working login shell after this step is shown
end-to-end in §5.

### 1.7 Proof there is no scp/sftp side channel

The entire transfer is over the controlling TTY via the DCS protocol; **no** `scp`/`sftp`/`rsync`
process is spawned. This was verified by placing logging shims for **all three** of `ssh`, `scp` and `sftp` first on
`PATH` and running one full canonical session, then counting invocations:

```
# /tmp/kssh-obs/cap/noscp.log
# child argv: ./kitty/launcher/kitten ssh --kitten share_connections=no localhost
$ grep -c ' ssh invoked '  noscp.log   ->  2
$ grep -c ' scp invoked '  noscp.log   ->  0
$ grep -c ' sftp invoked ' noscp.log   ->  0

# the two ssh invocations, by argv length:
=== 1783491114.361949156 ssh invoked (argc=0) ===   # bare ssh: SSH-option discovery (kittens/ssh/utils.go:26,40)
=== 1783491114.386658068 ssh invoked (argc=8) ===   # bootstrap: ssh -t -- localhost exec sh -c <unwrap> <encoded>
```

So the only child processes are `ssh` itself: one bare `ssh` the kitten runs to parse the client's
supported options (`SSHOptions`, `kittens/ssh/utils.go:40`), and one `ssh … exec sh -c …` carrying
the bootstrap (§4, §5). `scp`/`sftp` are invoked **zero** times. (With `share_connections=yes` an
extra `ssh -O check` appears — §3 — and an `ssh -V` version probe appears only when the askpass
sentinel is absent, `kittens/ssh/main.go:152`; neither carries the archive.) The actual data path is
`dcs_to_kitty` / `KITTY_DATA_*` over `/dev/tty` (`shell-integration/ssh/bootstrap.sh:75,113`).

**Cause → effect.** Because the archive is streamed as escape‑framed base64 over the same PTY the
shell uses, the kitten needs no second authenticated channel and works even where `scp`/`sftp` are
disabled — at the cost of the 254‑byte line chunking that dodges the macOS `stty` input‑queue limit.

---

## §2 — Per‑connection state  (Q2: "how does the kitten keep track of everything it needs for a connection?")

**Direct answer.** All per‑session state lives in one struct, `connection_data`
(`kittens/ssh/main.go:171-189`). It carries the parsed SSH args, the resolved host options, the
match hostname and username, the echo state, whether the remote must request the data, the literal
env, the control‑master listen socket, an optional test script, the shared‑memory object name, the
`sh`/`py` script type, the final remote command `rcmd`, the substitution map `replacements`, the
`request_id` (defaulting to `KITTY_PID-KITTY_WINDOW_ID`), and the prepared `bootstrap_script`. The
`replacements` map is what gets textually substituted into the bootstrap template.

### 2.1 The `connection_data` struct — every field

```go
// kittens/ssh/main.go:171 (struct declaration)
type connection_data struct {
```

Field-by-field, **reconstructed from externally observed artifacts** of one canonical `kitten ssh`
run. The struct is internal Go state, so each value is read off the `PATH`-shim argv
(`cap/argv_fresh.json`), the PTY driver log (`cap/fresh.log`), the JSON envelope the driver read from
the SHM object, and the decoded transmitted bootstrap (`ARGV[19]`). Every temporary artifact is
removed in [Cleanup](#cleanup). The run and its reconstruction:

```
# canonical invocation (fresh master -> ssh -O check rc=255 -> request_data=true; see §3)
$ KITTY_PID=555003 KITTY_WINDOW_ID=7 \
    ./kitty/launcher/kitten ssh --kitten askpass=ssh --kitten share_connections=yes localhost

# reconstructed connection_data (kittens/ssh/main.go:171-189)   [obs]=observed  [def]=source default
  remote_args        = []                                                                    # no server command                 [def]
  host_opts          = *Config{Interpreter:"sh", Share_connections:true,
                               Remote_dir:".local/share/kitty-ssh-kitten", Remote_kitty:if-needed}   # sh + ControlMaster args    [obs]
  hostname_for_match = "localhost"                                                            # envelope + argv                   [obs]
  username           = "root"                                                                 # envelope username                 [obs]
  echo_on            = true                                                                   # echo_on="1" in bootstrap          [obs]
  request_data       = true                                                                   # request_data="1"; remote DCS      [obs]
  literal_env        = map[string]string(nil)                                                 # no --env passed                   [def]
  listen_on          = ""                                                                      # forward_remote_control=no         [def]
  test_script        = ""                                                                      # no test hook                      [def]
  dont_create_shm    = false                                                                  # SHM object was created            [obs]
  shm_name           = "kssh-106105-II5YDDIU6M3DG"                                            # /dev/shm object; == pwfile        [obs]
  script_type        = "sh"                                                                    # rcmd = exec sh -c <unwrap><enc>   [obs]
  rcmd               = ["exec" "sh" "-c" <unwrap:67 bytes> <encoded:5264 bytes>]              # shim ARGV[15:20]                  [obs]
  replacements       = <8 keys; see §2.3>
  request_id         = "555003-7"                                                             # == KITTY_PID-KITTY_WINDOW_ID      [obs]
  bootstrap_script   = <5262 bytes, prepared from bootstrap.sh>                               # decoded ARGV[19]                  [obs]
```

Each field, named and grounded:

| Field | Observed value | Role | `file:line` |
|-------|----------------|------|-------------|
| `remote_args` | `[]string{}` | SSH args destined for the remote command | `kittens/ssh/main.go:172` |
| `host_opts` | `*Config` (Interpreter=`sh`, Share_connections=`true`, Remote_dir=`.local/share/kitty-ssh-kitten`, Remote_kitty=`if-needed`) | resolved per‑host options | `kittens/ssh/main.go:173` |
| `hostname_for_match` | `"localhost"` | hostname used for config matching | `kittens/ssh/main.go:174` |
| `username` | `"root"` | remote user | `kittens/ssh/main.go:175` |
| `echo_on` | `true` | whether TTY echo was on originally | `kittens/ssh/main.go:176` |
| `request_data` | `true` | whether the **remote** must request the tar | `kittens/ssh/main.go:177` |
| `literal_env` | `nil` | literal env overrides | `kittens/ssh/main.go:178` |
| `listen_on` | `""` | control‑master forward socket | `kittens/ssh/main.go:179` |
| `test_script` | `""` | test hook script | `kittens/ssh/main.go:180` |
| `dont_create_shm` | `false` | suppress SHM creation (tests) | `kittens/ssh/main.go:181` |
| `shm_name` | `"kssh-106105-II5YDDIU6M3DG"` | POSIX SHM object holding the JSON envelope | `kittens/ssh/main.go:183` |
| `script_type` | `"sh"` | `sh` or `py` bootstrap family | `kittens/ssh/main.go:184` |
| `rcmd` | `["exec" "sh" "-c" <unwrap> <encoded>]` | final wrapped remote command | `kittens/ssh/main.go:185` |
| `replacements` | (see §2.3) | template substitution map | `kittens/ssh/main.go:186` |
| `request_id` | `"555003-7"` | request identity `KITTY_PID-KITTY_WINDOW_ID` | `kittens/ssh/main.go:187` |
| `bootstrap_script` | `<5262 bytes>` | fully substituted bootstrap text | `kittens/ssh/main.go:188` |

### 2.2 `request_id` defaults to `KITTY_PID-KITTY_WINDOW_ID`

```go
// kittens/ssh/main.go:424
cd.request_id = os.Getenv("KITTY_PID") + "-" + os.Getenv("KITTY_WINDOW_ID")
```

**Observed:** with `KITTY_PID=555003` and `KITTY_WINDOW_ID=7`, the dump above shows
`request_id = "555003-7"`. This identity is later checked by the local responder before it releases the
tar (`get_ssh_data`, §6), and it is exactly the pair `kitty/window.py:1291` builds on the receiving
side (`f'{os.getpid()}-{self.id}'`, §7).

### 2.3 The `replacements` substitution map

Built in `bootstrap_script()` at `kittens/ssh/main.go:461-479` and applied to the bootstrap template.
**Observed** (the sensitive `DATA_PASSWORD`/`PASSWORD_FILENAME`/`REQUEST_ID` are shown as the kitten
produced them; `DATA_PASSWORD` is an instance‑specific throwaway credential to loopback):

```
=== replacements map (kittens/ssh/main.go L461-479) ===
  replacements["DATA_PASSWORD"] = <64 hex chars> (secrets.TokenHex)
  replacements["ECHO_ON"] = "1"
  replacements["EXEC_CMD"] = ""
  replacements["EXPORT_HOME_CMD"] = ""
  replacements["PASSWORD_FILENAME"] = "kssh-106105-II5YDDIU6M3DG"
  replacements["REQUEST_DATA"] = "1"
  replacements["REQUEST_ID"] = "555003-7"
  replacements["TEST_SCRIPT"] = ""
```

- `EXPORT_HOME_CMD`, `EXEC_CMD`, `TEST_SCRIPT` are the plain string replacements
  (`kittens/ssh/main.go:462-464`).
- `REQUEST_DATA` and `ECHO_ON` are booleans rendered as `"1"`/`"0"` via the `add_bool` helper
  (`kittens/ssh/main.go:473-474`); here both are `"1"`.
- `REQUEST_ID`, `DATA_PASSWORD`, `PASSWORD_FILENAME` are the sensitive triple, defined as
  `sensitive_data` (`kittens/ssh/main.go:460`) and copied into `cd.replacements` **unconditionally**
  at `kittens/ssh/main.go:479`; `PASSWORD_FILENAME` equals the struct's `shm_name` (both
  `kssh-106105-II5YDDIU6M3DG` here), tying the bootstrap's DCS request (§7) to the SHM object the
  local side reads (§6).

**Two maps — an important subtlety.** `cd.replacements` (dumped above) always holds all eight keys,
but it is **not** the map used to fill the script that is actually sent. `bootstrap_script()` clones
`replacements` into a second map `sd` **before** the sensitive triple is added
(`kittens/ssh/main.go:475`), then copies `sensitive_data` into `sd` **only if `cd.request_data`**
(`kittens/ssh/main.go:476-478`); it is `sd`, not `replacements`, that substitutes the transmitted
`bootstrap_script` (`kittens/ssh/main.go:482`). In this `request_data=true` run the real
`id`/`pwfile`/`pw` are therefore baked into the sent script (observed in `ARGV[19]`, §7); in the
default reused path (`request_data=false`) the sent script keeps the literal `"REQUEST_ID"` /
`"PASSWORD_FILENAME"` / `"DATA_PASSWORD"` placeholders — the crux of the Q6 security property (§6).

These substitutions are what turn the embedded `bootstrap.<script_type>` template
(`kittens/ssh/main.go:481`) into the concrete `bootstrap_script`, which is then wrapped into `rcmd`
(§4).

**Cause → effect.** One struct threads through the entire flow: `run_ssh()` fills it, `make_tarfile()`
reads `host_opts`/`script_type` from it, `bootstrap_script()` writes `shm_name`/`replacements`/
`request_id` into it, and `wrap_bootstrap_script()` reads `script_type`/`bootstrap_script` to produce
`rcmd`. Tracking "everything for a connection" is therefore literally tracking this one value.


---

## §3 — Fresh vs. reused connection  (Q3: "how it decides whether to start a fresh connection or piggyback on an existing one")

**Direct answer.** The kitten leans on OpenSSH connection multiplexing. By default
(`share_connections = yes`) it adds `-o ControlMaster=auto -o ControlPath=<runtimedir>/kssh-<kitty_pid>-%C
-o ControlPersist=yes` (plus keepalive options). The reuse decision is made by running **`ssh -O
check`** in `master_is_functional()`: if a master is already live **and** sharing is on **and** the
kitten would otherwise ask the remote for data, it flips `need_to_request_data` to **false** (the
local side pushes the data instead of the remote requesting it). If no master is live, `ssh -O check`
fails and the value stays **true**. OpenSSH itself decides fresh‑vs‑piggyback at connect time from
`ControlMaster=auto` + the `%C` socket; the kitten's own gate governs *who requests the tar*.

### 3.1 Defaults and the sharing arguments

Default `share_connections` is `yes`:

```
# kittens/ssh/main.py:183
opt('share_connections', 'yes', option_type='to_bool', ...)
```

`connection_sharing_args()` builds the multiplexing options (quoted verbatim, no elision):

```go
// kittens/ssh/main.go:121-145
func connection_sharing_args(kitty_pid int) ([]string, error) {
	rd := utils.RuntimeDir()
	// Bloody OpenSSH generates a 40 char hash and in creating the socket
	// appends a 27 char temp suffix to it. Socket max path length is approx
	// ~104 chars. And on idiotic Apple the path length to the runtime dir
	// (technically the cache dir since Apple has no runtime dir and thinks it's
	// a great idea to delete files in /tmp) is ~48 chars.
	if len(rd) > 35 {
		idiotic_design := fmt.Sprintf("/tmp/kssh-rdir-%d", os.Geteuid())
		if err := utils.AtomicCreateSymlink(rd, idiotic_design); err != nil {
			return nil, err
		}
		rd = idiotic_design
	}
	cp := strings.Replace(kitty.SSHControlMasterTemplate, "{kitty_pid}", strconv.Itoa(kitty_pid), 1)
	cp = strings.Replace(cp, "{ssh_placeholder}", "%C", 1)
	return []string{
		"-o", "ControlMaster=auto",
		"-o", "ControlPath=" + filepath.Join(rd, cp),
		"-o", "ControlPersist=yes",
		"-o", "ServerAliveInterval=60",
		"-o", "ServerAliveCountMax=5",
		"-o", "TCPKeepAlive=no",
	}, nil
}
```

The `ControlPath` template is `kssh-{kitty_pid}-{ssh_placeholder}`:

```python
# kitty/constants.py:188
ssh_control_master_template = 'kssh-{kitty_pid}-{ssh_placeholder}'
```

with `{ssh_placeholder}` replaced by SSH's own `%C` connection-tuple hash (`kittens/ssh/main.go:136`).

**Observed (canonical runtime, instance-specific).** The exact six `-o` options the kitten passed, captured by a transparent `PATH` shim (it logs argv, then runs/execs the real `/usr/bin/ssh` — see the shim listing in §0). These are argv indices `[3..14]` of the `argc=17` `ssh -O check` probe from a real fresh run with `KITTY_PID=555003` (the full probe and its exit code are in §3.3):

```
$ rm -f /root/.cache/kitty/run/kssh-*            # ensure no live master lingers
$ PATH=/tmp/kssh-obs/shim:$PATH KSSH_SHIM_LOG=/tmp/kssh-obs/cap/shim_fresh.log \
    python3 /tmp/kssh-obs/ptydrv.py --kitty-pid 555003 --kitty-window-id 7 \
      --raw /tmp/kssh-obs/cap/fresh.raw --log /tmp/kssh-obs/cap/fresh.log \
      -- ./kitty/launcher/kitten ssh --kitten askpass=ssh --kitten share_connections=yes localhost
$ sed -n '/argc=17/,/=== END ===/p' /tmp/kssh-obs/cap/shim_fresh.log | sed -n '5,16p'
ARGV[3]=-o
ARGV[4]=ControlMaster=auto
ARGV[5]=-o
ARGV[6]=ControlPath=/root/.cache/kitty/run/kssh-555003-%C
ARGV[7]=-o
ARGV[8]=ControlPersist=yes
ARGV[9]=-o
ARGV[10]=ServerAliveInterval=60
ARGV[11]=-o
ARGV[12]=ServerAliveCountMax=5
ARGV[13]=-o
ARGV[14]=TCPKeepAlive=no
```

These are exactly the six `-o` pairs `connection_sharing_args()` returns (`kittens/ssh/main.go:138-143`): `ControlMaster=auto`, `ControlPath=<runtimedir>/kssh-555003-%C`, `ControlPersist=yes`, `ServerAliveInterval=60`, `ServerAliveCountMax=5`, `TCPKeepAlive=no`. The `%C` token is still literal in the argv the kitten emits — OpenSSH itself expands it to the connection-tuple hash when it creates the socket. **Observed** on disk once a master exists (the reused-master run of §3.3):

```
$ ls -1 /root/.cache/kitty/run/ | grep kssh-555003-
kssh-555003-b8f03188b595d83f936586075e8de938d9a67f21
```

i.e. `ControlPath = /root/.cache/kitty/run/kssh-<kitty_pid>-<%C hash>` (the 40-char `%C` hash is instance-specific to the connection tuple).

### 3.2 The over-long-runtime-dir workaround (`/tmp/kssh-rdir-<euid>`)

Unix-domain socket paths are limited to ~104 bytes and OpenSSH appends a ~27-char temp suffix to the `%C` socket, so when the runtime dir exceeds 35 chars the kitten shortens it by symlinking it. The branch is quoted verbatim in §3.1 (`kittens/ssh/main.go:128-133`); `utils.RuntimeDir()` honours `KITTY_RUNTIME_DIRECTORY` first (`tools/utils/paths.go:193`), and `AtomicCreateSymlink(oldname, newname)` creates `newname` as a symlink pointing to `oldname` via `os.Symlink` (`tools/utils/atomic-write.go:15`).

**Observed (canonical runtime).** Forcing a 60-char runtime dir via `KITTY_RUNTIME_DIRECTORY` (> 35) makes the kitten resolve `ControlPath` under `/tmp/kssh-rdir-0` (euid 0) and create the symlink:

```
$ LONGDIR=/root/.cache/kitty/kssh-overlong-runtime-dir-observation-000   # len=60, > 35
$ mkdir -p "$LONGDIR"
$ PATH=/tmp/kssh-obs/shim:$PATH KSSH_SHIM_LOG=/tmp/kssh-obs/cap/overlong.log \
    KITTY_RUNTIME_DIRECTORY="$LONGDIR" \
    python3 /tmp/kssh-obs/ptydrv.py --kitty-pid 888001 --kitty-window-id 7 \
      -- ./kitty/launcher/kitten ssh --kitten askpass=ssh --kitten share_connections=yes localhost
$ grep -m1 ControlPath= /tmp/kssh-obs/cap/overlong.log
ARGV[6]=ControlPath=/tmp/kssh-rdir-0/kssh-888001-%C
$ readlink /tmp/kssh-rdir-0
/root/.cache/kitty/kssh-overlong-runtime-dir-observation-000
$ ls -ld /tmp/kssh-rdir-0
lrwxrwxrwx 1 root root 60 Jul  8 07:04 /tmp/kssh-rdir-0 -> /root/.cache/kitty/kssh-overlong-runtime-dir-observation-000
```

So the 60-char runtime dir is replaced by the 15-char `/tmp/kssh-rdir-0` symlink before `%C` and OpenSSH's ~27-char suffix are appended, keeping the socket path within the ~104-byte limit. The limit itself is external OpenSSH/POSIX behavior (background context, corroborated by the OpenSSH docs cited in 0.2.2); the kitten's response to it is the symlink observed above.

### 3.3 The reuse gate: `ssh -O check` -> `need_to_request_data`

The decision code, quoted verbatim (no elision):

```go
// kittens/ssh/main.go:649-664
need_to_request_data := true
if use_kitty_askpass {
	need_to_request_data = set_askpass()
}
master_is_functional := func() bool {
	if master_checked {
		return master_is_alive
	}
	master_checked = true
	check_cmd := slices.Insert(cmd, 1, "-O", "check")
	master_is_alive = exec.Command(check_cmd[0], check_cmd[1:]...).Run() == nil
	return master_is_alive
}

if need_to_request_data && host_opts.Share_connections && master_is_functional() {
	need_to_request_data = false
}
```

To exercise **both** states with `need_to_request_data` starting `true`, I set `--kitten askpass=ssh` (this makes `use_kitty_askpass=false`, so `set_askpass()` is not called and the value stays `true`; `kittens/ssh/main.go:648-651`). I ran the kitten with **no** master, and again with a real master pre-started at the kitten's exact `ControlPath`. The shim RUNs the real `ssh -O check` and records the exit code that `master_is_functional()` observes; the driver preserves the transmitted bootstrap so `request_data` and the emitted `@kitty-ssh` line can be read back.

**FRESH (no live master).** The full `-O check` probe the kitten issued (`argc=17`) and the exit code the shim recorded — `rc=255`:

```
$ sed -n '/argc=17/,/=== END ===/p' /tmp/kssh-obs/cap/shim_fresh.log
=== 1783490068.175010536 ssh invoked (argc=17) ===
ARGV[0]=-O
ARGV[1]=check
ARGV[2]=-t
ARGV[3]=-o
ARGV[4]=ControlMaster=auto
ARGV[5]=-o
ARGV[6]=ControlPath=/root/.cache/kitty/run/kssh-555003-%C
ARGV[7]=-o
ARGV[8]=ControlPersist=yes
ARGV[9]=-o
ARGV[10]=ServerAliveInterval=60
ARGV[11]=-o
ARGV[12]=ServerAliveCountMax=5
ARGV[13]=-o
ARGV[14]=TCPKeepAlive=no
ARGV[15]=--
ARGV[16]=localhost
ACTION=RAN_O_CHECK rc=255
=== END ===
```

With `rc=255`, `master_is_functional()` returns false, so `need_to_request_data` stays `true`. The bootstrap the kitten then transmits carries `request_data="1"`, and — because `request_data=1` — its `@kitty-ssh` DCS line has the **real substituted** id/pwfile/pw (the remote will issue the request). Note the transmitted `sh` bootstrap encodes newlines as `\r`, so I decode CRs before grepping (pw redacted here; the live command prints the real 64-hex value):

```
$ awk '/argc=20/{f=1} /=== END ===/{if(f)exit} f' /tmp/kssh-obs/cap/shim_fresh.log \
    | tr '\r' '\n' | grep -E '^request_data=|dcs_to_kitty "ssh" "id='
request_data="1"
    dcs_to_kitty "ssh" "id="555003-7":pwfile="kssh-106105-II5YDDIU6M3DG":pw="<redacted 64-hex, instance-specific>""
```

**REUSED (persisted master pre-started at the same `ControlPath`).** I started a real master, then re-ran the identical invocation. The `-O check` probe argv is **byte-for-byte identical** to the fresh probe above (verified by `diff`); only the recorded exit code differs — `rc=0`:

```
$ ssh -o ControlMaster=auto -o ControlPath=/root/.cache/kitty/run/kssh-555003-%C \
      -o ControlPersist=yes -N -f -- localhost         # pre-start the master
$ diff <(sed -n '/argc=17/,/ACTION=/p' /tmp/kssh-obs/cap/shim_fresh.log  | grep '^ARGV') \
       <(sed -n '/argc=17/,/ACTION=/p' /tmp/kssh-obs/cap/shim_reused.log | grep '^ARGV')
$ grep '^ACTION=' /tmp/kssh-obs/cap/shim_reused.log | tail -1
ACTION=RAN_O_CHECK rc=0
```

(the empty `diff` output confirms identical argv). With `rc=0`, `master_is_functional()` returns true, and the gate flips `need_to_request_data` to `false`. The transmitted bootstrap now carries `request_data="0"` and — because `request_data` is false — its `@kitty-ssh` DCS line keeps the **literal placeholders** (`sensitive_data` was never substituted into it; see §2.3 and §6): the local kitten will emit the request itself (§7):

```
$ awk '/argc=20/{f=1} /=== END ===/{if(f)exit} f' /tmp/kssh-obs/cap/shim_reused.log \
    | tr '\r' '\n' | grep -E '^request_data=|dcs_to_kitty "ssh" "id='
request_data="0"
    dcs_to_kitty "ssh" "id="REQUEST_ID":pwfile="PASSWORD_FILENAME":pw="DATA_PASSWORD""
```

So the decision flips exactly as the code says: **`ssh -O check` rc `255 -> 0`  =>  `need_to_request_data` `true -> false`  =>  `request_data` `"1" -> "0"`  =>  the local kitten emits the `@kitty-ssh` DCS itself** (§7) instead of the remote asking. This is the before/after of the single state that changes. Security-relevant corollary (Q6): in the reused path the transmitted command line carries only the literal tokens `REQUEST_ID`/`PASSWORD_FILENAME`/`DATA_PASSWORD`, never the real password.

### 3.4 `run_control_master()` and `forward_remote_control`

The kitten can also *explicitly* start a master with `-N -f`:

```go
// kittens/ssh/main.go:666-670
run_control_master := func() error {
	cmcmd := slices.Clone(cmd[:insertion_point])
	cmcmd = append(cmcmd, control_master_args...)
	cmcmd = append(cmcmd, "-N", "-f")
	cmcmd = append(cmcmd, "--", hostname)
```

This path runs only when `forward_remote_control` is on and `KITTY_LISTEN_ON` is set (`kittens/ssh/main.go:681`) — but `forward_remote_control` defaults to **no**:

```
# kittens/ssh/main.py:212
opt('forward_remote_control', 'no', option_type='to_bool', ...)
```

so in the canonical/default configuration the explicit master is not started; reuse is driven purely by `ControlMaster=auto` + the `ssh -O check` gate above.


---

## §4 — Per‑shell bootstrap encoding  (Q4: "the bootstrap script encoding looks wild too with all those character substitutions for different shells")

**Direct answer.** `sshd` hands the remote command to the user's login shell as a single `-c`
argument, so the encoding must survive whatever shell that is. The kitten picks one of two schemes by
`script_type`: for **Python** it base64‑encodes the script and unwraps with a one‑liner
`eval(compile(base64.standard_b64decode(sys.argv[-1]), 'bootstrap.py', 'exec'))`; for **`sh`** it
cannot assume a remote `base64` binary, so it quote‑escapes by mapping four bytes
(`'`→`\v`, `\`→`\f`, `\n`→`\r`, `!`→`\b`), wraps the whole thing in single quotes, and unwraps on the
remote by reversing those substitutions with **`tr`**. The final remote command is always
`exec <interpreter> -c <unwrap_script> <encoded_script>`.

### 4.1 `script_type` selection

```go
// kittens/ssh/main.go:511-518
func get_remote_command(cd *connection_data) error {
	interpreter := cd.host_opts.Interpreter
	q := strings.ToLower(path.Base(interpreter))
	is_python := strings.Contains(q, "python")
	cd.script_type = "sh"
	if is_python {
		cd.script_type = "py"
	}
```

The default interpreter is `sh` (`kittens/ssh/main.py:87`), so `script_type` defaults to `sh`; an
interpreter whose basename contains `python` selects `py`.

### 4.2 The two encodings, verbatim from source

```go
// kittens/ssh/main.go:497-508
if cd.script_type == "py" {
	encoded_script = base64.StdEncoding.EncodeToString(utils.UnsafeStringToBytes(cd.bootstrap_script))
	unwrap_script = `"import base64, sys; eval(compile(base64.standard_b64decode(sys.argv[-1]), 'bootstrap.py', 'exec'))"`
} else {
	// We can't rely on base64 being available on the remote system, so instead
	// we quote the bootstrap script by replacing ' and \ with \v and \f
	// also replacing \n and ! with \r and \b for tcsh
	// finally surrounding with '
	encoded_script = "'" + strings.NewReplacer("'", "\v", "\\", "\f", "\n", "\r", "!", "\b").Replace(cd.bootstrap_script) + "'"
	unwrap_script = `'eval "$(echo "$0" | tr \\\v\\\f\\\r\\\b \\\047\\\134\\\n\\\041)"' `
}
cd.rcmd = []string{"exec", cd.host_opts.Interpreter, "-c", unwrap_script, encoded_script}
```

### 4.3 Observed `rcmd` for both interpreters (canonical `kitten ssh`)

Both invocations go through the identical canonical `get_remote_command()` / `wrap_bootstrap_script()`; only the interpreter differs. I captured the exact `rcmd` argv byte-for-byte from real `kitten ssh` runs using the `PATH` shim's JSON argv log (`KSSH_SHIM_JSON`), once with the default `sh` interpreter and once with `--kitten interpreter=python3` (both `share_connections=no`, so the ssh argv is simply `-t -- localhost` followed by `rcmd`).

**`sh` (default) -> `script_type='sh'`:**

```
$ PATH=/tmp/kssh-obs/shim:$PATH KSSH_SHIM_JSON=/tmp/kssh-obs/cap/argv_sh.json \
    python3 /tmp/kssh-obs/ptydrv.py --kitty-pid 700001 --kitty-window-id 7 \
      --raw /tmp/kssh-obs/cap/sh.raw --log /tmp/kssh-obs/cap/sh.log \
      -- ./kitty/launcher/kitten ssh --kitten share_connections=no localhost
$ python3 - <<'EOF'
import json
arr=[json.loads(l) for l in open("/tmp/kssh-obs/cap/argv_sh.json") if l.strip()]
rcmd=next(a[a.index("exec"):] for a in arr if "exec" in a)
print("rcmd[0..2] =", rcmd[:3])
print("interpreter (rcmd[1]) =", repr(rcmd[1]))
print("unwrap  (rcmd[3], %d bytes) = %s" % (len(rcmd[3]), rcmd[3]))
print("encoded (rcmd[4]) length =", len(rcmd[4]))
print("encoded first 24 bytes hex =", ' '.join('%02x'%(ord(c)&0xff) for c in rcmd[4][:24]))
EOF
rcmd[0..2] = ['exec', 'sh', '-c']
interpreter (rcmd[1]) = 'sh'
unwrap  (rcmd[3], 67 bytes) = 'eval "$(echo "$0" | tr \\\v\\\f\\\r\\\b \\\047\\\134\\\n\\\041)"' 
encoded (rcmd[4]) length = 5207
encoded first 24 bytes hex = 27 23 08 2f 62 69 6e 2f 73 68 0d 23 20 43 6f 70 79 72 69 67 68 74 20 28
```

The unwrap string (`rcmd[3]`, 67 bytes) is **byte-for-byte identical** to the Go raw-string literal at `kittens/ssh/main.go:506` (verified by direct comparison: `captured == source506` is `True`). On the remote, `tr \\\v\\\f\\\r\\\b \\\047\\\134\\\n\\\041` decodes the octal codes as `\047`=`'`, `\134`=`\`, `\n`=newline, `\041`=`!` — i.e. `tr` maps the escape bytes `\v \f \r \b` **back** to `' \ <newline> !`, exactly reversing the `strings.NewReplacer` at `kittens/ssh/main.go:505`. The comment at `:503` notes the `\r`/`\b` mappings exist "for tcsh".

**`python3` -> `script_type='py'`:**

```
$ PATH=/tmp/kssh-obs/shim:$PATH KSSH_SHIM_JSON=/tmp/kssh-obs/cap/argv_py.json \
    python3 /tmp/kssh-obs/ptydrv.py --kitty-pid 700002 --kitty-window-id 7 \
      --raw /tmp/kssh-obs/cap/py.raw --log /tmp/kssh-obs/cap/py.log \
      -- ./kitty/launcher/kitten ssh --kitten interpreter=python3 --kitten share_connections=no localhost
$ python3 - <<'EOF'
import json, base64
arr=[json.loads(l) for l in open("/tmp/kssh-obs/cap/argv_py.json") if l.strip()]
rcmd=next(a[a.index("exec"):] for a in arr if "exec" in a)
print("rcmd[0..2] =", rcmd[:3])
print("interpreter (rcmd[1]) =", repr(rcmd[1]))
print("unwrap  (rcmd[3], %d bytes) = %s" % (len(rcmd[3]), rcmd[3]))
print("encoded (rcmd[4]) length =", len(rcmd[4]))
print("encoded head (base64) =", rcmd[4][:40])
print("base64 decodes to     =", base64.standard_b64decode(rcmd[4])[:22])
EOF
rcmd[0..2] = ['exec', 'python3', '-c']
interpreter (rcmd[1]) = 'python3'
unwrap  (rcmd[3], 100 bytes) = "import base64, sys; eval(compile(base64.standard_b64decode(sys.argv[-1]), 'bootstrap.py', 'exec'))"
encoded (rcmd[4]) length = 13484
encoded head (base64) = IyEvdXNyL2Jpbi9lbnYgcHl0aG9uCiMgTGljZW5z
base64 decodes to     = b'#!/usr/bin/env python\n'
```

The Python `unwrap` string (`rcmd[3]`, 100 bytes) is the literal at `kittens/ssh/main.go:499`; the encoded payload is plain standard base64 (head `IyEv...`, which decodes to `#!/usr/bin/env python\n`). Its length (13484) is larger than the `sh` form (5207) even though both wrap the same logical bootstrap — base64 inflates by ~4/3, whereas the `sh` scheme substitutes only four byte values in place and adds two wrapping quotes.

### 4.4 The `sh` substitutions, byte-exact (real canonical payload)

Counting the four `strings.NewReplacer` mappings (`kittens/ssh/main.go:505`) directly in the real 5207-byte `sh` payload captured above (`argv_sh.json`, `rcmd[4]`); labels are spelled out to keep the command copy-paste safe:

```
$ python3 - <<'EOF'
import json
arr=[json.loads(l) for l in open("/tmp/kssh-obs/cap/argv_sh.json") if l.strip()]
enc=next(a[a.index("exec"):] for a in arr if "exec" in a)[4]
b=bytes(ord(c)&0xff for c in enc)
print("payload length =", len(enc), "bytes")
print("wrapper: first byte = 0x%02x   last byte = 0x%02x   (0x27 = single quote)" % (b[0], b[-1]))
print("substituted-byte counts in payload:")
print("  0x0b VT (was single-quote ')  :", b.count(0x0b))
print("  0x0c FF (was backslash \\)     :", b.count(0x0c))
print("  0x0d CR (was newline)         :", b.count(0x0d))
print("  0x08 BS (was bang !)          :", b.count(0x08))
print("  0x0a LF (raw newline)         :", b.count(0x0a), " (must be 0)")
print("first 24 bytes hex =", ' '.join('%02x'%x for x in b[:24]))
EOF
payload length = 5207 bytes
wrapper: first byte = 0x27   last byte = 0x27   (0x27 = single quote)
substituted-byte counts in payload:
  0x0b VT (was single-quote ')  : 10
  0x0c FF (was backslash \)     : 25
  0x0d CR (was newline)         : 164
  0x08 BS (was bang !)          : 3
  0x0a LF (raw newline)         : 0  (must be 0)
first 24 bytes hex = 27 23 08 2f 62 69 6e 2f 73 68 0d 23 20 43 6f 70 79 72 69 67 68 74 20 28
```

Reading the first bytes: the payload opens with `0x27` (the wrapping `'`), then `0x23 0x08` — a `#` followed by `\b` (`0x08`) which was originally `!` — i.e. the shebang `#!/bin/sh` has its `!` replaced by `\b`, and its trailing newline replaced by `\r` (`0x0d`, at offset 10). Every raw newline is gone (`0x0a` count is **0** — all became `\r`), and the wrapper's first and last bytes are both `0x27`. This is the "wild character substitution" the question asks about, shown at the byte level on the real transmitted payload.

**Cause -> effect.** The `sh` scheme is deliberately dependency-free: by turning the four bytes that are dangerous inside a single-quoted string (`'`, `\`, newline, `!`) into control bytes that pass through cleanly, the entire multi-kilobyte bootstrap becomes one safe single-quoted argument, and a single portable `tr` call on the remote restores it — no remote `base64` required. The Python scheme can assume a Python interpreter, so it uses ordinary base64 plus an `eval(compile(base64.standard_b64decode(sys.argv[-1]), 'bootstrap.py', 'exec'))` one-liner.


---

## §5 — End‑to‑end trace  (Q5: "trace through what happens from when a user initiates an SSH session all the way to when the bootstrap actually executes on the remote side")

**Direct answer.** `main()` validates it is running inside kitty (`KITTY_WINDOW_ID`+`KITTY_PID`) with
a TTY on stdin, parses the SSH args (a pure‑`ssh` passthrough branch just `exec`s `ssh`), and calls
`run_ssh()`. `run_ssh()` parses the destination, loads per‑host config, adds the ControlMaster args
(§3), sets up askpass (deciding `need_to_request_data`, §3/§7), builds the tar + JSON envelope +
SHM (§1/§6), wraps the bootstrap (§4), takes the terminal to no‑echo, and `exec`s
`ssh … <rcmd>`. On the remote, the wrapped command unwraps into `bootstrap.sh`, detects a base64
helper, (optionally) requests the data over the TTY, reads and untars the archive, compiles terminfo,
and finally `exec`s the login shell with shell integration. Everything below is observed from one
**real** `./kitty/launcher/kitten ssh localhost` session driven under a PTY against loopback `sshd`.

### 5.1 Local: `main()` guards and dispatch

`main()` first parses the SSH argument vector with `ParseSSHArgs(args, "--kitten")`
(`kittens/ssh/main.go:810`), which yields `ssh_args`, `server_args`, and a `passthrough` flag. It then
applies three guards and dispatches — quoted contiguously (no elision):

```go
// kittens/ssh/main.go:822-831
	if passthrough {
		return 1, unix.Exec(SSHExe(), utils.Concat([]string{"ssh"}, ssh_args, server_args), os.Environ())
	}
	if os.Getenv("KITTY_WINDOW_ID") == "" || os.Getenv("KITTY_PID") == "" {
		return 1, fmt.Errorf("The SSH kitten is meant to run inside a kitty window")
	}
	if !tty.IsTerminal(os.Stdin.Fd()) {
		return 1, fmt.Errorf("The SSH kitten is meant for interactive use only, STDIN must be a terminal")
	}
	return run_ssh(ssh_args, server_args, found_extra_args)
```

The `passthrough` branch (a plain‑`ssh` invocation carrying no kitten‑specific args) simply `exec`s
`ssh`. Otherwise the kitten *requires* `KITTY_WINDOW_ID`+`KITTY_PID` and a TTY on stdin, then calls
`run_ssh()`. The destination is parsed by `get_destination()` (called from `run_ssh` at
`kittens/ssh/main.go:614`; defined at `:46`) into `(username, hostname_for_match)` — observed as
`root` / `localhost` in the §2 dump. Every capture in this document is proof the guards passed: the
PTY driver sets `KITTY_WINDOW_ID`/`KITTY_PID` and provides a real controlling terminal, exactly as a
kitty window would.

### 5.2 The real session: `kitten ssh localhost` → remote bash login shell

I drove the **real** `./kitty/launcher/kitten ssh localhost` binary under a PTY. The driver emulates
*only* the terminal end — it answers the remote's `@kitty-ssh` request through the repository's own
`get_ssh_data()` (`kittens/ssh/utils.py:115`) and `kitty.shm.SharedMemory`; `kitten`, `ssh`, the
loopback `sshd`, and the remote `bootstrap.sh` are all the real programs. It sends an `exit` command
once the shell‑integration prompt‑start marker (`OSC 133 ; A`) appears:

```
$ python3 /tmp/kssh-obs/ptydrv.py \
    --raw /tmp/kssh-obs/cap/default.raw --log /tmp/kssh-obs/cap/default.log \
    --kitty-pid 4242 --kitty-window-id 7 \
    --send-after ']133;A' 'printf "E2E_MARKER=%s\n" ok; exit' \
    -- ./kitty/launcher/kitten ssh localhost
```

**Ordered driver log** (timestamped milestones, verbatim):

```
=== driver pid=104474 kitty_pid=4242 window_id=7 request_id=4242-7
child argv: ./kitty/launcher/kitten ssh localhost
[+ 0.024s] DCS @kitty-ssh recv (payload 148 b64 bytes)
           SHM object BEFORE read : /dev/shm/kssh-104475-2WANDUJQLBXQI  exists=True mode=0o600 uid=0 gid=0 size=31535
           answering get_ssh_data(request_id='4242-7')
           get_ssh_data streamed 31560 bytes back to child
           SHM object AFTER read  : exists=False  (one-shot unlink @ utils.py:106)
           framing chunk[0]=b'\nKITTY_DATA_START\n' chunk[1]=b'OK\n' last=b'KITTY_DATA_END\n'
           stream: 124 data lines; distinct lengths=[158, 254]; first=254 last=158
           dumped tar 23549 bytes -> /tmp/kssh-obs/cap/default.tar (gzip magic 1f8b)
[+ 0.329s] marker ']133;A' seen; sending 'printf "E2E_MARKER=%s\\n" ok; exit'
[+ 2.347s] child exited status=None
DCS seen: {"ssh": 1, "ask": 0, "echo": 1, "other": 1}
```

**Raw terminal output** (`default.raw`, rendered with `cat -v`; the two credential‑bearing DCS
payloads are redacted — they are shown decoded byte‑exact in §7 — everything else is verbatim,
including the raw‑PTY carriage‑return artifact `<rintf` where a `\r` overwrote the leading `p`):

```
^[[?s^[[?19997h^[P@kitty-ssh|<@kitty-ssh request payload redacted; decoded byte-exact in §7>^[\^[P@kitty-print|<@kitty-print debug payload redacted; base64 of a bash HISTCONTROL notice>^[\^[]7;kitty-shell-cwd://reverse-code-generator-a966ddcf-cwgm9/root^G^[]133;k;start_kitty^G^[]133;D;0^G^[]133;A^G^[]133;k;end_kitty^Groot@reverse-code-generator-a966ddcf-cwgm9:~# ^[]133;k;start_suffix_kitty^G^[[5 q^[]2;reverse-code-generator-a966ddcf-cwgm9: ~^G^[]133;k;end_suffix_kitty^G^M<rintf "E2E_MARKER=%s\n" ok; exit                                                                                                                                                                           ^M^M<rintf "E2E_MARKER=%s\n" ok; exit^M
^[]2;reverse-code-generator-a966ddcf-cwgm9: printf "E2E_MARKER=%s\n" ok; exit^G^[]133;C;cmdline=printf\ \"E2E_MARKER=%s\\n\"\ ok\;\ exit^G^A^[]133;k;start_kitty^G^B^A^[]133;k;end_kitty^G^B^A^[]133;k;start_suffix_kitty^G^B^A^[[0 q^B^A^[]133;k;end_suffix_kitty^G^BE2E_MARKER=ok^M
logout^M
Shared connection to localhost closed.^M^M
^[P@kitty-echo|<@kitty-echo canary payload redacted; 64-hex sync token, not a credential>^[\^[[?r^[[?19997l
```

This is the whole trace in one shot, and it maps directly onto `run_ssh()`'s ordered steps
(`kittens/ssh/main.go:597`):

- `get_destination()` → `load_config(hostname_for_match, …)` (`:614`, `:619`).
- insert the ControlMaster args when sharing → `connection_sharing_args()` (`:642`, §3).
- `set_askpass()` sets up askpass and the initial `need_to_request_data`; the `ssh -O check` reuse
  gate can then flip it (`:651`, `:663‑664`, §3).
- open the controlling terminal with `SetNoEcho`, saving the original echo state (`:718`, `:722`).
- build the tar / JSON envelope / SHM and wrap the bootstrap via `get_remote_command()` (`:749`,
  §1/§4/§6).
- `exec` `ssh … <rcmd>` (`:753‑757`).
- in the default (`!request_data`) path, emit the data request itself with
  `tui.DCSToKitty("ssh", rq)` (`rq` built at `:762`, sent at `:766`), then drain any TTY garbage with
  an `@kitty-echo` canary via `tui.DCSToKitty("echo", canary)` (`:539`).

The two DCS strings visible at the head and tail of the raw trace (`@kitty-ssh` … `@kitty-echo`) are
exactly those two emissions. In between, the remote `bootstrap.sh` pulled the tar over the TTY (the
driver's `get_ssh_data()` streamed **31560 bytes**; the decoded tar is **23549 bytes**, gzip magic
`1f8b`), untarred it, and `exec`d the login shell: the `OSC 7` cwd report, the `OSC 133 ; A/;C/;D`
prompt/command markers, and the live bash prompt `root@reverse-code-generator-a966ddcf-cwgm9:~#` are
all emitted by the *integrated* shell. The injected command ran (`E2E_MARKER=ok`), the shell
`logout`ed, and OpenSSH printed `Shared connection to localhost closed.` The SHM lifecycle for this
same run — present at `0o600` before the read, unlinked after — is detailed in §6.

### 5.3 The repository's own end‑to‑end SSH test (independent corroboration)

The project ships an end‑to‑end SSH test that likewise drives the real `kitten ssh` over a PTY
against a loopback endpoint and exercises the remote bootstrap. Re‑running the whole module:

```
$ CI=true LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 ./kitty/launcher/kitty +launch test.py --module ssh
test_basic_pty_operations (kitty_tests.ssh.SSHKitten.test_basic_pty_operations) ... ok
test_ssh_bootstrap_with_different_launchers (kitty_tests.ssh.SSHKitten.test_ssh_bootstrap_with_different_launchers) ... ok
test_ssh_connection_data (kitty_tests.ssh.SSHKitten.test_ssh_connection_data) ... ok
test_ssh_copy (kitty_tests.ssh.SSHKitten.test_ssh_copy) ... ok
test_ssh_env_vars (kitty_tests.ssh.SSHKitten.test_ssh_env_vars) ... ok
test_ssh_leading_data (kitty_tests.ssh.SSHKitten.test_ssh_leading_data) ... ok
test_ssh_login_shell_detection (kitty_tests.ssh.SSHKitten.test_ssh_login_shell_detection) ... ok
test_ssh_shell_integration (kitty_tests.ssh.SSHKitten.test_ssh_shell_integration) ... ok

----------------------------------------------------------------------
Ran 8 tests in 17.040s

OK
```

The 8/8 pass is stable across runs; the absolute duration is instance‑specific (observed `10.660s`
and `17.040s` on two runs).

### 5.4 Remote: unwrap → base64 detect → data request → untar → terminfo → login shell

On the remote, `sshd` hands the wrapped `-c` argument to the user's shell, which unwraps it (the
`tr`/`base64` step of §4) into `bootstrap.sh`. The script then:

1. **Detects a base64 helper** via a fallback chain — `base64` → `openssl` → `b64encode` → Python →
   Perl → `die` (`shell-integration/ssh/bootstrap.sh:55‑72`); which helper wins on this endpoint is
   shown in §7.4.
2. **Requests data when `request_data="1"`** — turning off echo and emitting the `@kitty-ssh` DCS.
   The source template (before Go substitution) is:

   ```sh
   # shell-integration/ssh/bootstrap.sh:90-95
   request_data="REQUEST_DATA"
   trap "cleanup_on_bootstrap_exit" EXIT
   [ "$request_data" = "1" ] && {
       command stty "-echo" < /dev/tty
       dcs_to_kitty "ssh" "id="REQUEST_ID":pwfile="PASSWORD_FILENAME":pw="DATA_PASSWORD""
   }
   ```

   The real substituted form of this line — fresh (`request_data="1"`, real credentials) vs. reused
   (`request_data="0"`, literal placeholders) — is shown byte‑for‑byte in §3.3.
3. **Reads the stream and untars** — `read_base64_from_tty | base64_decode | tar xpzf - -C $tdir`,
   then `mv_files_and_dirs` into `$HOME` (`shell-integration/ssh/bootstrap.sh:104-113`, quoted in §1.6).
4. **Compiles terminfo** with `command tic -x -o "$1/$tname" "$1/.terminfo/kitty.terminfo"`
   (`shell-integration/ssh/bootstrap-utils.sh:44`).
5. **`exec`s the login shell** — `exec_login_shell` (called at `shell-integration/ssh/bootstrap.sh:164`;
   defined at `shell-integration/ssh/bootstrap-utils.sh:221`), which dispatches to
   `exec_zsh_with_integration` / `exec_fish_with_integration` / `exec_bash_with_integration`
   (definitions at `shell-integration/ssh/bootstrap-utils.sh:102,118,128`).

**Observed** — the raw trace in §5.2 *is* the proof these remote steps executed: the `OSC 133`
markers and the `root@…:~#` bash prompt can only appear after `exec_login_shell` reached
`exec_bash_with_integration` (the login shell on this endpoint is `/bin/bash`), and
`Shared connection to localhost closed.` confirms the whole session ran and tore down cleanly.

**Cause → effect.** The local side never runs a shell on the remote directly; it hands `sshd` a
single self‑unwrapping `-c` argument. That argument reconstitutes the bootstrap, which pulls the
archive over the very TTY it is attached to, lays down shell integration + terminfo, and only then
replaces itself with the user's login shell — so by the time the user sees a prompt, integration is
already in place.


---

## §6 — Shared‑memory credential security  (Q6: "how the shared memory piece keeps things secure")

**Direct answer.** Two guarantees, one universal and one path‑dependent. **(1) The archive is never on any command line** — in both paths the base64 JSON envelope (tar + password) is written into a POSIX shared‑memory object created at mode **`0600`** in `/dev/shm` and streamed to the remote over the TTY only *after* validation. **(2) Whether the *password* touches a command line depends on `request_data`** (`kittens/ssh/main.go:649`, decided by askpass/reuse — §3/§7): in the **default `request_data=false` path** the remote command line carries only the literal placeholders `REQUEST_ID`/`PASSWORD_FILENAME`/`DATA_PASSWORD`, and the real password is delivered by the *local* kitten via an `@kitty-ssh` DCS over its own controlling terminal (`kittens/ssh/main.go:761‑766`) — so it never appears in any remote process's `argv`; in the **`request_data=true` path** (no shared master to reuse) the real password *is* substituted into the wrapped bootstrap and therefore does appear on the **remote** command line briefly (visible via `ps` on the remote during bootstrap) so the remote can echo it back. In every path, before releasing the tar the local responder validates the object's **owner** (`st_uid`/`st_gid` must equal the current euid/egid), validates its **permissions** (must be exactly `0o600`), **unlinks** the object immediately (one‑shot), and checks the supplied `pw` and `request_id` match. Any mismatch raises and aborts.

### 6.1 Fresh random password per session (`secrets.TokenHex()`)

```go
// kittens/ssh/main.go:431
pw, err := secrets.TokenHex()
```

**Observed** — two real `kitten ssh` runs produce different passwords. The raw 64‑hex values are credentials and are redacted; their SHA‑256 digests (one‑way, safe to print) differ, and the script confirms the raw values differ without printing them:

```
$ python3 - <<'PY'   # extract pw from two real captured kitten-ssh bootstrap DCS args; compare
import re, hashlib, io
def pw(p):
    b = io.open(p, errors='replace').read()
    return re.findall(r'[0-9a-f]{64}', b)[0]   # the sole 64-hex per capture is the session password
a = pw('/tmp/kssh-obs/cap/argv_pw1.json')
b = pw('/tmp/kssh-obs/cap/argv_pw2.json')
print('run1 pw = <redacted 64-hex, instance-specific>  sha256[:16]=', hashlib.sha256(a.encode()).hexdigest()[:16])
print('run2 pw = <redacted 64-hex, instance-specific>  sha256[:16]=', hashlib.sha256(b.encode()).hexdigest()[:16])
print('differ  =', a != b, ' (each raw pw is 32 bytes = 64 hex chars)')
PY
run1 pw = <redacted 64-hex, instance-specific>  sha256[:16]= 1a1e9c64b62b5698
run2 pw = <redacted 64-hex, instance-specific>  sha256[:16]= c3f00e52265af391
differ  = True  (each raw pw is 32 bytes = 64 hex chars)
```

### 6.2 The SHM object: created, written with a size prefix, mode `0600`

```go
// kittens/ssh/main.go:446-450
		data_shm, err = shm.CreateTemp(fmt.Sprintf("kssh-%d-", os.Getpid()), uint64(len(encoded_data)+8))
		if err == nil {
			err = shm.WriteWithSize(data_shm, encoded_data, 0)
			if err == nil {
				err = data_shm.Flush()
```

The Go backend creates the tmpfs object with `O_EXCL|O_CREATE|O_RDWR` at mode `0600` in `/dev/shm`
(`tools/utils/shm/shm_fs.go:130`, dir `tools/utils/shm/specific_linux.go:11`); the Python side uses
the same `0o600` default (`kitty/shm.py:51`). The `WriteWithSize` size prefix is a 4‑byte big‑endian
length (Go `binary.BigEndian.PutUint32`; matches Python's `'!I'` format), which is why the local
responder can `read_data_with_size()` back exactly.

**Observed (canonical runtime — the same `kitten ssh` session as §5.2).** The driver stat’d the object the instant the `@kitty-ssh` request arrived, *before* answering it — it exists at `0o600`, owned by the invoking uid:

```
$ sed -n '3,4p' /tmp/kssh-obs/cap/default.log     # driver log of the §5.2 real kitten-ssh session
[+ 0.024s] DCS @kitty-ssh recv (payload 148 b64 bytes)
           SHM object BEFORE read : /dev/shm/kssh-104475-2WANDUJQLBXQI  exists=True mode=0o600 uid=0 gid=0 size=31535
```

### 6.3 One‑shot unlink on read

The local responder opens the object read‑only and **unlinks it immediately**, then validates:

```python
# kittens/ssh/utils.py:100-112
def read_data_from_shared_memory(shm_name: str) -> Any:
    import json
    import stat

    from kitty.shm import SharedMemory
    with SharedMemory(shm_name, readonly=True) as shm:
        shm.unlink()
        if shm.stats.st_uid != os.geteuid() or shm.stats.st_gid != os.getegid():
            raise ValueError(f'Incorrect owner on pwfile: uid={shm.stats.st_uid} gid={shm.stats.st_gid}')
        mode = stat.S_IMODE(shm.stats.st_mode)
        if mode != stat.S_IREAD | stat.S_IWRITE:
            raise ValueError(f'Incorrect permissions on pwfile: 0o{mode:03o}')
        return json.loads(shm.read_data_with_size())
```

**Observed (canonical runtime — same session).** Immediately after the driver’s `get_ssh_data()` streamed the tar back to the child, the object is gone:

```
$ sed -n '6,7p' /tmp/kssh-obs/cap/default.log
           get_ssh_data streamed 31560 bytes back to child
           SHM object AFTER read  : exists=False  (one-shot unlink @ utils.py:106)
```

### 6.4 The owner and permission guards actually fire

Owner/permission tampering cannot arise in a normal canonical run, so these two guards are driven by a `[SUPPLEMENTAL]` read‑only harness (`/tmp/kssh-obs/q6_guards.py`) that `import`s and calls the **real** `kittens.ssh.utils.get_ssh_data` and `read_data_from_shared_memory` (no reimplementation), tampering throwaway `/dev/shm` objects. Its full stdout follows — the owner/permission guards are rows `[4]`/`[5]`; the `pw`/`request_id` rejections of §6.5 are rows `[2]`/`[3]` of the same run:

```
$ REPO=$PWD python3 /tmp/kssh-obs/q6_guards.py
=== [SUPPLEMENTAL harness — REAL kittens/ssh/utils.py functions] ===

[1] HAPPY PATH (correct pw + id + 0600 + owner):
    first markers : ['', 'KITTY_DATA_START']
    OK present    : True
    tar streamed  : 472 chars after OK (env tar was 456 chars)
    NOTE responder yields KITTY_DATA_START then OK then 254-byte lines (utils.py:116,135,143)

[2] WRONG PASSWORD:
    ['Incorrect password'] 

[3] WRONG REQUEST ID (responder expected 'MISMATCH-99', DCS carried id=4242-7):
    ["Incorrect request id: '4242-7' expecting the KITTY_PID-KITTY_WINDOW_ID for the current kitty window"] 

[4] WRONG PERMISSIONS (0o644): Incorrect permissions on pwfile: 0o644

[5] WRONG OWNER (chown->uid=1 gid=1): Incorrect owner on pwfile: uid=1 gid=1
```

These are the exact error strings at `kittens/ssh/utils.py:108` and `:111`.

### 6.5 Password and request‑id checks in `get_ssh_data()`

Even with a valid object, the responder refuses to release the tar unless the caller‑supplied `pw`
and `request_id` match:

```python
# kittens/ssh/utils.py:129-133
            env_data = read_data_from_shared_memory(pwfilename)
            if pw != env_data['pw']:
                raise ValueError('Incorrect password')
            if rq_id != request_id:
                raise ValueError(f'Incorrect request id: {rq_id!r} expecting the KITTY_PID-KITTY_WINDOW_ID for the current kitty window')
```

**Observed** — both checks reject bad input:

```
# rows [2] and [3] of the q6_guards.py run shown in §6.4 (same [SUPPLEMENTAL] harness):
[2] WRONG PASSWORD:
    ['Incorrect password'] 
[3] WRONG REQUEST ID (responder expected 'MISMATCH-99', DCS carried id=4242-7):
    ["Incorrect request id: '4242-7' expecting the KITTY_PID-KITTY_WINDOW_ID for the current kitty window"] 
```

with the two raised `ValueError`s captured verbatim on the harness’s stderr (one per rejected input):

```
ValueError: Incorrect password
ValueError: Incorrect request id: '4242-7' expecting the KITTY_PID-KITTY_WINDOW_ID for the current kitty window
```

(There is also a Go‑side equivalent `read_data_from_shared_memory` in `kittens/ssh/main.go:72-85`
with the analogous "Incorrect owner on SHM file"/"Incorrect permissions on SHM file" guards, used
when the kitten reads an SHM object itself.)


### 6.6 Where the password actually travels — the two paths (`request_data`)

The tar envelope is *always* confined to the local SHM object; the *password's* exposure, however, depends on `request_data`. `bootstrap_script()` puts the real credentials in `sensitive_data` (`kittens/ssh/main.go:460`), then copies that map into the bootstrap‑script substitutions (`sd`) **only when `cd.request_data` is true**, while *always* copying it into `cd.replacements`:

```go
// kittens/ssh/main.go:475-482
	sd := maps.Clone(replacements)
	if cd.request_data {
		maps.Copy(sd, sensitive_data)
	}
	maps.Copy(replacements, sensitive_data)
	cd.replacements = replacements
	cd.bootstrap_script = utils.UnsafeBytesToString(shell_integration.Data()["shell-integration/ssh/bootstrap."+cd.script_type].Data)
	cd.bootstrap_script = prepare_script(cd.bootstrap_script, sd)
```

`sd` feeds `prepare_script` — which produces the bootstrap that becomes the **remote** `ssh -c` argument — so the real credentials reach the remote `argv` only when `cd.request_data` is true (`:476‑477`). `cd.replacements` always holds them (`:479`), and in the default path the kitten emits them itself over its **local** controlling terminal:

```go
// kittens/ssh/main.go:761-766
	if !cd.request_data {
		rq := fmt.Sprintf("id=%s:pwfile=%s:pw=%s", cd.replacements["REQUEST_ID"], cd.replacements["PASSWORD_FILENAME"], cd.replacements["DATA_PASSWORD"])
		err := term.ApplyOperations(tty.TCSANOW, tty.SetNoEcho)
		if err == nil {
			var dcs string
			dcs, err = tui.DCSToKitty("ssh", rq)
```

**Observed** — the `PATH` shim that captured the wrapped bootstrap `argv` confirms both paths (the full captures are in §3.3):

- `request_data="1"` (fresh, no master to reuse): the real credentials are substituted into the remote bootstrap's `dcs_to_kitty "ssh" …` line — `id="…":pwfile="kssh-…":pw="<redacted 64‑hex, instance-specific>"` — i.e. the password is on the **remote** command line (briefly visible via `ps` on the remote).
- `request_data="0"` (default / reused master): the same line carries the **literal placeholders** `id="REQUEST_ID":pwfile="PASSWORD_FILENAME":pw="DATA_PASSWORD"`; the real password is delivered only by the local `@kitty-ssh` DCS (`kittens/ssh/main.go:766`, the `@kitty-ssh` at the head of the §5.2 raw trace) and never appears in any remote `argv`.

**Cause → effect.** The archive itself never touches a command line in either path — it lives in an owner‑only `0600` tmpfs object and is streamed over the TTY only after validation — so a co‑located unprivileged user cannot recover the *tar* from `ps`/`/proc`. The *password* is likewise kept off the command line in the default `request_data=false` path (delivered by the local `@kitty-ssh` DCS); only in the `request_data=true` path is it briefly present on the **remote** `argv` so the remote can echo it back. In both paths the object is unlinked on first read (a captured name is useless afterward) and the responder verifies owner, permissions, `pw`, and `request_id`, so a forged or tampered object is rejected before any tar byte is emitted. The password is freshly minted per session, so replay across sessions fails.


---

## §7 — Bidirectional TTY handshake  (Q7: "how the terminal communicates back and forth with the remote shell during setup")

**Direct answer.** All back‑and‑forth happens over the controlling TTY (`/dev/tty`) using two of
kitty's DCS (Device Control String) escape protocols, each framed as `ESC P @kitty-<verb>|<payload>
ESC \`. **(a)** `@kitty-ssh` carries the data request `id=…:pwfile=…:pw=…`; kitty routes it to the
local responder `get_ssh_data()`. **(b)** `@kitty-ask` carries an askpass shared‑memory object name;
kitty reads the question from that object, prompts the user, writes the answer back, and flips a
sentinel byte the kitten is polling. Terminal echo is turned off (`stty -echo`) around the handshake
and restored (`stty echo`) on exit. Which side *originates* the `@kitty-ssh` request depends on
`request_data` (remote when true; local when false — see §3), which I exercise both ways below.

### 7.1 The DCS framer

```sh
# shell-integration/ssh/bootstrap.sh:75
dcs_to_kitty() { printf "\033P@kitty-$1|%s\033\134" "$(printf "%s" "$2" | base64_encode)" > /dev/tty; }
```

Note it **base64‑encodes the payload** before framing — so the `@kitty-ssh` payload on the wire is
base64. (`\033`=ESC, `\134`=`\`, i.e. the `ESC \` string terminator.) Because that base64 blob
encodes the credential, it is masked in the captures below; the **decoded** payload is shown
byte‑exact with only the 64‑hex `pw` redacted.

### 7.2 `@kitty-ssh` data request — byte‑exact, both origins

The **remote** issues the request from this gate (byte‑exact from source); the `REQUEST_DATA` /
`REQUEST_ID` / `PASSWORD_FILENAME` / `DATA_PASSWORD` tokens are the `replacements` substituted by
`bootstrap_script()` (§2.3):

```sh
# shell-integration/ssh/bootstrap.sh:90-95
request_data="REQUEST_DATA"
trap "cleanup_on_bootstrap_exit" EXIT
[ "$request_data" = "1" ] && {
    command stty "-echo" < /dev/tty
    dcs_to_kitty "ssh" "id="REQUEST_ID":pwfile="PASSWORD_FILENAME":pw="DATA_PASSWORD""
}
```

The **local** counterpart is `tui.DCSToKitty("ssh", rq)` at `kittens/ssh/main.go:766`, emitted only
under the `if !cd.request_data` guard at `kittens/ssh/main.go:760`. So exactly one side emits the
frame, selected by `request_data`. I exercised both, capturing the local PTY master with `ptydrv.py`
(a faithful terminal that answers `@kitty-ssh` via the repo's own `get_ssh_data()`; kitten + ssh +
remote `sshd` + remote `bootstrap.sh` are all real).

**(i) `request_data=true` → the REMOTE bootstrap issues it.** Forced with `--kitten askpass=ssh`,
which makes `use_kitty_askpass=false` (`kittens/ssh/main.go:648`) so `set_askpass()` is never called
and `need_to_request_data` stays `true` (`kittens/ssh/main.go:649`). A `PATH` shim in front of the
real `ssh` records the substituted bootstrap's `request_data` value:

```
$ PATH=/tmp/kssh-obs/shim:$PATH KSSH_SHIM_JSON=/tmp/kssh-obs/cap/q7r.json \
    timeout 60 python3 /tmp/kssh-obs/ptydrv.py \
      --raw /tmp/kssh-obs/cap/q7r.raw --log /tmp/kssh-obs/cap/q7r.drvlog \
      --kitty-pid 4242 --kitty-window-id 7 --send-after ']133;A' 'exit' \
      -- ./kitty/launcher/kitten ssh --kitten askpass=ssh --kitten share_connections=no localhost
$ # shim invocation 1 (the bootstrap ssh command; argc=8 because share_connections=no drops the ControlMaster opts):
invocation 1: argc=8; contains request_data="1": True
```

The frame appears on the TTY only *after* the ssh round‑trip and the remote `bootstrap.sh` reaching
its gate — the driver timestamps it at **+0.216s**:

```
[+ 0.216s] DCS @kitty-ssh recv (payload 148 b64 bytes)
```

Byte‑exact frame (envelope verbatim; the base64 blob encodes the credential so it is masked; decoded
payload shown with only the 64‑hex `pw` redacted):

```
^[P@kitty-ssh|<base64 payload, 148 chars — encodes the decoded line below>^[\
  decoded: id=4242-7:pwfile=kssh-157155-K5EPSGVT5WJ64:pw=<redacted 64-hex, instance-specific>
```

**(ii) `request_data=false` (default here — this OpenSSH supports askpass‑require, §5) → the LOCAL
kitten issues it** via `tui.DCSToKitty("ssh", rq)` (`kittens/ssh/main.go:766`). Same driver, default
invocation; the kitten writes the frame *before* it `exec`s `ssh`, so the driver timestamps it at
**+0.024s**:

```
$ timeout 60 python3 /tmp/kssh-obs/ptydrv.py \
      --raw /tmp/kssh-obs/cap/default.raw --log /tmp/kssh-obs/cap/default.log \
      --kitty-pid 4242 --kitty-window-id 7 --send-after ']133;A' 'printf "E2E_MARKER=%s\n" ok; exit' \
      -- ./kitty/launcher/kitten ssh localhost
[+ 0.024s] DCS @kitty-ssh recv (payload 148 b64 bytes)
```

```
^[P@kitty-ssh|<base64 payload, 148 chars — encodes the decoded line below>^[\
  decoded: id=4242-7:pwfile=kssh-104475-2WANDUJQLBXQI:pw=<redacted 64-hex, instance-specific>
```

**Origin is proven by `request_data` + timing, NOT by byte position.** Both frames sit at the same
early offset (~byte 13) because `ssh` emits nothing during connect, so the remote's first output
lands immediately after the local kitten's terminal‑setup escapes `^[[?s^[[?19997h` — the
`HANDLE_TERMIOS_SIGNALS` private mode `19997` (`kittens/tui/operations.py:48`), emitted locally and
present in neither `ptydrv.py` nor `bootstrap.sh`. What actually distinguishes the two is (a) the
shim's `request_data` value and (b) the **+0.024s** (local, pre‑`exec`) vs **+0.216s** (remote,
post‑connect) arrival time.

**Stability (observed, two local runs).** The envelope `^[P@kitty-ssh|<b64>^[\` and the payload
structure `id=<KITTY_PID>-<KITTY_WINDOW_ID>:pwfile=<shm_name>:pw=<64hex>` are invariant; only the
per‑run values (and hence the base64 length) vary:

```
default.raw  (run 1): id=4242-7  :pwfile=kssh-104475-2WANDUJQLBXQI:pw=<redacted 64-hex>  (b64 148)
default2.raw (run 2): id=812001-7:pwfile=kssh-111145-MPQXC2YQZPNIE:pw=<redacted 64-hex>  (b64 152)
```

### 7.3 Local dispatch: `@kitty-ssh` → `get_ssh_data()`

kitty's VT parser routes DCS `@kitty-ssh|` to `handle_remote_ssh` and `@kitty-ask|` to
`handle_remote_askpass` (`kitty/vt-parser.c:608-609`). The SSH handler calls the local responder with
the receiving window's identity:

```python
# kitty/window.py:1289-1292
def handle_remote_ssh(self, msg: memoryview) -> None:
    from kittens.ssh.utils import get_ssh_data
    for line in get_ssh_data(msg, f'{os.getpid()}-{self.id}'):
        self.write_to_child(line)
```

The `f'{os.getpid()}-{self.id}'` is exactly the `request_id` the responder checks against (§6.5) —
tying the DCS request back to the SHM credential.

### 7.4 The base64‑helper fallback chain (a named "e.g." item)

The remote picks its base64 implementation from an ordered chain; the **first** available wins
(byte‑exact from source):

```sh
# shell-integration/ssh/bootstrap.sh:55-72
if command -v base64 > /dev/null 2> /dev/null; then
    base64_encode() { command base64 | command tr -d \\n\\r; }
    base64_decode() { command base64 -d; }
elif command -v openssl > /dev/null 2> /dev/null; then
    base64_encode() { command openssl enc -A -base64; }
    base64_decode() { command openssl enc -A -d -base64; }
elif command -v b64encode > /dev/null 2> /dev/null; then
    base64_encode() { command b64encode - | command sed '1d;$d' | command tr -d \\n\\r; }
    base64_decode() { command fold -w 76 | command b64decode -r; }
elif detect_python; then
    pybase64() { command "$python" -c "import sys, base64; getattr(sys.stdout, 'buffer', sys.stdout).write(base64.standard_b64$1(getattr(sys.stdin, 'buffer', sys.stdin).read()))"; }
    base64_encode() { pybase64 "encode"; }
    base64_decode() { pybase64 "decode"; }
elif detect_perl; then
    base64_encode() { command "$perl" -MMIME::Base64 -0777 -ne 'print encode_base64($_)'; }
    base64_decode() { command "$perl" -MMIME::Base64 -ne 'print decode_base64($_)'; }
else
    die "base64 executable not present on remote host, ssh kitten cannot function."
```

**Observed** — on this endpoint `base64` is present, so it is chosen (openssl is also present but the
chain stops at the first match at L55):

```
$ for h in base64 openssl b64encode; do printf "  command -v %-10s -> " "$h"; command -v "$h" 2>/dev/null || echo "(absent)"; done
  command -v base64     -> /usr/bin/base64
  command -v openssl    -> /usr/bin/openssl
  command -v b64encode  -> (absent)
  => FIRST match wins: 'base64' present -> base64_encode/base64_decode use 'base64' (L55-57)
```

### 7.5 `@kitty-ask` askpass handshake — byte‑exact, all three sub‑types

`trigger_ask()` writes the **raw** SHM object name (no base64, unlike `@kitty-ssh`):

```go
// kittens/ssh/askpass.go:24-35
func trigger_ask(name string) {
	term, err := tty.OpenControllingTerm()
	if err != nil {
		fatal(err)
	}
	defer term.Close()
	_, err = term.WriteString("\x1bP@kitty-ask|" + name + "\x1b\\")
	if err != nil {
		fatal(err)
	}

}
```

`RunSSHAskpass()` derives the question, writes it to a fresh SHM object with a sentinel byte set to
`0`, triggers the DCS, then polls the sentinel until the local terminal flips it to `1`:

```go
// kittens/ssh/askpass.go:37-75
func RunSSHAskpass() {
	msg := os.Args[len(os.Args)-1]
	prompt := os.Getenv("SSH_ASKPASS_PROMPT")
	is_confirm := prompt == "confirm"
	q_type := "get_line"
	if is_confirm {
		q_type = "confirm"
	}
	is_fingerprint_check := strings.Contains(msg, "(yes/no/[fingerprint])")
	q := map[string]any{
		"message":     msg,
		"type":        q_type,
		"is_password": !is_fingerprint_check,
	}
	data, err := json.Marshal(q)
	if err != nil {
		fatal(err)
	}
	data_shm, err := shm.CreateTemp("askpass-*", uint64(len(data)+32))
	if err != nil {
		fatal(fmt.Errorf("Failed to create SHM file with error: %w", err))
	}
	defer data_shm.Close()
	defer func() { _ = data_shm.Unlink() }()

	data_shm.Slice()[0] = 0
	if err = shm.WriteWithSize(data_shm, data, 1); err != nil {
		fatal(fmt.Errorf("Failed to write to SHM file with error: %w", err))
	}
	if err = data_shm.Flush(); err != nil {
		fatal(fmt.Errorf("Failed to flush SHM file with error: %w", err))
	}
	trigger_ask(data_shm.Name())
	for {
		time.Sleep(50 * time.Millisecond)
		if data_shm.Slice()[0] == 1 {
			break
		}
	}
```

The `type` and `is_password` fields follow `kittens/ssh/askpass.go:40-50`: `type="confirm"` iff
`SSH_ASKPASS_PROMPT=="confirm"` (else `"get_line"`), and `is_password = !is_fingerprint_check`, where
`is_fingerprint_check` is true iff the message contains `"(yes/no/[fingerprint])"`.

**Observed** — the real `RunSSHAskpass` entry (`KITTY_KITTEN_RUN_MODULE=ssh_askpass`), driven for
each sub‑type; the driver reads the question JSON from the SHM object (at offset 1, after the
sentinel) and reports the byte‑exact `@kitty-ask` object name and the sentinel state:

```
--- get_line (SSH_ASKPASS_PROMPT unset; not a fingerprint) ---   [cap/ask_get_line.log]
  child argv: ./kitty/launcher/kitten Enter passphrase for key '/root/.ssh/id_ed25519':
  DCS @kitty-ask recv name='askpass-VUBPWPGFDTUX4'      (raw frame: ^[P@kitty-ask|askpass-VUBPWPGFDTUX4^[\)
  askpass sentinel byte is 0 before answer (askpass.go:62)
  askpass question JSON: {"is_password": true, "message": "Enter passphrase for key '/root/.ssh/id_ed25519':", "type": "get_line"}
--- confirm (SSH_ASKPASS_PROMPT=confirm; not a fingerprint) ---   [cap/ask_confirm.log]
  child argv: ./kitty/launcher/kitten Please confirm the operation (yes/no)?
  DCS @kitty-ask recv name='askpass-BKSG6UMXX4YIW'      (raw frame: ^[P@kitty-ask|askpass-BKSG6UMXX4YIW^[\)
  askpass sentinel byte is 0 before answer (askpass.go:62)
  askpass question JSON: {"is_password": true, "message": "Please confirm the operation (yes/no)?", "type": "confirm"}
--- fingerprint (SSH_ASKPASS_PROMPT=confirm; IS a fingerprint) ---   [cap/ask_fingerprint.log]
  child argv: ./kitty/launcher/kitten The authenticity of host 'localhost' can't be established.
  ED25519 key fingerprint is SHA256:abcdEXAMPLEfingerprintNOTreal.
  Are you sure you want to continue connecting (yes/no/[fingerprint])?
  DCS @kitty-ask recv name='askpass-CF25MDEPUTZZA'      (raw frame: ^[P@kitty-ask|askpass-CF25MDEPUTZZA^[\)
  askpass sentinel byte is 0 before answer (askpass.go:62)
  askpass question JSON: {"is_password": false, "message": "The authenticity of host 'localhost' can't be established.\nED25519 key fingerprint is SHA256:abcdEXAMPLEfingerprintNOTreal.\nAre you sure you want to continue connecting (yes/no/[fingerprint])?", "type": "confirm"}
```

The observed JSON matches the code exactly: `confirm`/`fingerprint` set `type="confirm"` (both were
invoked with `SSH_ASKPASS_PROMPT=confirm`, as OpenSSH does for host‑key confirmation), while the
fingerprint message — containing `(yes/no/[fingerprint])` — sets `is_password=false`; the passphrase
prompt (`SSH_ASKPASS_PROMPT` unset) stays `type="get_line"` with `is_password=true`. The object name
is `askpass-<random>` (`CreateTemp("askpass-*")`, `kittens/ssh/askpass.go:55`) and differs per run;
the sentinel is `0` while awaiting the answer (`kittens/ssh/askpass.go:62`, polled at `:72`). On the
receiving side, `handle_remote_askpass` (`kitty/window.py:1351-1378`) reads the question at offset 1,
prompts, writes the response at offset 1, and flips the sentinel byte to `\x01` at offset 0.

### 7.6 Echo toggling around the handshake

Echo state is a substituted replacement (`echo_on="ECHO_ON"` → `"1"`, §2.3), disabled while the
request is issued and restored in the exit trap (each block byte‑exact from source):

```sh
# shell-integration/ssh/bootstrap.sh:8   (echo_on is the substituted ECHO_ON replacement)
echo_on="ECHO_ON"
```

```sh
# shell-integration/ssh/bootstrap.sh:10-11   (exit trap restores echo)
cleanup_on_bootstrap_exit() {
    [ "$echo_on" = "1" ] && command stty "echo" 2> /dev/null < /dev/tty
```

```sh
# shell-integration/ssh/bootstrap.sh:92-93   (request gate disables echo)
[ "$request_data" = "1" ] && {
    command stty "-echo" < /dev/tty
```

**Observed (discrete before/after — the exact toggles the bootstrap runs).** A temporary PTY harness
(removed afterwards — see [§Cleanup](#cleanup)) runs the two commands from `bootstrap.sh` —
`command stty "-echo" < /dev/tty` (`shell-integration/ssh/bootstrap.sh:93`) then
`command stty "echo" < /dev/tty` (`shell-integration/ssh/bootstrap.sh:11`) — reading the termios
`ECHO` lflag directly at each stage. The off→on transition is captured discretely and is stable
across two runs:

```
$ cat /tmp/kssh-obs/echo_state.sh
#!/bin/sh
# mirrors shell-integration/ssh/bootstrap.sh echo toggles (:93 disable, :11 restore)
show() { python3 -c 'import termios; print("echo (ECHO on)" if termios.tcgetattr(0)[3] & termios.ECHO else "-echo (ECHO off)")' < /dev/tty; }
printf 'state BEFORE (initial)        : '; show
command stty "-echo" < /dev/tty     # bootstrap.sh:93 - disable echo before the handshake
printf 'state AFTER  stty -echo       : '; show
command stty "echo"  < /dev/tty     # bootstrap.sh:11 - exit trap restores echo
printf 'state AFTER  stty echo        : '; show

$ python3 /tmp/kssh-obs/run_pty.py    # pty.fork() → exec /bin/sh echo_state.sh under a PTY; read master to EOF
state BEFORE (initial)        : echo (ECHO on)
state AFTER  stty -echo       : -echo (ECHO off)
state AFTER  stty echo        : echo (ECHO on)
```

This is the discrete `stty`-attribute before/after snapshot for the echo off→on transition, on par
with the SHM (`0o600`→unlinked, §6) and `need_to_request_data` (true→false, §3.3) state captures.

On the local side the kitten opens the controlling terminal with `SetNoEcho`
(`kittens/ssh/main.go:718`, §5). So echo is off during the credential/askpass exchange
(before → `stty -echo`) and on again afterward (after → `stty echo`).

**Cause → effect.** The DCS protocol lets the remote and the local kitty exchange structured
messages *inline on the same byte stream* as the shell session, with echo suppressed so the escape
frames and any typed secret never appear on screen. `@kitty-ssh` pulls the archive credential;
`@kitty-ask` proxies OpenSSH's password/confirm/fingerprint prompts through kitty's native askpass —
all without a second channel.


---

## Coverage Pass

Every sub‑question and every named item / "e.g. / such as" example, mapped to the section and the
observed evidence that answers it.

### Q1 — Archive build & TTY transfer → §1

| Named item | Where | Observed evidence |
|------------|-------|-------------------|
| `make_tarfile()` | §1.1 | `kittens/ssh/main.go:255`; runtime member listing |
| gzip **BestCompression** | §1.1 | `main.go:259`; first bytes `1f8b0800` |
| **PAX** format | §1.1 | `main.go:297,310`; "PAX format? True" |
| `data.sh` | §1.2 | `main.go:321`; member `data.sh` (195 B) |
| `bootstrap-utils.sh` (sh only) | §1.2 | `main.go:324-325`; member present for `sh` |
| `FilesMatching` exclusions (`shell-integration/ssh/.+`, `zsh/kitty.zsh`) | §1.2 | `main.go:332-333`; those paths absent from listing |
| wrapper binaries (`kitty/version`, `kitty/bin/{kitty,kitten}`, `Remote_kitty!=no`) | §1.2 | `main.go:342-353`; members at `0o755` |
| terminfo (`kitty.terminfo`, `x/<DefaultTermName>`) | §1.2 | `main.go:355,357`; `x/xterm-kitty` observed |
| `go:embed` source | §1.3 | `tools/tui/shell_integration/data.go:19-20,49` |
| JSON envelope `{tarfile,pw,hostname,username}` | §1.4 | `main.go:439-442`; observed keys |
| 254‑byte TTY streaming, `KITTY_DATA_START`/`OK`/`END` | §1.5 | `utils.py:117,138,143,148`; observed frames + 254 lengths, stable ×2 |
| **no scp/sftp** | §1.7 | shim log (only `ssh` argv); TTY DCS path |

### Q2 — Per‑connection state → §2

All 16 `connection_data` fields (`remote_args`, `host_opts`, `hostname_for_match`, `username`,
`echo_on`, `request_data`, `literal_env`, `listen_on`, `test_script`, `dont_create_shm`, `shm_name`,
`script_type`, `rcmd`, `replacements`, `request_id`, `bootstrap_script`) — §2.1 table
(`main.go:171-189`), dumped for a real run. `request_id` default `KITTY_PID-KITTY_WINDOW_ID` — §2.2
(`main.go:424`; observed `"555003-7"`). `replacements` map (all 8 keys incl. `EXPORT_HOME_CMD`,
`EXEC_CMD`, `TEST_SCRIPT`, `REQUEST_DATA`, `ECHO_ON`, `REQUEST_ID`, `DATA_PASSWORD`,
`PASSWORD_FILENAME`) — §2.3 (`main.go:461-479`).

### Q3 — Fresh vs. reused → §3

`share_connections` default yes — §3.1 (`main.py:183`). `ControlMaster`/`ControlPath`/
`ControlPersist` (+ `ServerAliveInterval=60`/`ServerAliveCountMax=5`/`TCPKeepAlive=no`) — §3.1
(`main.go:137-143`; observed argv). `%C` — §3.1 (`main.go:136`; observed socket hash
`b8f03188…`). `ssh_control_master_template` — §3.1 (`kitty/constants.py:188`).
`/tmp/kssh-rdir-<euid>` symlink for over‑long runtime dir — §3.2 (`main.go:128-133`; observed).
`master_is_functional()` / `ssh -O check` — §3.3 (`main.go:653-662`; observed `rc=255`→`rc=0`).
`need_to_request_data` flip — §3.3 (`main.go:663-664`; observed `request_data "1"→"0"` + local
`@kitty-ssh` emission). `run_control_master -N -f` — §3.4 (`main.go:666-668`).
`forward_remote_control` default no — §3.4 (`main.py:212`).

### Q4 — Per‑shell encoding → §4

`script_type` selection (`py` iff basename contains "python", else `sh`) — §4.1 (`main.go:511-518`).
Python base64 path + `eval(compile(base64.standard_b64decode(...)))` — §4.2/§4.3 (`main.go:498-499`;
observed `rcmd`). `sh` `tr` substitution: exact mappings `'`→`\v`, `\`→`\f`, `\n`→`\r`, `!`→`\b` and
the unwrap string `'eval "$(echo "$0" | tr \\\v\\\f\\\r\\\b \\\047\\\134\\\n\\\041)"' ` — §4.2/§4.4
(`main.go:505-506`; observed byte counts + shim `ARGV[17]`). Final
`rcmd = {exec, interpreter, -c, unwrap, encoded}` — §4.2 (`main.go:508`; observed both interpreters).

### Q5 — End‑to‑end trace → §5

`main()` guards (`KITTY_WINDOW_ID`/`KITTY_PID`, stdin TTY, passthrough) — §5.1
(`main.go:822-831`). `get_destination` — §5.1 (`main.go:46`; observed `root`/`localhost`).
`run_ssh` steps (`load_config`, ControlMaster args, `set_askpass`, `make_tarfile`, `SetNoEcho`,
wrap+exec, DCS emit, drain) — §5.2 (`main.go:597,642,651,718,749,753-757,766,539`; observed ordered
trace). Remote unwrap → base64 detect → data request → untar → `compile_terminfo` → `exec_login_shell`
(→ `exec_{zsh,fish,bash}_with_integration`) — §5.4 (`bootstrap.sh:55-72,90-95,104-113,164`;
`bootstrap-utils.sh:44,102,118,128,221`; observed). Canonical test module 8/8 — §5.3.

### Q6 — Shared‑memory security → §6

`secrets.TokenHex()` (differs per run) — §6.1 (`main.go:431`). `shm.CreateTemp`/`WriteWithSize`/
`Flush`, `0600` — §6.2 (`main.go:446-450`; `shm_fs.go:130`; `shm.py:51`; observed
`mode=0o600`). Owner/permission guards + exact error strings — §6.4 (`utils.py:107-111`; both
raised). One‑shot unlink — §6.3 (`utils.py:106`; observed `exists=False` after). `pw`/`rq_id` checks
— §6.5 (`utils.py:129-133`; both rejected). Two‑path password travel — §6.6 (`main.go:460,475-482,761-766`): the tar is never on any command line in either path; the real password reaches the **remote** `argv` only when `request_data=true` (`:476-477`), while the default `request_data=false` path delivers it via the local `@kitty-ssh` DCS (`:766`) and keeps it off the remote `argv`.

### Q7 — Bidirectional TTY handshake → §7

`@kitty-ssh` DCS byte‑exact (remote + local origin, base64 payload) — §7.2 (remote gate
`bootstrap.sh:90-95` + framer `:75`; local emit `main.go:760,766`; origin selected by
`use_kitty_askpass`/`need_to_request_data` at `main.go:648-649`; observed both, stable ×2). `@kitty-ask` DCS byte‑exact (raw shm name) — §7.5
(`askpass.go:30`; observed 3 sub‑types). `RunSSHAskpass`/`trigger_ask` + sentinel polling — §7.5
(`askpass.go:24,37,55,62,72`; observed sentinel=0). askpass confirm/get‑line/fingerprint — §7.5
(`askpass.go:40-50`; observed question JSON for each). `stty -echo`/`stty echo` — §7.6
(`bootstrap.sh:8,10-11,92-93`; `main.go:718`; observed `echo_on="1"`). base64 fallback chain (`base64` →
`openssl` → `b64encode`/`b64decode` → python `pybase64` → perl `MIME::Base64` → `die`) — §7.4
(`bootstrap.sh:55-72`; observed `base64` selected). `window.py` dispatch — §7.3
(`window.py:1289-1292`; `vt-parser.c:608-609`). `request_data` true vs false (who issues the request)
— §7.2 (observed both origins).

### Variant cross‑product (all exercised)

- Interpreter **`sh`** and **`python`** → two `rcmd` forms (§4).
- Connection **fresh** and **reused** → `ssh -O check` `rc 255→0`, `request_data "1"→"0"` (§3).
- `request_data` **true** and **false** → remote‑issued vs local‑issued `@kitty-ssh` (§7.2, §5).
- base64 helper chain → `base64` selected on this endpoint, ordered fallback documented (§7.4).
- askpass **confirm** / **get‑line** / **fingerprint** → all three captured (§7.5).
- Before/after for state changes → SHM `0o600` then unlinked (§6); echo off then on (§7.6);
  `need_to_request_data` true then false (§3).

### Note on inferred vs observed

Nearly every claim is backed by captured runtime output. The only items presented as background
rather than observed are external‑mechanism facts (the ~104‑byte Unix‑socket path limit motivating
the `/tmp/kssh-rdir` symlink, §3.2, and general OpenSSH `ControlMaster`/POSIX `shm_open` semantics),
which corroborate — but are not the source of — the observed behavior. No value in this document was
adjusted toward an expected result; instance‑specific values (`pw`, `shm_name`, `%C`, chosen
`KITTY_PID`/`KITTY_WINDOW_ID`) are reported exactly as emitted and labelled as varying per run.

---

## Cleanup

All observation instrumentation and captures were created **outside** the repository, under
`/tmp/kssh-obs/`, and are removed once the document is complete. Nothing was ever written into the
source tree.

**Temporary artifacts created (all outside the repo):**

- `/tmp/kssh-obs/ptydrv.py` — the terminal-emulation PTY driver (§0.4).
- `/tmp/kssh-obs/shim/{ssh,scp,sftp}` — the transparent `PATH` shims (§0.4).
- `/tmp/kssh-obs/q6_guards.py` — the `[SUPPLEMENTAL]` guard-firing harness (§6).
- `/tmp/kssh-obs/echo_state.sh` + `/tmp/kssh-obs/run_pty.py` — the TTY-echo toggle harness and its PTY runner (§7.6).
- `/tmp/kssh-obs/cap/*` and the top-level `*.raw`/`*.log`/`*.tar`/`*.json`/`*.txt` captures — the raw
  session recordings, driver logs, decoded tarballs, byte-exact argv dumps, and toolchain/build/sshd
  transcripts quoted throughout.

**Removal and verification:**

```
$ rm -rf /tmp/kssh-obs
$ rm -f /dev/shm/kssh-* /root/.cache/kitty/run/kssh-*   # any stray SHM objects / control sockets

$ ls -d /tmp/kssh-obs 2>&1
ls: cannot access '/tmp/kssh-obs': No such file or directory

$ ls /dev/shm/ | grep -c kssh
0

$ git -C <repo> status --porcelain
(no output — working tree clean)
```

The empty `git status --porcelain` (explicit empty-output marker above) is the final read-only proof:
the branch's single commit adds only `blitzy/documentation/kitty_815df1e210e0.md`, no source file was
modified, added, or deleted, and no temporary artifact remains inside the repository tree. This
matches the pre-work baseline (§0.5).

