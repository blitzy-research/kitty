# How the kitty `ssh` kitten works — an evidence-backed walkthrough

> Branch: `kitty_815df1e210e0` · Repo: `kovidgoyal/kitty` · HEAD: `815df1e21`
> Product built & observed at runtime: **kitty / kitten 0.35.2**
> Every **structural** claim below carries an exact `file:line` reference; every **behavioral** claim carries the **actual command** that produced it and the **unedited observed output**. The only claims that could not be observed at runtime are labelled **_inferred from source_** (there are exactly two — see the appendix).

---

## Executive summary

The kitty `ssh` kitten is a thin **Go wrapper around the system `ssh` binary**. It does **not** re-implement the SSH transport; instead it shells out to `ssh` (`SSHExe = sync.OnceValue(func() string { return utils.FindExe("ssh") })` [kittens/ssh/utils.go:L22]) and adds three things on top:

1. **Connection sharing** — it injects six OpenSSH `-o` options that enable `ControlMaster` multiplexing so repeated connections to the same host reuse one authenticated TCP/SSH channel [kittens/ssh/main.go:L121-L145].
2. **A remotely-executed bootstrap** — it generates a small POSIX-`sh` (or Python) script, encodes it so it survives an `ssh` command line, and runs it as the remote login command. That bootstrap pulls a gzipped tar archive of kitty's shell-integration files + terminfo onto the remote host and then hands off to the user's real login shell [kittens/ssh/main.go:L422-L518; shell-integration/ssh/bootstrap.sh:L104-L164].
3. **A secure data channel** — the archive plus a data password is written to a **POSIX shared-memory object** (`/dev/shm/kssh-*`, mode `0600`), never onto the command line. The remote bootstrap requests the data over the terminal itself using a **DCS (Device Control String) handshake**; kitty answers by streaming the tar back in ≤254-byte lines after validating six security checks [kitty/shm.py:L62; kittens/ssh/utils.py:L100-L148; shell-integration/ssh/bootstrap.sh:L75-L95].

The whole thing is driven off **two independent decisions**: whether an SSH master connection is already alive (reuse vs. fresh), and whether the data must be requested at all (which can be short-circuited either by connection reuse or by the kitty-askpass path). The rest of this document answers ten specific questions about these mechanisms, each with a runtime demonstration.

---

## §0 Methodology & Build

### 0.1 Toolchain actually observed

All work was performed in the task's Linux container (Ubuntu 25.10, `uname` `Linux 6.6.122+ x86_64`). The exact tool versions present were captured with:

```
$ go version; python3 --version; ssh -V; zsh --version; fish --version; bash --version | head -1; tar --version | head -1; base64 --version | head -1; tr --version | head -1
```

Observed (verbatim):

```
go version go1.23.4 linux/amd64
Python 3.13.7
OpenSSH_10.0p2 Ubuntu-5ubuntu5.4, OpenSSL 3.5.3 16 Sep 2025
zsh 5.9 (x86_64-ubuntu-linux-gnu)
fish, version 4.0.6
GNU bash, version 5.2.37(1)-release
tar (GNU tar) 1.35
base64 (uutils coreutils) 0.2.2
tr (uutils coreutils) 0.2.2
```

Two honest deltas from the anchors in the task brief:

* The brief anticipated **Python 3.12.x** and **OpenSSH 9.6p1**; the container actually runs **Python 3.13.7** and **OpenSSH_10.0p2**. `go.mod` declares `go 1.22`; the installed `go1.23.4` satisfies that.
* `base64`/`tr` are provided by **uutils coreutils 0.2.2**, not GNU coreutils. This matters for Q7 (the `tr`-based decoding), so the byte-level reversal was re-verified against this exact `tr` (it handles the `\v\f\r\b` / octal escape set correctly — see §7).

### 0.2 Build commands (Run-First)

kitty's single build system is `setup.py`. The product was built from source **before any observation**, with the exact canonical commands:

```
$ export PATH=/usr/local/go/bin:$PATH
$ python3 setup.py build --debug --ignore-compiler-warnings --skip-building-kitten   # C-extension + launcher + code-generation
$ python3 setup.py build --debug --ignore-compiler-warnings --skip-code-generation   # the Go kitten
```

Both steps exited `0`. Step 1 produces `kitty/fast_data_types.so`, `kitty/launcher/kitty`, `constants_generated.go`, and the embedded shell-integration blob `tools/tui/shell_integration/data_generated.bin`; step 2 produces `kitty/launcher/kitten`. Runnable binaries confirmed:

```
$ ./kitty/launcher/kitten --version   ->  kitten 0.35.2 created by Kovid Goyal
$ ./kitty/launcher/kitty  --version   ->  kitty 0.35.2 created by Kovid Goyal
```

**About `--ignore-compiler-warnings` (honest note).** The brief states the flag is required because a newer `wayland-protocols` trips `-Werror=switch` in `glfw/wl_window.c`. In *this* container `wayland-protocols` is **not** installed, so `setup.py` prints `Disabling building of wayland backend` and never compiles `glfw/wl_window.c`; the `-Werror=switch` condition therefore does **not** fire here and the flag is effectively a no-op for this build. It is kept because it is the documented canonical invocation and is harmless. This is reported as observed rather than asserting the brief's stated cause.

All build outputs are git-ignored; `git status --porcelain` is empty immediately after building (confirmed with `git check-ignore`). The product tree is never modified.

### 0.3 Canonical entry points used (no mocks of kitten logic)

Every behavioral result below comes from one of four **real** entry points:

| # | Entry point | What it exercises | Key source |
|---|-------------|-------------------|------------|
| 1 | `kitten ssh …` (driven through a real PTY) | the actual user command → `run_ssh` | [kittens/ssh/main.go:L597,L800] |
| 2 | `printf '<conf>' \| kitten __pytest__ ssh '<test-script>'` | the real `get_remote_command` path; prints JSON `{"cmd":…, "shm_name":…}`; reads an ssh-kitten config from **stdin**; fixes `request_id="testing"`, `request_data=true`, `echo_on=true` | [kittens/ssh/main.go:L847-L886] |
| 3 | `kitty +launch test.py --module ssh` | the PTY round-trip suite `kitty_tests/ssh.py` (`check_bootstrap` [L227]) | — |
| 4 | system `ssh` invoked with the kitten's **own** six sharing args | the OpenSSH ControlMaster lifecycle, against a local `sshd` | — |

The **only** shim used is a *record-only* fake `ssh` that appends its `argv` to a log and, for `-O check`, returns an exit code we control (`FAKE_SSH_OCHECK_RC`) so the kitten's *own* decision branch executes. It re-implements **no** kitten logic. Its complete source:

```sh
#!/bin/sh
LOG="${FAKE_SSH_LOG:-/tmp/kssh_obs/fakessh.log}"
{
  echo "=== invocation $(date +%s.%N) argc=$# ==="
  i=0
  for a in "$@"; do echo "argv[$i]=$a"; i=$((i+1)); done
} >> "$LOG"
prev=""
for a in "$@"; do
  if [ "$prev" = "-O" ] && [ "$a" = "check" ]; then
    echo "  -> -O check seen; returning FAKE_SSH_OCHECK_RC=${FAKE_SSH_OCHECK_RC:-0}" >> "$LOG"
    exit "${FAKE_SSH_OCHECK_RC:-0}"
  fi
  prev="$a"
done
exit 0
```

