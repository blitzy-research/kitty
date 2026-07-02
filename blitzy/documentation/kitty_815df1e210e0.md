# How the Kitty SSH kitten establishes a secure remote session and shares SSH connections

This document traces, **end-to-end and from directly observed runtime behavior**, how the Kitty
terminal emulator's **SSH kitten** (`kitten ssh`) establishes a secure remote session and shares
SSH connections. It answers eight questions (Q1–Q8). Every behavioral claim is paired with the
exact command/code that produced it, an adjacent **verbatim** evidence block, and a `file:line`
citation into the source at HEAD `815df1e21`.

---

## 0. Build & Run methodology

### 0.1 Platform, toolchain and canonical build

All observation was performed inside the user-specified container image
`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`
(`andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`),
which supplies the Go and C toolchains.

Toolchain versions (verbatim):

```text
$ go version
go version go1.22.12 linux/amd64          # satisfies go.mod:3  →  go 1.22
$ python3 --version
Python 3.13.7                              # satisfies pyproject.toml:2  →  requires-python = ">=3.8"
$ ssh -V
OpenSSH_10.0p2 Ubuntu-5ubuntu5.4, OpenSSL 3.5.3 16 Sep 2025
```

The canonical build is `./dev.sh build` (the command documented at `docs/build.rst:19`; `dev.sh`
forwards to `go run bypy/devenv.go`). It was run as a normal user in the default configuration:

```text
$ export CFLAGS="${CFLAGS:+$CFLAGS }-Wno-error=switch"   # see note below; does NOT alter kitten behavior
$ ./dev.sh build ; echo "BUILD_EXIT_CODE=$?"
Build successful. Run kitty as: kitty/launcher/kitty
BUILD_EXIT_CODE=0
```

This produced the two launcher binaries used throughout:

```text
$ ls -la kitty/launcher/kitty kitty/launcher/kitten
-rwxr-xr-x ... 15765764 ... kitty/launcher/kitten
-rwxr-xr-x ...    40384 ... kitty/launcher/kitty
$ ./kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
```

> **`CFLAGS=-Wno-error=switch` rationale (no behavior change):** on Ubuntu 25.10 the newer
> `wayland-protocols` adds `XDG_TOPLEVEL_STATE_CONSTRAINED_*` enum values that `glfw/wl_window.c`'s
> `switch` does not handle; `setup.py` compiles C with `-Werror` by default (`setup.py:517`) and
> appends `CFLAGS` **after** it, so only that library-drift warning is de-promoted. All genuine
> warnings still error, no source is modified, and the C VT parser and the Go kitten are unaffected.

The build/HEAD anchor for the "default, canonical configuration" claim (verbatim):

```text
$ git rev-parse --abbrev-ref HEAD     # my working branch
blitzy-7d489156-59de-439e-9b2f-075b4512d760
$ git rev-parse --short HEAD
815df1e21
$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1     # == the container image tag → confirms the canonical source commit
```

The deliverable filename is fixed as `<source_branch_name>.md` = `kitty_815df1e210e0.md`, matching
the full HEAD `815df1e210e0…`.

### 0.2 The SSH target and how the real path was driven

An in-container OpenSSH server was reachable at `localhost` (root pubkey auth). Plain connectivity,
verbatim:

```text
$ ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new localhost 'echo SSH_OK; whoami; uname -s'
SSH_OK
root
Linux
```

The SSH kitten's real entry point requires two runtime env vars and a terminal `stdin` (see Q1). The
kitten also depends on the **kitty terminal being present** to serve credential data over a custom
DCS escape protocol (Q8). To exercise the **real** `kitten ssh` entry point non-interactively, a PTY
observation harness was written under `/tmp` that plays *exactly* kitty's terminal role:

- it is the PTY master; the **real** `kitty/launcher/kitten ssh` binary (and the real `ssh` child and
  the real remote `bootstrap.(sh|py)`) run in the slave;
- when it receives the DCS request `ESC P @ kitty-ssh | <base64> ESC \` it calls the **real**
  `kittens.ssh.utils.get_ssh_data(payload, request_id)` and writes each yielded line back — this is
  precisely what `kitty/window.py:1289-1292` `handle_remote_ssh` does;
- it answers the drain canary `kitty-echo|` by echoing back the printable payload, exactly as
  `kitty/window.py:1282-1287` `handle_remote_echo` does;
- a PATH shim named `ssh` logs the exact child `argv` the kitten builds, then `exec`s `/usr/bin/ssh`
  so behavior is unchanged.

The kitten's Python terminal-side module is loaded with the **bundled** interpreter the build
produced (`dependencies/linux-amd64/bin/python3` with `LD_LIBRARY_PATH=dependencies/linux-amd64/lib`),
because Kitty's compiled extension links `libpython3.14`. Everything exercised — the kitten binary,
`get_ssh_data`, `ssh`, the remote bootstrap, the real POSIX `sshd` — is the real built code; only
kitty's event loop (its `Boss`) byte-pump is reimplemented, faithfully, in the harness.

### 0.3 Labeling convention used in this document

- **(observed)** — captured from the real `kitten ssh localhost` entry point at runtime. This is the
  default; evidence blocks show the actual command and output.
- **NON-CANONICAL** — a value obtained from a fallback/synthetic stand-in rather than the real
  `kitten ssh` push. Used sparingly (e.g. the wrong-permissions SHM rejection, which the real kitten
  never produces). Where used, the real path is still described.
- **(inferred)** — a statement derived from *reading* the source, not observed at runtime.

Every measured magnitude (the `254`-byte transfer line, the `2`-second drain timeout,
`ServerAliveInterval=60`) is reported with the run scale and confirmed stable across ≥2 runs.

---

## Q1 — End-to-end secure-session and connection-sharing trace

**Question.** Trace the full path from a user invoking `kitten ssh <host>` to the bootstrap executing
on the remote host.

The chain is `kitten ssh <host>` → `func main` [kittens/ssh/main.go:800] → `func run_ssh`
[kittens/ssh/main.go:597] → host `Config` load → connection-sharing injection (Q6) →
askpass/request-data decision (Q8) → `func get_remote_command` [kittens/ssh/main.go:511] → launch of
the `ssh` child via `exec.Command` [main.go:754] → TTY data exchange (Q4/Q8) → remote
`bootstrap.(sh|py)` (Q3) → tar extraction → `exec_login_shell`.

### Hop 1 — `func main` and the two entry guards

`main` dispatches the legacy `use-python` arg [main.go:804], parses args with
`ParseSSHArgs(args, "--kitten")` [main.go:810], handles `passthrough` by `exec`-ing plain `ssh`
[main.go:822], and then enforces two guards before doing anything else:

```go
// kittens/ssh/main.go:825-831
if os.Getenv("KITTY_WINDOW_ID") == "" || os.Getenv("KITTY_PID") == "" {
    return 1, fmt.Errorf("The SSH kitten is meant to run inside a kitty window")
}
if !tty.IsTerminal(os.Stdin.Fd()) {
    return 1, fmt.Errorf("The SSH kitten is meant for interactive use only, STDIN must be a terminal")
}
return run_ssh(ssh_args, server_args, found_extra_args)
```

**Cause → effect.** These guards guarantee the kitten only runs where it can talk to the kitty
terminal (it needs `KITTY_PID`/`KITTY_WINDOW_ID` to form the request-id, and a real TTY to exchange
data). Both fire exactly as written (observed):

```text
$ env -u KITTY_WINDOW_ID -u KITTY_PID ./kitty/launcher/kitten ssh localhost   # guard 1
Error: The SSH kitten is meant to run inside a kitty window

