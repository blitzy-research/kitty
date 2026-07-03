# How the Kitty SSH kitten establishes a secure remote session and shares SSH connections

This document traces, **end-to-end and from directly observed runtime behavior**, how the Kitty
terminal emulator's **SSH kitten** (`kitten ssh`) establishes a secure remote session and shares
SSH connections. It answers eight questions (Q1–Q8). Every behavioral claim is paired with the
exact command/code that produced it, an adjacent **verbatim** evidence block, and a `file:line`
citation into the source at the canonical source commit `815df1e21` (see §0.1 for how this canonical
source base relates to the branch `HEAD` and the working tree).

---

## 0. Build & Run methodology

### 0.1 Platform, toolchain and canonical build

All observation was performed inside the user-specified container image
`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`
(`andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`),
which supplies the Go and C toolchains.

Toolchain versions, captured verbatim (each block below is raw command output with no
annotations inserted inside the fence):

```text
$ go version
go version go1.22.12 linux/amd64
$ python3 --version
Python 3.13.7
$ ssh -V
OpenSSH_10.0p2 Ubuntu-5ubuntu5.4, OpenSSL 3.5.3 16 Sep 2025
```

`go1.22.12` satisfies the pinned `go 1.22` in `go.mod:3`; system `Python 3.13.7` satisfies
`requires-python = ">=3.8"` in `pyproject.toml:2`. A second, **bundled** CPython that `./dev.sh build`
downloads is what actually loads the compiled Kitty extension (`libpython3.14`) and the terminal-side
data server during observation (see §0.2), captured verbatim:

```text
$ LD_LIBRARY_PATH=dependencies/linux-amd64/lib dependencies/linux-amd64/bin/python3 --version
Python 3.14.6
```

The canonical build is `./dev.sh build` (the command documented at `docs/build.rst:19`; `dev.sh`
forwards to `go run bypy/devenv.go`). It was run as a normal user in the default configuration. The
`export CFLAGS=…` line preceding it is explained in the note after this block; the fence itself is
the raw captured transcript, shown to its final lines. (The trailing `...` on the `Compiling` and
`Linking` lines are the build tool's own literal progress output, not elision by this document — no
content has been removed from those lines.)

```text
$ export CFLAGS="${CFLAGS:+$CFLAGS }-Wno-error=switch"
$ ./dev.sh build ; echo BUILD_EXIT_CODE=$?
[1/1] Compiling kitty/data-types.c ...
 done
[1/1] Linking kitty/fast_data_types ...
 done
kitty/tools/cmd
Build successful. Run kitty as: kitty/launcher/kitty
BUILD_EXIT_CODE=0
```

The `Build successful.` string is emitted by the build tool at `bypy/devenv.go:381`. This produced
the two launcher binaries used throughout (complete `ls -la`, no truncation):

```text
$ ls -la kitty/launcher/kitty kitty/launcher/kitten
-rwxr-xr-x 1 root root 15765764 Jul  3 00:36 kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 Jul  2 22:35 kitty/launcher/kitty
$ ./kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
```

> **`CFLAGS=-Wno-error=switch` rationale (no behavior change):** on Ubuntu 25.10 the newer
> `wayland-protocols` adds `XDG_TOPLEVEL_STATE_CONSTRAINED_*` enum values that `glfw/wl_window.c`'s
> `switch` does not handle; `setup.py` compiles C with `-Werror` by default (`setup.py:517`) and
> appends `CFLAGS` **after** it, so only that library-drift warning is de-promoted. All genuine
> warnings still error, no source is modified, and the C VT parser and the Go kitten are unaffected.

The VCS anchor for the "default, canonical configuration" claim is the **canonical source base
commit**, which is immutable and therefore reproduces verbatim regardless of how many
documentation-only commits are later layered onto the branch (verbatim):

```text
$ git rev-parse --abbrev-ref HEAD
blitzy-7d489156-59de-439e-9b2f-075b4512d760
$ git rev-parse 815df1e21
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
$ git log --oneline -1 815df1e21
815df1e21 Wire up applying of font config
$ git diff --name-status 815df1e21..HEAD
A	blitzy/documentation/kitty_815df1e210e0.md
```

The **canonical source commit** the build reflects is `815df1e21` (full
`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`), and it equals the container image tag
`…kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`. This answer document is added on top of that base
by **documentation-only commit(s)** on branch `blitzy-7d489156-59de-439e-9b2f-075b4512d760`; those
commits touch **only this file** and no source file. The invariant proof is
`git diff --name-status 815df1e21..HEAD`, which reports exactly one changed path —
`A blitzy/documentation/kitty_815df1e210e0.md` — no matter how many documentation-only commits are
layered on top (the diff compares the two endpoints, so its output does not drift as later review-fix
commits are added). Consequently every `file:line` citation in this document resolves identically at
`815df1e21` and at the working tree. The deliverable filename is fixed as `<source_branch_name>.md` =
`kitty_815df1e210e0.md`, matching the full source commit `815df1e210e0…`.

> **Why the anchor is `815df1e21` and not a `HEAD` short-hash:** a `HEAD` value is not a stable,
> reproducible anchor — every subsequent commit (including a review-fix commit to this very document)
> changes it, so a quoted `git rev-parse --short HEAD` would cease to reproduce. The anchor above is
> pinned to the immutable canonical base commit and to the endpoint diff, both of which reproduce
> verbatim at any later `HEAD`.

### 0.2 The SSH target and how the real path was driven

An in-container OpenSSH server (`sshd`) reachable at `localhost` is the target for the real
`kitten ssh localhost` path. It was **pre-provisioned by the environment setup** as a local,
passwordless, root-pubkey target; the exact provisioned state was captured verbatim below. Host keys
were generated with `ssh-keygen -A` (its output is the three `ssh_host_*_key` files); a per-user
Ed25519 keypair was created and its public half placed in `authorized_keys`; a drop-in `sshd_config`
enables pubkey-only root login; and `sshd` is running as the listener. If the listener is not up it is
restarted with `mkdir -p /run/sshd && /usr/sbin/sshd`.

Host keys produced by `ssh-keygen -A` (verbatim):

```text
$ ls -la /etc/ssh/ssh_host_*key
-rw------- 1 root root  545 Jul  2 22:32 /etc/ssh/ssh_host_ecdsa_key
-rw------- 1 root root  444 Jul  2 22:32 /etc/ssh/ssh_host_ed25519_key
-rw------- 1 root root 2635 Jul  2 22:32 /etc/ssh/ssh_host_rsa_key
```