A local `sshd` (used only for Q1's real ControlMaster lifecycle) was started on a high port with a throwaway key:

```
config /tmp/sshd_test_config: Port 2222, ListenAddress 127.0.0.1,
  PermitRootLogin prohibit-password, PubkeyAuthentication yes,
  PasswordAuthentication no, UsePAM no, PidFile /run/sshd_test.pid
$ ssh-keygen -A; mkdir -p /run/sshd; setsid /usr/sbin/sshd -f /tmp/sshd_test_config
$ ssh -p 2222 -o StrictHostKeyChecking=no -i /root/.ssh/id_ed25519 root@localhost 'echo REMOTE_OK; id'
REMOTE_OK
uid=0(root) gid=0(root) groups=0(root)
```

### 0.4 Run-to-run variance (stated up front)

Several values are **freshly random every run** and are shown at two values so the variance is explicit rather than implied:

* the shared-memory object name suffix (`kssh-<pid>-<13-char base32>`), e.g. `kssh-58194-3NZCHVVAXYVHS` vs `kssh-58359-IWBTBR2SRVWPI`;
* the data password `pw`, a fresh `secrets.TokenHex()` 64-hex string every run (shown truncated `first8…last4` throughout — these are ephemeral and already unlinked, never real provider credentials);
* process ids (`KITTY_PID`, master pid) and the OpenSSH `%C` connection hash.

### 0.5 Read-only guarantee

The source tree is treated as read-only. All observation scripts, logs, the local `sshd` + keys, the fake `ssh` shim, `/dev/shm/kssh-*` objects, and every build artifact are removed at the end. The final `git status --porcelain` proving that only this document was added is pasted in §0.6.

### 0.6 Final read-only proof

After all observation and the removal of every temporary artifact (the local `sshd`, its config/pidfile, the fake `ssh` shim, all `/dev/shm/kssh-*` objects, and every git-ignored build output — launchers, `*.so`, `constants_generated.go`, `data_generated.bin`, the `build/` tree, etc.), the repository contains exactly one new file — this document:

```
$ git status --porcelain -uall
?? blitzy/documentation/kitty_815df1e210e0.md

$ git status --porcelain -uall | grep -E '^( M|M | D|D |A |R |C )'    # tracked changes?
(no output — no tracked file was modified, added, or deleted)

$ find blitzy -type f
blitzy/documentation/kitty_815df1e210e0.md
```

No existing source file was modified; no product code was added. The read-only mandate is satisfied.

---

## §1 How does the kitten set up a secure session and share connections?

**Direct answer.** The kitten never re-implements SSH. It resolves the system `ssh` binary once (`SSHExe` [kittens/ssh/utils.go:L22], version probed via `ssh -V` in `GetSSHVersion` [kittens/ssh/utils.go:L211]) and, when connection sharing is enabled (the default), prepends **six** OpenSSH `-o` options that turn on `ControlMaster` multiplexing. The first connection to a host opens a background *master* channel; every later connection to the same host piggybacks on it, so authentication and TCP setup happen only once. The exact six options come from `connection_sharing_args` [kittens/ssh/main.go:L121-L145].

### 1.1 The six options, captured verbatim

Captured from a real `kitten ssh` run (driven through a PTY with the record-only fake `ssh`, `KITTY_PID=55002`). This is the `-O check` probe invocation the kitten made, with its six sharing options — reproduced exactly as the kitten emitted them:

```
$ # fake-ssh argv log from: kitten ssh --kitten askpass=ssh -- host.test echo hello
argv[0]=-O
argv[1]=check
argv[2]=-o
argv[3]=ControlMaster=auto
argv[4]=-o
argv[5]=ControlPath=/root/.cache/kitty/run/kssh-55002-%C
argv[6]=-o
argv[7]=ControlPersist=yes
argv[8]=-o
argv[9]=ServerAliveInterval=60
argv[10]=-o
argv[11]=ServerAliveCountMax=5
argv[12]=-o
argv[13]=TCPKeepAlive=no
argv[14]=--
argv[15]=host.test
```

So the six options, verbatim, are:

| # | Option | Effect | Source |
|---|--------|--------|--------|
| 1 | `-o ControlMaster=auto` | reuse a master if present, else become the master | [main.go:L137] |
| 2 | `-o ControlPath=<runtime_dir>/kssh-<KITTY_PID>-%C` | per-connection master socket path | [main.go:L135-L136,L138] |
| 3 | `-o ControlPersist=yes` | keep the master alive after the first client exits | [main.go:L139] |
| 4 | `-o ServerAliveInterval=60` | keepalive probe every 60 s | [main.go:L140] |
| 5 | `-o ServerAliveCountMax=5` | drop after 5 missed probes | [main.go:L141] |
| 6 | `-o TCPKeepAlive=no` | rely on SSH-level keepalive, not TCP | [main.go:L142] |

### 1.2 The `ControlPath` template and why it is short

The socket name is built from a template that is generated at build time:

* `ssh_control_master_template = 'kssh-{kitty_pid}-{ssh_placeholder}'` [kitty/constants.py:L188]
* emitted into Go by `generate_constants` as `const SSHControlMasterTemplate` [gen/go_code.py:L571,L599], landing in `constants_generated.go`
* at runtime `{kitty_pid}` → the pid [main.go:L135] and `{ssh_placeholder}` → the literal `%C` [main.go:L136]

The prefix is deliberately terse. The comment at [main.go:L123-L127] explains that OpenSSH appends a 40-character connection hash plus a ~27-character temporary suffix to `ControlPath`, and a UNIX socket path is capped near ~104 bytes; keeping `kssh-<pid>-` short leaves room for those. `%C` is OpenSSH's **connection hash** of the tuple (local host, remote host, port, user) — _inferred from source_ as to its exact inputs (they are OpenSSH-internal), but its behavior is confirmed empirically in §1.3.

### 1.3 The real ControlMaster lifecycle (against a local `sshd`)

Driving the system `ssh` with the kitten's exact six options against the local `sshd` (port 2222), with `ControlPath=/root/.cache/kitty/run/kssh-77001-%C`:

**(1) First connection — becomes the master:**

```
$ ssh -o ControlMaster=auto -o ControlPath=$RD/kssh-77001-%C -o ControlPersist=yes \
      -o ServerAliveInterval=60 -o ServerAliveCountMax=5 -o TCPKeepAlive=no \
      -p 2222 -i /root/.ssh/id_ed25519 root@localhost 'echo MASTER_CONNECTION_OK; hostname'
Warning: Permanently added '[localhost]:2222' (ED25519) to the list of known hosts.
MASTER_CONNECTION_OK
reverse-code-generator-1a3e34e0-6h99z
```

**(2) The master socket appeared** — note it is a UNIX socket (`s`), mode `0600`, and its name ends in the 40-hex `%C` hash:

```
$ ls -l /root/.cache/kitty/run/kssh-77001-*
srw------- 1 root root 0 ... kssh-77001-105960d1763861baa6c0ba26d2941ddc34a18f30
```

**(3) `-O check` reports the master alive (exit 0):**

```
$ ssh <same 6 opts> -O check -p 2222 -i /root/.ssh/id_ed25519 root@localhost ; echo exit=$?
Master running (pid=56364)
exit=0
```

**(4) Second connection reuses it — no re-auth, ~7 ms:**

```
$ time ssh <same 6 opts> -p 2222 -i /root/.ssh/id_ed25519 root@localhost 'echo REUSED_CONNECTION_OK'
REUSED_CONNECTION_OK

real	0m0.007s
```

The absence of the `Permanently added …` host-key line and the 7 ms wall time (vs. a fresh TCP+auth handshake) demonstrate the second client rode the existing master.

**(5) `-O exit` tears it down; `ControlPersist=yes` had kept it alive until then:**

```
$ ssh <same 6 opts> -O exit -p 2222 -i /root/.ssh/id_ed25519 root@localhost
Exit request sent.
$ ls /root/.cache/kitty/run/kssh-77001-*   ->  (socket gone; master pid 56364 exited)
```

### 1.4 Relevant configuration defaults

| Option | Default | Source |
|--------|---------|--------|
| `share_connections` | `yes` | [kittens/ssh/main.py:L183] |
| `forward_remote_control` | `no` | [kittens/ssh/main.py:L212] |
| `interpreter` | `sh` | [kittens/ssh/main.py:L87] |
| `askpass` | `unless-set` (choices: `unless-set` / `ssh` / `native`) | [kittens/ssh/main.py:L192] |

`forward_remote_control=yes` is only allowed together with `share_connections=yes`; otherwise the kitten errors with `Cannot use forward_remote_control=yes without share_connections=yes` [kittens/ssh/main.go:L681-L684].

### 1.5 The macOS long-runtime-dir workaround (_inferred from source_)

If the runtime directory path is longer than 35 bytes, `connection_sharing_args` symlinks it to `/tmp/kssh-rdir-<euid>` to stay under the socket-path limit [kittens/ssh/main.go:L128-L134]. On Linux the runtime dir here is `/root/.cache/kitty/run` (21 bytes < 35), so this branch does **not** fire — confirmed, because the observed `ControlPath` used the real runtime dir with no `/tmp` symlink. This branch is therefore labelled **_inferred from source_**.

---

## §2 How does it use shared memory to pass credentials securely?

**Direct answer.** When the kitten must ship data to the remote (the "fresh" path, §6), it serializes a JSON blob — the gzipped tar archive (base64), the **data password**, the hostname and username — into a **POSIX shared-memory object** created with `O_CREAT | O_EXCL` and mode `0600`, named `kssh-<pid>-<random>` (so it appears at `/dev/shm/kssh-*` on Linux). The password and the object name are the *only* things told to the remote, and they travel over the terminal via the DCS handshake (§10) — never on the `ssh` command line. The remote hands the password + object name back; kitty opens the object, validates six things, and streams the tar out. The object is unlinked on first read (single-use).

### 2.1 A live `/dev/shm/kssh-*` object

`kitten __pytest__ ssh` writes the shm object and prints its name without consuming it, so it can be inspected live:

```
$ printf '' | ./kitty/launcher/kitten __pytest__ ssh 'echo UNTAR_DONE'   # prints {"cmd":…,"shm_name":…}
$ ls -l /dev/shm/kssh-58194-3NZCHVVAXYVHS
-rw------- 1 root root 31655 Jul 14 22:15 /dev/shm/kssh-58194-3NZCHVVAXYVHS

$ stat /dev/shm/kssh-58194-3NZCHVVAXYVHS
  File: /dev/shm/kssh-58194-3NZCHVVAXYVHS
  size: 31655   ...   regular file
Access: (0600/-rw-------)  Uid: (    0/    root)   Gid: (    0/    root)
```

Programmatic proof that owner/group equal this process's effective ids and the mode is exactly `0600`:

```
st_uid=0  geteuid=0  match=True
st_gid=0  getegid=0  match=True
mode=0o600  is_0600=True
```

### 2.2 Creation guarantees

The Python creator (`kitty/shm.py`) opens with `flags = os.O_CREAT | os.O_EXCL` [kitty/shm.py:L62] — the `O_EXCL` prevents an attacker pre-creating (squatting) the object — and a default mode of `stat.S_IREAD | stat.S_IWRITE` = `0600` [kitty/shm.py:L51]. On the Go side the kitten allocates the object via `shm.CreateTemp("kssh-<pid>-", len(encoded_data)+8)` [kittens/ssh/main.go:L446] using the POSIX backends in `tools/utils/shm/` (`shm.go`, `shm_syscall.go:L162`, `specific_linux.go:L11`).

### 2.3 The 4-byte size prefix and the payload shape

The object is framed as **`[4-byte big-endian size][JSON]`**. The size format is `size_fmt = '!I'` [kitty/shm.py:L46] so `num_bytes_for_size = struct.calcsize('!I') = 4` [kitty/shm.py:L47]; `write_data_with_size` packs `struct.pack('!I', len(data))` then the bytes [kitty/shm.py:L115-L119, pack at L118]. Decoding the live object confirms the framing exactly, and shows the four JSON keys:

```
prefix_width=4  size=31647  total=31655  after_json_offset=31651  trailing_bytes=4
trailing bytes hex: 00000000            # slack from the Go len+8 allocation
JSON keys: ['hostname', 'pw', 'tarfile', 'username']
  hostname: 'host.test'
  username: 'testuser'
  pw: 64-char hex secret = b8af6e29…7d9b     # ephemeral secrets.TokenHex, redacted tail
  tarfile: base64 len=31516 -> decoded 23636 bytes, gzip_magic=1f8b
```

(The Go side over-allocates by 8 bytes; the 4-byte size prefix + JSON leave 4 trailing zero bytes of slack, shown above.)

### 2.4 Run-to-run variance

Two consecutive runs produce different object names (fresh pid + random suffix, fresh `pw`):

```
run1 shm_name: kssh-58194-3NZCHVVAXYVHS
run2 shm_name: kssh-58359-IWBTBR2SRVWPI
```

### 2.5 Why credentials never hit the command line

Because the data lives in the shm object and only a **password + object name** cross to the remote — and even those travel over the tty via DCS, not as `argv` — a process listing (`ps`) on either host never reveals the archive or the password. The single exception is the fully self-contained *reused* path, where no data channel is opened at all (§6). Security enforcement on the read side is covered in §9; the handshake that carries the password is §10.

---


## §3 How are the bootstrap scripts that run on the remote machine generated?

**Direct answer.** The kitten picks one of **two** template scripts based on the `interpreter` option, fills in a set of named placeholders, and that filled-in script becomes the command run on the remote host. `get_remote_command` [kittens/ssh/main.go:L511-L518] chooses `script_type = "py"` if the lower-cased basename of `interpreter` contains `"python"`, otherwise `"sh"` (default `interpreter` is `sh` [kittens/ssh/main.py:L87]). The chosen template is loaded from kitty's embedded shell-integration data — `shell-integration/ssh/bootstrap.sh` for `sh`, `shell-integration/ssh/bootstrap.py` for `py`, with shared helpers in `shell-integration/ssh/bootstrap-utils.sh` — and `prepare_script` [kittens/ssh/main.go:L407] substitutes the placeholders.

### 3.1 Generation pipeline

`bootstrap_script` [kittens/ssh/main.go:L422] assembles everything:

* the per-connection `request_id` (falling back to `KITTY_PID-KITTY_WINDOW_ID` [main.go:L424]);
* a fresh data password `pw = secrets.TokenHex()` [main.go:L432];
* the tar archive via `make_tarfile` [main.go:L436] (see §4);
* a data map `{tarfile(base64), pw, hostname, username}` [main.go:L442-L445] which is written to the shm object [main.go:L446-L459];
* a set of sensitive replacements `{REQUEST_ID, DATA_PASSWORD, PASSWORD_FILENAME}` [main.go:L461] that are only merged into the substitution set when `request_data` is true [main.go:L479];
* the template body is fetched by key `shell-integration/ssh/bootstrap.<type>` and passed to `prepare_script` [main.go:L483-L484].

`prepare_script` replaces each `\bKEY\b` token and defaults `EXEC_CMD` / `EXPORT_HOME_CMD` to empty strings.

### 3.2 The placeholders, and their observed filled-in values

Substituted tokens in `bootstrap.sh` and the concrete values seen in a captured, generated `sh` script (`kitten __pytest__ ssh 'echo UNTAR_DONE'`), obtained by `diff`-ing the generated script against the template:

| Placeholder | Template line | Observed replacement |
|-------------|---------------|----------------------|
| `ECHO_ON` | [bootstrap.sh:L8] | `1` |
| `EXPORT_HOME_CMD` | [bootstrap.sh:L79] | *(empty)* |
| `REQUEST_DATA` | [bootstrap.sh:L90] | `1` |
| `REQUEST_ID` / `PASSWORD_FILENAME` / `DATA_PASSWORD` | [bootstrap.sh:L94] | `id="testing":pwfile="kssh-49068-…":pw="0c8c…bb"` |
| `EXEC_CMD` | [bootstrap.sh:L159] | *(empty)* |
| `TEST_SCRIPT` | [bootstrap.sh:L162] | `echo UNTAR_DONE` |

The Python template shows the analogous substitutions at `bootstrap.py` L20 (`ECHO_ON`), L22 (`REQUEST_DATA`), L29 (`EXPORT_HOME_CMD`), L81 (the DCS `id=…:pwfile=…:pw=…`), L306 (`EXEC_CMD`), L311 (`TEST_SCRIPT`).

### 3.3 Both generated scripts, verified against their templates

**sh scheme** (`interpreter=sh`, default):

```
$ printf '' | ./kitty/launcher/kitten __pytest__ ssh 'echo UNTAR_DONE'
{"cmd":["exec","sh","-c","<tr-unwrap>","<quoted-encoded 5277 bytes>"],"shm_name":"kssh-49068-4WEGPPNMSCSF6"}
```

After removing the wrapping single-quotes and reversing the byte substitution (§7), the decoded script is byte-for-byte `shell-integration/ssh/bootstrap.sh` with only the placeholder lines of §3.2 changed (`diff` shows *only* those lines differ).

**py scheme** (forced via stdin config `interpreter python3`):

```
$ printf 'interpreter python3' | ./kitty/launcher/kitten __pytest__ ssh 'echo UNTAR_DONE'
{"cmd":["exec","python3","-c",
        "import base64, sys; eval(compile(base64.standard_b64decode(sys.argv[-1]), 'bootstrap.py', 'exec'))",
        "<base64 13580 bytes>"],
 "shm_name":"kssh-49219-SSYNI43WLJHM6"}
```

Base64-decoding the last argument yields 10183 bytes that equal the template `shell-integration/ssh/bootstrap.py` (both 318 lines; first line `#!/usr/bin/env python`) with the §3.2 placeholders filled.

---

## §4 How is the archive of all shell-integration files built and sent over?

**Direct answer.** `make_tarfile` [kittens/ssh/main.go:L255] builds an **in-memory gzip (BestCompression) + tar** archive containing kitty's shell-integration files, the terminfo database, small `kitty`/`kitten` launcher stubs, and an env script produced by `serialize_env` [main.go:L204]; every entry's mode is OR-ed with `0o600` (`h.Mode |= 0o600`) so no file is left unreadable/unwritable by its owner. The archive is base64-encoded, placed in the shm JSON (§2), and — when the remote asks for it (§10) — streamed back over the tty in **≤254-byte lines** framed by `KITTY_DATA_START` / `OK` / … / `KITTY_DATA_END` [kittens/ssh/utils.py:L117,L138-L148]. The remote reassembles and pipes it through `base64 -d | tar xpzf - -C <tmpdir>` and atomically moves files into place [shell-integration/ssh/bootstrap.sh:L104-L135, tar at L113].

### 4.1 Where the payload comes from

The shell-integration files are embedded into the kitten at build time by `generate_ssh_kitten_data` [gen/go_code.py:L840], written to `tools/tui/shell_integration/data_generated.bin` [gen/go_code.py:L848]. At runtime `make_tarfile` uses `gzip.BestCompression` [main.go:L259], a `tar.NewWriter` [main.go:L262], and an add-closure that sets `h.Mode |= 0o600` [main.go:L267].

### 4.2 The real archive, decoded and listed

Taking the `tarfile` value from a live shm object, base64-decoding and gunzipping it (`gzip_magic=1f8b`, decodes cleanly), then `tar tvf -` lists **15 members**:

```
-rw-r--r-- 0/0     195   data.sh
-rw-r--r-- 0/0    8468   bootstrap-utils.sh
-rw-r--r-- 0/0     286   home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_completions.d/kitten.fish
-rw-r--r-- 0/0   22557   home/.local/share/kitty-ssh-kitten/shell-integration/zsh/kitty-integration
-rw-r--r-- 0/0   10409   home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish
-rw-r--r-- 0/0    1880   home/.local/share/kitty-ssh-kitten/shell-integration/zsh/.zshenv
-rw-r--r-- 0/0     280   home/.local/share/kitty-ssh-kitten/shell-integration/zsh/completions/_kitty
-rw-r--r-- 0/0   17363   home/.local/share/kitty-ssh-kitten/shell-integration/bash/kitty.bash
-rw-r--r-- 0/0     285   home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_completions.d/kitty.fish
-rw-r--r-- 0/0     294   home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_completions.d/clone-in-kitty.fish
-rw-r--r-- 0/0       6   home/.local/share/kitty-ssh-kitten/kitty/version
-rwxr-xr-x 0/0    4377   home/.local/share/kitty-ssh-kitten/kitty/bin/kitty
-rwxr-xr-x 0/0    2761   home/.local/share/kitty-ssh-kitten/kitty/bin/kitten
-rw-r--r-- 0/0    4271   home/.terminfo/kitty.terminfo
-rw-r--r-- 0/0    3711   home/.terminfo/x/xterm-kitty
```

Every regular file's mode is ≥ `0o600` (data files `0644`, the two launcher stubs `0755`), confirming the `h.Mode |= 0o600` guarantee. The env script `data.sh` (from `serialize_env`) contains:

```
export TERM=dumb; COLORTERM=truecolor; KITTY_SHELL_INTEGRATION=enabled;
KITTY_SSH_KITTEN_DATA_DIR=.local/share/kitty-ssh-kitten; KITTY_REMOTE=if-needed
```

### 4.3 Transport framing (≤254-byte lines)

The stream is emitted by `get_ssh_data` with `line_sz = 254` [kittens/ssh/utils.py:L143] — the 254-byte cap accommodates macOS's 255-byte terminal input-queue limit. Measured directly from the real `get_ssh_data` output for the happy path (§9): the reply carried **4 data lines** whose **maximum length was exactly 254 bytes** (`max line len=254 (<=254)`). The remote consumes them in `read_base64_from_tty` (loop until `KITTY_DATA_END`) [shell-integration/ssh/bootstrap.sh:L97-L102] inside `untar_and_read_env`, which pipes `read_base64_from_tty | base64_decode | tar xpzf - -C "$tdir"` [bootstrap.sh:L113] and then moves the files into place. A real round-trip (§8) shows the untar completing: `~/.terminfo/kitty.terminfo` exists afterward and `data.sh` env vars are in effect.

---


## §5 How does the kitten keep track of everything it needs for a connection?

**Direct answer.** All per-connection state lives in one Go struct, `type connection_data struct` [kittens/ssh/main.go:L171-L189]. It has **exactly 16 fields**, populated as the kitten parses arguments, loads host config, and generates the remote command. Here is every field:

| # | Field | Purpose |
|---|-------|---------|
| 1 | `remote_args` | the command (if any) to run on the remote after bootstrap |
| 2 | `host_opts` | the resolved per-host ssh-kitten options (interpreter, share_connections, askpass, …) |
| 3 | `hostname_for_match` | hostname used to match `host` blocks in config |
| 4 | `username` | remote username |
| 5 | `echo_on` | whether tty echo should be restored (`ECHO_ON` placeholder) |
| 6 | `request_data` | **the core switch**: must the remote request the data blob? (§6) |
| 7 | `literal_env` | environment entries passed through literally |
| 8 | `listen_on` | remote-control listen socket (for `forward_remote_control`) |
| 9 | `test_script` | injected script used by the test harness (`TEST_SCRIPT`) |
| 10 | `dont_create_shm` | suppress shm creation (used when no data is needed) |
| 11 | `shm_name` | the `/dev/shm/kssh-*` object name once written |
| 12 | `script_type` | `"sh"` or `"py"` (§3) |
| 13 | `rcmd` | the final remote command vector (`exec <interp> -c <unwrap> <encoded>`) |
| 14 | `replacements` | the placeholder→value map fed to `prepare_script` |
| 15 | `request_id` | per-window id (`KITTY_PID-KITTY_WINDOW_ID`, or `"testing"` under the test hook) |
| 16 | `bootstrap_script` | the fully substituted (pre-encoding) bootstrap body |

### 5.1 Observed field values from a real run

Several fields are directly observable. From `kitten __pytest__ ssh` JSON you see `rcmd` (as `cmd`) and `shm_name`; from the generated bootstrap and the fake-ssh trace you see `request_id`, `script_type`, `request_data`, `echo_on`:

```
# sh run
shm_name    = kssh-49068-4WEGPPNMSCSF6
script_type = sh          (cmd[1] == "sh")
rcmd        = ["exec","sh","-c","<tr-unwrap>","<quoted-encoded>"]
request_id  = testing     (fixed by the __pytest__ hook)
request_data= 1           (REQUEST_DATA in the generated bootstrap)
echo_on     = 1

# py run
shm_name    = kssh-49219-SSYNI43WLJHM6
script_type = py          (cmd[1] == "python3")
```

And from a real `kitten ssh` PTY run (fresh path), `request_id` takes its production fallback value `KITTY_PID-KITTY_WINDOW_ID`, observed as `55002-7` (see §6.2). These observed values line up exactly with the 16-field definition at [main.go:L171-L189].

### 5.2 How the struct is filled end-to-end

Arguments are parsed by `parse_kitten_args` [main.go:L96] and the destination by `get_destination` [main.go:L46]; `host_opts` is resolved from config; then `get_remote_command` [main.go:L511] fills `script_type`, `bootstrap_script`, and `rcmd`, and `shm_name` is set when the data blob is written [main.go:L459]. `request_data` is decided in `run_ssh` (next section) and copied into the struct at [main.go:L724].

---

## §6 How does connection reuse decide between a fresh connection and piggybacking on an existing one?

**Direct answer.** One line decides it:

```
if need_to_request_data && host_opts.Share_connections && master_is_functional() {
    need_to_request_data = false
}
```
[kittens/ssh/main.go:L663-L665]

If the kitten still thinks it needs to request data **and** connection sharing is on **and** a master connection is already functional, it flips `need_to_request_data` to **false** — i.e. it piggybacks on the live master and does **not** open a data channel for this connection. Otherwise `need_to_request_data` stays **true** and the remote will request the data blob via the DCS handshake.

### 6.1 `master_is_functional()` — the `-O check` probe

`master_is_functional` [main.go:L653-L661] runs the system `ssh` with `-O check` inserted at `argv[1]` [main.go:L657] and treats **exit 0** as "master alive" [main.go:L659], caching the result in `master_checked`. When the master is absent, `run_control_master` [main.go:L666-L680] can start a background master by adding `-N -f` [main.go:L669]. Each `kitten ssh` run therefore makes up to three `ssh` calls: an options probe (`argc=0`, `SSHOptions()` [utils.go:L40]), the `-O check` probe (`argc=16`, [main.go:L659]), and the real connection (`argc=19`, [main.go:L754]).

### 6.2 Isolating the branch and observing BOTH outcomes

Because **two** independent mechanisms set `request_data=false` — a live master *and* the kitty-askpass path (`use_kitty_askpass` [main.go:L648]) — the connection-sharing branch is isolated by forcing `askpass=ssh` (so `use_kitty_askpass` is false and `set_askpass` [main.go:L147-L166] is not consulted). The fake `ssh` then controls the `-O check` exit code.

**Reused path — `FAKE_SSH_OCHECK_RC=0` (master alive).** The `-O check` invocation returns 0; the generated bootstrap has `request_data="0"` and its DCS line is left **unsubstituted** (kitty will send the DCS itself — §10):

```
  -> -O check seen; returning FAKE_SSH_OCHECK_RC=0
...
request_data="0"
[ "$request_data" = "1" ] && {
    command stty "-echo" < /dev/tty
    dcs_to_kitty "ssh" "id="REQUEST_ID":pwfile="PASSWORD_FILENAME":pw="DATA_PASSWORD""
}
```

**Fresh path — `FAKE_SSH_OCHECK_RC=1` (master absent).** The `-O check` invocation returns 1; the generated bootstrap has `request_data="1"` and its DCS line is **filled in** with the real request id, shm object name and password:

```
  -> -O check seen; returning FAKE_SSH_OCHECK_RC=1
...
request_data="1"
[ "$request_data" = "1" ] && {
    command stty "-echo" < /dev/tty
    dcs_to_kitty "ssh" "id="55002-7":pwfile="kssh-55940-KQOKWNYAOHR2Q":pw="54148b88…1673""
}
```

Note `id="55002-7"` = `KITTY_PID-KITTY_WINDOW_ID`, confirming the `request_id` fallback [main.go:L424].

### 6.3 Cause → effect summary

| `-O check` result | `need_to_request_data` | Observed bootstrap | Data channel |
|-------------------|------------------------|--------------------|--------------|
| exit 0 (master alive) | flipped to **false** [L664] | `request_data="0"`, DCS placeholders left literal | none opened; kitty sends DCS itself [L763-L772] |
| exit ≠ 0 (master absent) | stays **true** | `request_data="1"`, DCS line filled with `id/pwfile/pw` | remote requests data over DCS (§10) |

---


## §7 How does the bootstrap encoding work, with character substitutions for different shells?

**Direct answer.** The generated bootstrap is wrapped so it can survive being passed as a single `ssh` command-line argument, by `wrap_bootstrap_script` [kittens/ssh/main.go:L486-L509]. There are **exactly two** encoding schemes, selected by `script_type` (§3): a **Python** scheme (base64) and a **POSIX-sh** scheme (a four-byte character substitution reversed remotely with `tr`). In all cases the final remote command vector is `["exec", <interpreter>, "-c", <unwrap>, <encoded>]` [main.go:L508].

### 7.1 Python scheme — base64

[main.go:L497-L499] base64-encodes the script and the unwrap one-liner decodes+executes it:

```
encoded = base64.StdEncoding.EncodeToString(bootstrap)
unwrap  = "import base64, sys; eval(compile(base64.standard_b64decode(sys.argv[-1]), 'bootstrap.py', 'exec'))"
rcmd    = ["exec", <interpreter>, "-c", unwrap, encoded]
```

Proven by forcing `interpreter=python3` and decoding the last argument (see §3.3): the base64 payload decodes byte-for-byte to `shell-integration/ssh/bootstrap.py`.

### 7.2 POSIX-sh scheme — four-byte substitution + `tr`

base64 may be unavailable on a remote *before* the archive lands, so the sh scheme avoids it. Instead the script is single-quoted after **four control-safe byte substitutions** [main.go:L500-L506], and the remote reverses them with `tr`:

| Original byte | → substituted with | reversed remotely by `tr` |
|---------------|---------------------|---------------------------|
| `'` `0x27` (single quote) | `\v` `0x0b` (vertical tab) | `\v` → `\047` |
| `\` `0x5c` (backslash) | `\f` `0x0c` (form feed) | `\f` → `\134` |
| newline `0x0a` | `\r` `0x0d` (carriage return) | `\r` → `\n` |
| `!` `0x21` (bang) | `\b` `0x08` (backspace) | `\b` → `\041` |

The unwrap string embedded in the remote command is (from [main.go:L506]):

```
eval "$(echo "$0" | tr \v\f\r\b \047\134\n\041)"
```

i.e. it maps the four control bytes back to `'` (`\047`), `\` (`\134`), newline (`\n`), `!` (`\041`). Single-quoting works because the only byte that could terminate a single-quoted string — the single quote itself — has been removed by the substitution.

### 7.3 Byte-accurate proof (mandatory)

Taking the **exact bytes the kitten emitted** for the sh scheme (the encoded argument with its wrapping single-quotes stripped), the four control bytes are present and the four originals are entirely absent:

```
$ printf '' | ./kitty/launcher/kitten __pytest__ ssh 'echo UNTAR_DONE'   # shm_name: kssh-67470-DANQGI6LTRFZ2
--- byte histogram of the substituted script ---
VT(0x0b) count : 10
FF(0x0c) count : 25
CR(0x0d) count : 164
BS(0x08) count : 3
raw squote(0x27): 0
raw bslash(0x5c): 0
raw newline(0x0a):0
raw bang(0x21)   : 0
```

Reversing through the **real system `tr`** (uutils coreutils 0.2.2) exactly as the remote does, the counts swap over perfectly and no control bytes remain:

```
$ tr '\013\014\015\010' '\047\134\012\041' < S_sub.bin > B_rev.sh   # \v\f\r\b -> ' \ newline !
first line of reversed script: #!/bin/sh
reversed histogram: squote=10 bslash=25 newline=164 bang=3
residual control bytes in reversed (should be 0): VT=0 FF=0 CR=0 BS=0
```

The symmetry is exact: 10 `'`↔10 VT, 25 `\`↔25 FF, 164 newlines↔164 CR, 3 `!`↔3 BS. Finally, forward-substituting the reversed script and comparing to the original emitted bytes is **byte-identical**:

```
$ tr '\047\134\012\041' '\013\014\015\010' < B_rev.sh > S_roundtrip.bin
$ cmp S_sub.bin S_roundtrip.bin && echo "cmp: identical (exit=$?)"
cmp: identical (exit=0)
```

And the reversed script equals the template with only placeholders filled (164 lines each; only the six placeholder lines of §3.2 differ):

```
$ diff <(sed -n '1,200p' shell-integration/ssh/bootstrap.sh) B_rev.sh
< echo_on="ECHO_ON"
> echo_on="1"
< EXPORT_HOME_CMD
>
< request_data="REQUEST_DATA"
> request_data="1"
< dcs_to_kitty "ssh" "id="REQUEST_ID":pwfile="PASSWORD_FILENAME":pw="DATA_PASSWORD""
> dcs_to_kitty "ssh" "id="testing":pwfile="kssh-67470-…":pw="d50e5c73…317a""
< EXEC_CMD
>
< TEST_SCRIPT
> echo UNTAR_DONE
```

### 7.4 Two senses of "different shells" — both covered

The question names **sh/bash/zsh/fish launchers *and* the Python interpreter**; these are two distinct axes and both are exercised.

**Axis 1 — the `interpreter` option selects the *encoding scheme*** (`sh` vs `py`, §3/§7.1-7.2). The two produce visibly different `rcmd` shapes:

```
sh : ["exec","sh","-c","eval \"$(echo \"$0\" | tr \v\f\r\b \047\134\n\041)\"","'<substituted>'"]
py : ["exec","python3","-c","import base64, sys; eval(compile(base64.standard_b64decode(sys.argv[-1]), 'bootstrap.py', 'exec'))","<base64>"]
```

**Axis 2 — the remote *login shell / launcher* that actually runs the bootstrap.** Driving the real PTY round-trip (`check_bootstrap`) across launchers and login shells:

* Launchers **sh, dash, bash, zsh** (with `interpreter=sh`): each reached `UNTAR_DONE=True` and extracted terminfo (`terminfo=True`).
* Interpreter **python3**: `UNTAR_DONE=True`, `terminfo=True`.
* Login shells **fish, zsh, bash**: each reached the shell with shell-integration active (cursor became `BEAM`).
* The canonical named tests `test_ssh_bootstrap_with_different_launchers` [kitty_tests/ssh.py:L146-L155] and `test_ssh_shell_integration` [kitty_tests/ssh.py:L198-L201] (which iterates `all_possible_sh` [L63] = `dash, zsh, bash, posh, sh, python3`) both pass — see §8.

```
# per-launcher (real check_bootstrap, test_script='env; exit 0', interpreter=sh)
sh   -> UNTAR_DONE=True terminfo=True
dash -> UNTAR_DONE=True terminfo=True
bash -> UNTAR_DONE=True terminfo=True
zsh  -> UNTAR_DONE=True terminfo=True
# login-shell handoff
fish -> shell reached, cursor=BEAM
zsh  -> shell reached, cursor=BEAM
bash -> shell reached, cursor=BEAM
```

All of `dash`, `zsh`, `bash`, `fish`, and `python3` are installed in this container (§0.1), so every named variant was genuinely exercised — none had to be skipped.

---


## §8 Full trace: from the user initiating SSH to the bootstrap executing remotely

**Direct answer.** The end-to-end path is: `kitten ssh` → parse args/host config → compute sharing args → decide askpass and fresh-vs-reused → generate + encode the bootstrap → (if fresh) write the shm blob and launch `ssh` → on the remote, the login shell runs `interpreter -c <unwrap> <encoded>`, which self-decodes and runs the bootstrap → the bootstrap emits a DCS data request → kitty validates and streams the tar back → the remote untars, sets up terminfo + shell integration, and `exec`s the user's real login shell.

### 8.1 The spine, step by step (with `file:line`)

| Step | What happens | Source |
|------|--------------|--------|
| 1 | `main` → `run_ssh` | [main.go:L800; L597] |
| 2 | destination + args parsed | `get_destination` [L46]; `parse_kitten_args` [L96] |
| 3 | six sharing args computed and inserted into the ssh cmdline | [L121-L145]; inserted [L642] |
| 4 | askpass decision (`use_kitty_askpass` [L648]; `set_askpass` [L147-L166]); fresh-vs-reused decision | [L663-L665] |
| 5 | `get_remote_command` → `bootstrap_script` → `wrap_bootstrap_script` build `rcmd` | [L511; L422; L486-L509] |
| 6 | if data needed: shm written; ssh launched; controlling terminal opened | shm [L446-L459]; `OpenControllingTerm` [L718] |
| 7 | remote login shell runs `interpreter -c <unwrap> <encoded>`; the bootstrap self-decodes (sh: `tr`; py: base64) | §7 |
| 8 | remote emits the DCS request | `dcs_to_kitty` [bootstrap.sh:L75]; `request_data` gate [L90-L95] |
| 9 | kitty answers: DCS `@kitty-ssh` → `handle_remote_ssh` → `get_ssh_data`; streams the tar | [window.py:L1289-L1291; utils.py:L115-L148] |
| 10 | remote reassembles (`read_base64_from_tty` [bootstrap.sh:L97-L102]), untars (`untar_and_read_env` [L104-L135]), then `exec_login_shell` | [bootstrap.sh:L164] |

### 8.2 Primary evidence — the canonical PTY suite

`kitty_tests/ssh.py` performs exactly this round-trip over a real pty. The full module passes:

```
$ ./kitty/launcher/kitty +launch test.py --module ssh
test_basic_pty_operations ... ok
test_ssh_bootstrap_with_different_launchers ... ok
test_ssh_connection_data ... ok
test_ssh_copy ... ok
test_ssh_env_vars ... ok
test_ssh_leading_data ... ok
test_ssh_login_shell_detection ... ok
test_ssh_shell_integration ... ok
----------------------------------------------------------------------
Ran 8 tests in 10.066s

OK
```

### 8.3 Byte-level round-trip (real bootstrap + real `get_ssh_data`)

To observe the actual bytes crossing the tty, the real generated bootstrap was run under a PTY (remote side) while the real `get_ssh_data` answered the DCS (terminal side; the same call `handle_remote_ssh` makes). The complete chain fired:

```
--- DCS request captured (remote -> terminal) ---
raw: \x1bP@kitty-ssh|aWQ9dGVzdGluZzpwd2ZpbGU9a3NzaC01OTk3OS1XVTVWSzNOT0dPN1RFOnB3PWQ2Ni…=\x1b\\
DECODED: id=testing:pwfile=kssh-59979-WU5VK3NOGO7TE:pw=d668…48f0
--- framed reply markers delivered (terminal -> remote) ---
markers: ['KITTY_DATA_START', 'OK', 'KITTY_DATA_END']   reply_sent: True
--- end-to-end result on the remote ---
UNTAR_DONE present: True
terminfo written:  True         # ~/.terminfo/kitty.terminfo exists
env from data.sh:  KITTY_SHELL_INTEGRATION=enabled ; COLORTERM=truecolor
```

The remote reached the post-data section that precedes `exec_login_shell` [bootstrap.sh:L164]; the login-shell handoff itself is asserted by the canonical suite (cursor becomes `BEAM` once shell integration loads *after* the handoff, in `test_ssh_shell_integration`).

---

## §9 How does shared memory keep things secure?

**Direct answer.** The terminal-side data server refuses to hand out the archive unless **six** conditions hold, enforced across `read_data_from_shared_memory` [kittens/ssh/utils.py:L100-L112] and `get_ssh_data` [kittens/ssh/utils.py:L115-L148]. In order: the object is **unlinked on first read** (single-use); the **owner** must match the effective uid/gid; the **permissions** must be exactly `0600`; the request **message must parse**; the **password** must match; and the **request id** must match the current window. Only then does it emit `OK` and stream the tar. These checks are meaningful because creation used `O_EXCL` (no squatting) [kitty/shm.py:L62] and `0600` [kitty/shm.py:L51]. Note the owner/permission checks read `shm.stats`, captured by `os.fstat` at open time [kitty/shm.py:L89], so they remain valid even though the object is unlinked first.

### 9.1 The six checks, each triggered at runtime

Driving the **real** `get_ssh_data` exactly as `handle_remote_ssh` does (`get_ssh_data(memoryview(<base64 payload>), request_id)` with `request_id = f'{os.getpid()}-{self.id}'` [window.py:L1291]; observed `request_id='57935-1'`), each branch was provoked and its exact output captured:

**1 — Happy path** → `OK` + framed 254-byte data + `KITTY_DATA_END`:

```
[1 HAPPY PATH] OK present=True  DATA_END present=True  data lines=4  max line len=254 (<=254)
```

**2 — Wrong password** [utils.py:L131]:

```
[2 WRONG PASSWORD]
   Incorrect password
```

**3 — Wrong request id** [utils.py:L133]:

```
[3 WRONG REQUEST ID]
   Incorrect request id: 'NOT-57935-1' expecting the KITTY_PID-KITTY_WINDOW_ID for the current kitty window
```

**4 — Wrong permissions** (object created `0o644`) [utils.py:L109-L111]:

```
[4 setup] /dev/shm/kssh-q9-… mode = 0o644 (created 0644 to trip check)
[4 WRONG PERMISSIONS]
   Incorrect permissions on pwfile: 0o644
```

**5 — Wrong owner** (object chowned to uid=1, while euid=0) [utils.py:L107-L108]:

```
[5 setup] /dev/shm/kssh-q9-… chowned to uid=1 gid=1 (euid=0 egid=0)
[5 WRONG OWNER]
   Incorrect owner on pwfile: uid=1 gid=1
```

**6 — Invalid / garbled message** (not the `id=…:pwfile=…:pw=…` shape) [utils.py:L120,L124-L126]:

```
[6 INVALID MESSAGE]
   invalid ssh data request message
```

### 9.2 Single-use (unlink-on-read)

Calling twice on the same object: the first read succeeds and unlinks the object [utils.py:L106] (plus an `atexit.register(shm.unlink)` safety net [utils.py:L96]); the second read fails because the object is gone:

```
[7 SINGLE-USE first call]  OK present=True   exists_after=False
[7 SINGLE-USE second call (same shm)]
   [Errno 2] No such file or directory: '/kssh-q9-…'
```

### 9.3 Ordering note

`read_data_from_shared_memory` unlinks **first** [L106], then checks owner [L107-L108], then permissions [L109-L111], using the `os.fstat` snapshot taken when the object was opened [shm.py:L89]. So even a hostile object is removed from the filesystem before its metadata is judged, and the judgement is on the state at open time. On any rejection, `get_ssh_data` both logs a traceback (showing the exact `raise` site) and yields the clean one-line error to the remote — which the bootstrap surfaces via `die "$line"` [bootstrap.sh:L141].

---


## §10 How does the terminal communicate back and forth with the remote shell during setup?

**Direct answer.** The two sides talk over the tty itself using **DCS (Device Control String)** sequences of the form `ESC P @kitty-<type> | <base64-payload> ESC \`. The remote → terminal direction is emitted by `dcs_to_kitty` [shell-integration/ssh/bootstrap.sh:L75]; the terminal → remote direction is either kitty's framed reply (the tar stream) or, in the reused case, a DCS the terminal itself builds with `DCSToKitty` [tools/tui/dcs_to_kitty.go:L14]. kitty routes an incoming `@kitty-ssh` DCS to `handle_remote_ssh` [kitty/window.py:L1289-L1291] (and `@kitty-askpass` to `handle_remote_askpass` [kitty/window.py:L1351]).

### 10.1 The DCS frame is identical in both directions

Remote builder (POSIX sh) [bootstrap.sh:L75]:

```
dcs_to_kitty() { printf "\033P@kitty-$1|%s\033\134" "$(printf "%s" "$2" | base64_encode)" > /dev/tty; }
```

Terminal builder (Go) [tools/tui/dcs_to_kitty.go:L14]:

```
data := base64.StdEncoding.EncodeToString(payload)
ans  := "\x1bP@kitty-" + msgtype + "|" + data + "\033\\"   // (tmux passthrough wrap if under tmux)
```

Both produce `ESC P @kitty-<type> | base64 ESC \` (`\033\134` == `\x1b\x5c` == `ESC \`).

### 10.2 The captured request (remote → terminal)

From the real bootstrap under a PTY (`request_data=true`), the raw DCS bytes and the decoded payload:

```
raw:     \x1bP@kitty-ssh|aWQ9dGVzdGluZzpwd2ZpbGU9a3NzaC01OTk3OS1XVTVWSzNOT0dPN1RFOnB3PWQ2Ni…=\x1b\\
base64:  aWQ9dGVzdGluZzpwd2ZpbGU9…
DECODED: id=testing:pwfile=kssh-59979-WU5VK3NOGO7TE:pw=d668…48f0
```

This matches the template's request line `id="REQUEST_ID":pwfile="PASSWORD_FILENAME":pw="DATA_PASSWORD"` [bootstrap.sh:L94] (Python equivalent [bootstrap.py:L81]). `request_id` is `"testing"` under the test hook; in production it is `KITTY_PID-KITTY_WINDOW_ID` (observed `55002-7` in §6.2). `pw` is fresh per run (`d668…` here vs `ea95…` on another run — run-to-run variance).

### 10.3 The framed reply (terminal → remote)

kitty answers via `get_ssh_data`, which yields, in order [kittens/ssh/utils.py]: a leading `\nKITTY_DATA_START\n` to discard tty garbage [L117], then `OK\n` [L138], then the base64 tar in ≤254-byte lines [L143-L147], then `KITTY_DATA_END\n` [L148]. Observed markers delivered to the remote:

```
markers: ['KITTY_DATA_START', 'OK', 'KITTY_DATA_END']   reply_sent: True
```

The remote's `get_data` [bootstrap.sh:L137-L153] discards everything until it sees `KITTY_DATA_START`, then requires the next line to be `OK` (any other line becomes `die "$line"` — this is how the §9 error strings surface on the remote), then `untar_and_read_env` reads the base64 body until `KITTY_DATA_END`.

### 10.4 Two DCS senders, chosen by `request_data`

* **Fresh (`request_data=true`)** — the **remote** sends the request: guarded by `[ "$request_data" = "1" ]` it runs `stty -echo` then `dcs_to_kitty "ssh" …` [bootstrap.sh:L92-L95]. (Confirmed by the capture in §10.2.)
* **Reused (`request_data=false`)** — the **terminal** sends it instead: `if !cd.request_data { rq := fmt.Sprintf("id=%s:pwfile=%s:pw=%s", …) [main.go:L762]; term.ApplyOperations(TCSANOW, SetNoEcho) [L763]; DCSToKitty("ssh", rq) [L766]; term.WriteAllString(dcs) [L768] }`. The `SetNoEcho` here mirrors the remote's `stty -echo`.

### 10.5 Echo suppression and tty-garbage draining

`stty -echo` is run on the remote before the exchange [bootstrap.sh:L93] (and `SetNoEcho` on the terminal side [main.go:L763]) so the credential/data bytes are not echoed. This is directly observable: in the byte-level round-trip the reply markers `KITTY_DATA_START` / `KITTY_DATA_END` sent *to* the remote do **not** reappear in the remote's output stream (they were consumed, not echoed) — if echo were on they would bounce back. Leading garbage is handled on both ends: the terminal prefixes the reply with a `KITTY_DATA_START` discard marker [utils.py:L117]; the remote accumulates any pre-start bytes into `leading_data` [bootstrap.sh:L147]; and the kitten drains residual tty output after the child exits via `drain_potential_tty_garbage` [main.go:L530], which itself sends a `DCSToKitty("echo", canary)` probe [main.go:L539,L544] and is invoked at [main.go:L783]. `echo_on` is restored on exit by `cleanup_on_bootstrap_exit` [bootstrap.sh:L11].

### 10.6 Message types

| DCS type | Dispatch | Role |
|----------|----------|------|
| `@kitty-ssh` | `handle_remote_ssh` → `get_ssh_data(msg, f'{os.getpid()}-{self.id}')` [window.py:L1289-L1291] | the data request/response of this document |
| `@kitty-askpass` | `handle_remote_askpass` [window.py:L1351] | password prompts; relevant here only because it can also set `request_data=false` (§6) |
| `@kitty-print` | (via `dcs_to_kitty "print"`) [bootstrap.sh:L76] | debug output |

---

## Architecture diagram

Two channels run over the single `ssh` connection: the **shm credential channel** (local to the terminal host) and the **tty / DCS channel** (across the wire). The command line carries neither the archive nor the password (except the self-contained reused path).

```
   ┌─────────────────────────────── LOCAL (terminal host) ───────────────────────────────┐
   │                                                                                      │
   │  user: kitten ssh <host>                                                             │
   │        │  run_ssh                                     [main.go:L597]                 │
   │        ▼                                                                             │
   │  parse args/host cfg ─► connection_sharing_args (6× -o) [main.go:L121]               │
   │        │                                                                             │
   │        ▼   decide: askpass? master alive?  (need_to_request_data) [main.go:L663]     │
   │        │                                                                             │
   │        ├── fresh ──► make_tarfile (gzip+tar) [main.go:L255]                          │
   │        │             serialize_env  [main.go:L204]                                   │
   │        │             pw = secrets.TokenHex() [main.go:L432]                          │
   │        │             ┌───────────────── shm credential channel ──────────────────┐  │
   │        │             │  /dev/shm/kssh-<pid>-<rand>  (O_EXCL, 0600)                │  │
   │        │             │  [4-byte size][JSON {tarfile(b64), pw, hostname, user}]    │  │
   │        │             │  kitty/shm.py:L62,L51,L118    tools/utils/shm/*.go         │  │
   │        │             └───────────────────────────────────────────────────────────┘  │
   │        ▼                                                                             │
   │  wrap_bootstrap_script (sh: tr-substitution / py: base64) [main.go:L486]             │
   │        │  rcmd = exec <interp> -c <unwrap> <encoded>                                 │
   │        ▼                                                                             │
   │  system ssh  ── ControlMaster socket kssh-<pid>-%C (0600) ──►                        │
   └──────────┼──────────────────────────────────────────────────────────────────────────┘
              │                  tty / DCS channel  (never argv)
            ▼
   ┌─────────────────────────────── REMOTE (login shell) ────────────────────────────────┐
   │  <interp> -c <unwrap> <encoded>                                                      │
   │     └─ self-decode (sh: tr \v\f\r\b \047\134\n\041 / py: base64)  →  bootstrap.sh    │
   │            │ stty -echo; dcs_to_kitty "ssh" "id=…:pwfile=…:pw=…"  [bootstrap.sh:L75] │
   │            ▼                          ESC P @kitty-ssh|<b64> ESC \                    │
   │   ┌──────────────────────────────  DCS request  ──────────────────────────────────┐ │
   │   │  kitty: handle_remote_ssh → get_ssh_data   [window.py:L1289 / utils.py:L115]   │ │
   │   │    6 checks: unlink→owner→perms→parse→pw→id                                     │ │
   │   │    reply: KITTY_DATA_START / OK / <=254-byte b64 lines / KITTY_DATA_END         │ │
   │   └────────────────────────────────────────────────────────────────────────────────┘ │
   │            ▼                                                                          │
   │   get_data → read_base64_from_tty | base64 -d | tar xpzf -   [bootstrap.sh:L113]     │
   │   compile terminfo + shell-integration; . data.sh                                    │
   │            ▼                                                                          │
   │   exec_login_shell   [bootstrap.sh:L164]  ─►  user's real shell (kitty integrated)   │
   └──────────────────────────────────────────────────────────────────────────────────────┘
```

---


## Coverage matrix — the ten questions

| # | Question | Section | Key `file:line` anchors | Key observed evidence (command → result) |
|---|----------|---------|-------------------------|------------------------------------------|
| 1 | Secure session + connection sharing (ControlMaster/reuse) | §1 | main.go:L121-L145; utils.go:L22; constants.py:L188 | fake-ssh argv log → six `-o` options verbatim; real `sshd`: `-O check`→`Master running (pid=…)`, reuse in `0m0.007s`, socket `srw------- kssh-77001-<40hex>` |
| 2 | POSIX shm credential passing (`/dev/shm/kssh-*`) | §2 | shm.py:L46,L51,L62,L118; main.go:L446 | `ls -l`→`-rw------- root root`; `stat`→`0600`, uid/gid match; JSON keys `hostname,pw,tarfile,username`; `prefix_width=4` |
| 3 | Bootstrap-script generation | §3 | main.go:L407,L422,L511-L518; main.py:L87 | `kitten __pytest__ ssh` → sh & py `cmd`; `diff` vs templates shows only 6 placeholder lines filled |
| 4 | Archive build & transport | §4 | main.go:L255,L259,L267; gen/go_code.py:L840,L848; utils.py:L143 | `tar tvf` → 15 members; all modes ≥`0600`; `max line len=254` |
| 5 | Per-connection state tracking | §5 | main.go:L171-L189 | 16 fields enumerated; observed `shm_name`, `rcmd`, `script_type`, `request_id`, `request_data`, `echo_on` |
| 6 | Fresh-vs-reused decision | §6 | main.go:L653-L665; L648; L763-L772 | fake-ssh RC=0 → `request_data="0"` (placeholders literal); RC=1 → `request_data="1"` (`id="55002-7":pwfile=…:pw=…`) |
| 7 | Bootstrap encoding + per-shell substitutions | §7 | main.go:L486-L509; main.py:L87 | histogram `VT10/FF25/CR164/BS3`, raw originals `0`; `tr` reverse; `cmp … identical (exit=0)`; sh/dash/bash/zsh/fish/python3 all `UNTAR_DONE=True` |
| 8 | Full end-to-end trace | §8 | main.go:L597…; bootstrap.sh:L75,L104,L164; window.py:L1289 | `test.py --module ssh` → `Ran 8 tests … OK`; byte round-trip → `UNTAR_DONE=True`, terminfo written |
| 9 | Shared-memory security (six checks) | §9 | utils.py:L100-L148; L96; shm.py:L89 | six labelled captures + single-use double-call (`[Errno 2] No such file…`) |
| 10 | Terminal↔remote DCS handshake | §10 | bootstrap.sh:L75,L92-L95,L137-L153; dcs_to_kitty.go:L14; window.py:L1289,L1351 | raw `\x1bP@kitty-ssh|…\x1b\\` → `id=testing:pwfile=…:pw=…`; markers `KITTY_DATA_START/OK/KITTY_DATA_END` |

### Sub-table A — the six shared-memory security checks (§9)

| Order | Check | Rejection string | Source |
|-------|-------|------------------|--------|
| 1 | single-use unlink on read | *(2nd read)* `[Errno 2] No such file or directory` | utils.py:L106 (+ atexit L96) |
| 2 | owner == effective uid/gid | `Incorrect owner on pwfile: uid=… gid=…` | utils.py:L107-L108 |
| 3 | mode == `0600` | `Incorrect permissions on pwfile: 0o…` | utils.py:L109-L111 |
| 4 | message parses | `invalid ssh data request message` | utils.py:L120,L124-L126 |
| 5 | password matches | `Incorrect password` | utils.py:L130-L131 |
| 6 | request id matches | `Incorrect request id: … expecting the KITTY_PID-KITTY_WINDOW_ID …` | utils.py:L132-L133 |

### Sub-table B — the four sh-scheme byte substitutions (§7)

| Original | Hex | → Substituted | Hex | Observed count (this run) | Reverse (`tr`) |
|----------|-----|---------------|-----|---------------------------|-----------------|
| `'` | `0x27` | vertical tab `\v` | `0x0b` | 10 | `\v` → `\047` |
| `\` | `0x5c` | form feed `\f` | `0x0c` | 25 | `\f` → `\134` |
| newline | `0x0a` | carriage return `\r` | `0x0d` | 164 | `\r` → `\n` |
| `!` | `0x21` | backspace `\b` | `0x08` | 3 | `\b` → `\041` |

---

## Appendix — claims labelled _inferred from source_

Everything else in this document is backed by captured runtime output. Exactly two claims could not be directly observed and are labelled **_inferred from source_**:

1. **The macOS long-runtime-dir symlink workaround** [kittens/ssh/main.go:L128-L134]: when the runtime dir exceeds 35 bytes, the kitten symlinks it under `/tmp/kssh-rdir-<euid>`. On this Linux host the runtime dir is `/root/.cache/kitty/run` (21 bytes), so the branch does not fire — confirmed by the observed `ControlPath` using the real runtime dir. The branch's behavior on macOS is therefore read from source only.
2. **The exact inputs to OpenSSH's `%C` connection hash** (local host, remote host, port, user): these are internal to OpenSSH. Its *behavior* was confirmed empirically in §1.3 (a `kssh-<pid>-<40hex>` socket appears and is reused), but the precise hashed tuple is inferred, not observed.

---

## Appendix — reproduction commands (quick index)

```
# build
export PATH=/usr/local/go/bin:$PATH
python3 setup.py build --debug --ignore-compiler-warnings --skip-building-kitten
python3 setup.py build --debug --ignore-compiler-warnings --skip-code-generation

# generation / encoding (Q3, Q5, Q7)
printf '' | ./kitty/launcher/kitten __pytest__ ssh 'echo UNTAR_DONE'                 # sh scheme
printf 'interpreter python3' | ./kitty/launcher/kitten __pytest__ ssh 'echo UNTAR_DONE'  # py scheme

# full round-trip / launchers (Q4, Q7, Q8)
./kitty/launcher/kitty +launch test.py --module ssh

# connection sharing lifecycle (Q1) — kitten's own six -o options against a local sshd
ssh -o ControlMaster=auto -o ControlPath=$RD/kssh-<pid>-%C -o ControlPersist=yes \
    -o ServerAliveInterval=60 -o ServerAliveCountMax=5 -o TCPKeepAlive=no -p 2222 …
```