$ echo "" | env KITTY_WINDOW_ID=1 KITTY_PID=99999 ./kitty/launcher/kitten ssh localhost   # guard 2 (piped stdin)
Error: The SSH kitten is meant for interactive use only, STDIN must be a terminal
```

With both `KITTY_PID`/`KITTY_WINDOW_ID` set **and** a PTY on `stdin` (as the harness provides), `main`
proceeds into `run_ssh`.

### Hop 2 — `run_ssh` orchestration → the ssh child

`run_ssh` builds the `ssh` command starting from `SSHExe()` (which resolves via
`utils.FindExe("ssh")` [kittens/ssh/utils.go:22-24]), loads the per-host `Config`, injects the
connection-sharing `-o` options when `host_opts.Share_connections` is true [main.go:637-647] (Q6),
makes the askpass/request-data decision [main.go:648-651] (Q8), opens the controlling terminal with
echo disabled `tty.OpenControllingTerm(tty.SetNoEcho)` [main.go:718], populates the `connection_data`
struct (Q5), calls `get_remote_command` [main.go:752], and launches the child:

```go
// kittens/ssh/main.go:752-756
err = get_remote_command(&cd)
...
cmd = append(cmd, cd.rcmd...)
c := exec.Command(cmd[0], cmd[1:]...)
```

**Observed** — the ssh PATH-shim recorded exactly three `ssh` invocations for one
`kitten ssh localhost` run; the third is the real connection (its full `argv` appears under Q6/Q7):

```text
=== ssh invocation ===            # [1] no-arg options probe: exec.Command(SSHExe())  [utils.go:40]
=== ssh invocation ===            # [2] argv[0]=-V   → ssh -V, GetSSHVersion()
=== ssh invocation ===            # [3] the real connection: argv = -t -o ControlMaster=auto ... -- localhost exec sh -c <unwrap> <script>
  argv[0]=-t
  argv[14]=localhost
  argv[15]=exec
  argv[16]=sh
  argv[17]=-c
```

### Hop 3 — credential request over the TTY, then the remote bootstrap runs

After `c.Start()`, on the default (push) path the kitten writes the data-serving request straight
down the terminal [main.go:760-768] (see Q8). The harness (playing kitty) received it and — via the
**real** `get_ssh_data` — streamed the archive back. **Observed** for `kitten ssh localhost`:

```text
[sh] LAUNCH: kitty/launcher/kitten ssh localhost
[sh] <-- DCS kitty-ssh| req#1 (148 b64 bytes)
[sh]     decoded = id=58113-1:pwfile=kssh-58114-B6TON5DTJMRV4:pw=ca39a40f…  (64-hex random pw truncated here for hygiene; ephemeral, single-use, SHM already unlinked)
[sh]     --> get_ssh_data yielded 257 lines
```

Here `id=58113-1` is `KITTY_PID-KITTY_WINDOW_ID` (the harness pid `58113`, window `1`), and
`pwfile=kssh-58114-…` names the shared-memory object the kitten created (pid `58114` is the kitten
child). The remote `bootstrap.sh` consumed the streamed tar, extracted it, compiled terminfo, and
**exec'd the login shell** — the real remote prompt and kitty shell-integration markers appeared
(observed, from the raw PTY capture, control bytes rendered with `cat -v`):

```text
^[]7;kitty-shell-cwd://reverse-code-generator-63e78bca-hn5x7/root^G   # OSC-7 cwd (shell integration)
^[]133;A^G                                                            # OSC-133 prompt mark
root@reverse-code-generator-63e78bca-hn5x7:~#                         # the REAL remote login shell prompt
```

**Cause → effect.** The appearance of the remote prompt with kitty's OSC-133/OSC-7 marks proves the
whole chain succeeded: the archive was transported over the TTY, extracted on the remote, terminfo
compiled, and `exec_login_shell` [shell-integration/ssh/bootstrap.sh:164] replaced the bootstrap with
an interactive login shell that has kitty shell integration active. When the session then ends, the
kitten drains the TTY (Q8). Each subsequent question dissects one stage of this trace in full.

---

## Q2 — Shared memory (SHM) for secure credential passing

**Question.** How is SHM used to hand off sensitive data, and what specifically keeps it secure?

### The producer/consumer, local-only credential channel

The kitten (Go, producer) and the kitty terminal (Python, consumer) exchange credentials through a
POSIX shared-memory object that **never leaves the local machine**. The producer is `bootstrap_script`
[kittens/ssh/main.go:422]:

```go
// kittens/ssh/main.go (bootstrap_script)
pw, err := secrets.TokenHex()                                   // :431  random password
...
data := map[string]string{                                      // :439-443
    "tarfile":  base64.StdEncoding.EncodeToString(tfd),
    "pw":       pw,
    "hostname": cd.hostname_for_match,
    "username": cd.username,
}
encoded_data, err := json.Marshal(data)                         // :444
data_shm, err := shm.CreateTemp(fmt.Sprintf("kssh-%d-", os.Getpid()), uint64(len(encoded_data)+8))  // :446
...
shm.WriteWithSize(data_shm, encoded_data, 0)                    // :448
data_shm.Flush()                                                // :450
cd.shm_name = data_shm.Name()                                   // :456
```

The Go SHM helpers are `CreateTemp` [tools/utils/shm/shm.go:91], `WriteWithSize` [shm.go:120],
`ReadWithSize` [shm.go:129] and `ReadWithSizeAndUnlink` [shm.go:142]. The name (`kssh-<pid>-<random>`)
is placed into the request as `pwfile=` and pushed to the terminal (Q1/Q8).

The consumer/validator is on the kitty side: `read_data_from_shared_memory` [kittens/ssh/utils.py:100]
and `get_ssh_data` [kittens/ssh/utils.py:115]. The SHM abstraction it opens is `kitty/shm.py`'s
`SharedMemory` [kitty/shm.py:35], created (on the producer's Python-clone equivalent) with default
`mode = stat.S_IREAD | stat.S_IWRITE` (= `0o600`) [kitty/shm.py:51] and `flags = os.O_CREAT | os.O_EXCL`
[kitty/shm.py:62], with an 8-byte size prefix [kitty/shm.py:47] and a `def unlink` [kitty/shm.py:175].

**Observed** — the JSON payload actually carried by the real SHM (keys only; values are the secret
password/tar) contains exactly the four documented fields:

```text
D1 unlink: read_data_from_shared_memory OK; keys=['hostname', 'pw', 'tarfile', 'username']
```

### The five-part security model (each defense: literal + cause→effect + observed)

The validator opens the SHM by the name in the request and applies five checks, in this order. All
five were exercised against a **real** SHM created by the real `kitten ssh localhost` push
(`/tmp/kssh_obs/q2_shm.py` captured the pushed `pwfile`/`pw`, then drove the real validators).

**Defense 1 — immediate `unlink` on open.** `read_data_from_shared_memory` unlinks the SHM the moment
it opens it, before returning any bytes:

```python
# kittens/ssh/utils.py:104-106
with SharedMemory(shm_name, readonly=True) as shm:
    shm.unlink()
```

*Cause → effect:* the credential object exists on disk (`/dev/shm`) only for the instant between
creation and first read; a second reader (or an attacker who learns the name later) finds nothing.
**Observed** (the file is gone immediately after the single read):

```text
after read, /dev/shm/kssh-61471-C3POQXME36BHK exists? False  -> unlink CONFIRMED (gone)
```

**Defense 2 — owner UID/GID check.** 

```python
# kittens/ssh/utils.py:107-108
if shm.stats.st_uid != os.geteuid() or shm.stats.st_gid != os.getegid():
    raise ValueError(f'Incorrect owner on pwfile: uid={shm.stats.st_uid} gid={shm.stats.st_gid}')
```

*Cause → effect:* the terminal refuses any SHM object not owned by the same user, so another user
cannot substitute a forged credential file. **Observed** (real SHM passes):

```text
st_uid=0 st_gid=0  (my euid=0 egid=0)  -> owner check PASS
```

**Defense 3 — `0o600` permission check.**

```python
# kittens/ssh/utils.py:109-111
mode = stat.S_IMODE(shm.stats.st_mode)
if mode != stat.S_IREAD | stat.S_IWRITE:
    raise ValueError(f'Incorrect permissions on pwfile: 0o{mode:03o}')
```

*Cause → effect:* only owner-read/owner-write (`0o600`) is accepted, so a group- or world-readable
credential object is rejected. **Observed** (real SHM passes):

```text
mode=0o600  (expected 0o600) -> perm check PASS
```

The rejection branch fires with the exact message when the mode is wrong. This was demonstrated with
a **NON-CANONICAL** synthetic SHM `chmod`'d to `0o644` (the real kitten always creates `0o600`, so the
rejection cannot be produced from the canonical push):

```text
D3 rejection ValueError: Incorrect permissions on pwfile: 0o644
```

**Defense 4 — random password match.** After `read_data_from_shared_memory` returns, `get_ssh_data`
compares the `pw` from the request against the `pw` inside the SHM:

```python
# kittens/ssh/utils.py:130-131
if pw != env_data['pw']:
    raise ValueError('Incorrect password')