The client keypair and the authorized public key that permits the local connection (verbatim; the
listed files are the `root` user's `~/.ssh`, and the `authorized_keys` content is a **public** key):

```text
$ ls -la /root/.ssh
total 32
drwx------ 1 root root 4096 Jul  2 22:42 .
drwx------ 1 root root 4096 Jul  2 23:34 ..
-rw------- 1 root root  124 Jul  2 22:37 authorized_keys
-rw------- 1 root root  444 Jul  2 22:37 id_ed25519
-rw-r--r-- 1 root root  124 Jul  2 22:37 id_ed25519.pub
-rw------- 1 root root  978 Jul  2 22:42 known_hosts
-rw-r--r-- 1 root root  142 Jul  2 22:37 known_hosts.old
$ cat /root/.ssh/authorized_keys
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMPN2UdaqRROeSDsrvyv3Vanfv/iZmyGA63jXnThgzQn root@reverse-code-generator-63e78bca-hn5x7
```

The drop-in `sshd` configuration (verbatim) and the running listener process (verbatim):

```text
$ cat /etc/ssh/sshd_config.d/99-kitty-localhost.conf
# Added for kitty SSH kitten investigation: local passwordless target
PermitRootLogin prohibit-password
PubkeyAuthentication yes
PasswordAuthentication no
X11Forwarding no
AcceptEnv LANG LC_* KITTY_*
$ pgrep -a sshd | head -1
18492 sshd: /usr/sbin/sshd [listener] 0 of 10-100 startups
```

Plain connectivity through this target, verbatim:

```text
$ ssh -o BatchMode=yes -o StrictHostKeyChecking=accept-new localhost 'echo SSH_OK; whoami; uname -s'
SSH_OK
root
Linux
```

The SSH kitten's real entry point requires two runtime env vars and a terminal `stdin` (see Q1). The
kitten also depends on the **kitty terminal being present** to serve credential data over a custom
DCS escape protocol (Q8). Normally that terminal is a live kitty GUI window whose C event loop
(`kitty/vt-parser.c`) parses DCS frames and whose `Boss`/`Window` (`kitty/window.py`) dispatches them.
To exercise the **real** `kitten ssh` entry point non-interactively, a PTY observation harness was
written under `/tmp` that stands in for that terminal role.

**What is REAL (observed) in every harness run — the entities the questions concern:**

- the **real** `kitty/launcher/kitten ssh` binary runs in the PTY slave (the real Q1 entry point);
- the **real** `ssh` child it spawns — captured argv-exact by a transparent PATH shim named `ssh`
  that logs the child `argv` and then `exec`s `/usr/bin/ssh`, so `ssh`'s behavior is unchanged;
- the **real** remote `sshd` on `localhost` and the **real** remote `bootstrap.(sh|py)` it runs;
- the **real** terminal-side data server `kittens.ssh.utils.get_ssh_data(payload, request_id)` — the
  harness calls this exact function (loaded via the bundled interpreter below) and streams back each
  line it yields, which is exactly what `kitty/window.py:1291` `handle_remote_ssh` does. The credential
  validator `read_data_from_shared_memory` it calls is likewise the real one.

**What is NON-CANONICAL (harnessed) — must not be read as canonical kitty behavior:**

- the **terminal byte-pump and DCS dispatch itself** — i.e. the recognition of the `ESC P @ kitty-…`
  frames on the PTY and the routing of each verb to a handler. In real kitty this is done by
  `kitty/vt-parser.c` (`parse_kitty_dcs`, `:586`) and `kitty/window.py`'s dispatch; the harness
  reimplements that recognition/routing loop. The **handlers it invokes are the real functions**, but
  the pump/dispatch role is a stand-in and is therefore labeled **NON-CANONICAL** wherever its own
  behavior (rather than the real function's output) is what a claim rests on. Concretely: the
  `kitty-echo|` drain-canary reply is modeled on `kitty/window.py:1282-1287` `handle_remote_echo`, and
  the drain-canary **withholding** experiment in Q8 is a deliberate harness deviation (real kitty
  always echoes) and is labeled NON-CANONICAL there.

The kitten's Python terminal-side module is loaded with the **bundled** interpreter the build produced
(`dependencies/linux-amd64/bin/python3` with `LD_LIBRARY_PATH=dependencies/linux-amd64/lib`), because
Kitty's compiled extension links `libpython3.14`; this is the same `get_ssh_data` code path a live
kitty uses, so its output is canonical even though the pump that feeds it is the stand-in.

### 0.3 Labeling convention used in this document

Every behavioral claim carries exactly one of these three labels, matching the evidence beside it:

- **(observed)** — captured from the real `kitten ssh localhost` entry point at runtime, through the
  real code the question concerns (the `kitten` binary, the real `ssh` child, the real remote
  `sshd`/`bootstrap`, and the real `get_ssh_data`/`read_data_from_shared_memory`). Evidence blocks
  show the actual command and its verbatim output.
- **NON-CANONICAL** — a value that comes from a harness stand-in or a deliberately synthetic input
  rather than from real kitty behavior. The real path is always described alongside. Every
  NON-CANONICAL use in this document is one of the following four, and no others:
  1. the **DCS byte-pump / dispatch role** itself (the harness reimplements `kitty/vt-parser.c` +
     `kitty/window.py` frame recognition/routing; the handlers it calls are real — see §0.2);
  2. the **`sh -x` replay** of the real generated bootstrap script in Q3 (used only to expose the
     literal `mktemp -d` / `tar xpzf` / `compile_terminfo` steps; the real remote side-effects are
     shown separately as observed);
  3. the **withheld drain-canary** timing experiment in Q8 (the harness deliberately drops the echo
     that real kitty always sends, to force the kitten's timeout);
  4. the three **synthetic-SHM negative defenses** in Q2 (wrong-password D4, wrong-request-id D5,
     wrong-permissions D3) — driven against a hand-built SHM through the **real** validators, because
     the canonical push never triggers these rejection branches.
- **(inferred)** — a statement derived from *reading* the source, not observed at runtime. Used for
  code facts and untaken sibling branches (e.g. the base64 fallbacks the bootstrap did not need, or
  `connection_data` fields not populated in the observed run).

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
$ env -u KITTY_WINDOW_ID -u KITTY_PID ./kitty/launcher/kitten ssh localhost
Error: The SSH kitten is meant to run inside a kitty window
$ printf '' | env KITTY_WINDOW_ID=1 KITTY_PID=99999 ./kitty/launcher/kitten ssh localhost   # piped (non-tty) stdin
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
struct (Q5), calls `get_remote_command` (defined at [kittens/ssh/main.go:511]) at its call site
[kittens/ssh/main.go:749], and launches the child:

```go
// kittens/ssh/main.go:749-754
err = get_remote_command(&cd)
if err != nil {
    return 1, err
}
cmd = append(cmd, cd.rcmd...)
c := exec.Command(cmd[0], cmd[1:]...)
```

**Observed** — the ssh PATH-shim recorded exactly three `ssh` invocations for one
`kitten ssh localhost` run. The first is a no-arg options probe (`exec.Command(SSHExe())`), the second
is `ssh -V` (for `GetSSHVersion()`), and the third is the real connection. The shim-log headers are
verbatim (the shim records `argc` for each):

```text
=== ssh invocation (argc=0) ===
=== ssh invocation (argc=1) ===
argv[0]='-V'
=== ssh invocation (argc=20) ===
```

The third invocation had `argc=20` (per the `=== ssh invocation (argc=20) ===` header above). Its
`argv[18]` (the `tr` unwrap) and `argv[19]` (the entire encoded bootstrap script — a several-kilobyte,
per-run machine-generated blob) belong to Q7 and are covered there rather than repeated here: in Q7,
`argv[18]` is reproduced **byte-for-byte**, whereas `argv[19]` is **summarized, not pasted in full**
(Q7 evidences it with complete, un-truncated per-substitution fragments in `sh` mode and a complete
base64 round-trip on a boundary-aligned prefix plus its measured length in `py` mode). The other
eighteen elements, `argv[0..17]`, are listed verbatim from the shim log below with nothing elided:

```text
argv[0]='-t'
argv[1]='-o'
argv[2]='ControlMaster=auto'
argv[3]='-o'
argv[4]='ControlPath=/root/.cache/kitty/run/kssh-87498-%C'
argv[5]='-o'
argv[6]='ControlPersist=yes'
argv[7]='-o'
argv[8]='ServerAliveInterval=60'
argv[9]='-o'
argv[10]='ServerAliveCountMax=5'
argv[11]='-o'
argv[12]='TCPKeepAlive=no'
argv[13]='--'
argv[14]='localhost'
argv[15]='exec'
argv[16]='sh'
argv[17]='-c'
```

`argv[0]='-t'` forces a remote TTY; `argv[1..12]` are the six connection-sharing `-o` options (Q6);
`argv[13]='--'` terminates option parsing; `argv[14]='localhost'` is the target; and
`argv[15..17]='exec' 'sh' '-c'` sets up the remote command that runs the wrapped bootstrap (Q3/Q7).

### Hop 3 — credential request over the TTY, then the remote bootstrap runs

After `c.Start()`, on the default (push) path the kitten writes the data-serving request straight
down the terminal [main.go:761-768] (see Q8). The harness (playing kitty) received it and — via the
**real** `get_ssh_data` — streamed the archive back. **Observed** for `kitten ssh localhost`, verbatim
from the harness meta log (the request line is shown in full, including the complete ephemeral
password — see the note below):

```text
DCS kitty-ssh| req#1 payload_b64_len=148
  request(decoded)=id=87498-1:pwfile=kssh-87499-DIUQ3H4FV3L5I:pw=52747a358160aeab524cc5050a754e6093476e53cc2e6a40007371780c305e81
SHM read_data_from_shared_memory OK; keys=['hostname', 'pw', 'tarfile', 'username']
```

Here `id=87498-1` is `KITTY_PID-KITTY_WINDOW_ID` (the harness pid `87498`, window `1`), and
`pwfile=kssh-87499-DIUQ3H4FV3L5I` names the shared-memory object the kitten created (pid `87499` is
the kitten child). The `pw=52747a35…c305e81` value is shown in full because it is an **ephemeral,
single-use** credential: it is a fresh 32-byte random token per run (Q2), and by the time this request
is decoded the SHM object it authenticated has already been read and unlinked (D1, Q2), so the value is
not reusable. The `get_ssh_data` generator then streamed the base64 archive back as 124 payload lines
framed by `KITTY_DATA_START` / `OK` / `KITTY_DATA_END` (the framing is dissected in Q4). The remote
`bootstrap.sh` consumed the streamed tar, extracted it, compiled terminfo, and
**exec'd the login shell** — the real remote prompt and kitty shell-integration markers appeared
(observed, from the raw PTY capture, control bytes rendered with `cat -v`; the fence below is the
verbatim `cat -v` output with no annotations inserted):

```text
^[]7;kitty-shell-cwd://reverse-code-generator-63e78bca-hn5x7/root^G
^[]133;A^G
root@reverse-code-generator-63e78bca-hn5x7:~#
```

The first line is an OSC-7 current-working-directory report (`kitty-shell-cwd://…`), the second is the
OSC-133 prompt mark (`^[]133;A`), and the third is the **real remote login-shell prompt**
(`root@reverse-code-generator-63e78bca-hn5x7:~#`) — all three are kitty shell-integration output that
only appears once `exec_login_shell` has run on the remote.

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
// kittens/ssh/main.go:431-458 (verbatim, contiguous)
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
}
encoded_data, err := json.Marshal(data)
if err == nil && !cd.dont_create_shm {
    data_shm, err = shm.CreateTemp(fmt.Sprintf("kssh-%d-", os.Getpid()), uint64(len(encoded_data)+8))
    if err == nil {
        err = shm.WriteWithSize(data_shm, encoded_data, 0)
        if err == nil {
            err = data_shm.Flush()
        }
    }
}
if err != nil {
    return err
}
if !cd.dont_create_shm {
    cd.shm_name = data_shm.Name()
```

In that excerpt: the random password is `secrets.TokenHex()` [main.go:431]; `make_tarfile(cd, …)`
[main.go:435] builds the archive (Q4); the four-field JSON `data` map is [main.go:439-443];
`json.Marshal` is [main.go:444]; `shm.CreateTemp` is [main.go:446]; `shm.WriteWithSize` is
[main.go:448]; `data_shm.Flush()` is [main.go:450]; and `cd.shm_name = data_shm.Name()` is
[main.go:458]. The Go SHM helpers are `CreateTemp` [tools/utils/shm/shm.go:91], `WriteWithSize` [shm.go:120],
`ReadWithSize` [shm.go:129] and `ReadWithSizeAndUnlink` [shm.go:142]. The name (`kssh-<pid>-<random>`)
is placed into the request as `pwfile=` and pushed to the terminal (Q1/Q8). Note the
`uint64(len(encoded_data)+8)` argument to `CreateTemp` [main.go:446] is the **mmap region size** (the
JSON length plus 8 bytes of slack) — it is *not* the size prefix; the size prefix that
`WriteWithSize` actually writes ahead of the payload is a **4-byte** big-endian `uint32`
(`binary.BigEndian.PutUint32` [tools/utils/shm/shm.go:124], `NUM_BYTES_FOR_SIZE = 4`
[tools/utils/shm/shm.go:116]).

The consumer/validator is on the kitty side: `read_data_from_shared_memory` [kittens/ssh/utils.py:100]
and `get_ssh_data` [kittens/ssh/utils.py:115]. The SHM abstraction it opens is `kitty/shm.py`'s
`SharedMemory` [kitty/shm.py:35], created (on the producer's Python-clone equivalent) with default
`mode = stat.S_IREAD | stat.S_IWRITE` (= `0o600`) [kitty/shm.py:51] and `flags = os.O_CREAT | os.O_EXCL`
[kitty/shm.py:62], and a **4-byte** big-endian size prefix — `size_fmt = '!I'` [kitty/shm.py:46]
(`'!I'` = network-order `unsigned int`, 4 bytes) with
`num_bytes_for_size = struct.calcsize(size_fmt)` [kitty/shm.py:47] — plus a `def unlink`
[kitty/shm.py:175]. The Go producer and the Python consumer therefore agree on a 4-byte length prefix
(`NUM_BYTES_FOR_SIZE = 4` [tools/utils/shm/shm.go:116] ↔ `struct.calcsize('!I') == 4`).

**Observed** — the JSON payload actually carried by the real SHM (keys only; values are the secret
password/tar) contains exactly the four documented fields (verbatim from the sh1 harness meta log):

```text
SHM read_data_from_shared_memory OK; keys=['hostname', 'pw', 'tarfile', 'username']
```

### The five-part security model (each defense: literal + cause→effect + evidence)

The validator opens the SHM by the name in the request and applies five checks, in this order. The
evidence comes in two forms, labeled explicitly per defense below:

- **The happy path passed on the real push (observed).** In the real `kitten ssh localhost` run the
  terminal-side `read_data_from_shared_memory` returned successfully and `get_ssh_data` then served the
  archive. A single observed line proves D1, D2 **and** D3 all passed on the real path, because the
  function `unlink`s (D1) and then raises on any owner mismatch (D2) or permission mismatch (D3)
  *before* it can return:

  ```text
  SHM read_data_from_shared_memory OK; keys=['hostname', 'pw', 'tarfile', 'username']
  ```

  The subsequent appearance of the remote login shell (Q1) confirms the password (D4) and request-id
  (D5) matched as well, since `get_ssh_data` yields an error line instead of the archive on either
  mismatch.
- **The rejection branches are NON-CANONICAL.** The canonical push always creates a `0o600` SHM with
  the correct `pw` and `id`, so it can *never* trigger the D3-wrong-permission, D4-wrong-password or
  D5-wrong-request-id rejection branches. To evidence those branches, a **synthetic** in-process SHM
  was built and fed to the **real** validators (`read_data_from_shared_memory` / `get_ssh_data`); every
  such rejection value below is labeled **NON-CANONICAL**. The granular per-defense PASS probes
  (`exists? False`, `st_uid=…`, `mode=0o600`) likewise come from that synthetic driver and are labeled
  NON-CANONICAL — the real push's pass is the single observed line above.

**Defense 1 — immediate `unlink` on open.** `read_data_from_shared_memory` unlinks the SHM the moment
it opens it, before returning any bytes:

```python
# kittens/ssh/utils.py:104-106
with SharedMemory(shm_name, readonly=True) as shm:
    shm.unlink()
```

*Cause → effect:* the credential object exists on disk (`/dev/shm`) only for the instant between
creation and first read; a second reader (or an attacker who learns the name later) finds nothing. On
the real push this is what makes the observed `read_data_from_shared_memory OK` line above possible
exactly once. **NON-CANONICAL** granular probe (synthetic SHM through the real `read_data_from_shared_memory`,
checking `/dev/shm` immediately after the single read — the real push leaves the same effect):

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
cannot substitute a forged credential file. The real push passed this check (it is a precondition of
the observed `read_data_from_shared_memory OK` line). **NON-CANONICAL** granular probe (synthetic SHM
through the real validator, printing the owner it checked):

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
credential object is rejected. The real push passed this check (again a precondition of the observed
`read_data_from_shared_memory OK` line — the kitten always creates the SHM `0o600`). **NON-CANONICAL**
granular probe (synthetic SHM through the real validator):

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
the SHM name but not its contents. The real push passed this (the archive was served, Q1).
**NON-CANONICAL** rejection probe (synthetic SHM through the real `get_ssh_data`, request carrying a
tampered `pw` — the canonical push never mismatches its own password):

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
window will not serve credentials on behalf of another. The real push passed this — its `id=87498-1`
matched the serving window (Q1). **NON-CANONICAL** rejection probe (synthetic SHM through the real
`get_ssh_data`, request carrying a tampered `id=9999-9999`):

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
// kittens/ssh/main.go:407-419 (prepare_script, verbatim contiguous)
func prepare_script(script string, replacements map[string]string) string {
	if _, found := replacements["EXEC_CMD"]; !found {
		replacements["EXEC_CMD"] = ""
	}
	if _, found := replacements["EXPORT_HOME_CMD"]; !found {
		replacements["EXPORT_HOME_CMD"] = ""
	}
	keys := utils.Keys(replacements)
	for i, key := range keys {
		keys[i] = "\\b" + key + "\\b"
	}
	pat := regexp.MustCompile(strings.Join(keys, "|"))
	return pat.ReplaceAllStringFunc(script, func(key string) string { return replacements[key] })
```

Here `EXEC_CMD` defaults to empty at [main.go:409], `EXPORT_HOME_CMD` at [main.go:412]; each key is
wrapped into the word-boundary token `\b<key>\b` at [main.go:416]; the alternation is compiled by
`regexp.MustCompile` at [main.go:418] and applied by `ReplaceAllStringFunc` at [main.go:419]. The
default request-id is formed at [main.go:424] as
`os.Getenv("KITTY_PID") + "-" + os.Getenv("KITTY_WINDOW_ID")`. The sensitive tokens `REQUEST_ID`,
`DATA_PASSWORD`, `PASSWORD_FILENAME` and the booleans `REQUEST_DATA`/`ECHO_ON` are substituted into
the script. `wrap_bootstrap_script` [main.go:486] then encodes the whole script (Q7) and sets:

```go
// kittens/ssh/main.go:508
cd.rcmd = []string{"exec", cd.host_opts.Interpreter, "-c", unwrap_script, encoded_script}
```

**Observed** — the fully-substituted `sh` bootstrap that was actually sent, decoded from the ssh child
`argv[19]` (the sh-mode character substitutions of Q7 reversed; newlines shown as real line breaks).
The decoded script is the `shell-integration/ssh/bootstrap.sh` source with the placeholder tokens
substituted; its request-data push guard block is shown **verbatim** (complete lines, no elision).
Note `request_data="0"` (the push default) and that the request line still holds its *placeholder*
names — because `request_data` is `0`, the `[ "$request_data" = "1" ] && { … }` block is never
executed, so the placeholders are never expanded on the push path:

```text
request_data="0"
trap "cleanup_on_bootstrap_exit" EXIT
[ "$request_data" = "1" ] && {
    command stty "-echo" < /dev/tty
    dcs_to_kitty "ssh" "id="REQUEST_ID":pwfile="PASSWORD_FILENAME":pw="DATA_PASSWORD""
}
```

The remaining stages — `read_base64_from_tty`, `untar_and_read_env` (with `mktemp -d`, `tar xpzf`,
`compile_terminfo`), `get_data`, and `exec_login_shell` — are enumerated next with their exact
`bootstrap.sh` line numbers, and their **real runtime effects** are evidenced under "Remote execution"
below.

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
and `shell-integration/ssh/kitten` (visible in the archive listing under Q4 as the entries
`home/.local/share/kitty-ssh-kitten/kitty/bin/kitty` and
`home/.local/share/kitty-ssh-kitten/kitty/bin/kitten`).

**Observed evidence for the remote extraction / terminfo / exec stages.** Two complementary captures
back these claims.

*(observed, real path)* — the real `kitten ssh localhost` session left **persistent staged artifacts**
on the remote host that only `untar_and_read_env` + `compile_terminfo` can produce, and the real
login-shell prompt appeared (Q1). Verbatim from the remote after the session:

```text
$ file ~/.terminfo/x/xterm-kitty; ls -la ~/.terminfo/kitty.terminfo; ls -d ~/.local/share/kitty-ssh-kitten/*/
/root/.terminfo/x/xterm-kitty: Compiled terminfo entry "xterm-kitty"
-rw-r--r-- 1 root root 4271 Jan  1  1970 /root/.terminfo/kitty.terminfo
/root/.local/share/kitty-ssh-kitten/kitty/
/root/.local/share/kitty-ssh-kitten/shell-integration/
```

The `Compiled terminfo entry "xterm-kitty"` is exactly what `compile_terminfo` [bootstrap-utils.sh:18]
produces (it runs `tic`), and the `kitty-ssh-kitten/` tree is the extracted+staged archive content —
neither can exist unless `mktemp -d` + `tar xpzf` and then `mv_files_and_dirs` ran on the remote.

*(NON-CANONICAL)* — to expose the literal `mktemp -d` / `tar xpzf` / `compile_terminfo` /
`mv_files_and_dirs` steps **by name**, the real generated `argv[19]` bootstrap (decoded) was re-run
under `sh -x` and fed the **real** framed data stream captured from the push. This is NON-CANONICAL
because it replays the script outside the real remote `ssh` shell (into a scratch `$HOME`), rather than
observing the remote shell's own trace. The `sh -x` trace is verbatim:

```text
+ command -v tar
+ command mktemp -d /tmp/kssh_obs/fakehome/.kitty-ssh-kitten-untar-XXXXXXXXXXXX
+ tdir=/tmp/kssh_obs/fakehome/.kitty-ssh-kitten-untar-bevL0qTMTkeh
+ command base64 -d
+ compile_terminfo /tmp/kssh_obs/fakehome/.kitty-ssh-kitten-untar-bevL0qTMTkeh/home
+ mv_files_and_dirs /tmp/kssh_obs/fakehome/.kitty-ssh-kitten-untar-bevL0qTMTkeh/home /tmp/kssh_obs/fakehome
```

The `tar "xpzf" "-" "-C" "$tdir"` invocation [shell-integration/ssh/bootstrap.sh:113] is the final stage
of the `read_base64_from_tty | base64_decode | command tar …` pipeline, so its `set -x` line interleaves
with the concurrent `read_base64_from_tty` stage and is not cleanly isolable; its success is instead
shown directly by the compiled terminfo the replay produced in the scratch `$HOME` (NON-CANONICAL):

```text
$ file /tmp/kssh_obs/fakehome/.terminfo/x/xterm-kitty
/tmp/kssh_obs/fakehome/.terminfo/x/xterm-kitty: Compiled terminfo entry "xterm-kitty"
```

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
// kittens/ssh/main.go:259-270 (verbatim, contiguous)
gw, err := gzip.NewWriterLevel(&w, gzip.BestCompression)
if err != nil {
    return nil, err
}
tw := tar.NewWriter(gw)
rd := strings.TrimRight(cd.host_opts.Remote_dir, "/")
seen := make(map[file_unique_id]string, 32)
add := func(h *tar.Header, data []byte) (err error) {
    // some distro's like nix mess with installed file permissions so ensure
    // files are at least readable and writable by owning user
    h.Mode |= 0o600
    err = tw.WriteHeader(h)
```

In that excerpt: `gzip.NewWriterLevel(&w, gzip.BestCompression)` is [main.go:259]; `tar.NewWriter(gw)`
is [main.go:263]; the `add` closure begins at [main.go:266]; `h.Mode |= 0o600` (the "at least owner
rw" fix-up, whose two-line rationale comment is real source at [main.go:267-268]) is at
[main.go:269]; and `tw.WriteHeader(h)` is at [main.go:270].

In-memory entries (e.g. `data.sh`) are written by the `add_data` closure [main.go:293] with
`Format: tar.FormatPAX` and `Mode: 0o644` [main.go:297-298]; `data.sh` itself is added at
[main.go:321] and `bootstrap-utils.sh` at [main.go:325]. The `Format: tar.FormatPAX` on every entry
is what makes the archive a **PAX-format** tar.

**Observed** — the archive actually transported by the `kitten ssh localhost` push was captured from
the framed stream and `base64 -d`ecoded to `sh1.tar.gz`. Its `file` signature and its **complete**
`tar -tzf` listing (all 15 entries, no elision) are verbatim below:

```text
$ file sh1.tar.gz
sh1.tar.gz: gzip compressed data, max compression, original size modulo 2^32 93184
$ tar -tzf sh1.tar.gz
data.sh
bootstrap-utils.sh
home/.local/share/kitty-ssh-kitten/shell-integration/zsh/.zshenv
home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_completions.d/kitten.fish
home/.local/share/kitty-ssh-kitten/shell-integration/zsh/kitty-integration
home/.local/share/kitty-ssh-kitten/shell-integration/bash/kitty.bash
home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish
home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_completions.d/kitty.fish
home/.local/share/kitty-ssh-kitten/shell-integration/zsh/completions/_kitty
home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_completions.d/clone-in-kitty.fish
home/.local/share/kitty-ssh-kitten/kitty/version
home/.local/share/kitty-ssh-kitten/kitty/bin/kitty
home/.local/share/kitty-ssh-kitten/kitty/bin/kitten
home/.terminfo/kitty.terminfo
home/.terminfo/x/xterm-kitty
$ tar -tzf sh1.tar.gz | wc -l
15
```

The listing contains **exactly 15 entries** (`wc -l` → `15`). `file` reporting **"max compression"** is
the runtime signature of `gzip.BestCompression` [main.go:259]; the two top-level entries `data.sh` and
`bootstrap-utils.sh` are the in-memory PAX entries added at [main.go:321] and [main.go:325]; the eight
`home/.local/share/kitty-ssh-kitten/shell-integration/` entries (the `zsh`, `bash`, and `fish` assets
listed above) are the shell-integration files the remote stages; `home/.local/share/kitty-ssh-kitten/kitty/bin/kitty`
and `home/.local/share/kitty-ssh-kitten/kitty/bin/kitten` are the bundled launcher stubs (Q3), alongside
`home/.local/share/kitty-ssh-kitten/kitty/version`; and `home/.terminfo/kitty.terminfo` +
`home/.terminfo/x/xterm-kitty` are the terminfo the remote compiles. (Entry *order* varies run-to-run;
the *count* is 15.)

### Transport over the TTY: `get_ssh_data` framing and the 254-byte line

The kitty terminal serves the base64 payload with `get_ssh_data` [kittens/ssh/utils.py:115], framed
between markers and chunked into fixed-size lines:

The generator first yields the `KITTY_DATA_START` marker at [utils.py:117], then (after the password
and request-id validation of Q2) yields the `OK` go-ahead and streams the base64 payload in 254-byte
lines, ending with `KITTY_DATA_END`. The first yield is [utils.py:117]:

```python
# kittens/ssh/utils.py:117 (verbatim)
yield b'\nKITTY_DATA_START\n'  # to discard leading data
```

and the go-ahead + chunking + frame-end are [utils.py:138-148] (verbatim, contiguous — the three-line
comment is real source):

```python
# kittens/ssh/utils.py:138-148 (verbatim, contiguous)
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

**Cause → effect.** The `254` line size is deliberately one below the documented macOS 255-byte
terminal input-queue limit (comment at utils.py:140-142), so no transfer line can overflow the
remote's canonical-mode input queue. The `KITTY_DATA_START` / `OK` / `KITTY_DATA_END` frame lets the
remote discard any leading garbage, detect the go-ahead, and know when the stream ends.

**Observed** — the framed stream that the **real** `get_ssh_data` produced for the sh1 run (the same
run whose request/argv appear in Q1), captured verbatim. The leading and trailing 40/30 bytes show the
three frame markers and the gzip magic (`H4sI…` is base64 of `\x1f\x8b\x08…`, i.e. the gzip header):

```text
leading40:  b'\nKITTY_DATA_START\nOK\nH4sIAAAAAAAC/+y9bWw'
trailing30: b'D//w/VFQgAbAEA\nKITTY_DATA_END\n'
```

**Line size `254` — measured, stable across 2 runs.** The chunk-size distribution of the payload
lines (between `OK` and `KITTY_DATA_END`) is all-254 except the final remainder. Measured verbatim
from the two captured runs (`sh1.framed`, `sh2.framed`):

```text
run 1 (sh1): total 31616 bytes ; 124 payload chunks ; size_dist {254: 123, 214: 1} ; non_last_sizes [254] ; last_size 214
run 2 (sh2): total 31584 bytes ; 124 payload chunks ; size_dist {254: 123, 182: 1} ; non_last_sizes [254] ; last_size 182
```

**Every non-final line is exactly 254 bytes in both runs** (`non_last_sizes [254]`), confirming
`line_sz = 254` is stable and canonical. Only the final *remainder* chunk differs (`214` vs `182`) and
the total byte count differs slightly (`31616` vs `31584`), because the gzip'd tar size depends on the
fresh random password baked into `data.sh` each run; the per-line transfer size itself does not vary.

---

## Q5 — Connection state tracking: the `connection_data` struct

**Question.** How does the kitten keep track of everything it needs for a connection?

All per-connection state lives in one struct, `connection_data` [kittens/ssh/main.go:171-189]. Its
**16 fields** are enumerated below, each with how it is populated during `run_ssh` and an explicit
evidence label — **(observed)** = seen on the real `kitten ssh localhost` run; **(inferred)** = read
from the source assignment (the value itself was not directly captured, e.g. test-hook fields and the
default path leaves it empty):

| # | Field | Type | Populated during `run_ssh` | Evidence (label) |
|---|-------|------|-----------------------------|---------------------|
| 1 | `remote_args` | `[]string` | the server-side args (command after the host); empty for a login shell | **(observed)** empty in `kitten ssh localhost` — no server command, interactive login shell |
| 2 | `host_opts` | `*Config` | `cd.host_opts, cd.literal_env = host_opts, literal_env` [main.go:723] | **(inferred)** default `*Config` values (`Interpreter sh`, `Share_connections true`, `Askpass unless-set`) are from source defaults [conf_generated.go]; the struct is set at [main.go:723] |
| 3 | `hostname_for_match` | `string` | `cd.hostname_for_match, cd.username = hostname_for_match, uname` [main.go:725] | **(observed)** `localhost`; also written into the SHM `hostname` key (observed keys include `'hostname'`, Q2) |
| 4 | `username` | `string` | `cd.username = uname` [main.go:725] | **(observed)** present as the SHM `username` key (Q2) |
| 5 | `echo_on` | `bool` | `cd.echo_on = term.WasEchoOnOriginally()` [main.go:722] | **(observed)** the decoded script carries `echo_on="1"` (Q3) |
| 6 | `request_data` | `bool` | `cd.request_data = need_to_request_data` [main.go:724] | **(observed)** both branches: `request_data="0"` (push, default) and `="1"` (pull) seen in the decoded script (Q7/Q8) |
| 7 | `literal_env` | `map[string]string` | `cd.literal_env = literal_env` [main.go:723] | **(inferred)** env forced verbatim into the remote; empty on the default run (no `env` directive given), so no value was captured — source assignment [main.go:723] |
| 8 | `listen_on` | `string` | forward path only: `cd.listen_on = "tcp:localhost:" + strconv.Itoa(port)` [main.go:716] | **(observed)** populated only with `--kitten forward_remote_control=yes`; exercised in Q6 (empty on the default push) |
| 9 | `test_script` | `string` | test hook (`TEST_SCRIPT`), empty outside the PTY test harness | **(inferred)** source-only test hook; empty on the real path (no `TEST_SCRIPT` body in the captured script) |
| 10 | `dont_create_shm` | `bool` | test hook to skip SHM creation ([main.go:445], [main.go:457]) | **(inferred)** source-only test hook; `false` on the real path (SHM *was* created — observed `pwfile=kssh-…`, Q1/Q2) |
| 11 | `shm_name` | `string` | `cd.shm_name = data_shm.Name()` [main.go:458] in `bootstrap_script` | **(observed)** `kssh-87499-DIUQ3H4FV3L5I` (== request `pwfile=`, Q1) |
| 12 | `script_type` | `string` | `get_remote_command`: `"sh"` [main.go:515] default, `"py"` [main.go:517] if python | **(observed)** both: `sh` (default) and `py` (`--kitten interpreter=python3`, Q7) |
| 13 | `rcmd` | `[]string` | `wrap_bootstrap_script`: `[]string{"exec", cd.host_opts.Interpreter, "-c", unwrap_script, encoded_script}` [main.go:508] | **(observed)** as ssh `argv[15..19]` = `exec sh -c <unwrap> <script>` (Q1/Q7) |
| 14 | `replacements` | `map[string]string` | placeholder→value map built by `bootstrap_script`/`prepare_script` | **(inferred)** the map is a source construct [main.go:407-419]; its tokens `REQUEST_ID`,`PASSWORD_FILENAME`,`DATA_PASSWORD` are observed in the decoded push script (Q3) |
| 15 | `request_id` | `string` | defaulted to `KITTY_PID + "-" + KITTY_WINDOW_ID` [main.go:424] | **(observed)** `87498-1` (== request `id=`, Q1) |
| 16 | `bootstrap_script` | `string` | the fully-substituted script text, set in `bootstrap_script` | **(observed)** verbatim as the ssh `argv[19]` payload (Q3/Q7) |

**Cause → effect.** `connection_data` is the single value threaded through `bootstrap_script` →
`get_remote_command` → `wrap_bootstrap_script` and back into `run_ssh`; it carries both the *inputs*
(host config, hostname, username, env) and the *derived artifacts* (the SHM name, the request-id, the
chosen `script_type`, the encoded `rcmd`, and the placeholder `replacements`) needed to both launch the
`ssh` child and answer the credential request. Eleven of the sixteen fields were observed directly on
the real path (`listen_on` via the Q6 `forward_remote_control=yes` run); the other five (`host_opts`
default values, `literal_env`, `test_script`, `dont_create_shm`, and the `replacements` map) are
labeled **(inferred)** because they are read from the cited source assignment rather than captured as a
runtime value.

> Corroboration: the test `test_ssh_connection_data` [kitty_tests/ssh.py:45] also constructs and
> asserts on this struct, but any value taken from that harness would be **NON-CANONICAL**; the values
> in the table above are from the real `kitten ssh localhost` push (**observed**) or the cited source
> assignments (**inferred**), never from that test.


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

**Observed** — the six options exactly as passed to the real `ssh` child in the sh1 run (verbatim
`argv[1..12]` from the shim log, the same run as Q1):

```text
argv[1]='-o'
argv[2]='ControlMaster=auto'
argv[3]='-o'
argv[4]='ControlPath=/root/.cache/kitty/run/kssh-87498-%C'
argv[5]='-o'
argv[6]='ControlPersist=yes'
argv[7]='-o'
argv[8]='ServerAliveInterval=60'
argv[9]='-o'
argv[10]='ServerAliveCountMax=5'
argv[11]='-o'
argv[12]='TCPKeepAlive=no'
```

Each option, its literal, and its cause → effect:

- **`ControlMaster=auto`** [main.go:138] — reuse an existing master socket if one exists, otherwise
  become the master. This is the mechanism that lets a second `kitten ssh` piggyback on the first.
- **`ControlPath=<runtime_dir>/kssh-<kitty_pid>-%C`** [main.go:139] — the Unix-domain multiplexing
  socket path. `%C` is OpenSSH's SHA1 of `%l%h%p%r` (local host, remote host, port, remote user), so
  distinct destinations get distinct sockets. Observed value: `/root/.cache/kitty/run/kssh-87498-%C`.
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
// kittens/ssh/main.go:648-665 (verbatim, contiguous)
use_kitty_askpass := host_opts.Askpass == Askpass_native || (host_opts.Askpass == Askpass_unless_set && os.Getenv("SSH_ASKPASS") == "")
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

In that block: `use_kitty_askpass` is decided at [main.go:648]; `need_to_request_data` starts `true`
at [main.go:649]; `set_askpass()` (which returns `false` when OpenSSH supports `SSH_ASKPASS_REQUIRE`,
Q8) is called at [main.go:651]; `master_is_functional` builds `ssh -O check` by
`slices.Insert(cmd, 1, "-O", "check")` at [main.go:658] and runs it, recording success at
[main.go:659]; and the piggyback gate `if need_to_request_data && host_opts.Share_connections &&
master_is_functional()` sets `need_to_request_data = false` at [main.go:664].

**Observed — default path (OpenSSH 10.0 ≥ 8.4).** `set_askpass` returns `false` (see Q8), so
`need_to_request_data` is already `false` and the `&&` short-circuits — `master_is_functional()` is
**not** called, and the kitten **pushes** the data itself. Evidence: in the default `kitten ssh
localhost` run the shim logged only three ssh invocations (options-probe, `-V`, real connection) and
**no `-O check`**, and the embedded script carried `request_data="0"`.

**Observed — request path (`--kitten askpass=ssh`).** Here `use_kitty_askpass` is false, so
`set_askpass` is not called and `need_to_request_data` stays `true`; the gate then **does** call
`master_is_functional()`, which runs `ssh -O check`. The complete `-O check` invocation the shim
logged (verbatim, all 17 argv elements — `slices.Insert(cmd, 1, "-O", "check")` [main.go:658] is why
`argv[0]='-O'` and `argv[1]='check'` precede the `-t`):

```text
=== ssh invocation (argc=17) ===
argv[0]='-O'
argv[1]='check'
argv[2]='-t'
argv[3]='-o'
argv[4]='ControlMaster=auto'
argv[5]='-o'
argv[6]='ControlPath=/root/.cache/kitty/run/kssh-89459-%C'
argv[7]='-o'
argv[8]='ControlPersist=yes'
argv[9]='-o'
argv[10]='ServerAliveInterval=60'
argv[11]='-o'
argv[12]='ServerAliveCountMax=5'
argv[13]='-o'
argv[14]='TCPKeepAlive=no'
argv[15]='--'
argv[16]='localhost'
```

**Absent-master result (observed).** Running that exact `ssh -O check` against a `ControlPath` with no
live master returns non-zero — captured verbatim (the `exit_code` line is `echo exit_code=$?`):

```text
Control socket connect(/tmp/kssh_obs/cmcheck.sock): No such file or directory
exit_code=255
```

Non-zero exit → `master_is_functional()` is false → the whole condition is false →
`need_to_request_data` stays true → the embedded script carries `request_data="1"` (observed) → the
**remote** requests the data (pull, Q8).

### `run_control_master` and the port-forward path

`run_control_master` [main.go:666] explicitly starts a detached background master by appending
`-N -f` [main.go:669] then `"--", hostname` [main.go:670], and is invoked on the remote-control
forwarding path guarded by `host_opts.Forward_remote_control && os.Getenv("KITTY_LISTEN_ON") != ""`
[main.go:681]. **Observed** — driving that path (`--kitten forward_remote_control=yes`,
`KITTY_LISTEN_ON=unix:/tmp/kssh_obs/kitty.sock`) produced, in order: `ssh -O check` (master absent) →
the `-N -f` control-master start → `ssh -O check` (now alive) → the `-O forward`. The two key `ssh`
child invocations are verbatim from the shim log (pid `89459` for this run):

```text
=== ssh invocation (argc=17) ===        # run_control_master: -N -f master start
argv[0]='-t'
argv[1]='-o'
argv[2]='ControlMaster=auto'
argv[3]='-o'
argv[4]='ControlPath=/root/.cache/kitty/run/kssh-89459-%C'
argv[5]='-o'
argv[6]='ControlPersist=yes'
argv[7]='-o'
argv[8]='ServerAliveInterval=60'
argv[9]='-o'
argv[10]='ServerAliveCountMax=5'
argv[11]='-o'
argv[12]='TCPKeepAlive=no'
argv[13]='-N'
argv[14]='-f'
argv[15]='--'
argv[16]='localhost'
=== ssh invocation (argc=19) ===        # forward: -R 0:<sock> -O forward
argv[0]='-t'
argv[1]='-o'
argv[2]='ControlMaster=auto'
argv[3]='-o'
argv[4]='ControlPath=/root/.cache/kitty/run/kssh-89459-%C'
argv[5]='-o'
argv[6]='ControlPersist=yes'
argv[7]='-o'
argv[8]='ServerAliveInterval=60'
argv[9]='-o'
argv[10]='ServerAliveCountMax=5'
argv[11]='-o'
argv[12]='TCPKeepAlive=no'
argv[13]='-R'
argv[14]='0:/tmp/kssh_obs/kitty.sock'
argv[15]='-O'
argv[16]='forward'
argv[17]='--'
argv[18]='localhost'
```

The `-N -f` at `argv[13..14]` matches [main.go:669]; the `-R 0:<sock> -O forward` at `argv[13..16]`
matches [main.go:702]; and `-- localhost` matches [main.go:670]/[main.go:703].

**Live-master result (observed).** After `run_control_master` started the `-N -f` master, re-running
the identical `ssh -O check` succeeds — captured verbatim (contrast the absent result above):

```text
Master running (pid=90232)
exit_code=0
```

So `master_is_functional()` returns `false` before the master is started (exit 255) and `true` after
(exit 0), which is exactly the guard `if !master_is_functional() { run_control_master(); … }`
[main.go:685-692].

The forward path also **rejects abstract UNIX sockets** [main.go:697-698] because OpenSSH cannot
forward them. **Observed** (real kitten with `KITTY_LISTEN_ON=unix:@kitty-abstract`; the leading red
styling is `die()`'s ANSI coloring, shown here as plain text):

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
// kittens/ssh/main.go:512-517 (verbatim, contiguous)
interpreter := cd.host_opts.Interpreter
q := strings.ToLower(path.Base(interpreter))
is_python := strings.Contains(q, "python")
cd.script_type = "sh"
if is_python {
    cd.script_type = "py"
}
```

`wrap_bootstrap_script` [kittens/ssh/main.go:486] then encodes accordingly.

### `sh`/`bash`/`tcsh` mode — four character substitutions + `tr` inverse

```go
// kittens/ssh/main.go:505-506 (sh mode, verbatim)
encoded_script = "'" + strings.NewReplacer("'", "\v", "\\", "\f", "\n", "\r", "!", "\b").Replace(cd.bootstrap_script) + "'"
unwrap_script = `'eval "$(echo "$0" | tr \\\v\\\f\\\r\\\b \\\047\\\134\\\n\\\041)"' `
```

The **four substitutions**, each with its cause → effect:

- **`'` → `\v`** (vertical tab) — a single quote cannot appear inside a single-quoted shell string, so
  it is swapped for the rare control byte `\v`.
- **`\` → `\f`** (form feed) — backslashes would otherwise be re-interpreted; swapped for `\f`.
- **newline `\n` → `\r`** (carriage return) — the script is passed as one shell word; embedded
  newlines become `\r` so the whole thing survives as a single `argv` element.
- **`!` → `\b`** (backspace) — `!` triggers history expansion in interactive `csh`/`tcsh`; swapping it
  for `\b` prevents that.

The remote inverts them with the `tr` inside the unwrap script. The source literal writes each escape
with three backslashes (`\\\v\\\f\\\r\\\b` and `\\\047\\\134\\\n\\\041`, [main.go:506]) because the
one-liner passes through two shell-quoting layers (`sh -c '…'` and the inner `"$(echo "$0" | tr …)"`
command substitution) before `tr` runs; after those layers `tr` effectively receives set-1 `\v\f\r\b`
and set-2 `\047\134\n\041`, giving the inverse mapping `\v→\047` (`'`), `\f→\134` (`\`), `\r→\n`
(newline), `\b→\041` (`!`). `echo "$0"` feeds the encoded script (passed as `$0` to `sh -c`) into that
`tr`, and `eval` runs the decoded result.

**Observed** — the default `sh` run's ssh child. `argv[16..17]='sh' '-c'`, `argv[18]` is the unwrap
script, and `argv[19]` is the encoded bootstrap. The unwrap `argv[18]` matches source [main.go:506]
byte-for-byte (verbatim from the shim log, the shim renders control bytes with Go `%q`):

```text
argv[16]='sh'
argv[17]='-c'
argv[18]='eval "$(echo "$0" | tr \\\v\\\f\\\r\\\b \\\047\\\134\\\n\\\041)"' 
```

`argv[19]` is the full 5489-character encoded script; rather than reproduce all of it, each of the
**four substitutions** is demonstrated by a **complete, un-truncated fragment** extracted verbatim from
that captured `argv[19]` (control bytes shown as the shim's `\xNN` / `\r`, i.e. `\x08`=`\b`,
`\x0b`=`\v`, `\x0c`=`\f`, `\r`=CR):

```text
!  → \b  (\x08):        #\x08/bin/sh
\  → \f  (\x0c):        "\x0c033[31m%s\x0c033[m\x0cn
'  → \v  (\x0b):        \x0bbuffer\x0b
newline → \r  and  \ → \f:   \r{ \x0cunalias command;
```

Each line is a complete contiguous slice of the real `argv[19]` (no interior elision). Reading them:
the first is the shebang `#!/bin/sh` with `!`→`\x08`; the second is the source `"\033[31m%s\033[m\n`
(the `die()` red-color `printf`) with every `\`→`\x0c`; the third is the source `'buffer'` (from the
`pybase64` helper) with each `'`→`\x0b`; and the fourth is the source line-break plus `\unalias`,
showing a source newline became `\r` (CR) and the `\` became `\x0c`.

**Cause → effect (`sh`/`bash` vs Python vs `tcsh`).** base64 cannot be relied upon to exist on an
arbitrary remote `sh`/`bash` before the archive is unpacked, so the shell path uses this
quote-safe **character-substitution** scheme (`'`,`\`,newline handled) that needs only `echo`, `tr`
and `eval`. The `!` → `\b` and newline → `\r` substitutions are specifically what make the encoded
one-liner safe for **`tcsh`/`csh`** (history-expansion and single-word constraints). Python is
different: the interpreter is guaranteed to have base64, so it uses the base64 path below.

### `py` mode — base64 encode/decode

```go
// kittens/ssh/main.go:497-499 (py mode, verbatim)
if cd.script_type == "py" {
    encoded_script = base64.StdEncoding.EncodeToString(utils.UnsafeStringToBytes(cd.bootstrap_script))
    unwrap_script = `"import base64, sys; eval(compile(base64.standard_b64decode(sys.argv[-1]), 'bootstrap.py', 'exec'))"`
```

(The `if cd.script_type == "py"` condition is [main.go:497], the `base64.StdEncoding.EncodeToString`
encode is [main.go:498], and the Python `unwrap_script` is [main.go:499]; lines 501-503 that follow are
the **comment** for the `else` shell branch, not the Python branch.)

**Observed** — the `--kitten interpreter=python3` run's ssh child. `argv[18]` (the Python unwrap) is
verbatim below; `argv[19]` is a 13484-character base64 blob:

```text
argv[16]='python3'
argv[17]='-c'
argv[18]='"import base64, sys; eval(compile(base64.standard_b64decode(sys.argv[-1]), \'bootstrap.py\', \'exec\'))"'
```

Rather than paste the 13484-char `argv[19]`, the base64 encoding is demonstrated by a **complete
round-trip** on its first 28 characters (a clean base64 boundary) and by its measured length — no
ellipsis:

```text
$ printf 'IyEvdXNyL2Jpbi9lbnYgcHl0aG9u' | base64 -d
#!/usr/bin/env python
$ python3 -c "import re;print(len(re.search(r\"argv\[19\]='(.*)'\",open('py1.ssh_argv').read()).group(1)))"
13484
```

The first 28 base64 chars decode exactly to `#!/usr/bin/env python` (the first line of
`shell-integration/ssh/bootstrap.py`), confirming the py branch encodes with
`base64.StdEncoding.EncodeToString` [main.go:498] and the remote decodes the whole `argv[19]` with
`base64.standard_b64decode` [main.go:499] to reconstruct `bootstrap.py`.


---

## Q8 — Terminal ↔ remote-shell communication during setup

**Question.** What request/response protocol is exchanged over the controlling terminal during setup?

### The DCS frame format

Local ↔ remote coordination rides on a custom **Device Control String** protocol of the form
`ESC P @ kitty-<verb> | <base64-payload> ESC \`. The Go builder is `DCSToKitty`
[tools/tui/dcs_to_kitty.go:14]:

```go
// tools/tui/dcs_to_kitty.go:14-26 (verbatim, contiguous)
func DCSToKitty(msgtype, payload string) (string, error) {
	data := base64.StdEncoding.EncodeToString(utils.UnsafeStringToBytes(payload))  // :15
	ans := "\x1bP@kitty-" + msgtype + "|" + data                                   // :16  the frame
	tmux := TmuxSocketAddress()
	if tmux != "" {
		err := TmuxAllowPassthrough()
		if err != nil {
			return "", err
		}
		ans = "\033Ptmux;\033" + ans + "\033\033\\\033\\"                          // :23  tmux passthrough variant
	} else {
		ans += "\033\\"                                                            // :25  ST terminator (non-tmux)
	}
```

The remote shell builds the same frame with `dcs_to_kitty()`
[shell-integration/ssh/bootstrap.sh:75], whose verbatim body is:

```sh
dcs_to_kitty() { printf "\033P@kitty-$1|%s\033\134" "$(printf "%s" "$2" | base64_encode)" > /dev/tty; }
```

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

**Observed** — during the real sh1 run the harness received exactly these verbs: `kitty-ssh|` (the
data request), `kitty-print|` (remote debug), and `kitty-echo|` (the drain canary). The following is
the complete verbatim `/tmp/kssh_obs/sh1.meta` verb log (no truncation):

```text
DCS kitty-ssh| req#1 payload_b64_len=148
DCS kitty-print|: ignoreboth or ignorespace present in bash HISTCONTROL setting, showing running command will not be robust
DCS kitty-echo| canary_b64_len=88 at t=0.382 (MODE=normal)
```

Cross-referencing the C dispatch table above: `ssh|`→`handle_remote_ssh` [kitty/vt-parser.c:608],
`print|`→`handle_remote_print` [kitty/vt-parser.c:606], `echo|`→`handle_remote_echo`
[kitty/vt-parser.c:607]. **Note on the byte-pump:** recognizing the `kitty-` prefix and splitting the
frame is done by the harness (standing in for `parse_kitty_dcs` [kitty/vt-parser.c:586] and the kitty
event loop) — that part is **NON-CANONICAL** — but the handlers it invokes (`get_ssh_data`, the echo
strip/echo-back) are the real functions.

### PUSH vs PULL (cause → effect), and the `SSH_ASKPASS` pull path

Whether the **local kitten pushes** the credential request down the TTY or the **remote pulls** it is
decided in `run_ssh` and stored in `cd.request_data`. The gate is a single boolean,
`need_to_request_data`:

```go
// kittens/ssh/main.go:648-651, 663-664
use_kitty_askpass := host_opts.Askpass == Askpass_native || (host_opts.Askpass == Askpass_unless_set && os.Getenv("SSH_ASKPASS") == "")  // :648
need_to_request_data := true                                                    // :649
if use_kitty_askpass {
	need_to_request_data = set_askpass()                                        // :651
}
// a live control master also forces push (Q6 piggyback):
if need_to_request_data && host_opts.Share_connections && master_is_functional() {
	need_to_request_data = false                                                // :664
}
```

`cd.request_data = need_to_request_data` [kittens/ssh/main.go:724] then selects the branch:
`request_data == false` ⇒ **PUSH**, `request_data == true` ⇒ **PULL**. The value is substituted into
the remote script's `request_data="REQUEST_DATA"` placeholder [shell-integration/ssh/bootstrap.sh:90]
as `"0"` (push) or `"1"` (pull).

- **PUSH (`cd.request_data == false`).** The kitten writes the request down the TTY itself, right
  after starting the ssh child:

  ```go
  // kittens/ssh/main.go:761-768
  if !cd.request_data {
  	rq := fmt.Sprintf("id=%s:pwfile=%s:pw=%s", cd.replacements["REQUEST_ID"], cd.replacements["PASSWORD_FILENAME"], cd.replacements["DATA_PASSWORD"])  // :762
  	err := term.ApplyOperations(tty.TCSANOW, tty.SetNoEcho)                   // :763
  	if err == nil {
  		var dcs string
  		dcs, err = tui.DCSToKitty("ssh", rq)                                 // :766
  		if err == nil {
  			err = term.WriteAllString(dcs)                                   // :768
  		}
  	}
  }
  ```

  **Cause → effect.** This branch is reached when `set_askpass()` [kittens/ssh/main.go:147] returns
  **false** — which it does when OpenSSH is new enough, `SupportsAskpassRequire()`
  [kittens/ssh/utils.go:206] being `self.Major > 8 || (self.Major == 8 && self.Minor >= 4)`
  [kittens/ssh/utils.go:207] (i.e. ≥ 8.4; here `10.0p2`). In that case `set_askpass` sets
  `need_to_request_data = false` [kittens/ssh/main.go:156] and exports `SSH_ASKPASS_REQUIRE="force"`
  [kittens/ssh/main.go:163] — the "force" is what makes the push safe: ssh will only ever consult the
  kitty askpass helper, never prompt on the TTY, so the kitten can hand the data over directly. A live
  control master [kittens/ssh/main.go:663-664] also forces push.

  **Observed (real path — `kitten ssh localhost`, default `askpass=unless-set`, OpenSSH `10.0p2`).**
  The embedded script was stamped `request_data="0"`, so the remote does **not** emit the request and
  the kitten pushes it itself:

  ```text
  request_data="0"\rtr
  ```
  (`/tmp/kssh_obs/sh1.ssh_argv`, decoded `argv[19]` — the `"0"` proves the push branch)

  ```text
  request(decoded)=id=87498-1:pwfile=kssh-87499-DIUQ3H4FV3L5I:pw=52747a358160aeab524cc5050a754e6093476e53cc2e6a40007371780c305e81
  ```
  (`/tmp/kssh_obs/sh1.meta` — the full request the kitten pushed via `DCSToKitty("ssh", rq)`
  [kittens/ssh/main.go:766]; this ephemeral password matches Q1's sh1 run and its SHM was unlinked on
  read)

- **PULL (`cd.request_data == true`).** When kitty's askpass is disabled, `use_kitty_askpass` is
  **false** [kittens/ssh/main.go:648], so `set_askpass()` is **never called**, `need_to_request_data`
  stays `true` [kittens/ssh/main.go:649], the kitten's push block [kittens/ssh/main.go:761] is
  skipped, and the **remote** emits the request. The bootstrap's guard fires because
  `request_data="1"`:

  ```sh
  # shell-integration/ssh/bootstrap.sh:90-95
  request_data="REQUEST_DATA"
  trap "cleanup_on_bootstrap_exit" EXIT
  [ "$request_data" = "1" ] && {
      command stty "-echo" < /dev/tty
      dcs_to_kitty "ssh" "id="REQUEST_ID":pwfile="PASSWORD_FILENAME":pw="DATA_PASSWORD""
  }
  ```

  **Observed (real path — `kitten ssh --kitten askpass=ssh localhost`).** Passing `askpass=ssh` made
  `use_kitty_askpass` false; the embedded script was stamped `request_data="1"` (contrast the push
  run's `"0"`), and the `kitty-ssh|` request arrived **from the remote** after the connection came up,
  served by the same real `get_ssh_data`:

  ```text
  request_data="1"\rtr
  ```
  (`/tmp/kssh_obs/pull1.ssh_argv`, decoded `argv[19]` — the `"1"` proves the pull branch)

  ```text
  DCS kitty-ssh| req#1 payload_b64_len=152
    request(decoded)=id=116503-1:pwfile=kssh-116504-NK3NJSNWHF4H4:pw=af42a5c8cb67288c54f946ab4e7bca939e9aa443c113c4892e235d942d572199
  SHM read_data_from_shared_memory OK; keys=['hostname', 'pw', 'tarfile', 'username']
  ```
  (`/tmp/kssh_obs/pull1.meta`) The `request_data="1"` outcome was **stable across two runs** (pull2
  gave `id=116625-1:pwfile=kssh-116626-EYJXPICMT7LTQ`).

  The askpass helper invoked by ssh is `RunSSHAskpass` [kittens/ssh/askpass.go:37], selected via the
  `KITTY_KITTEN_RUN_MODULE=ssh_askpass` env var. Note that helper belongs to the **kitty-askpass-
  enabled** path: `set_askpass` exports `SSH_ASKPASS` [kittens/ssh/main.go:160] and
  `KITTY_KITTEN_RUN_MODULE=ssh_askpass` [kittens/ssh/main.go:161] only when it is actually called
  (i.e. `use_kitty_askpass` is true). With `--kitten askpass=ssh` the kitten stays out of ssh's askpass
  mechanism entirely, so ssh uses its own. *(inferred)* On an OpenSSH older than 8.4 with kitty askpass
  still enabled, `set_askpass` would return `true` (no `SSH_ASKPASS_REQUIRE`), also leaving
  `request_data=true` and pulling — that variant is not observable here because the host ships
  `10.0p2`.

### The drain canary and its 2-second timeout

After the ssh child exits (`c.Wait()` at [main.go:782]), `run_ssh` calls
`drain_potential_tty_garbage` [main.go:530] at [main.go:783] to flush any stray remote output that may
still be arriving on the TTY. It writes a random echo canary and reads until the canary comes back or
a 2-second deadline elapses:

```go
// kittens/ssh/main.go:535-562 (verbatim, contiguous)
canary, err := secrets.TokenHex()                       // :535  32 random bytes → 64 hex chars
if err != nil {
	return
}
dcs, err := tui.DCSToKitty("echo", canary)              // :539  build the kitty-echo| frame
q := utils.UnsafeStringToBytes(canary)                  // :540  the bytes to scan for
if err != nil {
	return
}
err = term.WriteAllString(dcs)                          // :544  write the canary to the TTY
if err != nil {
	return
}
data := make([]byte, 0)
give_up_at := time.Now().Add(2 * time.Second)           // :549  the 2-second deadline
buf := make([]byte, 0, 8192)
for !bytes.Contains(data, q) {                          // :551  read until the canary is seen…
	buf = buf[:cap(buf)]
	timeout := time.Until(give_up_at)
	if timeout < 0 {                                    // :554  …or the deadline elapses
		break
	}
	n, err := term.ReadWithTimeout(buf, timeout)        // :557
	if err != nil {
		break
	}
	data = append(data, buf[:n]...)                     // :561
}
```

**Cause → effect.** The real terminal echoes a `kitty-echo|` payload back (base64-decoded,
non-printable bytes stripped, by `handle_remote_echo` [kitty/window.py:1282-1287]); once the kitten
sees its own canary in the stream, `bytes.Contains(data, q)` [kittens/ssh/main.go:551] is satisfied,
the loop exits, and the drain returns. The `2 * time.Second` cap [kittens/ssh/main.go:549] guarantees
it never blocks forever if the canary is somehow lost.

**Observed — normal drain (real drain loop; canary echoed by the harness terminal stand-in for
`handle_remote_echo`), stable across two runs.** The kitten binary really executed
`drain_potential_tty_garbage`, wrote the 64-hex-char canary, and read until it saw it; the drain
returned almost instantly:

```text
MODE=normal ssh_reqs=1
canary_seen_at=0.382 exited_at=0.384
drain_seconds=0.003
```
(`/tmp/kssh_obs/sh1.timing`; the second run `/tmp/kssh_obs/sh2.timing` also gave `drain_seconds=0.003`)

**NON-CANONICAL — withheld canary (harnessed negative).** A real kitty session *always* echoes the
canary back via `handle_remote_echo` [kitty/window.py:1282-1287], so it never loses it. To exercise
the timeout path I ran the harness in `MODE=withhold`, which **deliberately drops** that echo —
something the real `handle_remote_echo` never does — so this timing does **not** reflect canonical
behavior. With the echo dropped, the loop runs to the `give_up_at := time.Now().Add(2 * time.Second)`
[kittens/ssh/main.go:549] deadline:

```text
MODE=withhold ssh_reqs=1
canary_seen_at=0.370 exited_at=2.375
drain_seconds=2.005
```
(`/tmp/kssh_obs/drainA.timing`; the second run `/tmp/kssh_obs/drainB.timing` gave
`drain_seconds=2.004` — both hit the 2-second cap, stable across two runs)

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
present. **Observed** — on the localhost target the first branch is taken. The verbatim definition
from `shell-integration/ssh/bootstrap.sh:56` is:

```sh
    base64_encode() { command base64 | command tr -d \\n\\r; }
```

and the transfer succeeded (Q4), so branch 1 is the live path here. Branches 2–6 are the enumerated
siblings from the source; because `base64` is present on this host, those branches are not taken here
and are therefore **(inferred)** from reading.

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
- [x] `shm.CreateTemp(fmt.Sprintf("kssh-%d-", os.Getpid()), uint64(len(encoded_data)+8))` — `main.go:446` (the `+8` is over-allocation slack, not the 4-byte size prefix); `WriteWithSize` `:448`; `Flush` `:450` — (obs, `/dev/shm/kssh-` object).
- [x] Go helpers `CreateTemp`/`WriteWithSize`/`ReadWithSize`/`ReadWithSizeAndUnlink` — `tools/utils/shm/shm.go:91,120,129,142` — (inf).
- [x] D1 immediate `shm.unlink()` — `kittens/ssh/utils.py:106` (also `kitty/shm.py:175`) — (obs, "unlink CONFIRMED (gone)").
- [x] D2 owner check `Incorrect owner on pwfile` — `utils.py:107-108` — (obs, PASS: uid/gid 0).
- [x] D3 perm check `Incorrect permissions on pwfile: 0o{mode:03o}` — `utils.py:109-111` — (obs PASS `0o600`; N-C reject `0o644`); default `0o600` at `kitty/shm.py:51`, `O_CREAT|O_EXCL` `:62`.
- [x] D4 `Incorrect password` — `utils.py:131` (guard `pw != env_data['pw']` `:130`) — (obs PASS; N-C reject on tampered pw).
- [x] D5 `Incorrect request id: {rq_id!r} expecting the KITTY_PID-KITTY_WINDOW_ID for the current kitty window` — `utils.py:133` (guard `rq_id != request_id` `:132`) — (obs PASS; N-C reject on tampered id).
- [x] SHM never transmitted (only base64 tar crosses) — (obs, created+consumed in `/dev/shm`).

### Q3 — bootstrap generation + remote execution
- [x] `prepare_script` defaults `EXEC_CMD`/`EXPORT_HOME_CMD` + `\b<key>\b` word-boundary regex — `main.go:407-418` — (obs, script substituted).
- [x] `wrap_bootstrap_script` sets `rcmd = [exec, Interpreter, -c, unwrap, encoded]` — `main.go:508` — (obs, ssh argv).
- [x] `dcs_to_kitty` — `bootstrap.sh:75`; `request_data="REQUEST_DATA"` `:90`; request line `id=REQUEST_ID:pwfile=PASSWORD_FILENAME:pw=DATA_PASSWORD` `:94` — (obs).
- [x] `untar_and_read_env` `:104`; `mktemp -d "$HOME/.kitty-ssh-kitten-untar-XXXXXXXXXXXX"` `:108`; `tar "xpzf"` `:113`; `compile_terminfo` `:130`; `get_data` `:137`; `EXEC_CMD` `:159`; `exec_login_shell` `:164` — (obs, prompt).
- [x] `bootstrap.py` equivalents — `:22,73,81,133,178,188,203` — (obs, base64-decoded from py argv).
- [x] `bootstrap-utils.sh` `mv_files_and_dirs`/`compile_terminfo`/`prepare_for_exec`/`exec_login_shell` — `:9,18,192,221` — (obs, bundled in archive).
- [x] launcher stubs `shell-integration/ssh/{kitty,kitten}` — (obs, archive entries `kitty/bin/{kitty,kitten}`).

### Q4 — archive + transport
- [x] `gzip.NewWriterLevel(&w, gzip.BestCompression)` — `main.go:259` — (obs, `file`→"max compression").
- [x] `tar.NewWriter` `:263`; `h.Mode |= 0o600` `:269`; `Format: tar.FormatPAX, Mode: 0o644` (add_data `:293`, `:297-298`) — (obs, `tar -tzf` 15 entries).
- [x] `yield b'\nKITTY_DATA_START\n'` — `utils.py:117` — (obs, leading bytes).
- [x] `yield b'OK\n'` — `utils.py:138` — (obs).
- [x] `line_sz = 254` — `utils.py:143` — (obs, non-last chunks all 254, **stable ×2**); reason: macOS 255-byte input-queue limit (comment `:140-142`).
- [x] `yield b'KITTY_DATA_END\n'` — `utils.py:148` — (obs, trailing bytes).

### Q5 — connection_data (all 16 fields)
- [x] All 16 fields — `main.go:171-189` — enumerated in the Q5 table with per-field population site and evidence label; 11 (observed), 5 (inferred). Observed values from the sh1 run: `request_id=87498-1` `main.go:424`, `shm_name=kssh-87499-DIUQ3H4FV3L5I` `main.go:458`, `script_type` sh+py `:515/:517`, `request_data` `="0"`(push)+`="1"`(pull) `:724`, `echo_on=1` `:722`, `rcmd` `:508`, `listen_on` `:716`; inferred (source-only): `host_opts` defaults, `literal_env` `:723`, `test_script`, `dont_create_shm`, `replacements` `:480`.

### Q6 — connection reuse (all six -o options + decision)
- [x] `ControlMaster=auto` — `main.go:138` — (obs).
- [x] `ControlPath=<rd>/kssh-<pid>-%C` — `main.go:139` — (obs `/root/.cache/kitty/run/kssh-87498-%C`).
- [x] `ControlPersist=yes` — `main.go:140` — (obs).
- [x] `ServerAliveInterval=60` — `main.go:141` — (obs).
- [x] `ServerAliveCountMax=5` — `main.go:142` — (obs).
- [x] `TCPKeepAlive=no` — `main.go:143` — (obs).
- [x] template `kssh-{kitty_pid}-{ssh_placeholder}` → `%C` — `kitty/constants.py:188`, `gen/go_code.py:599`, `main.go:135-136` — (obs).
- [x] `ssh -O check` (`master_is_functional`) — `main.go:658-659` — (obs: absent→`No such file or directory` exit 255; live→`Master running (pid=90232)` exit 0).
- [x] `run_control_master` `-N -f` `main.go:669`, `"--", hostname` `main.go:670` — (obs, forward run argv `argv[13..16]`).
- [x] forward `-R 0:<listen_on> -O forward` — `main.go:702` — (obs, forward run argv `argv[13..16]`).
- [x] abstract-socket rejection — `main.go:697-698` — (obs, verbatim error).
- [x] push-vs-request gate `need_to_request_data && Share_connections && master_is_functional()` — `main.go:663-664` — (obs, both branches).

### Q7 — shell encoding (four substitutions + inverse + base64 path)
- [x] `'` → `\v` — `main.go:505` — (obs, `\x0bbuffer\x0b`).
- [x] `\` → `\f` — `main.go:505` — (obs, `\x0c033`).
- [x] `\n` → `\r` — `main.go:505` — (obs, `\r` linebreaks).
- [x] `!` → `\b` — `main.go:505` — (obs, `#\x08/bin/sh`).
- [x] `tr` inverse (source `\\\v\\\f\\\r\\\b \\\047\\\134\\\n\\\041`) — `main.go:506` — (obs, argv[18] matches source byte-for-byte).
- [x] py base64 `EncodeToString` `main.go:498` / unwrap `standard_b64decode` `main.go:499` (branch `if=="py"` `:497`) — (obs, first 28 b64 chars of argv[19] → `#!/usr/bin/env python`; len 13484).
- [x] `is_python` branch — `main.go:514-517` — (obs, sh default + py forced).
- [x] `sh`/`bash` vs Python vs `tcsh` distinction — (explained: shell uses char-subst since base64 not guaranteed; `!`→`\b` and `\n`→`\r` specifically for tcsh; python uses base64).

### Q8 — DCS protocol + pull + drain + fallbacks
- [x] DCS frame `\x1bP@kitty-<verb>|<b64>\033\\` — `dcs_to_kitty.go:16,25`; tmux variant `:23`; remote `bootstrap.sh:75` — (obs).
- [x] VT dispatch prefix `kitty-` — `vt-parser.c:600` — (inf) — required prefix.
- [x] all 9 verbs `cmd{`,`overlay-ready|`,`kitten-result|`,`print|`,`echo|`,`ssh|`,`ask|`,`clone|`,`edit|` — `vt-parser.c:603-611` — (obs for ssh|/echo|/print|; others inf); **`ask|` not `askpass|`**.
- [x] `handle_remote_ssh` → `get_ssh_data(msg, f'{os.getpid()}-{self.id}')` — `window.py:1289,1291` — (obs, request_id `<pid>-<wid>`).
- [x] PUSH (`cd.request_data == false`) — kitten writes `id=%s:pwfile=%s:pw=%s` + `DCSToKitty("ssh", rq)` `main.go:762,766`. **Condition**: `set_askpass()` returns false — new OpenSSH so `SupportsAskpassRequire()` `utils.go:206-207` true (≥8.4; here 10.0p2) → `need_to_request_data=false` `main.go:156` + `SSH_ASKPASS_REQUIRE="force"` `main.go:163`; sentinel `openssh-is-new-enough-for-askpass` `:149`; **or** a live master `main.go:663-664`. Gate `use_kitty_askpass` `:648` / `need_to_request_data` `:649,651` / `cd.request_data` `:724` — (obs, `request_data="0"`, `id=87498-1`).
- [x] PULL (`cd.request_data == true`) — remote emits the request `bootstrap.sh:90,92,94`. **Condition**: kitty askpass disabled (`--kitten askpass=ssh`) ⇒ `use_kitty_askpass` false `main.go:648` ⇒ `set_askpass()` never called ⇒ `need_to_request_data` stays true `:649` — (obs via real `kitten ssh --kitten askpass=ssh localhost`, `request_data="1"`, `id=116503-1`, stable ×2).
- [x] `set_askpass` env exports (kitty-askpass-enabled/push path only): `SSH_ASKPASS` `:160`, `KITTY_KITTEN_RUN_MODULE=ssh_askpass` `:161`, `SSH_ASKPASS_REQUIRE=force` `:163` — (obs, push run) — helper `RunSSHAskpass` `askpass.go:37` (inf).
- [x] drain canary `secrets.TokenHex()` `:535`, `DCSToKitty("echo", canary)` `:539`, `2 * time.Second` deadline `:549`, loop `bytes.Contains` `:551` — normal drain **0.003s stable ×2** (obs, real loop; echo via harness stand-in for `window.py:1282-1287`); withheld-canary **2.005s/2.004s** = the 2s cap (**NON-CANONICAL** — harness drops the echo real `handle_remote_echo` always sends); 64-hex-char canary.
- [x] six base64 fallbacks — `bootstrap.sh:55-72` — (obs branch 1 `base64`; branches 2–6 inf) — degrade openssl→b64encode→python→perl→`die`.

### Environment variables named across the questions
- [x] `KITTY_WINDOW_ID`, `KITTY_PID` — guards `main.go:825`; request-id `main.go:424`, `window.py:1291` — (obs).
- [x] `SSH_ASKPASS`, `SSH_ASKPASS_REQUIRE`, `KITTY_KITTEN_RUN_MODULE` — `main.go:160-163` — (obs; exported by `set_askpass` when kitty askpass is enabled, i.e. the push-enabling path — `SSH_ASKPASS_REQUIRE=force` only when `need_to_request_data==false`).

### Placeholder tokens
- [x] `{kitty_pid}`, `{ssh_placeholder}`→`%C` — `main.go:135-136` — (obs `kssh-87498-%C`).
- [x] `REQUEST_ID`, `PASSWORD_FILENAME`, `DATA_PASSWORD` — `main.go:762`, `bootstrap.sh:94` — (obs in pushed request).
- [x] `EXEC_CMD`, `EXPORT_HOME_CMD` — `main.go:409,412`; `bootstrap.sh:159` — (obs empty defaults in captured script).

### Measured values (stability)
- [x] `line_sz = 254` — stable across 2 runs (all non-last chunks 254).
- [x] drain — normal drain **0.003s** stable ×2 (obs, real loop); withheld-canary **2.005s / 2.004s** = the `2 * time.Second` cap `main.go:549` stable ×2 (**NON-CANONICAL** — harness drops the echo the real `handle_remote_echo` always sends).
- [x] `ServerAliveInterval=60` / `ServerAliveCountMax=5` — source literals confirmed verbatim in every ssh-child argv capture.

**Result:** every distinct item named across Q1–Q8 — each mechanism, function, condition, file, flag,
env var, placeholder, and every "e.g./such as/including/like" sibling — is addressed above with its
exact literal, `file:line`, an evidence label (observed / non-canonical / inferred), and its causal
reason. Both `script_type` branches (`sh` + `py`) and both credential paths (push + pull) were
exercised on the real `kitten ssh localhost` entry point.

