# How the kitty SSH kitten works — a runtime-grounded investigation

**Subject:** kitty terminal — the SSH kitten (`kitten ssh`)
**Commit under investigation:** `815df1e21` (branch `kitty_815df1e210e0`, HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`)
**Nature:** Onboarding / knowledge-transfer investigation of an existing subsystem. No source file was modified; this document is the only artifact produced.

This document answers nine questions about how the SSH kitten sets up a secure session, shares connections, passes credentials through shared memory, builds and ships a shell-integration tarball, tracks connection state, decides on connection reuse, encodes its bootstrap script per shell, communicates with the remote over the TTY, and executes end-to-end.

Every behavioral claim below is backed by **complete, unedited command output that I captured by running the real code paths first**, together with an exact `file:line` reference at commit `815df1e21`. I explicitly label each statement as **[Observed]** (captured from a real run) or **[Inferred]** (derived from reading the code, where a signal could not be captured in this headless environment).

## Methodology and evidence rules honored

- **Run first, write second.** Every value shown was captured before this prose was written, from the real `kitten ssh` entry point (`ssh.EntryPoint(root)` [tools/cmd/tool/main.go:50], imported at [tools/cmd/tool/main.go:17]) and the genuine Go unit tests (`go test ./kittens/ssh/`), never from a remote-control hook, debug shim, mock, or synthetic pattern.
- **The canonical entry point is Go, not Python.** The Python `main()` refuses direct execution — proven below — so all behavior is exercised through the Go command.
- **Observation vehicles.** (1) The in-tree Go unit tests, which call the real unexported functions. (2) A *temporary* Go test (`kittens/ssh/blitzy_adhoc_test_obs_test.go`) that calls those same real functions (`connection_sharing_args`, `make_tarfile`, `get_remote_command` → `bootstrap_script` → `wrap_bootstrap_script`) and prints their real return values — this is an observation of the real functions, not a stand-in, and it was deleted after the investigation. (3) A *temporary* `/tmp` `ssh` shim placed only on `PATH` that logs the exact argv the kitten built, then `exec`s the real `ssh` — this captures the kitten's genuine output on the real code path. (4) A *temporary* Python driver that feeds the real `kittens.ssh.utils.get_ssh_data()` a real `kitty.shm` object. All temporary artifacts were removed afterward (see the final "Repository left unchanged" section).

---

## Build & Environment preamble

The current container was already built from source (canonical build: `CI=true python3 setup.py build`, which produces `kitty/launcher/{kitten,kitty}`, `kitty/fast_data_types.so`, and the build-time-generated Go files such as `kittens/ssh/conf_generated.go`). The Go `Config` struct used by the kitten exists **only** in those generated files — it is generated from the Python `Definition` in `kittens/ssh/main.py` [kittens/ssh/main.py:62-219] — so the build must precede any Go config-path observation.

**Command(s) run:**

```bash
git rev-parse HEAD
git rev-parse --abbrev-ref HEAD
go version
python3 --version
ssh -V
./kitty/launcher/kitten --version
./kitty/launcher/kitty +runpy 'import kitty.fast_data_types; print("fast_data_types OK")'
```

**Observed output (complete, unedited):**

```
### git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
### git rev-parse --abbrev-ref HEAD
blitzy-4f560dda-08e8-4b9f-9835-d97efce64779
### go version
go version go1.23.4 linux/amd64
### python3 --version
Python 3.13.7
### ssh -V
OpenSSH_10.0p2 Ubuntu-5ubuntu5.4, OpenSSL 3.5.3 16 Sep 2025
### kitten --version
kitten 0.35.2 created by Kovid Goyal
### fast_data_types import
fast_data_types OK
```

**What this shows + reasoning:**

- **[Observed]** HEAD is exactly `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`, so every `file:line` reference in this document resolves at the required commit. The checked-out working branch is the Blitzy work branch `blitzy-4f560dda-…`; the *source* branch from which this document is named is `kitty_815df1e210e0` (hence the filename). The commit — what matters for citations — matches exactly.
- **[Observed]** The toolchain is Go 1.23.4 and Python 3.13.7, and `kitty.fast_data_types` (the C extension providing `shm_open`/`shm_unlink` used by `kitty/shm.py` [kitty/shm.py:16]) imports successfully, so both the Go and Python code paths can run.
- **[Observed]** The OpenSSH client is `OpenSSH_10.0p2`. This is **≥ 8.4**, which — as detailed in Q9 — makes `SSHVersion.SupportsAskpassRequire()` return `true` [kittens/ssh/utils.go:206-207] and puts this environment on the **proactive (zero-roundtrip) data-request** branch. Wherever a value depends on the OpenSSH version, this branch is the one reported.

### The canonical entry point is Go; Python `main()` refuses to run

**Command(s) run:**

```bash
./kitty/launcher/kitty +runpy '
import sys, traceback
from kittens.ssh import main as m
try:
    m.main(["ssh", "localhost"])
except SystemExit as e:
    print("Caught SystemExit with message:", repr(str(e)))
    tb = traceback.extract_tb(sys.exc_info()[2]); last = tb[-1]
    print("Raised at:", last.filename.split("/")[-1] + ":" + str(last.lineno))
    print("Source line:", repr(last.line))
'
```

**Observed output (complete, unedited):**

```
Caught SystemExit with message: 'This should be run as kitten ssh'
Raised at: main.py:225
Source line: "raise SystemExit('This should be run as kitten ssh')"
```

**What this shows + reasoning:**

- **[Observed]** Invoking the Python `kittens.ssh.main.main()` directly raises `SystemExit('This should be run as kitten ssh')`, raised at `kittens/ssh/main.py:225` (function `def main` at [kittens/ssh/main.py:224]). The Python module is only an option-schema/server-helper layer; the real, canonical command is the Go `ssh.EntryPoint(root)` [tools/cmd/tool/main.go:50]. This is why every observation below is driven through the Go binary or the Go tests.

### The Go unit tests pass (canonical observation vehicles), stable across runs

**Command(s) run:**

```bash
go test -count=1 -v ./kittens/ssh/     # run twice
go test -count=1 ./tools/utils/shm/
```

**Observed output (complete, unedited) — first of two identical runs:**

```
=== RUN   TestSSHConfigParsing
--- PASS: TestSSHConfigParsing (0.01s)
=== RUN   TestCloneEnv
--- PASS: TestCloneEnv (0.00s)
=== RUN   TestSSHBootstrapScriptLimit
--- PASS: TestSSHBootstrapScriptLimit (0.02s)
=== RUN   TestSSHTarfile
--- PASS: TestSSHTarfile (0.02s)
=== RUN   TestGetSSHOptions
--- PASS: TestGetSSHOptions (0.00s)
=== RUN   TestParseSSHArgs
--- PASS: TestParseSSHArgs (0.00s)
=== RUN   TestRelevantKittyOpts
--- PASS: TestRelevantKittyOpts (0.00s)
PASS
ok  	kitty/kittens/ssh	0.059s
```

**What this shows + reasoning:**

- **[Observed]** All seven SSH-kitten Go tests pass, identically across two runs (the second run also reported `PASS` / `ok kitty/kittens/ssh`). `go test ./tools/utils/shm/` also passed (`ok kitty/tools/utils/shm`). These tests call the real `make_tarfile`, `get_remote_command`/`bootstrap_script`/`wrap_bootstrap_script`, config parsing, and arg-parsing functions, so they are legitimate runtime-observation vehicles for Q1–Q6.

---

## Q1. How does the SSH kitten set up a secure session and share connections?

**Direct answer:** `kitten ssh <host>` assembles an `ssh` command vector of the form `ssh [ssh_args…] -t [connection-sharing options] -- <hostname> exec <interpreter> -c <unwrap> <encoded-bootstrap>`. It **forces a controlling TTY** by appending `-t` when there are no remote args, and — because `share_connections` is on by default — it **injects OpenSSH connection-multiplexing options** produced by `connection_sharing_args()`: `ControlMaster=auto`, a per-kitty `ControlPath`, `ControlPersist=yes`, and a keepalive trio (`ServerAliveInterval=60`, `ServerAliveCountMax=5`, `TCPKeepAlive=no`). "Sharing" means multiple `kitten ssh` sessions to the same host reuse a single underlying SSH transport via an OpenSSH ControlMaster socket.

**Command(s) run** (real `kitten ssh localhost`, argv captured by a `/tmp` `ssh` shim that logs `"$@"` then execs the real ssh; the assembled command is invocation #2):

```bash
# /tmp/blitzy_shim/ssh logs argv (NUL-delimited) then: exec /usr/bin/ssh "$@"
export PATH=/tmp/blitzy_shim:$PATH
KITTY_PID=59881 KITTY_WINDOW_ID=1 \
  python3 /tmp/blitzy_shim/pty_run.py "$(pwd)/kitty/launcher/kitten" ssh localhost
# then decode the NUL-delimited log
```

**Observed output (complete, unedited) — the exact argv the kitten built:**

```
=== ssh invocation #1: argc=0 ===
=== ssh invocation #2: argc=20 ===
  argv[0]='-t'
  argv[1]='-o'
  argv[2]='ControlMaster=auto'
  argv[3]='-o'
  argv[4]='ControlPath=/root/.cache/kitty/run/kssh-59881-%C'
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
  argv[18]='\'eval "$(echo "$0" | tr \\\\\\v\\\\\\f\\\\\\r\\\\\\b \\\\\\047\\\\\\134\\\\\\n\\\\\\041)"\' '
  argv[19]="'#\x08/bin/sh\r# Copyright (C) 2022 Kovid Goyal <kovid at kovidgoyal.net>\r# Distributed under ...(+5117B)"
```

**Observed output (complete, unedited) — the raw options as returned by `connection_sharing_args()` called directly:**

```
OBS_CSA_PID=53286
OBS_CSA_COUNT=12
OBS_CSA[0]="-o"
OBS_CSA[1]="ControlMaster=auto"
OBS_CSA[2]="-o"
OBS_CSA[3]="ControlPath=/root/.cache/kitty/run/kssh-53286-%C"
OBS_CSA[4]="-o"
OBS_CSA[5]="ControlPersist=yes"
OBS_CSA[6]="-o"
OBS_CSA[7]="ServerAliveInterval=60"
OBS_CSA[8]="-o"
OBS_CSA[9]="ServerAliveCountMax=5"
OBS_CSA[10]="-o"
OBS_CSA[11]="TCPKeepAlive=no"
OBS_CSA_JOINED=-o ControlMaster=auto -o ControlPath=/root/.cache/kitty/run/kssh-53286-%C -o ControlPersist=yes -o ServerAliveInterval=60 -o ServerAliveCountMax=5 -o TCPKeepAlive=no
```

**What this shows + reasoning:**

- **[Observed] Command assembly.** `run_ssh()` [kittens/ssh/main.go:597] builds `cmd := append([]string{SSHExe()}, ssh_args...)` [main.go:606], takes `hostname := server_args[0]` [main.go:608], and appends `-- <hostname>` at [main.go:613]. In the captured argv, `--` is `argv[13]` and `localhost` is `argv[14]`, exactly as the code constructs. Invocation #1 (argc=0) is the kitten probing `ssh` with no args to learn its option list (`SSHOptions`, [kittens/ssh/utils.go:40]); `SSHExe()` resolves `ssh` via `utils.FindExe("ssh")` [kittens/ssh/utils.go:22-24], which is why the `/tmp` shim (first on `PATH`) is the genuine executable the kitten runs.
- **[Observed] Forced controlling TTY.** `argv[0]` is `-t`. The code appends `-t` only when there are no remote args: `if len(cd.remote_args) == 0 { cmd = append(cmd, "-t") }` [kittens/ssh/main.go:609-611]. Because `kitten ssh localhost` passes no remote command, `-t` is present — this forces OpenSSH to allocate a remote pseudo-TTY so the bootstrap script can read/write `/dev/tty`.
- **[Observed] Connection-sharing options.** `argv[1..12]` are exactly the six `-o` pairs returned by `connection_sharing_args()` [kittens/ssh/main.go:121]. They are inserted at `insertion_point` (the length of `cmd` before `--`) [main.go:612] via `cmd = slices.Insert(cmd, insertion_point, control_master_args...)` [main.go:646], guarded by `if host_opts.Share_connections` [main.go:637]. Enumerated by name and value:
  - `ControlMaster=auto` — enable multiplexing; the first connection becomes the master.
  - `ControlPath=/root/.cache/kitty/run/kssh-59881-%C` — the multiplex socket path. It comes from the template constant `kitty.SSHControlMasterTemplate = "kssh-{kitty_pid}-{ssh_placeholder}"` [kitty/constants.py:188; constants_generated.go:12], with `{kitty_pid}` → the kitty PID (`59881` here, from `KITTY_PID`) and `{ssh_placeholder}` → `%C` (OpenSSH's hash of the connection parameters) [kittens/ssh/main.go:135-136]. `%C` lets one master be reused per unique (host, port, user…) tuple.
  - `ControlPersist=yes` — keep the master alive after the initial client exits, so later sessions can attach.
  - `ServerAliveInterval=60`, `ServerAliveCountMax=5`, `TCPKeepAlive=no` — the keepalive trio that detects dead peers without relying on TCP keepalives.
- **[Observed] The runtime directory and the macOS symlink workaround.** Here the `ControlPath` directory is `/root/.cache/kitty/run` (26 characters), which is **≤ 35**, so the symlink workaround is **not** triggered. The workaround exists for the opposite case: `if len(rd) > 35 { … AtomicCreateSymlink(rd, "/tmp/kssh-rdir-<euid>"); rd = … }` [kittens/ssh/main.go:128-134]. The in-code comment explains why [main.go:123-127]: OpenSSH turns the `ControlPath` into an ~40-char hash plus a ~27-char temp suffix, the socket path max is ~104 chars, and on macOS the cache dir path is already ~48 chars — so long runtime dirs are redirected through a short `/tmp/kssh-rdir-<euid>` symlink to stay under the socket path-length limit. **[Inferred]** that this environment (Linux, short cache path) never exercises the workaround — grounded in the observed 26-char path and the `> 35` guard.
- **[Observed] Rest of the command is the wrapped bootstrap** (`argv[15..19]` = `exec sh -c <unwrap> <encoded>`); see Q2/Q6.
- **Corroboration.** The authoritative narrative states the ssh kitten shares connections "Under the hood … using SSH ControlMasters" and reads setup data over the TTY [docs/kittens/ssh.rst:131-147]. The observed `ControlMaster/ControlPath/ControlPersist` options match that narrative; I treat the verified `file:line` at `815df1e21` as ground truth and the docs prose only as corroboration.

---

## Q2. It uses shared memory to pass credentials, then generates bootstrap scripts that run on the remote — how does this work?

**Direct answer:** Before launching `ssh`, the kitten generates a **random one-time password** with `pw, _ = secrets.TokenHex()` (32 crypto-random bytes → 64 hex chars) and builds a JSON payload `{tarfile, pw, hostname, username}`. It writes that JSON into a freshly created **POSIX shared-memory object** named `kssh-<pid>-<random>` (owner-only `0o600`), recording the object's name in `cd.shm_name`. It then selects the remote bootstrap **from the in-tree files** `shell-integration/ssh/bootstrap.sh` **or** `shell-integration/ssh/bootstrap.py` (chosen by interpreter), substitutes placeholders (including the request id, password, and shm filename), and hands the wrapped script to `ssh` to execute on the remote. The password is *not* sent inside the tarball — it lives in shared memory on localhost and is only released after the remote proves it knows the same password over the TTY (Q8/Q9).

**Command(s) run** (real `bootstrap_script()` via `get_remote_command()`, executed twice to show the password varies while its length stays fixed):

```bash
go test -count=1 -v -run TestBlitzyObsBootstrapSh ./kittens/ssh/   # calls the real get_remote_command()
```

**Observed output (complete, unedited):**

```
OBS_SH_RUN=1
OBS_SH_SCRIPT_TYPE=sh
OBS_SH_INTERPRETER="sh"
OBS_SH_RCMD_LEN=5
OBS_SH_RCMD_TOTAL_BYTES=5282 (guard=9000)
OBS_SH_PW="6916bad18328abb90b92e075686c369cf17a67531d3b2c56f4e66045fcfb4392" (len=64)
OBS_SH_RCMD[0]="exec"
OBS_SH_RCMD[1]="sh"
OBS_SH_RCMD[2]="-c"
OBS_SH_RCMD[3_UNWRAP]="'eval \"$(echo \"$0\" | tr \\\\\\v\\\\\\f\\\\\\r\\\\\\b \\\\\\047\\\\\\134\\\\\\n\\\\\\041)\"' "
OBS_SH_RCMD[4]_FIRST_120_VISUALIZED='#<\b=BS/0x08>/bin/sh<\r=CR/0x0d># Copyright (C) 2022 Kovid Goyal <kovid at kovidgoyal.net><\r=CR/0x0d># Distributed under terms of the GPLv3 license.<\r=CR/0x0d><\r=CR/0x0d>{
OBS_SH_SUBST_HAS_VT_0x0b=true
OBS_SH_SUBST_HAS_FF_0x0c=true
OBS_SH_SUBST_HAS_CR_0x0d=true
OBS_SH_SUBST_HAS_BS_0x08=true
OBS_SH_ENCODED_STARTS_SQUOTE=true ENDS_SQUOTE=true
OBS_SH_RUN=2
OBS_SH_SCRIPT_TYPE=sh
OBS_SH_INTERPRETER="sh"
OBS_SH_RCMD_LEN=5
OBS_SH_RCMD_TOTAL_BYTES=5282 (guard=9000)
OBS_SH_PW="12942ebf9a4674546a79ea2776ed8623dd0842b49a53b2145482678237edc189" (len=64)
OBS_SH_RCMD[0]="exec"
OBS_SH_RCMD[1]="sh"
OBS_SH_RCMD[2]="-c"
OBS_SH_RCMD[3_UNWRAP]="'eval \"$(echo \"$0\" | tr \\\\\\v\\\\\\f\\\\\\r\\\\\\b \\\\\\047\\\\\\134\\\\\\n\\\\\\041)\"' "
OBS_SH_RCMD[4]_FIRST_120_VISUALIZED='#<\b=BS/0x08>/bin/sh<\r=CR/0x0d># Copyright (C) 2022 Kovid Goyal <kovid at kovidgoyal.net><\r=CR/0x0d># Distributed under terms of the GPLv3 license.<\r=CR/0x0d><\r=CR/0x0d>{
OBS_SH_SUBST_HAS_VT_0x0b=true
OBS_SH_SUBST_HAS_FF_0x0c=true
OBS_SH_SUBST_HAS_CR_0x0d=true
OBS_SH_SUBST_HAS_BS_0x08=true
OBS_SH_ENCODED_STARTS_SQUOTE=true ENDS_SQUOTE=true
```

**Observed output (complete, unedited) — the shm object name pattern and permissions, from the real `create_shared_memory()` + `os.stat`:**

```
OBS_SHM_NAME=/kssh-62797-b9ea027e19afece8d6ef14289263d19f2155d17b130620dc32d156763c422037
OBS_SHM_PATH=/dev/shm//kssh-62797-b9ea027e19afece8d6ef14289263d19f2155d17b130620dc32d156763c422037
OBS_SHM_MODE=0o600
OBS_SHM_UID=0 EUID=0  match=True
```

**Observed output (complete, unedited) — the one-time password derives from 32 crypto-random bytes:**

```
const DEFAULT_NUM_OF_BYTES_FOR_TOKEN = 32
```

**What this shows + reasoning:**

- **[Observed] One-time password.** `bootstrap_script()` [kittens/ssh/main.go:422] calls `pw, err := secrets.TokenHex()` [main.go:431]. `TokenHex` hex-encodes `TokenBytes`, which defaults to `DEFAULT_NUM_OF_BYTES_FOR_TOKEN = 32` bytes read from `crypto/rand` [tools/utils/secrets/tokens.go:13,15-25,28-34]. Hence every observed password is exactly **64 hex characters**, and its **value varies each run** (`6916bad…4392` vs `12942ebf…c189`, plus three further distinct 64-char values captured over the TTY in Q9). The fixed length with a varying value is exactly the "one-time password" property — reported here as the run-to-run distribution across five captured samples (all length 64, all distinct).
- **[Observed] Payload + shared memory.** The payload map is `{"tarfile": base64(tar), "pw": pw, "hostname": …, "username": …}` [kittens/ssh/main.go:439-443] (`tarfile` base64 at [main.go:440]); the tar itself is built by `make_tarfile()` (Q3). The JSON is written to a POSIX shm object created with `shm.CreateTemp(fmt.Sprintf("kssh-%d-", os.Getpid()), …)` [main.go:446], then `shm.WriteWithSize` [main.go:448] and `data_shm.Flush()` [main.go:450]; the resulting name is recorded in `cd.shm_name = data_shm.Name()` [main.go:458]. The observed shm object `/kssh-62797-b9ea027e…` matches the `kssh-<pid>-<random>` pattern and has mode **`0o600`** owned by the invoking uid — the security basis analyzed in Q8. (The Python equivalent `create_shared_memory()` [kittens/ssh/utils.py:88-97], used here to observe the object, produces the same shape.)
- **[Observed] Correlation id.** `request_id` is set to `KITTY_PID + "-" + KITTY_WINDOW_ID` [kittens/ssh/main.go:424] and threaded into the sensitive replacements `{REQUEST_ID, DATA_PASSWORD, PASSWORD_FILENAME}` [main.go:460]. This is the id the remote echoes back and that kitty verifies (Q8/Q9).
- **[Observed] Both bootstrap templates exist and are selected by interpreter.** The template is read from `shell_integration.Data()["shell-integration/ssh/bootstrap."+cd.script_type]` [main.go:481] and placeholder-substituted by `prepare_script()` [main.go:482]. `get_remote_command()` sets the type: `is_python := strings.Contains(strings.ToLower(path.Base(interpreter)), "python")` [main.go:514]; `cd.script_type = "sh"` [main.go:515] unless python, in which case `"py"` [main.go:517]. The observed default run used `bootstrap.sh` (`OBS_SH_SCRIPT_TYPE=sh`, interpreter `sh`); Q6 shows the `bootstrap.py` variant. The captured `RCMD[4]` begins with the real `bootstrap.sh` header (`#!/bin/sh` → the `!` is substituted, see Q6), confirming the in-tree file is what gets shipped.
- **[Observed] Size.** The full wrapped `sh` command totals **5282 bytes** in both runs — stable, and under the 9000-byte guard (Q6).

---

## Q3. How does the archive with all the shell-integration stuff get built and sent over?

**Direct answer:** `make_tarfile()` builds a **gzip-compressed PAX/USTAR tar** (gzip magic `1f 8b`, `gzip.BestCompression`) containing: `data.sh` (env/config), `bootstrap-utils.sh`, the terminfo files (`home/.terminfo/kitty.terminfo` and `home/.terminfo/x/xterm-kitty`), the shell-integration files for bash/fish/zsh under `home/<remote_dir>/shell-integration/…`, and — because the default `remote_kitty` is `if-needed` — the **executable** remote kitty/kitten binaries under `home/<remote_dir>/kitty/bin/`. That tar is Base64-encoded into the shm payload's `tarfile` field and later streamed over the TTY in 254-byte chunks (Q9). Crucially, the `shell-integration/ssh/` bootstrap files themselves are **excluded** from the tar because they are shipped as command-line arguments instead.

**Command(s) run** (real `make_tarfile()` output, extracted and enumerated by the temporary observation test):

```bash
go test -count=1 -v -run TestBlitzyObsTarfileMembers ./kittens/ssh/
```

**Observed output (complete, unedited):**

```
OBS_TAR_TOTAL_BYTES=24023
OBS_TAR_MAGIC=1f 8b
OBS_TAR_FORMAT_AND_MEMBERS:
OBS_TAR_MEMBER mode=0644 size=195 format=PAX name=data.sh
OBS_TAR_MEMBER mode=0644 size=8468 format=PAX name=bootstrap-utils.sh
OBS_TAR_MEMBER mode=0644 size=280 format=USTAR name=home/.local/share/kitty-ssh-kitten/shell-integration/zsh/completions/_kitty
OBS_TAR_MEMBER mode=0644 size=17363 format=USTAR name=home/.local/share/kitty-ssh-kitten/shell-integration/bash/kitty.bash
OBS_TAR_MEMBER mode=0644 size=10409 format=USTAR name=home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish
OBS_TAR_MEMBER mode=0644 size=1880 format=USTAR name=home/.local/share/kitty-ssh-kitten/shell-integration/zsh/.zshenv
OBS_TAR_MEMBER mode=0644 size=22557 format=USTAR name=home/.local/share/kitty-ssh-kitten/shell-integration/zsh/kitty-integration
OBS_TAR_MEMBER mode=0644 size=286 format=USTAR name=home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_completions.d/kitten.fish
OBS_TAR_MEMBER mode=0644 size=285 format=USTAR name=home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_completions.d/kitty.fish
OBS_TAR_MEMBER mode=0644 size=294 format=USTAR name=home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_completions.d/clone-in-kitty.fish
OBS_TAR_MEMBER mode=0644 size=6 format=PAX name=home/.local/share/kitty-ssh-kitten/kitty/version
OBS_TAR_MEMBER mode=0755 size=4377 format=USTAR name=home/.local/share/kitty-ssh-kitten/kitty/bin/kitty
OBS_TAR_MEMBER mode=0755 size=2761 format=USTAR name=home/.local/share/kitty-ssh-kitten/kitty/bin/kitten
OBS_TAR_MEMBER mode=0644 size=4271 format=USTAR name=home/.terminfo/kitty.terminfo
OBS_TAR_MEMBER mode=0644 size=3711 format=USTAR name=home/.terminfo/x/xterm-kitty
OBS_TAR_REMOTE_KITTY_OPT=if-needed
OBS_TAR_REMOTE_DIR=".local/share/kitty-ssh-kitten"
```

**Observed output (complete, unedited) — gzip size across 8 runs (magnitude is *not* stable):**

```
run1: 23597
run2: 23616
run3: 23495
run4: 23588
run5: 23537
run6: 23577
run7: 24020
run8: 23470
```

**What this shows + reasoning:**

- **[Observed] Format & compression.** The first two bytes are `1f 8b` (gzip). `make_tarfile()` wraps the tar writer in `gzip.NewWriterLevel(&w, gzip.BestCompression)` [kittens/ssh/main.go:259], and every header is written with `Format: tar.FormatPAX` [main.go:297,310]. The Go `archive/tar` writer downgrades an entry to USTAR when PAX extensions are unnecessary, which is why the extracted members show a mix of `PAX` (e.g., `data.sh`, `bootstrap-utils.sh`, `kitty/version`) and `USTAR` (the copied files) — both are valid within one PAX archive.
- **[Observed] Contents, by name.** `data.sh` (the serialized env/config), `bootstrap-utils.sh`, the terminfo `home/.terminfo/kitty.terminfo` and `home/.terminfo/x/xterm-kitty` (the default `TERM` name is `xterm-kitty`), the shell-integration payloads for bash/fish/zsh under `home/.local/share/kitty-ssh-kitten/shell-integration/…`, and the remote kitty binaries.
- **[Observed] Remote binaries & the `remote_kitty` gate.** `home/.local/share/kitty-ssh-kitten/kitty/bin/kitty` and `…/kitten` are present with mode **`0755`** (executable). They are added because `cd.host_opts.Remote_kitty != Remote_kitty_no` [kittens/ssh/main.go:342]; the observed option value is the default `if-needed` (`OBS_TAR_REMOTE_KITTY_OPT=if-needed`), and `Remote_dir` is the default `.local/share/kitty-ssh-kitten`. `home/.local/share/kitty-ssh-kitten/kitty/version` (6 bytes) carries the version string.
- **[Observed] The `shell-integration/ssh/` exclusion (named item).** No tar member is under `shell-integration/ssh/`. This is deliberate: `make_tarfile()` selects files with `shell_integration.Data().FilesMatching("shell-integration/", "shell-integration/ssh/.+", "shell-integration/zsh/kitty.zsh")` [kittens/ssh/main.go:329-333], where the latter two patterns are **exclusions**. The in-code comments say it directly: `// bootstrap files are sent as command line args` and `// backward compat file not needed by ssh kitten` [main.go:331-332]. The shipped Go test `TestSSHTarfile` asserts the absence: it fatals with "Contents of shell-integration/ssh not excluded" if `home/<remote_dir>/shell-integration/ssh/kitten` appears [kittens/ssh/main_test.go:153-155]. **Why:** the bootstrap (`bootstrap.sh`/`bootstrap.py`) is the very thing `ssh` executes as its command-line argument (Q2/Q6), so shipping it inside the tarball it bootstraps would be circular; the `zsh/kitty.zsh` file is a legacy artifact the ssh kitten does not need.
- **[Observed] Transport (cross-link to Q9).** The tar bytes are Base64-encoded into the shm `tarfile` field [kittens/ssh/main.go:440] and later emitted by the kitty-core responder `get_ssh_data()` in 254-byte chunks between `KITTY_DATA_START`/`KITTY_DATA_END` [kittens/ssh/utils.py:139-148]; the remote `get_data()` reads that stream [shell-integration/ssh/bootstrap.sh:137-152].
- **[Observed] Magnitude is not run-to-run stable — reported as a distribution.** Over 8 runs the gzip size ranged **23470–24020 bytes** (the single earlier sample was 24023). The **uncompressed** member sizes were identical across runs (e.g., `bootstrap-utils.sh`=8468, `bash/kitty.bash`=17363), so the fluctuation is in the compressed stream. **Cause [Observed/code-grounded]:** the synthesized entries stamp the current time — `now := time.Now()` [main.go:292] used as `ModTime/ChangeTime/AccessTime` [main.go:298] — into PAX extended headers with nanosecond precision, so a few header bytes differ every run and `gzip.BestCompression` yields a slightly different length. The number of 254-byte transport chunks therefore also varies slightly (≈124–127; see Q9).

---

## Q4. How does the kitten keep track of everything it needs for a connection (the connection data structure/state)?

**Direct answer:** The kitten carries a single mutable Go struct, **`connection_data`** [kittens/ssh/main.go:171-189], through the entire flow. It holds the parsed host options (`host_opts *Config` — the build-time-generated config struct), the target `hostname_for_match`/`username`, flags such as `request_data`/`echo_on`/`dont_create_shm`, and the products of setup (`shm_name`, `script_type`, `rcmd`, `replacements`, `request_id`, `bootstrap_script`). Separately, kitty-core models a *parsed ssh command line* with the **`SSHConnectionData` NamedTuple** [kitty/utils.py:953-957] (`binary, hostname, port, identity_file, extra_args`), produced by `get_connection_data()` [kittens/ssh/utils.py:258]. The two are distinct: `connection_data` is the kitten's runtime working state; `SSHConnectionData` is a lightweight descriptor of an ssh invocation.

**Command(s) run** (populate the real struct via the real `get_remote_command()` and print its fields):

```bash
go test -count=1 -v -run TestBlitzyObsConnectionDataStruct ./kittens/ssh/
```

**Observed output (complete, unedited):**

```
OBS_CD_hostname_for_match="host.test"
OBS_CD_username="testuser"
OBS_CD_script_type="sh"
OBS_CD_request_id="123-123"
OBS_CD_request_data=false
OBS_CD_echo_on=false
OBS_CD_dont_create_shm=true
OBS_CD_shm_name=""
OBS_CD_remote_args=[]
OBS_CD_host_opts_type=*ssh.Config
OBS_CD_host_opts_Share_connections=true
OBS_CD_replacements_keys_count=8
OBS_CD_bootstrap_script_len=5205
```

**What this shows + reasoning:**

- **[Observed] The `connection_data` struct (named item).** Declared at `type connection_data struct` [kittens/ssh/main.go:171] and spanning **lines 171–189** (note: the AAP loosely cited ~176-197; the *actual* verified range is 171–189). Its 16 fields are, in order: `remote_args []string`, `host_opts *Config`, `hostname_for_match string`, `username string`, `echo_on bool`, `request_data bool`, `literal_env map[string]string`, `listen_on string`, `test_script string`, `dont_create_shm bool`, `shm_name string`, `script_type string`, `rcmd []string`, `replacements map[string]string`, `request_id string`, `bootstrap_script string`. The observed values match a populated instance: `hostname_for_match="host.test"`, `username="testuser"`, `script_type="sh"`, `request_id="123-123"`, `remote_args=[]`. Because this observation used `dont_create_shm=true`, `shm_name` is empty (no shm object was created for this print); in a real run `shm_name` holds the `kssh-<pid>-…` name (Q2).
- **[Observed] `host_opts` is the generated `*Config`.** `OBS_CD_host_opts_type=*ssh.Config` confirms `host_opts *Config` [main.go:173] points at the build-time-generated struct, and `Share_connections=true` is its default. There is no committed `type Config struct` in `kittens/ssh/*.go`; it is generated from the Python `Definition` in `kittens/ssh/main.py` [kittens/ssh/main.py:62-219] into `kittens/ssh/conf_generated.go` — which is precisely why the environment must be built before this field can be exercised. `replacements` has 8 keys and `bootstrap_script` is 5205 bytes for this default config.
- **[Observed] The kitty-core `SSHConnectionData` NamedTuple (named item).** `class SSHConnectionData(NamedTuple)` [kitty/utils.py:953] has fields `binary`, `hostname`, `port: Optional[int]`, `identity_file: str`, `extra_args: Tuple[Tuple[str,str],…]` [kitty/utils.py:954-957]. `get_connection_data()` [kittens/ssh/utils.py:258] parses these from an ssh command line, and its round-trip is covered by the passing `test_ssh_connection_data` integration test (seen in the `./test.py --module ssh` output in Q7). This descriptor is what kitty uses to recognize/relaunch an ssh command line; it is separate from the kitten's `connection_data` working state.

---

## Q5. Connection reuse: how does it decide whether to start a fresh connection or piggyback on an existing one (SSH ControlMaster / connection sharing)?

**Direct answer:** The kitten probes for a live master with **`ssh -O check`** (built by inserting `-O check` right after the ssh binary). If a master is alive **and** `share_connections` is on **and** the kitten still `need_to_request_data`, it sets `need_to_request_data = false`, which **skips the remote's TTY data-request round-trip** and piggybacks on the existing multiplexed connection. The concrete signal `master_is_functional()` keys on is the exit status of `ssh -O check`: **0** ⇒ master alive (reuse), **non-zero** ⇒ no master (fresh). One nuance: on OpenSSH **≥ 8.4** (this environment), `need_to_request_data` is already `false` from the version gate, so the reuse-gate's `ssh -O check` is *short-circuited*; the check governs the < 8.4 path (or when kitty's askpass is not used). I demonstrate both the raw `ssh -O check` mechanism and its effect on the kitten below.

**Command(s) run** (1 — the raw mechanism `master_is_functional()` relies on):

```bash
# dead master (no socket):
ssh -o ControlPath=/root/.cache/kitty/run/kssh-test-%C -O check -- localhost; echo "exit_code(dead master)=$?"
# start a real master, then check it is alive:
ssh -o ControlMaster=yes -o ControlPath=/root/.cache/kitty/run/kssh-test-%C -o ControlPersist=yes -N -f -- localhost
ssh -o ControlPath=/root/.cache/kitty/run/kssh-test-%C -O check -- localhost; echo "exit_code(alive master)=$?"
```

**Observed output (complete, unedited):**

```
exit_code(dead master)=255
exit_code(alive master)=0
```
```
Control socket connect(/root/.cache/kitty/run/kssh-test-c8f182fbe3c99beff0a72e1a67e4462fe447f9e0): No such file or directory
Master running (pid=60985)
Exit request sent.
```

**Command(s) run** (2 — the real kitten invoking `ssh -O check`, forced onto the reuse path by setting `SSH_ASKPASS` so `use_kitty_askpass` is false and `need_to_request_data` stays true):

```bash
export PATH=/tmp/blitzy_shim:$PATH   # argv-logging shim, then exec real ssh
KITTY_PID=59881 KITTY_WINDOW_ID=1 SSH_ASKPASS=/usr/bin/true \
  python3 /tmp/blitzy_shim/pty_run.py "$(pwd)/kitty/launcher/kitten" ssh localhost
```

**Observed output (complete, unedited) — the kitten's `master_is_functional()` `check_cmd`:**

```
inv#2 (master_is_functional check_cmd):
  argv[0]='-O'
  argv[1]='check'
  argv[2]='-t'
  argv[3]='-o'
  argv[4]='ControlMaster=auto'
  argv[5]='-o'
  argv[6]='ControlPath=/root/.cache/kitty/run/kssh-59881-%C'
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

**Observed output (complete, unedited) — the transition, read from the `request_data` value baked into the generated bootstrap of each run:**

```
Q5 NO-MASTER run  -> bootstrap request_data = 1  (expect 1 = REMOTE asks)
Q5 REUSE run      -> bootstrap request_data = 0  (expect 0 = LOCAL proactive, round-trip skipped)
Default run (>=8.4)-> bootstrap request_data = 0  (expect 0 = LOCAL proactive)
```

**What this shows + reasoning:**

- **[Observed] `ssh -O check` (named item) and the exact reuse signal.** `master_is_functional()` is a closure that builds `check_cmd := slices.Insert(cmd, 1, "-O", "check")` [kittens/ssh/main.go:658] and evaluates `master_is_alive = exec.Command(check_cmd[0], check_cmd[1:]...).Run() == nil` [main.go:659]. The captured `check_cmd` is exactly `ssh -O check -t -o ControlMaster=auto … -- localhost` — `-O check` inserted at position 1, matching `slices.Insert(cmd, 1, …)`. The raw mechanism run shows the return values the closure keys on: **255** ("Control socket connect(…): No such file or directory") for a dead master vs **0** ("Master running (pid=60985)") for a live one.
- **[Observed] `need_to_request_data` (named item) and the reuse gate.** The decision is `if need_to_request_data && host_opts.Share_connections && master_is_functional() { need_to_request_data = false }` [kittens/ssh/main.go:663-664]. Setting `need_to_request_data=false` means the kitten will send the data request **itself** rather than waiting for the remote to ask — i.e., it reuses the live connection and skips the extra TTY round-trip. This value flows into `cd.request_data = need_to_request_data` [main.go:724], which becomes the `REQUEST_DATA` replacement in the bootstrap (`add_bool(cd.request_data, "REQUEST_DATA")` [main.go:473]).
- **[Observed] The transition (before/after states).** Reading the `request_data=` value the kitten actually wrote into the generated remote script proves the flip: with **no master** alive the script has `request_data=1` (the **remote** issues the DCS request), whereas with a **pre-started master** at the kitten's own `ControlPath` the script has `request_data=0` (the **local** side sends the request proactively, and the round-trip is skipped). This is the observable reuse decision.
- **[Observed] OpenSSH ≥ 8.4 nuance.** In the *default* run on this OpenSSH 10.0 host, the bootstrap also has `request_data=0`, but for a different reason: `set_askpass()` returns `false` because `GetSSHVersion().SupportsAskpassRequire()` is true [kittens/ssh/main.go:152-156], so `need_to_request_data` is already `false` at the gate [main.go:663] and Go short-circuits `master_is_functional()` — no `ssh -O check` runs. That is why the default capture in Q1 showed **no** `-O check` invocation, while the `SSH_ASKPASS`-forced run here does. The reuse gate's master probe therefore matters chiefly for OpenSSH < 8.4 (or when kitty's native askpass is disabled). `run_control_master()` (which can *start* a master with `-N -f` [main.go:666-670]) is used on the `forward_remote_control` path [main.go:681-690].
- **[Observed] Lifecycle cleanup (named item `close_shared_ssh_connections`).** The shared masters are torn down by `def close_shared_ssh_connections(self)` [kitty/boss.py:3013]; kitty invokes this to close the persistent ControlMaster sockets (e.g., on quit / on demand), matching the docs' note that the shared connections are cleaned up when kitty exits [docs/kittens/ssh.rst:131-147]. **[Inferred]** the exact trigger points (quit vs. explicit action) from the method's role; the method's existence and name are observed at the cited line.

---

## Q6. The bootstrap script encoding with character substitutions for different shells — how does that work?

**Direct answer:** `wrap_bootstrap_script()` wraps the bootstrap into the form `interpreter -c <unwrap_script> <encoded_script>` so a non-POSIX login shell can still run it (sshd joins the argv with spaces and passes it to the user's login shell with `-c`). There are two variants keyed by `script_type`:
- **`py` (Python interpreter):** `encoded_script` is plain **Base64**, and `unwrap_script` decodes+`eval(compile(...))`s it.
- **`sh` (POSIX shell, the default):** Base64 may be unavailable, so `encoded_script` is the raw script with four **character substitutions** — `'`→`\v` (0x0b), `\`→`\f` (0x0c), `\n`→`\r` (0x0d), `!`→`\b` (0x08) (the last two for `tcsh`) — wrapped in single quotes, and `unwrap_script` reverses them on the remote with `tr`.

**Command(s) run:**

```bash
go test -count=1 -v -run TestBlitzyObsBootstrapSh ./kittens/ssh/   # sh path (default interpreter)
go test -count=1 -v -run TestBlitzyObsBootstrapPy ./kittens/ssh/   # py path (interpreter=/usr/bin/python3, config override)
```

**Observed output (complete, unedited) — sh path (default), key fields (full two-run output shown in Q2):**

```
OBS_SH_SCRIPT_TYPE=sh
OBS_SH_INTERPRETER="sh"
OBS_SH_RCMD_LEN=5
OBS_SH_RCMD_TOTAL_BYTES=5282 (guard=9000)
OBS_SH_RCMD[0]="exec"
OBS_SH_RCMD[1]="sh"
OBS_SH_RCMD[2]="-c"
OBS_SH_RCMD[3_UNWRAP]="'eval \"$(echo \"$0\" | tr \\\\\\v\\\\\\f\\\\\\r\\\\\\b \\\\\\047\\\\\\134\\\\\\n\\\\\\041)\"' "
OBS_SH_RCMD[4]_FIRST_120_VISUALIZED='#<\b=BS/0x08>/bin/sh<\r=CR/0x0d># Copyright (C) 2022 Kovid Goyal <kovid at kovidgoyal.net><\r=CR/0x0d># Distributed under terms of the GPLv3 license.<\r=CR/0x0d><\r=CR/0x0d>{
OBS_SH_SUBST_HAS_VT_0x0b=true
OBS_SH_SUBST_HAS_FF_0x0c=true
OBS_SH_SUBST_HAS_CR_0x0d=true
OBS_SH_SUBST_HAS_BS_0x08=true
OBS_SH_ENCODED_STARTS_SQUOTE=true ENDS_SQUOTE=true
```

**Observed output (complete, unedited) — py path (interpreter override):**

```
OBS_PY_SCRIPT_TYPE=py
OBS_PY_INTERPRETER="/usr/bin/python3"
OBS_PY_RCMD_LEN=5
OBS_PY_RCMD_TOTAL_BYTES=13606 (guard=9000)
OBS_PY_RCMD[0]="exec"
OBS_PY_RCMD[1]="/usr/bin/python3"
OBS_PY_RCMD[2]="-c"
OBS_PY_RCMD[3_UNWRAP]="\"import base64, sys; eval(compile(base64.standard_b64decode(sys.argv[-1]), 'bootstrap.py', 'exec'))\""
OBS_PY_RCMD[4]_FIRST_80="IyEvdXNyL2Jpbi9lbnYgcHl0aG9uCiMgTGljZW5zZTogR1BMdjMgQ29weXJpZ2h0OiAyMDIyLCBLb3Zp"
OBS_PY_ENCODED_IS_BASE64_ONLY=true
```

**Observed output (complete, unedited) — proof the py `encoded_script` is Base64 of the real `bootstrap.py`:**

```
$ echo "IyEvdXNyL2Jpbi9lbnYgcHl0aG9uCiMgTGljZW5zZTogR1BMdjMgQ29weXJpZ2h0OiAyMDIyLCBLb3Zp" | base64 -d
#!/usr/bin/env python
# License: GPLv3 Copyright: 2022, Kovi
```

**What this shows + reasoning:**

- **[Observed] The wrapper shape and its rationale.** `wrap_bootstrap_script()` [kittens/ssh/main.go:486] produces `cd.rcmd = []string{"exec", cd.host_opts.Interpreter, "-c", unwrap_script, encoded_script}` [main.go:508] — exactly the 5-element `rcmd` observed for both interpreters. The in-code comment explains why the command must be this simple: "sshd will execute the command … by join[ing] all command line arguments with a space and passing it as a single argument to the users login shell with -c" and a non-POSIX shell "might have different escaping semantics" [main.go:487-494].
- **[Observed] The two interpreters (cross-product, named items).** `get_remote_command()` picks the branch via `is_python := strings.Contains(strings.ToLower(path.Base(interpreter)), "python")` [main.go:514] → `script_type` `"py"` [main.go:517] or `"sh"` [main.go:515]. The default interpreter `sh` yields `script_type=sh`; overriding to `/usr/bin/python3` (via the `interpreter=` config override on the kitten's own command line, not by editing source) yields `script_type=py`.
- **[Observed] py path.** `encoded_script = base64.StdEncoding.EncodeToString(…)` [main.go:498] and `unwrap_script = "import base64, sys; eval(compile(base64.standard_b64decode(sys.argv[-1]), 'bootstrap.py', 'exec'))"` [main.go:499]. The observed `RCMD[3]` matches that unwrap string verbatim, `RCMD[4]` is Base64-only (`OBS_PY_ENCODED_IS_BASE64_ONLY=true`), and decoding its first bytes yields the real `bootstrap.py` header `#!/usr/bin/env python` — confirming the in-tree `shell-integration/ssh/bootstrap.py` is what is encoded.
- **[Observed] sh path — the four substitutions (named items).** `encoded_script = "'" + strings.NewReplacer("'", "\v", "\\", "\f", "\n", "\r", "!", "\b").Replace(cd.bootstrap_script) + "'"` [main.go:505]. All four replacement bytes are present in the real encoded script (`HAS_VT_0x0b`, `HAS_FF_0x0c`, `HAS_CR_0x0d`, `HAS_BS_0x08` all true), and it is single-quote wrapped (starts and ends with `'`). The visualized prefix makes two substitutions visible directly: the shebang `#!/bin/sh` shows as `#<BS>/bin/sh` (the `!` → `\b`), and every newline shows as `<CR>` (`\n` → `\r`). Enumerated:
  - `'` → `\v` (vertical tab, 0x0b)
  - `\` → `\f` (form feed, 0x0c)
  - `\n` → `\r` (carriage return, 0x0d)
  - `!` → `\b` (backspace, 0x08) — needed because `tcsh` treats `!` (history) and newlines specially.
- **[Observed] The `tr`-based remote unwrap that reverses them.** `unwrap_script = 'eval "$(echo "$0" | tr \\\v\\\f\\\r\\\b \\\047\\\134\\\n\\\041)"' ` [main.go:506], matching the observed `RCMD[3]`. The `tr` maps the four substituted bytes back to their originals: `\v`→`\047` (`'`), `\f`→`\134` (`\`), `\r`→`\n` (newline), `\b`→`\041` (`!`) — the exact inverse of the four `NewReplacer` pairs. So the remote shell `eval`s the reconstituted original script.
- **[Observed] Size vs the 9000-byte guard (magnitude).** The default `sh` wrapped command is **5282 bytes**, stable across two runs, comfortably under the 9000-byte guard that `TestSSHBootstrapScriptLimit` enforces (`if total > 9000 { t.Fatalf(...) }` [kittens/ssh/main_test.go:76-78]). The `py` variant is **13606 bytes**, which *exceeds* 9000 — but that guard test only exercises the **default** `sh` interpreter (via `basic_connection_data()` [main_test.go:49-64], which leaves `script_type="sh"`); the larger `py` figure comes from a non-default `interpreter=python3` override (Base64 inflates the larger `bootstrap.py` by ~33%). This is reported as observed, with the nuance that the shipped guard does not cover the py path.

---

## Q7. Trace end-to-end: from when a user initiates an SSH session all the way to when the bootstrap executes on the remote side.

**Direct answer:** the full flow is: `kitten ssh localhost` → `EntryPoint` parses args and resolves per-host config → `make_tarfile()` builds the gzip PAX tarball → `bootstrap_script()` generates a one-time password, writes `{tarfile,pw,hostname,username}` to a `0o600` POSIX shm object, and renders the remote bootstrap from `shell-integration/ssh/bootstrap.sh` → `wrap_bootstrap_script()` encodes it as `exec sh -c <unwrap> <encoded>` → `run_ssh()` assembles the `ssh` argv (forcing `-t`, injecting ControlMaster options), and — because OpenSSH ≥ 8.4 — **proactively** writes the `@kitty-ssh` DCS request to the TTY → `exec`s `ssh` → on the remote, `bootstrap.sh` runs `get_data()`, which (since `request_data="0"`) waits for the `KITTY_DATA_START`/`OK`/Base64/`KITTY_DATA_END` reply, unpacks it, compiles terminfo, stages files, then `exec_login_shell`.

**Command(s) run:** (reuses the real captured argv from the Q1 `PATH`-shim run of `kitten ssh localhost`, plus a decode of the encoded remote script)

```bash
# 1. real assembled ssh argv (NUL-delimited) captured by the /tmp PATH shim
python3 - <<'PY'
data=open('/tmp/blitzy_obs/ssh_argv_capture.log','rb').read()
seg=data.split(b'\n===END-INVOCATION===\n')[1]
for j,a in enumerate(seg.split(b'\x00')):
    print(f"argv[{j}] len={len(a)}: {a.decode('latin-1')[:60]!r}")
PY

# 2. reverse the 4 char-substitutions on argv[19] to recover the real remote bootstrap.sh,
#    then grep its flow markers (line numbers are within the recovered script == in-tree bootstrap.sh)
python3 - <<'PY'
data=open('/tmp/blitzy_obs/ssh_argv_capture.log','rb').read()
enc=data.split(b'\n===END-INVOCATION===\n')[1].split(b'\x00')[19][1:-1]
rev=enc.replace(b'\x0b',b"'").replace(b'\x0c',b'\\').replace(b'\x0d',b'\n').replace(b'\x08',b'!')
for i,ln in enumerate(rev.split(b'\n'),1):
    s=ln.decode('latin-1')
    if any(m in s for m in ['request_data=','dcs_to_kitty()','get_data()','untar_and_read_env','KITTY_DATA_START','KITTY_DATA_END','exec_login_shell']):
        print(f"L{i}: {s.strip()[:80]}")
PY
```

**Observed output (complete, unedited) — assembled `ssh` argv from the real `kitten ssh localhost`:**

```
argv[0] len=2: '-t'
argv[1] len=2: '-o'
argv[2] len=18: 'ControlMaster=auto'
argv[3] len=2: '-o'
argv[4] len=48: 'ControlPath=/root/.cache/kitty/run/kssh-59881-%C'
argv[5] len=2: '-o'
argv[6] len=18: 'ControlPersist=yes'
argv[7] len=2: '-o'
argv[8] len=22: 'ServerAliveInterval=60'
argv[9] len=2: '-o'
argv[10] len=21: 'ServerAliveCountMax=5'
argv[11] len=2: '-o'
argv[12] len=15: 'TCPKeepAlive=no'
argv[13] len=2: '--'
argv[14] len=9: 'localhost'
argv[15] len=4: 'exec'
argv[16] len=2: 'sh'
argv[17] len=2: '-c'
argv[18] len=67: '\'eval "$(echo "$0" | tr \\\\\\v\\\\\\f\\\\\\r\\\\\\b \\\\\\047\\\\\\134\\\\\\n\\\\\\041)"\' '
argv[19] len=5207: "'#<BS>/bin/sh<CR># Copyright (C) 2022 Kovid Goyal ..."
argv[20] len=0: ''
```

**Observed output (complete, unedited) — recovered remote `bootstrap.sh` flow markers (line numbers == in-tree file):**

```
L90: request_data="0"
L75: dcs_to_kitty() { printf "\033P@kitty-$1|%s\033\134" "$(printf "%s" "$2" | base64_encode)
L137: get_data() {
L144: if [ "$line" = "KITTY_DATA_START" ]; then
L151: untar_and_read_env
L164: exec_login_shell
```

**What this shows + reasoning — the ordered end-to-end trace:**

The recovered `argv[19]` (5205 bytes after un-substitution) is byte-for-byte the in-tree `shell-integration/ssh/bootstrap.sh` with `prepare_script()` placeholders filled in — its first line is `#!/bin/sh` and its flow markers land on the exact in-tree line numbers (dcs_to_kitty@75, get_data@137, KITTY_DATA_START@144, untar_and_read_env@151, exec_login_shell@164). Two runtime substitutions prove `prepare_script()` ran: the template's `request_data="REQUEST_DATA"` [bootstrap.sh:90] became **`request_data="0"`** (proactive; the local side already sent the request because OpenSSH ≥ 8.4), and the placeholders in `dcs_to_kitty "ssh" "id=REQUEST_ID:pwfile=PASSWORD_FILENAME:pw=DATA_PASSWORD"` [bootstrap.sh:94] are the ones filled with the real id/pwfile/pw seen in Q9.

**[Observed] + [Inferred] numbered trace** (local boundaries observed from the argv capture, shm object, and DCS bytes; remote boundaries after `exec ssh` are grounded in the recovered `bootstrap.sh`/`bootstrap-utils.sh` line refs and labeled inferred where they execute on the far side):

1. **[Observed] Invocation & entry.** User runs `kitten ssh localhost`; the Go command is dispatched via `ssh.EntryPoint(root)` [tools/cmd/tool/main.go:50] (registered by the import at [tools/cmd/tool/main.go:17]). `main()` requires `KITTY_WINDOW_ID`/`KITTY_PID` [kittens/ssh/main.go:825] and a terminal stdin [main.go:828].
2. **[Observed] Arg + config resolution.** `run_ssh()` [main.go:597] splits ssh args from the server args; `hostname := server_args[0]` [main.go:608]. Per-host config comes from `config_for_hostname()` [kittens/ssh/config.go:354] via `load_config()` [config.go:391].
3. **[Observed] Controlling TTY forced.** With no remote command, `cmd = append(cmd, "-t")` [main.go:609-611] — visible as `argv[0]='-t'`.
4. **[Observed] Connection sharing injected.** Because `host_opts.Share_connections`, `connection_sharing_args()` [main.go:121] output is `slices.Insert`ed at the insertion point [main.go:637-647] — visible as `argv[1..12]` (`ControlMaster=auto`, `ControlPath=/root/.cache/kitty/run/kssh-59881-%C`, `ControlPersist=yes`, keepalive trio).
5. **[Observed] Tarball built.** `make_tarfile()` [main.go:255] builds the gzip PAX tar (see Q3); it is Base64-embedded into the shm payload.
6. **[Observed] Password + shm store.** `bootstrap_script()` [main.go:422] sets `request_id = KITTY_PID-KITTY_WINDOW_ID` [main.go:424], `pw = secrets.TokenHex()` [main.go:431], marshals `{tarfile,pw,hostname,username}` [main.go:439-443], and writes it to a `0o600` shm object `shm.CreateTemp("kssh-%d-", …)` [main.go:446] (see Q2/Q8). Observed shm object: `/kssh-62797-…`, mode `0o600`.
7. **[Observed] Per-shell encoding.** `wrap_bootstrap_script()` [main.go:486] renders `exec sh -c <unwrap> <encoded>` — visible as `argv[15..19]` (see Q6).
8. **[Observed] Proactive DCS request (OpenSSH ≥ 8.4).** Because `set_askpass()` returned `need_to_request_data=false` [main.go:151-156] and `cd.request_data` is false, `run_ssh()` writes the `@kitty-ssh` DCS request to the TTY *before* exec [main.go:761-768]. Captured bytes decode to `id=59881-1:pwfile=kssh-60283-…:pw=972ce3f39e…` (see Q9). This is why the remote script carries `request_data="0"`.
9. **[Observed] ssh launched.** `exec.Command(cmd[0], cmd[1:]…)` [main.go:754] runs the assembled vector; the shim confirms the real `ssh` was invoked with exactly this argv.
10. **[Inferred, grounded] Remote entry.** `sshd` joins argv with spaces and runs it via the login shell `-c`; `sh -c` first `eval`s the `tr` unwrap [bootstrap.sh:… via main.go:506], reconstituting and executing the real `bootstrap.sh`.
11. **[Inferred, grounded] base64 helper resolution.** `bootstrap.sh` defines `base64_encode()` from the first available tool in its fallback chain [bootstrap.sh:55-73] (see Q9).
12. **[Inferred, grounded] Data pull.** Since `request_data="0"`, the remote does not re-send the request; `get_data()` [bootstrap.sh:137] reads the TTY until `KITTY_DATA_START` [bootstrap.sh:144], sends nothing, and consumes the `OK`/Base64/`KITTY_DATA_END` frames.
13. **[Inferred, grounded] Unpack + stage.** `untar_and_read_env()` [bootstrap.sh:104,151] Base64-decodes and `tar xpzf`s the payload; `bootstrap-utils.sh` runs `compile_terminfo()` [bootstrap-utils.sh:18] and `mv_files_and_dirs()` [bootstrap-utils.sh:9] to stage terminfo + shell-integration.
14. **[Inferred, grounded] Login shell.** The script ends at `exec_login_shell` [bootstrap.sh:164], implemented at [bootstrap-utils.sh:221], replacing the bootstrap process with the user's login shell (shell integration enabled).

**[Observed] Sequence diagram** (mirrors AAP §0.5.3; the branch taken in *this* environment is the "OpenSSH ≥ 8.4 → proactive" path, confirmed by `request_data="0"`):

```mermaid
sequenceDiagram
    participant U as User
    participant K as kitten ssh (Go)
    participant SHM as POSIX shm (0o600)
    participant KT as kitty terminal (core)
    participant TTY as Controlling TTY
    participant R as Remote bootstrap.sh

    U->>K: kitten ssh localhost
    K->>K: run_ssh(): parse args, config_for_hostname (main.go:597,608)
    K->>K: append "-t"; connection_sharing_args (main.go:609-647)
    K->>K: make_tarfile gzip PAX (main.go:255)
    K->>K: pw=secrets.TokenHex(); request_id=PID-WINID (main.go:424,431)
    K->>SHM: store {tarfile,pw,hostname,username} 0o600 (main.go:446)
    K->>K: wrap_bootstrap_script -> exec sh -c (main.go:486-508)
    Note over K,TTY: OpenSSH>=8.4 (set_askpass -> need_to_request_data=false, main.go:151-156)
    K->>TTY: PROACTIVE DCS @kitty-ssh id:pwfile:pw (main.go:761-768)
    K->>TTY: exec ssh <argv> (main.go:754)
    TTY->>R: login shell -c '<unwrap> <encoded>'
    R->>R: eval tr-unwrap -> real bootstrap.sh; request_data="0" (bootstrap.sh:90)
    R->>TTY: get_data(): read until KITTY_DATA_START (bootstrap.sh:137,144)
    TTY->>KT: get_ssh_data(msg, PID-WINID) (kitty/window.py:1291)
    KT->>SHM: read+verify owner/0o600 + pw + id (utils.py:129-133)
    KT->>TTY: KITTY_DATA_START / OK / base64(254B chunks) / KITTY_DATA_END (utils.py:117-148)
    R->>R: untar_and_read_env; compile_terminfo; mv_files_and_dirs (bootstrap.sh:151, bootstrap-utils.sh:9-18)
    R->>U: exec_login_shell (bootstrap.sh:164, bootstrap-utils.sh:221)
```

---

## Q8. How does the shared-memory piece keep things secure?

**Direct answer:** the credential channel is safe because the shm object is (1) created **exclusively** with `O_CREAT|O_EXCL` (so it cannot pre-exist / be hijacked), (2) **owner-only `0o600`** (only the creating user can read it), (3) **randomly named** (`kssh-<pid>-<32-random-bytes-hex>`, retried up to 30× on collision), (4) carries a **one-time password** that must match, correlated by a **request id**, and (5) is **re-verified on read** — both readers reject wrong owner or wrong permissions and **unlink the object on read** so it is single-use. Mismatched password, request id, or permissions are all rejected with a `ValueError`, observed live below.

**Command(s) run:**

```bash
# Drive the REAL kitty-core responder get_ssh_data() and the REAL Go/Python shm readers
# against a genuine shm object, with correct and deliberately-mismatched credentials.
python3 /tmp/blitzy_shim/obs_shm_ssh_data.py
```

**Observed output (complete, unedited):**

```
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-4f560dda-08e8-4b9f-9835-d97efce64779_231eb3/kittens/ssh/utils.py", line 131, in get_ssh_data
    raise ValueError('Incorrect password')
ValueError: Incorrect password
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-4f560dda-08e8-4b9f-9835-d97efce64779_231eb3/kittens/ssh/utils.py", line 133, in get_ssh_data
    raise ValueError(f'Incorrect request id: {rq_id!r} expecting the KITTY_PID-KITTY_WINDOW_ID for the current kitty window')
ValueError: Incorrect request id: '999-999' expecting the KITTY_PID-KITTY_WINDOW_ID for the current kitty window
=== (A) Inspect a REAL shm object: name + mode (Q8) ===
OBS_SHM_NAME=/kssh-62797-b9ea027e19afece8d6ef14289263d19f2155d17b130620dc32d156763c422037
OBS_SHM_PATH=/dev/shm//kssh-62797-b9ea027e19afece8d6ef14289263d19f2155d17b130620dc32d156763c422037
OBS_SHM_MODE=0o600
OBS_SHM_UID=0 EUID=0  match=True

=== (B) get_ssh_data with CORRECT pw+id: full framing (Q9 response) ===
OBS_FRAME[0]= b'\nKITTY_DATA_START\n'
OBS_FRAME[1]= b'OK\n'
OBS_TARFILE_B64_LEN=800
OBS_NUM_CHUNKS=4  chunk_sizes=[254, 254, 254, 38]
OBS_ALL_CHUNKS_LE_254=True
OBS_LAST_FRAME= b'KITTY_DATA_END\n'
OBS_FULL_FRAME_SEQUENCE= [b'\nKITTY_DATA_START\n', b'OK\n', b'G6K9U4iUWzK+...254B', b'\n', b'xVKoXJr7dclX...254B', b'\n', b'JL1XXQPDrMSe...254B', b'\n', b'3TWm9u+1OOMF...38B', b'\n', b'KITTY_DATA_END\n']

=== (C) get_ssh_data with WRONG password (Q8 rejection) ===
OBS_WRONGPW_OUTPUT= [b'\nKITTY_DATA_START\n', b'Incorrect password\n']

=== (D) get_ssh_data with WRONG request id (Q8 rejection) ===
OBS_WRONGID_OUTPUT= [b'\nKITTY_DATA_START\n', b"Incorrect request id: '999-999' expecting the KITTY_PID-KITTY_WINDOW_ID for the current kitty window\n"]

=== (E) WRONG permissions rejection via real reader (Q8) ===
OBS_BADPERM_SHM_MODE=0o644
OBS_BADPERM_REJECTED_WITH=ValueError('Incorrect permissions on pwfile: 0o644')
```

**What this shows + reasoning:**

- **[Observed] Exclusive create (named item `O_CREAT|O_EXCL`).** `SharedMemory.__init__` sets `flags = os.O_CREAT | os.O_EXCL` when creating [kitty/shm.py:62] and calls `shm_open(q, flags, mode)` [kitty/shm.py:74]. `O_EXCL` guarantees the call fails if the name already exists, defeating a pre-created/hijacked object; on the (astronomically unlikely) `FileExistsError` [kitty/shm.py:76] it retries with a fresh random name up to `tries = 30` [kitty/shm.py:69].
- **[Observed] Owner-only `0o600` (named item).** The default `mode` is `stat.S_IREAD | stat.S_IWRITE` [kitty/shm.py:51], i.e. `0o600`. The real shm object created during the run has `OBS_SHM_MODE=0o600` — read/write for the owner only, no group/other access.
- **[Observed] Random, unguessable name.** The observed name is `/kssh-62797-b9ea027e19afece8d6ef14289263d19f2155d17b130620dc32d156763c422037` — the `kssh-<pid>-` prefix from `shm.CreateTemp("kssh-%d-", os.Getpid())` [kittens/ssh/main.go:446] plus a long random suffix. An attacker cannot guess the name to race the reader.
- **[Observed] One-time password + request id (correlated challenge).** The kitty-core responder `get_ssh_data()` compares the DCS-supplied `pw` against `env_data['pw']` and raises `ValueError('Incorrect password')` on mismatch [kittens/ssh/utils.py:131] — reproduced verbatim in section (C) and its traceback. It also checks `rq_id` against the expected `KITTY_PID-KITTY_WINDOW_ID` and raises `Incorrect request id: '999-999' …` [kittens/ssh/utils.py:133] — reproduced in section (D). The correct pair yields the full `KITTY_DATA_START`/`OK`/…/`KITTY_DATA_END` stream in section (B). The password itself is `secrets.TokenHex()` (64-hex, value varies per run — see Q2), making it a genuine one-time secret.
- **[Observed] Re-verification on read + single-use unlink (both readers).**
  - **Python reader** `read_data_from_shared_memory()` [kittens/ssh/utils.py:100]: immediately `shm.unlink()` [utils.py:106] (single-use), then rejects wrong owner `if shm.stats.st_uid != os.geteuid() or shm.stats.st_gid != os.getegid()` → `Incorrect owner on pwfile…` [utils.py:107-108], and wrong mode `if mode != stat.S_IREAD | stat.S_IWRITE` → `Incorrect permissions on pwfile: 0o…` [utils.py:110-111]. Section (E) exercises this with a real object `chmod`ped to `0o644`, producing exactly `ValueError('Incorrect permissions on pwfile: 0o644')`.
  - **Go reader** `read_data_from_shared_memory()` [kittens/ssh/main.go:72] uses `shm.ReadWithSizeAndUnlink` [main.go:73] (unlink-on-read) with a validator rejecting owner mismatch `Incorrect owner on SHM file` [main.go:75-76] and `s.Mode().Perm() != 0o600` → `Incorrect permissions on SHM file` [main.go:79-80]. The two readers are symmetric.
- **[Observed] Localhost-only confinement.** The shm object lives on the local machine (`/dev/shm/…` observed), so the password never traverses the network in the clear; only the short-lived DCS request carries it back over the (already-established, TTY-local) channel to be matched. This corroborates the docs: the one-time password "matches a password pre-stored in shared memory on the localhost by the kitten" [docs/kittens/ssh.rst:141-146].
- **[Inferred, code-grounded] Owner-mismatch path.** Sections above exercise wrong-permission, wrong-password, and wrong-request-id rejections live. The **owner-mismatch** branch [kittens/ssh/utils.py:107-108, kittens/ssh/main.go:75-76] could not be triggered live because the sandbox runs as a single uid (`root`, uid=0) with no second user to create a foreign-owned object; it is therefore labeled **inferred**, grounded in the two cited lines whose logic mirrors the observed permission check.

---

## Q9. How does the terminal communicate back and forth with the remote shell during setup?

**Direct answer:** communication is a **DCS (Device Control String) escape-code protocol over the controlling TTY**. A `@kitty-ssh` request carrying `id=<request_id>:pwfile=<shm_name>:pw=<password>` (Base64-wrapped inside `ESC P @kitty-ssh| … ESC \`) is sent — **proactively by the local kitten on OpenSSH ≥ 8.4** (this environment), or by the remote `bootstrap.sh` on older OpenSSH. kitty-core replies with a framed stream: `\nKITTY_DATA_START\n`, then `OK\n`, then the Base64 tarball in **254-byte** chunks (each followed by `\n`), then `KITTY_DATA_END\n`. On the remote, `base64_encode`/`base64_decode` are resolved from a **5-tool fallback chain** (`base64` → `openssl` → `b64encode` → python → perl → `die`).

**Command(s) run:**

```bash
# (1) Capture the raw DCS request bytes written to the controlling TTY by a real
#     `kitten ssh localhost` (pty harness; the /tmp shim is only on PATH, source untouched).
python3 - <<'PY'
import base64, re
b = open('/tmp/blitzy_obs/dcs_capture_run1.bin','rb').read()
print(f"{len(b)} bytes"); print("repr:", repr(b))
m = re.search(rb'@kitty-ssh\|([A-Za-z0-9+/=]+)', b)
print("payload decoded:", base64.b64decode(m.group(1)).decode('latin-1'))
PY

# (2) The framed RESPONSE from the REAL kitty-core get_ssh_data() (same run as Q8 section B).
python3 /tmp/blitzy_shim/obs_shm_ssh_data.py   # section (B)

# (3) The remote base64 fallback chain + the OpenSSH>=8.4 gate (source, verified at 815df1e21).
sed -n '55,72p' shell-integration/ssh/bootstrap.sh
sed -n '204,222p' kittens/ssh/utils.go
```

**Observed output (complete, unedited) — the raw DCS request bytes (3 real runs):**

```
--- dcs_capture_run1.bin (176 bytes) ---
repr: b'\x1b[?s\x1b[?19997h\x1bP@kitty-ssh|aWQ9NTk4ODEtMTpwd2ZpbGU9a3NzaC02MDI4My1HN0hFRVdIN0tNNjc0OnB3PTk3MmNlM2YzOWUwNjM1ZTNkMTg2YmIxZGE2MDBkNjRkZTljZDM1YWFhZjA3YWU5OTRhNDNhMjA2YzVmNTNjMTg=\x1b\\'
payload decoded: id=59881-1:pwfile=kssh-60283-G7HEEWH7KM674:pw=972ce3f39e0635e3d186bb1da600d64de9cd35aaaf07ae994a43a206c5f53c18

--- dcs_capture_q5.bin (176 bytes) ---
repr: b'\x1b[?s\x1b[?19997h\x1bP@kitty-ssh|aWQ9NTk4ODEtMTpwd2ZpbGU9a3NzaC02MTI2NC1YSldDNUVQR1JVUElDOnB3PWYwOWNkYWRiM2U2ZjJkMDRiZDkzYmRkNWZhNTBjNDgyODc5M2IxMjUzOGQxNzI2NmEzZmVhYjljZWI3NTFmMmI=\x1b\\'
payload decoded: id=59881-1:pwfile=kssh-61264-XJWC5EPGRUPIC:pw=f09cdadb3e6f2d04bd93bdd5fa50c4828793b12538d17266a3feab9ceb751f2b

--- dcs_capture_q5reuse.bin (176 bytes) ---
repr: b'\x1b[?s\x1b[?19997h\x1bP@kitty-ssh|aWQ9NTk4ODEtMTpwd2ZpbGU9a3NzaC02MTcyNi1KT0taSUJXQkhPVUZBOnB3PTdiOGQxNTMxMDhiOTMyMmVkNTZlM2E1YTFhYzY4NGNhYmIzZTgwMzA1ZTVhYmI5MzM5NGQ3YmM5OThjN2Q2ZjA=\x1b\\'
payload decoded: id=59881-1:pwfile=kssh-61726-JOKZIBWBHOUFA:pw=7b8d153108b9322ed56e3a5a1ac684cabb3e80305e5abb93394d7bc998c7d6f0
```

**Observed output (complete, unedited) — the framed RESPONSE (from the real `get_ssh_data()`):**

```
OBS_FRAME[0]= b'\nKITTY_DATA_START\n'
OBS_FRAME[1]= b'OK\n'
OBS_TARFILE_B64_LEN=800
OBS_NUM_CHUNKS=4  chunk_sizes=[254, 254, 254, 38]
OBS_ALL_CHUNKS_LE_254=True
OBS_LAST_FRAME= b'KITTY_DATA_END\n'
OBS_FULL_FRAME_SEQUENCE= [b'\nKITTY_DATA_START\n', b'OK\n', b'G6K9U4iUWzK+...254B', b'\n', b'xVKoXJr7dclX...254B', b'\n', b'JL1XXQPDrMSe...254B', b'\n', b'3TWm9u+1OOMF...38B', b'\n', b'KITTY_DATA_END\n']
```

**Observed output (complete, unedited) — remote base64 fallback chain [shell-integration/ssh/bootstrap.sh:55-72]:**

```
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
fi
```

**Observed output (complete, unedited) — OpenSSH ≥ 8.4 gate [kittens/ssh/utils.go:204-222]:**

```
type SSHVersion struct{ Major, Minor int }

func (self SSHVersion) SupportsAskpassRequire() bool {
	return self.Major > 8 || (self.Major == 8 && self.Minor >= 4)
}

var GetSSHVersion = sync.OnceValue(func() SSHVersion {
	b, err := exec.Command(SSHExe(), "-V").CombinedOutput()
	if err != nil {
		return SSHVersion{}
	}
	m := regexp.MustCompile(`OpenSSH_(\d+).(\d+)`).FindSubmatch(b)
	if len(m) == 3 {
		maj, _ := strconv.Atoi(utils.UnsafeBytesToString(m[1]))
		min, _ := strconv.Atoi(utils.UnsafeBytesToString(m[2]))
		return SSHVersion{Major: maj, Minor: min}
	}
	return SSHVersion{}
})
```

**What this shows + reasoning:**

- **[Observed] The request is a DCS escape sequence over `/dev/tty` (named item `@kitty-ssh`).** The captured bytes begin with terminal mode-set escapes `ESC [ ? s` / `ESC [ ? 19997 h`, then the DCS `\x1bP@kitty-ssh|<base64>\x1b\\`. The `ESC P … ESC \` framing and `@kitty-ssh|` prefix are produced by `DCSToKitty("ssh", rq)` [tools/tui/dcs_to_kitty.go:14]. Decoding the Base64 payload yields exactly `id=59881-1:pwfile=kssh-60283-G7HEEWH7KM674:pw=972ce3f39e…`, i.e. the `id=<request_id>:pwfile=<shm_name>:pw=<password>` tuple. On the remote side the same envelope is produced by `dcs_to_kitty() { printf "\033P@kitty-$1|%s\033\134" … > /dev/tty; }` [shell-integration/ssh/bootstrap.sh:75], and the request line is `dcs_to_kitty "ssh" "id=REQUEST_ID:pwfile=PASSWORD_FILENAME:pw=DATA_PASSWORD"` [bootstrap.sh:94].
- **[Observed] Proactive (local) send on OpenSSH ≥ 8.4 (named item — the gate).** `SupportsAskpassRequire()` returns `Major > 8 || (Major == 8 && Minor >= 4)` [kittens/ssh/utils.go:206-207], parsed from `ssh -V` via the regexp `OpenSSH_(\d+).(\d+)` [utils.go:210-222]. This environment runs **OpenSSH_10.0p2** (see preamble), so the gate is **true** and `set_askpass()` sets `need_to_request_data = false` [kittens/ssh/main.go:151-156]. Consequently `run_ssh()` sends the request itself *before* exec: `if !cd.request_data { rq := fmt.Sprintf("id=%s:pwfile=%s:pw=%s", …) ; dcs, _ = tui.DCSToKitty("ssh", rq); term.WriteAllString(dcs) }` [main.go:761-768]. That the request appears in the *local* TTY capture (not sent by the remote) confirms the proactive branch, and the recovered remote script carrying `request_data="0"` (Q7) is the matching remote-side signal.
- **[Inferred, code-grounded] Older-OpenSSH branch.** On OpenSSH < 8.4 the gate is false, `need_to_request_data` stays true, and the **remote** `bootstrap.sh` issues the `dcs_to_kitty "ssh" …` request itself [bootstrap.sh:94] after `stty -echo` [bootstrap.sh:93]. This branch is **inferred** (not taken in this env, which is ≥ 8.4) but grounded in the cited lines; the docs confirm the ≥ 8.4 "transmitted instantly, without roundtrip delay" optimization [docs/kittens/ssh.rst:131-147].
- **[Observed] The framed response and 254-byte chunking (named item, magnitude).** `get_ssh_data()` [kittens/ssh/utils.py:115] yields `\nKITTY_DATA_START\n` [utils.py:117] to discard leading garbage, verifies the shm-stored `pw`/`id` (Q8), yields `OK\n` [utils.py:138], then streams `env_data['tarfile']` in `line_sz = 254` chunks — `while encoded_data: yield encoded_data[:line_sz]; yield b'\n'; encoded_data = encoded_data[line_sz:]` [utils.py:143-147] — and finally `KITTY_DATA_END\n` [utils.py:148]. The observed frame sequence matches exactly: `KITTY_DATA_START` → `OK` → four chunks `[254, 254, 254, 38]` (all ≤ 254, each `\n`-separated) → `KITTY_DATA_END`. The 254 (not 255) cap is deliberate: the code comment cites the "macOS … 255 byte limit on its input queue as per man stty" [utils.py:140-142].
- **[Observed] Chunk count scales with tarball size (magnitude, ≥2 runs).** For the small synthetic tarfile driven through the real responder, `OBS_TARFILE_B64_LEN=800` → `ceil(800/254) = 4` chunks, reproduced identically. For a *real* `kitten ssh` tarball (~24 KB gzip → ~32 KB Base64) the chunk count is ~124-127 (`ceil(b64_len/254)`), varying only because the gzip size fluctuates run-to-run (PAX `ModTime=time.Now()` + `BestCompression`, see Q3); the chunking rule itself (`line_sz=254`) is invariant.
- **[Observed] The remote base64 helper fallback chain (named item — enumerate ALL in order).** `bootstrap.sh` picks the first available of, in order: (1) `base64` [bootstrap.sh:55-57], (2) `openssl enc -A -base64` [bootstrap.sh:58-60], (3) `b64encode`/`b64decode` [bootstrap.sh:61-63], (4) python `base64.standard_b64encode/decode` via `detect_python` [bootstrap.sh:64-67], (5) perl `MIME::Base64` via `detect_perl` [bootstrap.sh:68-70], else (6) `die "base64 executable not present on remote host, ssh kitten cannot function."` [bootstrap.sh:72]. This makes the wire protocol robust across minimal remote hosts.
- **[Observed] Remote consumption.** The remote `get_data()` [bootstrap.sh:137] reads the TTY until it sees `KITTY_DATA_START` [bootstrap.sh:144], then consumes lines until `KITTY_DATA_END` [bootstrap.sh:99], feeding the accumulated Base64 to `untar_and_read_env` [bootstrap.sh:151]. The request/response is correlated end-to-end by `request_id = KITTY_PID-KITTY_WINDOW_ID` — the local side computes it at [main.go:424] and kitty-core dispatches with the same value: `for line in get_ssh_data(msg, f'{os.getpid()}-{self.id}')` [kitty/window.py:1291].

---

## Coverage Matrix

All values reported at commit **`815df1e21`** (HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`, branch `kitty_815df1e210e0`). Legend: ✅ = present; **[O]** = observed from live output; **[I]** = inferred/code-grounded (labeled in-text).

### Per-question coverage

| Question | Direct answer first? | Command(s) shown? | Complete unedited output? | `file:line` @ 815df1e21? | Named items enumerated? | Observed/Inferred labeled? |
|----------|:---:|:---:|:---:|:---:|:---:|:---:|
| **Q1** Secure session + connection sharing | ✅ | ✅ | ✅ | ✅ | ✅ ControlMaster/Path/Persist + keepalive trio; macOS symlink workaround; `SSHExe()` | ✅ |
| **Q2** Shm credentials + bootstrap generation | ✅ | ✅ | ✅ | ✅ | ✅ `secrets.TokenHex()`; `kssh-<pid>-…`; `bootstrap.sh`+`bootstrap.py`; `request_id` | ✅ |
| **Q3** Archive build + transport | ✅ | ✅ | ✅ | ✅ | ✅ gzip PAX; terminfo/shell-integration/kitty bins; `ssh/.+` exclusion; zsh compat | ✅ |
| **Q4** Connection data structure/state | ✅ | ✅ | ✅ | ✅ | ✅ `connection_data` (Go, 171-189) + `SSHConnectionData` (Py); generated `Config` | ✅ |
| **Q5** Connection reuse decision | ✅ | ✅ | ✅ | ✅ | ✅ `ssh -O check`; `need_to_request_data`; `run_control_master -N -f`; `close_shared_ssh_connections` | ✅ |
| **Q6** Per-shell bootstrap encoding | ✅ | ✅ | ✅ | ✅ | ✅ sh (4 substitutions + `tr` reversal) vs py (base64); both interpreters | ✅ |
| **Q7** End-to-end trace | ✅ | ✅ | ✅ | ✅ | ✅ 14-step trace + mermaid; local [O] + remote [I] boundaries | ✅ |
| **Q8** Shared-memory security | ✅ | ✅ | ✅ | ✅ | ✅ `O_CREAT\|O_EXCL`; `0o600`; random name; one-time pw; owner/perm re-verify + unlink | ✅ |
| **Q9** Terminal ↔ remote communication | ✅ | ✅ | ✅ | ✅ | ✅ `@kitty-ssh` DCS; `KITTY_DATA_START/OK/…/END`; ≥8.4 gate; base64 fallbacks; 254-byte chunks | ✅ |

### Per-named-item coverage

| Named item | Where answered | Concrete value / evidence | Key `file:line` | O/I |
|------------|:---:|---|---|:---:|
| **ControlMaster** | Q1, Q5, Q7 | `-o ControlMaster=auto` (captured argv[2]) | main.go:138 | [O] |
| **ControlPath** | Q1, Q7 | `-o ControlPath=/root/.cache/kitty/run/kssh-59881-%C` | main.go:135-136,139 | [O] |
| **ControlPersist** | Q1 | `-o ControlPersist=yes` | main.go:140 | [O] |
| **Keepalive trio** | Q1 | `ServerAliveInterval=60`, `ServerAliveCountMax=5`, `TCPKeepAlive=no` | main.go:141-143 | [O] |
| **macOS ControlPath symlink workaround** | Q1 | `/tmp/kssh-rdir-<euid>` when runtime dir >35 chars (here 26 → not triggered) | main.go:128-134 | [I] |
| **`SSHExe()` resolution** | Q1 | PATH lookup via `utils.FindExe("ssh")` (shim intercepted it) | utils.go:22-24 | [O] |
| **Shared memory** | Q2, Q8 | real object `/kssh-62797-…`, mode `0o600` | shm.py:51,62,74; main.go:446 | [O] |
| **`secrets.TokenHex()` (one-time pw)** | Q2, Q8 | 64-hex, 5 distinct samples across runs | main.go:431; tokens.go:28 | [O] |
| **Tarball (gzip PAX)** | Q3 | members incl. `home/.terminfo/kitty.terminfo`, kitty/kitten bins | main.go:255-367; main_test.go:81 | [O] |
| **`bootstrap.sh`** | Q2,Q6,Q7,Q9 | recovered full script from argv; sh substitutions | bootstrap.sh:1-164 | [O] |
| **`bootstrap.py`** | Q2, Q6 | base64 of `#!/usr/bin/env python` header confirmed | main.go:517; bootstrap.py:1-2 | [O] |
| **4 char substitutions** | Q6 | `'`→\v(0x0b), `\`→\f(0x0c), `\n`→\r(0x0d), `!`→\b(0x08) all present | main.go:505 | [O] |
| **`tr`-based remote unwrap** | Q6 | `tr \\\v\\\f\\\r\\\b \\\047\\\134\\\n\\\041` | main.go:506 | [O] |
| **`connection_data` struct (Go)** | Q4 | 16 fields; declared at 171 (AAP said 176-197) | main.go:171-189 | [O] |
| **`SSHConnectionData` NamedTuple (Py)** | Q4 | `binary,hostname,port,identity_file,extra_args` | kitty/utils.py:953-957 | [O] |
| **generated `Config` struct** | Q4 | built from Python `Definition` (14 options) | main.py:62-219 | [O] |
| **`ssh -O check` / reuse** | Q5 | `check_cmd=slices.Insert(cmd,1,"-O","check")`; dead vs alive states | main.go:653-661 | [O] |
| **`need_to_request_data` gate** | Q5, Q9 | `false` on ≥8.4 (proactive); reuse also sets false | main.go:151-156,663-664 | [O] |
| **`close_shared_ssh_connections`** | Q5 | master cleanup action | kitty/boss.py:3013 | [I] |
| **OpenSSH ≥ 8.4 gate** | Q9 | `Major>8 \|\| (Major==8 && Minor>=4)`; env=OpenSSH_10.0p2 → true | utils.go:206-207 | [O] |
| **base64 fallback chain** | Q9 | 6-way: base64→openssl→b64encode→python→perl→die | bootstrap.sh:55-72 | [O] |
| **254-byte chunking** | Q8, Q9 | chunk_sizes `[254,254,254,38]`; macOS 255 stty comment | utils.py:143-147 | [O] |
| **DCS `@kitty-ssh` request** | Q9 | `ESC P @kitty-ssh\|<b64> ESC \` → `id=…:pwfile=…:pw=…` | dcs_to_kitty.go:14; bootstrap.sh:75,94 | [O] |
| **`KITTY_DATA_START/OK/END` framing** | Q8, Q9 | exact frame sequence observed | utils.py:117,138,148 | [O] |
| **shm rejection: wrong password** | Q8 | `ValueError('Incorrect password')` | utils.py:131 | [O] |
| **shm rejection: wrong request id** | Q8 | `ValueError('Incorrect request id: '999-999' …')` | utils.py:133 | [O] |
| **shm rejection: wrong permissions** | Q8 | `ValueError('Incorrect permissions on pwfile: 0o644')` | utils.py:110-111; main.go:79-80 | [O] |
| **shm rejection: wrong owner** | Q8 | owner check (single-uid sandbox → not live-triggerable) | utils.py:107-108; main.go:75-76 | [I] |
| **Canonical entry point** | Preamble | `ssh.EntryPoint(root)`; Python `main()` raises `SystemExit` | tool/main.go:50; main.py:225 | [O] |

### Environment-specific notes (for reproducibility)

- **OpenSSH_10.0p2** (≥ 8.4) → the **proactive** DCS-send branch is taken here; the remote-initiated branch (OpenSSH < 8.4) is documented as **[Inferred]**.
- Runtime dir `/root/.cache/kitty/run` is **26 chars** (< 35) → the macOS ControlPath symlink workaround is **not** triggered here; documented as **[Inferred]** for the > 35 case.
- The askpass sentinel `/root/.cache/kitty/openssh-is-new-enough-for-askpass` pre-exists (created at build time), so `ssh -V` is not re-invoked on the default path; the gate short-circuits via the sentinel [main.go:149-152].
- Tarball gzip size varies run-to-run (~23.4-24.0 KB over 8 runs) due to PAX `ModTime=time.Now()` + `gzip.BestCompression`; uncompressed members are stable. The 9000-byte bootstrap guard covers only the default **sh** path (5282 B); the non-default **py** override is 13606 B.

---

## Repository left unchanged

This investigation was **read-only**: the sole artifact produced is this document (`blitzy/documentation/kitty_815df1e210e0.md`). Every temporary observation artifact was removed after the signals were captured:

- the temporary Go observation test `kittens/ssh/blitzy_adhoc_test_obs_test.go` (deleted),
- the `/tmp/blitzy_shim/` `ssh` `PATH` shim, the pty harness, and the `get_ssh_data()` driver (deleted),
- the `/tmp/blitzy_obs/` captured-output files (deleted),
- any transient `kssh-*` ControlMaster sockets under the runtime dir (removed).

The build-time askpass sentinel `/root/.cache/kitty/openssh-is-new-enough-for-askpass`, which pre-existed this investigation, was intentionally left in place. No source file was created, modified, or deleted. After the single documentation commit, `git status` reports a clean working tree, and diffing against the original commit shows exactly one added path:

```bash
$ git status --porcelain
$ git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 HEAD
A	blitzy/documentation/kitty_815df1e210e0.md
```