```

*Cause → effect:* the terminal only serves the archive if the requester presents the exact random
password (`secrets.TokenHex()`) that the kitten wrote into the SHM, defeating a request that guesses
the SHM name but not its contents. **Observed** (request with a tampered `pw`):

```text
D4 wrong-pw -> get_ssh_data yielded: [b'Incorrect password']
```

**Defense 5 — request-id match.**

```python
# kittens/ssh/utils.py:132-133
if rq_id != request_id:
    raise ValueError(f'Incorrect request id: {rq_id!r} expecting the KITTY_PID-KITTY_WINDOW_ID for the current kitty window')
```

*Cause → effect:* the request's `id` must equal the serving window's `KITTY_PID-KITTY_WINDOW_ID`
(passed by `handle_remote_ssh` as `f'{os.getpid()}-{self.id}'` [kitty/window.py:1291]), so one kitty
window will not serve credentials on behalf of another. **Observed** (request with a tampered `id`):

```text
D5 wrong-id -> get_ssh_data yielded: [b"Incorrect request id: '9999-9999' expecting the KITTY_PID-KITTY_WINDOW_ID for the current kitty window"]
```

### Why the SHM is secure: it is never transmitted

The SHM payload `{tarfile, pw, hostname, username}` is created (`shm.CreateTemp` on the local kitten)
and consumed (`read_data_from_shared_memory` on the local kitty terminal) **entirely on the local
machine's `/dev/shm`**. Only the base64-encoded *tar* (extracted from the payload) is streamed over
the wire, over the TTY (Q4). The credential channel therefore relies on kernel-enforced filesystem
semantics (owner-only `0o600`, `O_CREAT|O_EXCL` creation, immediate `unlink`) plus the two
application checks (password and request-id), all evidenced above.

---

## Q3 — Bootstrap script generation and remote execution

**Question.** How are the bootstrap scripts generated locally and how do they run on the remote host?

### Local generation: `prepare_script` and `bootstrap_script`

`bootstrap_script` [kittens/ssh/main.go:422] embeds the shell-integration bootstrap source for the
chosen `script_type` (`shell-integration/ssh/bootstrap.sh` or `bootstrap.py`), computes the SHM/tar
(Q2/Q4), and substitutes a set of placeholder tokens via `prepare_script` [kittens/ssh/main.go:407].
`prepare_script` defaults `EXEC_CMD` and `EXPORT_HOME_CMD` to empty and performs **word-boundary**
substitution — each key becomes `\b<key>\b` and is replaced by regex:

```go
// kittens/ssh/main.go:407-419 (prepare_script)
replacements["EXEC_CMD"] = ""            // :409
replacements["EXPORT_HOME_CMD"] = ""     // :412
keys := make([]string, len(replacements))
... keys[i] = "\\b" + key + "\\b" ...    // :416  word-boundary tokens
pat := regexp.MustCompile(strings.Join(keys, "|"))   // :417
return pat.ReplaceAllStringFunc(script, ...)         // :418
```

The default request-id is formed at [main.go:424] as
`os.Getenv("KITTY_PID") + "-" + os.Getenv("KITTY_WINDOW_ID")`. The sensitive tokens `REQUEST_ID`,
`DATA_PASSWORD`, `PASSWORD_FILENAME` and the booleans `REQUEST_DATA`/`ECHO_ON` are substituted into
the script. `wrap_bootstrap_script` [main.go:486] then encodes the whole script (Q7) and sets:

```go
// kittens/ssh/main.go:509
cd.rcmd = []string{"exec", cd.host_opts.Interpreter, "-c", unwrap_script, encoded_script}
```

**Observed** — the fully-substituted `sh` bootstrap that was actually sent (captured verbatim from
the ssh child `argv[19]`; shown decoded, control bytes as `\r`). Note `request_data="0"` (push,
default), the request line still holding its placeholder names, and the terminal stages:

```text
...
request_data="0"
[ "$request_data" = "1" ] && {
    command stty "-echo" < /dev/tty
    dcs_to_kitty "ssh" "id=REQUEST_ID:pwfile=PASSWORD_FILENAME:pw=DATA_PASSWORD"
}
...
untar_and_read_env() { ... tdir=$(command mktemp -d "$HOME/.kitty-ssh-kitten-untar-XXXXXXXXXXXX") ... 
    read_base64_from_tty | base64_decode | command tar "xpzf" "-" "-C" "$tdir" ...
    . "$tdir/bootstrap-utils.sh"; . "$tdir/data.sh" ...
    compile_terminfo "$tdir/home"; mv_files_and_dirs "$tdir/home" "$HOME" ... }
get_data() { ... [ "$line" = "KITTY_DATA_START" ] ... [ "$line" = "OK" ] && break ... untar_and_read_env; }
get_data
cleanup_on_bootstrap_exit
prepare_for_exec
exec_login_shell
```

### Remote execution: `bootstrap.sh` (POSIX) and `bootstrap.py` (Python)

The remote interpreter runs `exec <interpreter> -c <unwrap> <encoded>`; the unwrap reconstructs the
script and executes it. The `sh` bootstrap flow, in order, with the exact `file:line`:

- `dcs_to_kitty()` frame emitter [shell-integration/ssh/bootstrap.sh:75];
- `request_data="REQUEST_DATA"` placeholder [bootstrap.sh:90] and the request line
  `dcs_to_kitty "ssh" "id=REQUEST_ID:pwfile=PASSWORD_FILENAME:pw=DATA_PASSWORD"` [bootstrap.sh:94]
  (only emitted when `request_data = "1"`, i.e. the pull path);
- `read_base64_from_tty()` [bootstrap.sh:97], which stops at the `KITTY_DATA_END` sentinel;
- `untar_and_read_env()` [bootstrap.sh:104]: `mktemp -d "$HOME/.kitty-ssh-kitten-untar-XXXXXXXXXXXX"`
  [bootstrap.sh:108]; `tar "xpzf" "-" "-C" "$tdir"` [bootstrap.sh:113]; sources the bundled
  `bootstrap-utils.sh` and `data.sh`; `compile_terminfo "$tdir/home"` [bootstrap.sh:130];
- `get_data()` [bootstrap.sh:137] drives the framed read (`KITTY_DATA_START` … `OK` … `KITTY_DATA_END`);
- `EXEC_CMD` [bootstrap.sh:159] (empty in the login-shell case), `TEST_SCRIPT` [bootstrap.sh:162]
  (empty outside tests), and finally `exec_login_shell` [bootstrap.sh:164].

The staging/terminfo utilities live in `shell-integration/ssh/bootstrap-utils.sh` — `mv_files_and_dirs`
[bootstrap-utils.sh:9], `compile_terminfo` [bootstrap-utils.sh:18], `prepare_for_exec`
[bootstrap-utils.sh:192], `exec_login_shell` [bootstrap-utils.sh:221] — and are bundled into the
archive in `sh` mode. The archive also carries the remote launcher stubs `shell-integration/ssh/kitty`
and `shell-integration/ssh/kitten` (visible in the archive listing under Q4 as
`.../kitty/bin/kitty` and `.../kitty/bin/kitten`).

**Observed** — the remote actually reached `exec_login_shell` (the real login-shell prompt appeared,
Q1). The extraction/terminfo steps are the ones whose outputs feed that prompt.

When the remote **interpreter is Python**, `bootstrap.py` is used instead. Its equivalents were
captured by base64-decoding the py-path `argv[19]` (Q7): `request_data = int('0')` [bootstrap.py:22],
`dcs_to_kitty(payload, type='ssh')` [bootstrap.py:73], the request write
`id=REQUEST_ID:pwfile=PASSWORD_FILENAME:pw=DATA_PASSWORD` [bootstrap.py:81], `compile_terminfo`
[bootstrap.py:133], the `KITTY_DATA_START`/`KITTY_DATA_END` markers [bootstrap.py:178,188], and
`get_data()` [bootstrap.py:203]; it finishes by `os.execlp`-ing the login shell.


---

## Q4 — Shell-integration archive build and TTY transport

**Question.** How is the tar archive containing shell-integration assets assembled, and how is it sent
over the wire?

### Building the archive: `make_tarfile` → gzip-compressed PAX tar

`make_tarfile` [kittens/ssh/main.go:255] serializes the environment, then writes a **gzip
best-compression** stream wrapping a **PAX-format** tar:

```go
// kittens/ssh/main.go:259-268
gw, err := gzip.NewWriterLevel(&w, gzip.BestCompression)   // gzip, maximum compression
...
tw := tar.NewWriter(gw)
...
add := func(h *tar.Header, data []byte) (err error) {
    h.Mode |= 0o600                                        // ensure owner rw even on nix-mangled perms
    err = tw.WriteHeader(h)
```

In-memory entries (e.g. `data.sh`) are written with `Format: tar.FormatPAX, Mode: 0o644` in the
`add_data` closure (same function), confirming the PAX format.

**Observed** — decoding the archive actually transported (captured from the framed stream, then
`base64 -d`) confirms gzip max-compression and lists the shell-integration payload:

```text
$ file captured.tar.gz
captured.tar.gz: gzip compressed data, max compression, original size modulo 2^32 93184
$ tar -tzf captured.tar.gz            # 15 entries
data.sh
bootstrap-utils.sh
home/.local/share/kitty-ssh-kitten/shell-integration/zsh/.zshenv
home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish
home/.local/share/kitty-ssh-kitten/shell-integration/bash/kitty.bash
home/.local/share/kitty-ssh-kitten/shell-integration/zsh/kitty-integration
home/.local/share/kitty-ssh-kitten/kitty/version
home/.local/share/kitty-ssh-kitten/kitty/bin/kitty
home/.local/share/kitty-ssh-kitten/kitty/bin/kitten
home/.terminfo/kitty.terminfo
home/.terminfo/x/xterm-kitty
```

`file` reporting **"max compression"** is the runtime signature of `gzip.BestCompression`
[main.go:259]; the `home/.terminfo/...` and `shell-integration/...` entries are the assets the remote
stages; `kitty/bin/{kitty,kitten}` are the bundled launcher stubs (Q3).

### Transport over the TTY: `get_ssh_data` framing and the 254-byte line

The kitty terminal serves the base64 payload with `get_ssh_data` [kittens/ssh/utils.py:115], framed
between markers and chunked into fixed-size lines:

```python
# kittens/ssh/utils.py:117,138,140-148
yield b'\nKITTY_DATA_START\n'   # :117  discards any leading data on the line
...
yield b'OK\n'                   # :138  emitted only after pw + request-id validation pass
# macOS has a 255 byte limit on its input queue as per man stty.  (:140-142 comment)
line_sz = 254                   # :143
while encoded_data:
    yield encoded_data[:line_sz]  # :145
    yield b'\n'
    encoded_data = encoded_data[line_sz:]
yield b'KITTY_DATA_END\n'       # :148
```

**Cause → effect.** The `254` line size is deliberately one below the documented macOS 255-byte
terminal input-queue limit (comment at utils.py:140-142), so no transfer line can overflow the
remote's canonical-mode input queue. The `KITTY_DATA_START` / `OK` / `KITTY_DATA_END` frame lets the
remote discard any leading garbage, detect the go-ahead, and know when the stream ends.

**Observed** — the framed stream that `get_ssh_data` actually produced (captured verbatim), showing
the markers and the gzip magic (`H4sI…` is base64 of `\x1f\x8b\x08…`):

```text
leading 40 bytes: b'\nKITTY_DATA_START\nOK\nH4sIAAAAAAAC/+z9fWw'
trailing 30 bytes: b'YTpCQAGwB\nAA==\nKITTY_DATA_END\n'
```

**Line size `254` — measured, stable across 2 runs.** The chunk-size distribution of the payload
lines (between `OK` and `KITTY_DATA_END`) is all-254 except the final remainder:

```text
run 1: num payload chunks: 125 ; chunk-size distribution {4: 1, 254: 124} ; all non-last sizes = {254} ; last = 4
run 2: num payload chunks: 127 ; all non-last sizes = {254} ; last = 80
```

The count varies run-to-run (the gzip'd tar size depends on the random password baked into `data.sh`),
but **every non-final line is exactly 254 bytes in both runs**, confirming `line_sz = 254` is stable
and canonical.

---

## Q5 — Connection state tracking: the `connection_data` struct

**Question.** How does the kitten keep track of everything it needs for a connection?

All per-connection state lives in one struct, `connection_data` [kittens/ssh/main.go:171-189]. Its
**16 fields**, each with how it is populated during `run_ssh`:

| # | Field | Type | Populated during `run_ssh` | Observed / evidence |
|---|-------|------|-----------------------------|---------------------|
| 1 | `remote_args` | `[]string` | the server-side args (command after the host); empty for a login shell | empty in `kitten ssh localhost` (interactive login shell) |
| 2 | `host_opts` | `*Config` | `cd.host_opts = host_opts` [main.go:723] from the per-host config load | defaults: `Interpreter sh`, `Share_connections true`, `Askpass unless-set` [conf_generated.go] |
| 3 | `hostname_for_match` | `string` | `cd.hostname_for_match = hostname_for_match` [main.go:725] | `localhost`; also written into the SHM `hostname` key (observed keys `['hostname',...]`, Q2) |
| 4 | `username` | `string` | `cd.username = uname` [main.go:725] | SHM `username` key (observed, Q2) |
| 5 | `echo_on` | `bool` | `cd.echo_on = term.WasEchoOnOriginally()` [main.go:722] | script carries `echo_on="1"` (observed in captured bootstrap, Q3) |
| 6 | `request_data` | `bool` | `cd.request_data = need_to_request_data` [main.go:724] | `false` (push, default) / `true` (pull); observed as `request_data="0"`/`"1"` in the script (Q7/Q8) |
| 7 | `literal_env` | `map[string]string` | `cd.literal_env = literal_env` [main.go:723] | env forced verbatim into the remote; empty in the default run |
| 8 | `listen_on` | `string` | set only on the forward path: `cd.listen_on = "tcp:localhost:"+port` [main.go:716] | populated only with `forward_remote_control=yes` (Q6) |
| 9 | `test_script` | `string` | test hook (`TEST_SCRIPT`), empty outside the PTY test harness | empty (observed: no `TEST_SCRIPT` body in captured script) |
| 10 | `dont_create_shm` | `bool` | test hook to skip SHM creation | `false` on the real path (SHM was created — observed `/dev/shm/kssh-…`, Q2) |
| 11 | `shm_name` | `string` | `cd.shm_name = data_shm.Name()` [main.go:456] in `bootstrap_script` | observed `kssh-58114-B6TON5DTJMRV4` (== request `pwfile=`, Q1) |
| 12 | `script_type` | `string` | `get_remote_command`: `"sh"` [main.go:515] default, `"py"` [main.go:517] if python | observed both: `sh` (default) and `py` (`--kitten interpreter=python3`, Q7) |
| 13 | `rcmd` | `[]string` | `wrap_bootstrap_script`: `[]string{"exec", Interpreter, "-c", unwrap, encoded}` [main.go:509] | observed as ssh `argv[15..19]` = `exec sh -c <unwrap> <script>` (Q7) |
| 14 | `replacements` | `map[string]string` | placeholder→value map built by `bootstrap_script`/`prepare_script` | keys `REQUEST_ID`,`PASSWORD_FILENAME`,`DATA_PASSWORD` used in the push `rq` [main.go:762] |
| 15 | `request_id` | `string` | defaulted to `KITTY_PID + "-" + KITTY_WINDOW_ID` [main.go:424] | observed `58113-1` (== request `id=`, Q1) |
| 16 | `bootstrap_script` | `string` | the fully-substituted script text, set in `bootstrap_script` | observed verbatim as the ssh `argv[19]` payload (Q3/Q7) |

**Cause → effect.** `connection_data` is the single value threaded through `bootstrap_script` →
`get_remote_command` → `wrap_bootstrap_script` and back into `run_ssh`; it carries both the *inputs*
(host config, hostname, username, env) and the *derived artifacts* (the SHM name, the request-id, the
chosen `script_type`, the encoded `rcmd`, and the placeholder `replacements`) needed to both launch the
`ssh` child and answer the credential request. Every field above was either observed directly on the
real path or is populated by the cited assignment during `run_ssh`.

> Corroboration: the test `test_ssh_connection_data` [kitty_tests/ssh.py:45] also constructs and
> asserts on this struct, but any value taken from that harness would be **NON-CANONICAL**; the values
> in the table above are from the real `kitten ssh localhost` push (observed) or the cited source
> assignments.


---

## Q6 — Connection reuse: fresh connection vs. piggyback via `ControlMaster`

**Question.** How does the kitten decide whether to start a fresh connection or reuse an existing one
via OpenSSH `ControlMaster` multiplexing?

### The six `-o` options emitted by `connection_sharing_args`

When `host_opts.Share_connections` is true (the default), `run_ssh` inserts the options built by
`connection_sharing_args` [kittens/ssh/main.go:121] into the ssh command line [main.go:642-646]. The
function returns **exactly six `-o` options** [main.go:137-144]:

```go
// kittens/ssh/main.go:137-144
return []string{
    "-o", "ControlMaster=auto",
    "-o", "ControlPath=" + filepath.Join(rd, cp),
    "-o", "ControlPersist=yes",
    "-o", "ServerAliveInterval=60",
    "-o", "ServerAliveCountMax=5",
    "-o", "TCPKeepAlive=no",
}, nil
```

**Observed** — the six options exactly as passed to the real `ssh` child (from the shim `argv`):

```text
  argv[1]=-o   argv[2]=ControlMaster=auto
  argv[3]=-o   argv[4]=ControlPath=/root/.cache/kitty/run/kssh-58114-%C
  argv[5]=-o   argv[6]=ControlPersist=yes
  argv[7]=-o   argv[8]=ServerAliveInterval=60
  argv[9]=-o   argv[10]=ServerAliveCountMax=5
  argv[11]=-o  argv[12]=TCPKeepAlive=no
```

Each option, its literal, and its cause → effect:

- **`ControlMaster=auto`** [main.go:138] — reuse an existing master socket if one exists, otherwise
  become the master. This is the mechanism that lets a second `kitten ssh` piggyback on the first.
- **`ControlPath=<runtime_dir>/kssh-<kitty_pid>-%C`** [main.go:139] — the Unix-domain multiplexing
  socket path. `%C` is OpenSSH's SHA1 of `%l%h%p%r` (local host, remote host, port, remote user), so
  distinct destinations get distinct sockets. Observed value: `/root/.cache/kitty/run/kssh-58114-%C`.
- **`ControlPersist=yes`** [main.go:140] — the master detaches and stays alive after the first client
  session ends, so later sessions reuse it.
- **`ServerAliveInterval=60`** [main.go:141] — send a keepalive probe after 60 s of inactivity.
- **`ServerAliveCountMax=5`** [main.go:142] — give up after 5 unanswered probes (≈5 min).
- **`TCPKeepAlive=no`** [main.go:143] — disable OS-level TCP keepalives (the app-level
  ServerAlive probes are used instead, which also work through NAT/firewalls).

### The `ControlPath` template and how it is built

The socket path derives from the shared template constant
`ssh_control_master_template = 'kssh-{kitty_pid}-{ssh_placeholder}'` [kitty/constants.py:188], exported
to Go by codegen at [gen/go_code.py:599] as `const SSHControlMasterTemplate`
(`constants_generated.go:12`). `connection_sharing_args` substitutes the tokens:

```go
// kittens/ssh/main.go:135-136
cp := strings.Replace(kitty.SSHControlMasterTemplate, "{kitty_pid}", strconv.Itoa(kitty_pid), 1)
cp = strings.Replace(cp, "{ssh_placeholder}", "%C", 1)
```

The directory `rd` comes from `utils.RuntimeDir()` [main.go:122]; there is an Apple-specific workaround
that symlinks the runtime dir under `/tmp/kssh-rdir-<euid>` when the path exceeds 35 chars
[main.go:128-134], because OpenSSH's socket path length limit (~104) collides with macOS's long cache
paths and the 40-char hash + 27-char temp suffix OpenSSH appends.

### The push-vs-request (fresh-vs-piggyback) decision

The decision is driven by three things: `Share_connections`, the askpass capability, and whether a
functional master already exists.

```go
// kittens/ssh/main.go:648-664
use_kitty_askpass := host_opts.Askpass == Askpass_native ||
    (host_opts.Askpass == Askpass_unless_set && os.Getenv("SSH_ASKPASS") == "")
need_to_request_data := true
if use_kitty_askpass {
    need_to_request_data = set_askpass()          // false when OpenSSH supports SSH_ASKPASS_REQUIRE
}
master_is_functional := func() bool {
    ...
    check_cmd := slices.Insert(cmd, 1, "-O", "check")     // :658  → `ssh -O check ...`
    master_is_alive = exec.Command(check_cmd[0], check_cmd[1:]...).Run() == nil   // :659
    return master_is_alive
}
if need_to_request_data && host_opts.Share_connections && master_is_functional() {
    need_to_request_data = false                  // :664  piggyback: let the live master serve
}
```

**Observed — default path (OpenSSH 10.0 ≥ 8.4).** `set_askpass` returns `false` (see Q8), so
`need_to_request_data` is already `false` and the `&&` short-circuits — `master_is_functional()` is
**not** called, and the kitten **pushes** the data itself. Evidence: in the default `kitten ssh
localhost` run the shim logged only three ssh invocations (options-probe, `-V`, real connection) and
**no `-O check`**, and the embedded script carried `request_data="0"`.

**Observed — request path (`--kitten askpass=ssh`).** Here `use_kitty_askpass` is false, so
`set_askpass` is not called and `need_to_request_data` stays `true`; the gate then **does** call
`master_is_functional()`, which runs `ssh -O check`:

```text
=== ssh invocation ===
  argv[0]=-O
  argv[1]=check
  argv[2]=-t
  argv[4]=ControlMaster=auto ...
```

No master exists, so `ssh -O check` returns non-zero → `master_is_functional()` is false → the whole
condition is false → `need_to_request_data` stays true → the embedded script carries
`request_data="1"` (observed) → the **remote** requests the data (pull, Q8).

### `run_control_master` and the port-forward path

`run_control_master` [main.go:666] explicitly starts a detached background master by appending
`-N -f`, and is invoked on the remote-control forwarding path (`Forward_remote_control=yes` **and**
`KITTY_LISTEN_ON` set) [main.go:682]. **Observed** — driving that path
(`--kitten forward_remote_control=yes`, `KITTY_LISTEN_ON=unix:/tmp/kittytest.sock`) produced, in order,
`ssh -O check` (master absent) → the control-master start → `ssh -O check` (now alive) → the forward:

```text
# run_control_master  (main.go:669 appends "-N","-f"; main.go:672 "--", hostname)
ssh -t -o ControlMaster=auto -o ControlPath=/root/.cache/kitty/run/kssh-62150-%C -o ControlPersist=yes \
    -o ServerAliveInterval=60 -o ServerAliveCountMax=5 -o TCPKeepAlive=no -N -f -- localhost

# forward  (main.go:702 appends "-R","0:"+listen_on,"-O","forward")
ssh -t -o ControlMaster=auto -o ControlPath=/root/.cache/kitty/run/kssh-62150-%C -o ControlPersist=yes \
    -o ServerAliveInterval=60 -o ServerAliveCountMax=5 -o TCPKeepAlive=no -R 0:/tmp/kittytest.sock -O forward -- localhost
```

The forward path also **rejects abstract UNIX sockets** [main.go:701] because OpenSSH cannot forward
them. **Observed** (with `KITTY_LISTEN_ON=unix:@kitty-abstract`):

```text
Error: Cannot forward kitty remote control socket when an abstract UNIX socket (@kitty-abstract) is used, due to limitations in OpenSSH. Use either a path based one or a TCP socket
```

*(Background framing only, not a substitute for the above observations: OpenSSH's `ControlMaster` /
`ControlPath` / `ControlPersist` are the standard multiplexing directives; `ControlMaster auto` means
"reuse if a master exists, else become one", `%C` is the SHA1 of `%l%h%p%r`, and `ssh -O check`
interrogates a running master.)*

---

## Q7 — Shell-specific bootstrap encoding

**Question.** What character-substitution encoding is applied to the bootstrap script for different
shells (`sh`/`bash` vs. Python vs. `tcsh`)?

The branch is chosen in `get_remote_command` [kittens/ssh/main.go:511] by inspecting the remote
interpreter's basename:

```go
// kittens/ssh/main.go:514-517
is_python := strings.Contains(strings.ToLower(path.Base(interpreter)), "python")
cd.script_type = "sh"
if is_python { cd.script_type = "py" }
```

`wrap_bootstrap_script` [kittens/ssh/main.go:486] then encodes accordingly.

### `sh`/`bash`/`tcsh` mode — four character substitutions + `tr` inverse

```go
// kittens/ssh/main.go:505-506 (sh mode)
encoded_script = "'" + strings.NewReplacer("'", "\v", "\\", "\f", "\n", "\r", "!", "\b").Replace(cd.bootstrap_script) + "'"
unwrap_script  = `'eval "$(echo "$0" | tr \v\f\r\b \047\134\n\041)"'`
```

The **four substitutions**, each with its cause → effect:

- **`'` → `\v`** (vertical tab) — a single quote cannot appear inside a single-quoted shell string, so
  it is swapped for the rare control byte `\v`.
- **`\` → `\f`** (form feed) — backslashes would otherwise be re-interpreted; swapped for `\f`.
- **newline `\n` → `\r`** (carriage return) — the script is passed as one shell word; embedded
  newlines become `\r` so the whole thing survives as a single `argv` element.
- **`!` → `\b`** (backspace) — `!` triggers history expansion in interactive `csh`/`tcsh`; swapping it
  for `\b` prevents that.

The remote inverts them with `tr \v\f\r\b \047\134\n\041` — mapping `\v→\047` (`'`), `\f→\134` (`\`),
`\r→\n` (newline), `\b→\041` (`!`) — then `eval`s the result.

**Observed** — the default `sh` run's ssh child shows both the unwrap and all four substitutions live
in the encoded payload (`argv[18]` unwrap, `argv[19]` encoded; `%q`-quoted by the shim):

```text
argv[16]=sh
argv[17]=-c
argv[18]='eval "$(echo "$0" | tr \v\f\r\b \047\134\n\041)"'
argv[19]=$'\'#\b/bin/sh\r# Copyright (C) 2022 Kovid Goyal ...\r ... printf "\f033[31m%s\f033[m ... \vbuffer\v ... \''
```

Reading the substitutions straight out of that captured payload:

- `#\b/bin/sh` — the shebang `#!/bin/sh` with `!` → `\b`.
- `\r` between every line — newline → `\r`.
- `\f033` (was `\033`) and `\f134` — `\` → `\f`.
- `\vbuffer\v` (was `'buffer'`) — `'` → `\v`.

**Cause → effect (`sh`/`bash` vs Python vs `tcsh`).** base64 cannot be relied upon to exist on an
arbitrary remote `sh`/`bash` before the archive is unpacked, so the shell path uses this
quote-safe **character-substitution** scheme (`'`,`\`,newline handled) that needs only `echo`, `tr`
and `eval`. The `!` → `\b` and newline → `\r` substitutions are specifically what make the encoded
one-liner safe for **`tcsh`/`csh`** (history-expansion and single-word constraints). Python is
different: the interpreter is guaranteed to have base64, so it uses the base64 path below.

### `py` mode — base64 encode/decode

```go
// kittens/ssh/main.go:501-502 (py mode)
encoded_script = base64.StdEncoding.EncodeToString([]byte(cd.bootstrap_script))
unwrap_script  = "import base64, sys; eval(compile(base64.standard_b64decode(sys.argv[-1]), 'bootstrap.py', 'exec'))"
```

**Observed** — the `--kitten interpreter=python3` run's ssh child:

```text
argv[16]=python3
argv[17]=-c
argv[18]="import base64, sys; eval(compile(base64.standard_b64decode(sys.argv[-1]), 'bootstrap.py', 'exec'))"
argv[19]=IyEvdXNyL2Jpbi9lbnYgcHl0aG9uCiMgTGljZW5zZTogR1BMdjMg...   # base64; decodes to bootstrap.py
```

Base64-decoding `argv[19]` yields the real `bootstrap.py` source (it begins `#!/usr/bin/env python`
and contains `request_data = int('0')`), confirming the py branch encodes with
`base64.StdEncoding.EncodeToString` and the remote decodes with `base64.standard_b64decode`.


---

## Q8 — Terminal ↔ remote-shell communication during setup

**Question.** What request/response protocol is exchanged over the controlling terminal during setup?

### The DCS frame format

Local ↔ remote coordination rides on a custom **Device Control String** protocol of the form
`ESC P @ kitty-<verb> | <base64-payload> ESC \`. The Go builder is `DCSToKitty`
[tools/tui/dcs_to_kitty.go:14]:

```go
// tools/tui/dcs_to_kitty.go:15-25
data := base64.StdEncoding.EncodeToString(utils.UnsafeStringToBytes(payload))
ans := "\x1bP@kitty-" + msgtype + "|" + data          // :16  the frame
...
ans = "\033Ptmux;\033" + ans + "\033\033\\\033\\"      // :23  tmux passthrough variant
...
ans += "\033\\"                                        // :25  ST terminator (non-tmux)
```

The remote shell builds the same frame with `dcs_to_kitty()`
[shell-integration/ssh/bootstrap.sh:75] (`printf "\033P@kitty-$1|%s\033\134" ...` to `/dev/tty`).

### Dispatch: the C VT parser recognizes `kitty-` and routes each verb

The native parser `parse_kitty_dcs` [kitty/vt-parser.c:586] requires the `kitty-` prefix and then
dispatches by verb:

```c
// kitty/vt-parser.c:600-611
if (!starts_with("kitty-")) return false;
inc("kitty-");
dispatch("cmd{", handle_remote_cmd, 1);
dispatch("overlay-ready|", handle_overlay_ready, 0)
dispatch("kitten-result|", handle_kitten_result, 0)
dispatch("print|", handle_remote_print, 0)
dispatch("echo|", handle_remote_echo, 0)
dispatch("ssh|", handle_remote_ssh, 0)
dispatch("ask|", handle_remote_askpass, 0)
dispatch("clone|", handle_remote_clone, 0)
dispatch("edit|", handle_remote_edit, 0)
```

**All sibling verbs** in that dispatch block (enumerated): `cmd{`, `overlay-ready|`, `kitten-result|`,
`print|`, `echo|`, `ssh|`, `ask|`, `clone|`, `edit|`. **Note precisely:** the askpass verb literal is
**`ask|`, not `askpass|`** [vt-parser.c:609]. The `ssh|` verb routes to `Window.handle_remote_ssh`
[kitty/window.py:1289], which calls `get_ssh_data(msg, f'{os.getpid()}-{self.id}')`
[kitty/window.py:1291] — this is where the request-id `<KITTY_PID>-<KITTY_WINDOW_ID>` (Q2 defense 5)
comes from.

**Observed** — during real runs the harness received exactly these verbs: `kitty-ssh|` (the data
request), `kitty-echo|` (the drain canary), and `kitty-print|` (remote debug):

```text
[sh] <-- DCS kitty-ssh| req#1 (148 b64 bytes)
[sh] <-- DCS kitty-print|: 'debug: ignoreboth or ignorespace present in bash HISTCONTROL setting, ...'
[sh] <-- DCS kitty-echo| DRAIN CANARY (64 bytes)
```

### PUSH vs PULL (cause → effect), and the `SSH_ASKPASS` pull path

- **PUSH (default).** When `cd.request_data` is false the kitten writes the request down the TTY
  itself, right after starting the ssh child:

  ```go
  // kittens/ssh/main.go:760-768
  if !cd.request_data {
      rq := fmt.Sprintf("id=%s:pwfile=%s:pw=%s",
          cd.replacements["REQUEST_ID"], cd.replacements["PASSWORD_FILENAME"], cd.replacements["DATA_PASSWORD"])
      ...
      dcs, err = tui.DCSToKitty("ssh", rq)
      err = term.WriteAllString(dcs)
  }
  ```

  **Observed** — the pushed request arrives with `request_data="0"` baked into the script:

  ```text
  [sh]     decoded = id=58113-1:pwfile=kssh-58114-B6TON5DTJMRV4:pw=ca39a40f...
  ```

- **PULL.** `cd.request_data` becomes false only when `set_askpass` [kittens/ssh/main.go:147] returns
  false, which happens when OpenSSH supports `SSH_ASKPASS_REQUIRE`:

  ```go
  // kittens/ssh/main.go:149-163 (set_askpass)
  sentinel := filepath.Join(utils.CacheDir(), "openssh-is-new-enough-for-askpass")   // :149
  if sentinel_exists || GetSSHVersion().SupportsAskpassRequire() {                   // :152
      need_to_request_data = false
  }
  os.Setenv("SSH_ASKPASS", exe)                          // :160
  os.Setenv("KITTY_KITTEN_RUN_MODULE", "ssh_askpass")    // :161
  if !need_to_request_data {
      os.Setenv("SSH_ASKPASS_REQUIRE", "force")          // :163
  }
  ```

  `SupportsAskpassRequire()` [kittens/ssh/utils.go:206] is true for OpenSSH ≥ 8.4 (here `10.0p2`).
  When the askpass path is disabled (`--kitten askpass=ssh`), `need_to_request_data` stays true, so
  the **remote** emits the request instead [shell-integration/ssh/bootstrap.sh:90,94]. **Observed** —
  the embedded script then carries `request_data="1"`, and the request arrives from the remote after
  the connection is up:

  ```text
  # embedded script (pull): request_data="1"
  [pull] <-- DCS kitty-ssh| req#1 (148 b64 bytes)   (arrives at t≈0.21s, after the ssh connection is established)
  ```

  The askpass helper itself is `RunSSHAskpass` [kittens/ssh/askpass.go:37], selected via the
  `KITTY_KITTEN_RUN_MODULE=ssh_askpass` env var set above.

### The drain canary and its 2-second timeout

After the ssh child exits (`c.Wait()` at [main.go:782]), `run_ssh` calls
`drain_potential_tty_garbage` [main.go:530] at [main.go:783] to flush any stray remote output that may
still be arriving on the TTY. It writes a random echo canary and reads until the canary comes back or
a 2-second deadline elapses:

```go
// kittens/ssh/main.go:535-549
canary, err := secrets.TokenHex()               // :535  → 32 random bytes = 64 hex chars
dcs, err := tui.DCSToKitty("echo", canary)       // :539  kitty-echo| frame
...
give_up_at := time.Now().Add(2 * time.Second)    // :549
for !bytes.Contains(data, q) { ... }
```

**Cause → effect.** Kitty echoes a `kitty-echo|` payload back (base64-decoded, non-printable stripped,
by `handle_remote_echo` [kitty/window.py:1282-1287]); once the kitten sees its own canary it knows the
TTY is drained and returns. The 2-second cap ensures it never blocks forever if the canary is lost.

**Observed — measured, stable across 2 runs each.** The canary is 64 bytes (32 hex-encoded bytes). When
the terminal echoes the canary back, drain completes almost instantly; when the canary echo is
withheld, the kitten waits **exactly the 2-second deadline** before giving up:

```text
normal (canary echoed):   run A  0.003s ;  run B  0.003s     # completes as soon as canary returns
withheld (canary dropped): run A  2.005s ;  run B  2.005s     # == time.Now().Add(2 * time.Second)
```

### The remote base64 fallback chain (all six outcomes)

Because the remote must base64-decode the streamed archive before `tar` can read it, the bootstrap
defines `base64_encode`/`base64_decode` from the first available tool, falling back in order
[shell-integration/ssh/bootstrap.sh:55-73]:

1. **`base64`** — `command base64 | command tr -d \n\r` / `command base64 -d` [bootstrap.sh:55-57].
2. **`openssl`** — `openssl enc -A -base64` / `openssl enc -A -d -base64` [bootstrap.sh:58-60].
3. **`b64encode`/`b64decode`** — BSD tools; encode strips first/last line via `sed '1d;$d'`, decode via
   `fold -w 76 | b64decode -r` [bootstrap.sh:61-63].
4. **Python** (`detect_python`) — `pybase64()` calling `base64.standard_b64encode`/`decode`
   [bootstrap.sh:64-67].
5. **Perl** (`detect_perl`) — `MIME::Base64` `encode_base64`/`decode_base64` [bootstrap.sh:68-70].
6. **else `die`** — `die "base64 executable not present on remote host, ssh kitten cannot function."`
   [bootstrap.sh:72].

**Cause → effect.** A fallback chain exists because the remote host is not controlled by kitty and may
lack any given tool; the kitten degrades from the canonical `base64` binary through `openssl`, BSD
`b64encode`, Python, and Perl, and only aborts (defense against a silently broken transfer) if none is
present. **Observed** — on the localhost target the first branch is taken (the captured script shows
`base64_encode() { command base64 | command tr -d ... }`), and the transfer succeeded (Q4), so branch
1 is the live path here; branches 2–6 are the enumerated siblings from the source (the remaining
branches are **(inferred)** from reading, as `base64` is present on this host).

---

## Coverage pass — every named item across Q1–Q8

Each row: the named item, its exact literal, `file:line`, the evidence label, and the causal reason.
**(obs)** = observed on the real `kitten ssh localhost` path; **(N-C)** = non-canonical; **(inf)** =
inferred from reading.

### Q1 — end-to-end trace
- [x] `func main` — `kittens/ssh/main.go:800` — (obs, entry reached) — dispatch + guards + `run_ssh`.
- [x] `use-python` case — `main.go:804` — (inf) — backwards-compat arg strip.
- [x] `ParseSSHArgs(args, "--kitten")` — `main.go:810` — (obs) — splits ssh/server/override args.
- [x] passthrough `unix.Exec` — `main.go:822` — (inf) — plain ssh when not a kitten invocation.
- [x] Guard 1 `The SSH kitten is meant to run inside a kitty window` — `main.go:825-826` — (obs).
- [x] Guard 2 `The SSH kitten is meant for interactive use only, STDIN must be a terminal` — `main.go:829` — (obs).
- [x] `func run_ssh` — `main.go:597` — (obs) — orchestration.
- [x] `exec.Command(cmd[0], cmd[1:]...)` — `main.go:754` — (obs, 3 ssh invocations) — launches ssh child.
- [x] `tty.OpenControllingTerm(tty.SetNoEcho)` — `main.go:718` — (obs, no-echo TTY) — controlling terminal.
- [x] remote `exec_login_shell` — `shell-integration/ssh/bootstrap.sh:164` — (obs, prompt appeared).

### Q2 — SHM five-part defense
- [x] `bootstrap_script` producer — `main.go:422` — (obs) — creates SHM + payload.
- [x] `secrets.TokenHex()` password — `main.go:431` — (obs, pw in request) — random secret.
- [x] payload `{tarfile, pw, hostname, username}` — `main.go:439-443` — (obs, keys `['hostname','pw','tarfile','username']`).
- [x] `shm.CreateTemp("kssh-%d-", pid, ...)` — `main.go:446`; `WriteWithSize` `:448`; `Flush` `:450` — (obs, `/dev/shm/kssh-…`).
- [x] Go helpers `CreateTemp`/`WriteWithSize`/`ReadWithSize`/`ReadWithSizeAndUnlink` — `tools/utils/shm/shm.go:91,120,129,142` — (inf).
- [x] D1 immediate `shm.unlink()` — `kittens/ssh/utils.py:106` (also `kitty/shm.py:175`) — (obs, "unlink CONFIRMED (gone)").
- [x] D2 owner check `Incorrect owner on pwfile` — `utils.py:107-108` — (obs, PASS: uid/gid 0).
- [x] D3 perm check `Incorrect permissions on pwfile: 0o{mode:03o}` — `utils.py:109-111` — (obs PASS `0o600`; N-C reject `0o644`); default `0o600` at `kitty/shm.py:51`, `O_CREAT|O_EXCL` `:62`.
- [x] D4 `Incorrect password` — `utils.py:130-131` — (obs).
- [x] D5 `Incorrect request id: ...` — `utils.py:132-133` — (obs).
- [x] SHM never transmitted (only base64 tar crosses) — (obs, created+consumed in `/dev/shm`).

### Q3 — bootstrap generation + remote execution
- [x] `prepare_script` defaults `EXEC_CMD`/`EXPORT_HOME_CMD` + `\b<key>\b` word-boundary regex — `main.go:407-418` — (obs, script substituted).
- [x] `wrap_bootstrap_script` sets `rcmd = [exec, Interpreter, -c, unwrap, encoded]` — `main.go:509` — (obs, ssh argv).
- [x] `dcs_to_kitty` — `bootstrap.sh:75`; `request_data="REQUEST_DATA"` `:90`; request line `id=REQUEST_ID:pwfile=PASSWORD_FILENAME:pw=DATA_PASSWORD` `:94` — (obs).
- [x] `untar_and_read_env` `:104`; `mktemp -d "$HOME/.kitty-ssh-kitten-untar-XXXXXXXXXXXX"` `:108`; `tar "xpzf"` `:113`; `compile_terminfo` `:130`; `get_data` `:137`; `EXEC_CMD` `:159`; `exec_login_shell` `:164` — (obs, prompt).
- [x] `bootstrap.py` equivalents — `:22,73,81,133,178,188,203` — (obs, base64-decoded from py argv).
- [x] `bootstrap-utils.sh` `mv_files_and_dirs`/`compile_terminfo`/`prepare_for_exec`/`exec_login_shell` — `:9,18,192,221` — (obs, bundled in archive).
- [x] launcher stubs `shell-integration/ssh/{kitty,kitten}` — (obs, archive entries `kitty/bin/{kitty,kitten}`).

### Q4 — archive + transport
- [x] `gzip.NewWriterLevel(&w, gzip.BestCompression)` — `main.go:259` — (obs, `file`→"max compression").
- [x] `tar.NewWriter` `:263`; `h.Mode |= 0o600` `:268`; `Format: tar.FormatPAX, Mode: 0o644` (add_data) — (obs, `tar -tzf`).
- [x] `yield b'\nKITTY_DATA_START\n'` — `utils.py:117` — (obs, leading bytes).
- [x] `yield b'OK\n'` — `utils.py:138` — (obs).
- [x] `line_sz = 254` — `utils.py:143` — (obs, non-last chunks all 254, **stable ×2**); reason: macOS 255-byte input-queue limit (comment `:140-142`).
- [x] `yield b'KITTY_DATA_END\n'` — `utils.py:148` — (obs, trailing bytes).

### Q5 — connection_data (all 16 fields)
- [x] All 16 fields — `main.go:171-189` — enumerated in the Q5 table with per-field population site and observed value (`request_id=58113-1`, `shm_name=kssh-58114-…`, `script_type` sh+py, `request_data` 0+1, `echo_on=1`, `rcmd`, `replacements`, `host_opts` defaults, …).

### Q6 — connection reuse (all six -o options + decision)
- [x] `ControlMaster=auto` — `main.go:138` — (obs).
- [x] `ControlPath=<rd>/kssh-<pid>-%C` — `main.go:139` — (obs `/root/.cache/kitty/run/kssh-58114-%C`).
- [x] `ControlPersist=yes` — `main.go:140` — (obs).
- [x] `ServerAliveInterval=60` — `main.go:141` — (obs).
- [x] `ServerAliveCountMax=5` — `main.go:142` — (obs).
- [x] `TCPKeepAlive=no` — `main.go:143` — (obs).
- [x] template `kssh-{kitty_pid}-{ssh_placeholder}` → `%C` — `kitty/constants.py:188`, `gen/go_code.py:599`, `main.go:135-136` — (obs).
- [x] `ssh -O check` (`master_is_functional`) — `main.go:658-659` — (obs, PULL run).
- [x] `run_control_master` `-N -f` — `main.go:669` — (obs, forward run argv).
- [x] forward `-R 0:<listen_on> -O forward` — `main.go:702` — (obs, forward run argv).
- [x] abstract-socket rejection — `main.go:701` — (obs, verbatim error).
- [x] push-vs-request gate `need_to_request_data && Share_connections && master_is_functional()` — `main.go:663-664` — (obs, both branches).

### Q7 — shell encoding (four substitutions + inverse + base64 path)
- [x] `'` → `\v` — `main.go:505` — (obs, `\vbuffer\v`).
- [x] `\` → `\f` — `main.go:505` — (obs, `\f033`).
- [x] `\n` → `\r` — `main.go:505` — (obs, `\r` linebreaks).
- [x] `!` → `\b` — `main.go:505` — (obs, `#\b/bin/sh`).
- [x] `tr` inverse `tr \v\f\r\b \047\134\n\041` — `main.go:506` — (obs, argv[18]).
- [x] py base64 `EncodeToString` / `standard_b64decode` — `main.go:501-502` — (obs, argv[18]/[19] decode to bootstrap.py).
- [x] `is_python` branch — `main.go:514-517` — (obs, sh default + py forced).
- [x] `sh`/`bash` vs Python vs `tcsh` distinction — (explained: shell uses char-subst since base64 not guaranteed; `!`→`\b` and `\n`→`\r` specifically for tcsh; python uses base64).

### Q8 — DCS protocol + pull + drain + fallbacks
- [x] DCS frame `\x1bP@kitty-<verb>|<b64>\033\\` — `dcs_to_kitty.go:16,25`; tmux variant `:23`; remote `bootstrap.sh:75` — (obs).
- [x] VT dispatch prefix `kitty-` — `vt-parser.c:600` — (inf) — required prefix.
- [x] all 9 verbs `cmd{`,`overlay-ready|`,`kitten-result|`,`print|`,`echo|`,`ssh|`,`ask|`,`clone|`,`edit|` — `vt-parser.c:603-611` — (obs for ssh|/echo|/print|; others inf); **`ask|` not `askpass|`**.
- [x] `handle_remote_ssh` → `get_ssh_data(msg, f'{os.getpid()}-{self.id}')` — `window.py:1289,1291` — (obs, request_id `<pid>-<wid>`).
- [x] PUSH `id=%s:pwfile=%s:pw=%s` + `DCSToKitty("ssh", rq)` — `main.go:762,766` — (obs, `request_data="0"`).
- [x] PULL via `set_askpass` env vars `SSH_ASKPASS` `:160`, `KITTY_KITTEN_RUN_MODULE=ssh_askpass` `:161`, `SSH_ASKPASS_REQUIRE=force` `:163`; sentinel `openssh-is-new-enough-for-askpass` `:149`; `SupportsAskpassRequire()` `utils.go:206`; remote request `bootstrap.sh:90,94` — (obs, `request_data="1"`).
- [x] `RunSSHAskpass` — `askpass.go:37` — (inf).
- [x] drain canary `secrets.TokenHex()` `:535`, `DCSToKitty("echo", canary)` `:539`, `2 * time.Second` `:549` — (obs, **2.005s stable ×2**; 64-byte canary).
- [x] six base64 fallbacks — `bootstrap.sh:55-72` — (obs branch 1 `base64`; branches 2–6 inf) — degrade openssl→b64encode→python→perl→`die`.

### Environment variables named across the questions
- [x] `KITTY_WINDOW_ID`, `KITTY_PID` — guards `main.go:825`; request-id `main.go:424`, `window.py:1291` — (obs).
- [x] `SSH_ASKPASS`, `SSH_ASKPASS_REQUIRE`, `KITTY_KITTEN_RUN_MODULE` — `main.go:160-163` — (obs env set on pull decision).

### Placeholder tokens
- [x] `{kitty_pid}`, `{ssh_placeholder}`→`%C` — `main.go:135-136` — (obs `kssh-58114-%C`).
- [x] `REQUEST_ID`, `PASSWORD_FILENAME`, `DATA_PASSWORD` — `main.go:762`, `bootstrap.sh:94` — (obs in pushed request).
- [x] `EXEC_CMD`, `EXPORT_HOME_CMD` — `main.go:409,412`; `bootstrap.sh:159` — (obs empty defaults in captured script).

### Measured values (stability)
- [x] `line_sz = 254` — stable across 2 runs (all non-last chunks 254).
- [x] drain `2 * time.Second` — measured 2.005s across 2 withheld runs.
- [x] `ServerAliveInterval=60` / `ServerAliveCountMax=5` — source literals confirmed verbatim in every ssh-child argv capture.

**Result:** every distinct item named across Q1–Q8 — each mechanism, function, condition, file, flag,
env var, placeholder, and every "e.g./such as/including/like" sibling — is addressed above with its
exact literal, `file:line`, an evidence label (observed / non-canonical / inferred), and its causal
reason. Both `script_type` branches (`sh` + `py`) and both credential paths (push + pull) were
exercised on the real `kitten ssh localhost` entry point.

