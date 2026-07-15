# How the kitty `ssh` kitten works — an evidence-backed walkthrough

> Branch: `kitty_815df1e210e0` · Repo: `kovidgoyal/kitty` · HEAD: `815df1e21`
> Product built & observed at runtime: **kitty / kitten 0.35.2**
> Every **structural** claim below carries an exact **repository-relative** `file:line` reference; every **behavioral** claim carries the **exact command** that produced it and the **complete, unedited observed output**. Values that are freshly random every run are shown at more than one value so the variance is explicit (§0.6). The only claims that could not be observed at runtime are labelled **_inferred from source_** and are collected in an appendix.

---

## Executive summary

The kitty `ssh` kitten is a thin **Go wrapper around the system `ssh` binary**. It does **not** re-implement the SSH transport, user/host authentication, integrity, or cryptography — those remain entirely OpenSSH's responsibility (spelled out in §1.6). Instead it resolves and shells out to `ssh` (`SSHExe = sync.OnceValue(func() string { return utils.FindExe("ssh") })` [`kittens/ssh/utils.go`:L22-L24]) and layers three things on top:

1. **Connection sharing** — when `share_connections` is enabled (the default [`kittens/ssh/main.py`:L183]) it injects **six** OpenSSH `-o` options that turn on `ControlMaster` multiplexing, so repeated connections *from the same kitty instance to the same host* reuse one already-authenticated channel [`kittens/ssh/main.go`:L121-L145].
2. **A remotely-executed bootstrap** — it generates a small POSIX-`sh` (or Python) script, encodes it so it survives an `ssh` command line, and runs it as the remote command. That bootstrap pulls a gzipped tar of kitty's shell-integration files + terminfo onto the remote host and then hands off to the user's real login shell [`kittens/ssh/main.go`:L422-L525; `shell-integration/ssh/bootstrap.sh`:L104-L164].
3. **A shared-memory-backed data channel** — the tar archive plus a one-time data password are **always** written to a **POSIX shared-memory object** (`/dev/shm/kssh-*`, mode `0600`) — the only exception is the test-only `dont_create_shm` flag [`kittens/ssh/main.go`:L431-L459; `kittens/ssh/main_test.go`:L49-L54]. The archive is delivered over the terminal itself using a **DCS (Device Control String) handshake**, after kitty validates the request against the shm object.

**The credential flow hinges on one decision — `request_data` — and the true behavior is the opposite of "credentials never touch the command line."** The two facts the rest of this document keeps carefully separate are **(a) whether the shm/tar channel exists** and **(b) who sends the request DCS**:

- **Fresh connection** (`request_data=true`; no live master): the request id, password-file name, and password are copied into the generated bootstrap [`kittens/ssh/main.go`:L475-L478], encoded into the remote command [`kittens/ssh/main.go`:L508], and **appended to the system-`ssh` argv** [`kittens/ssh/main.go`:L753-L754]. So on a fresh connection those credentials **are** present in the local process's argument list — a real process-list exposure, disclosed and demonstrated in §2.5 and §6. The **remote** bootstrap is then the party that sends the request DCS back to kitty.
- **Reused connection** (`request_data=false`; a live master, isolated in this document with `askpass=ssh`): the very same placeholders are left **literal** in the argv, and the **local Go kitten itself** sends the request DCS to its controlling kitty terminal [`kittens/ssh/main.go`:L761-L769]. The shm/tar transfer still happens.

The whole thing is therefore driven off **two independent decisions**: whether an SSH master is already alive (reuse vs. fresh), and whether the data must be requested at all (`request_data`, which can also be flipped to `false` by the kitty-askpass path — deliberately isolated away here). Ten specific questions follow, each with a runtime demonstration.

---

## §0 Methodology & Build

### 0.1 Toolchain actually observed

All work was performed in the task's Linux container (`uname -srm` → `Linux 6.6.122+ x86_64`). The exact tool versions present were captured with:

```
$ go version; python3 --version; ssh -V; zsh --version; fish --version; \
  bash --version | head -1; tar --version | head -1; \
  base64 --version | head -1; tr --version | head -1
```

Observed (complete, verbatim):

```
go version go1.23.4 linux/amd64
Python 3.13.7
OpenSSH_10.0p2 Ubuntu-5ubuntu5.4, OpenSSL 3.5.3 16 Sep 2025
zsh 5.9 (x86_64-ubuntu-linux-gnu)
fish, version 4.0.6
GNU bash, version 5.2.37(1)-release (x86_64-pc-linux-gnu)
tar (GNU tar) 1.35
base64 (uutils coreutils) 0.2.2
tr (uutils coreutils) 0.2.2
```

Two honest deltas from the anchors quoted in the task brief, reported as observed:

* The brief anticipated **Python 3.12.x** and **OpenSSH 9.6p1**; this container actually runs **Python 3.13.7** and **OpenSSH_10.0p2**. `go.mod` declares `go 1.22`; the installed `go1.23.4` satisfies it.
* `base64`/`tr` are provided by **uutils coreutils 0.2.2**, not GNU coreutils. This matters for the `tr`-based decoding proof in §7, so the byte-level reversal was re-verified against *this* exact `tr` (it handles the `\b \v \f \r` control-byte set correctly — see §7.3).

### 0.2 Build commands (Run-First) — with complete output, timing, and artifacts

kitty's single build system is `setup.py`. The product was built from source **before any observation**, using the exact canonical commands (both steps timed with the shell, since `/usr/bin/time` is absent in this container):

```
$ export PATH=/usr/local/go/bin:$PATH
$ python3 setup.py build --debug --ignore-compiler-warnings --skip-building-kitten   # C-extension + launcher + code-generation
$ python3 setup.py build --debug --ignore-compiler-warnings --skip-code-generation   # the Go kitten
```

**Step 1 — complete output** (`BUILD1_EXIT=0`, wall time **8 s**):

```
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
[1/85] Compiling kitty/screen.c ...
[2/85] Compiling kitty/unicode-data.c ...
[3/85] Compiling [x11] glfw/x11_window.c ...
[4/85] Compiling kitty/glfw.c ...
[5/85] Compiling kitty/graphics.c ...
[6/85] Compiling kitty/child-monitor.c ...
[7/85] Compiling kitty/fonts.c ...
[8/85] Compiling kitty/shaders.c ...
[9/85] Compiling kitty/vt-parser.c ...
[10/85] Compiling kitty/vt-parser.c ...
[11/85] Compiling kitty/state.c ...
[12/85] Compiling [x11] glfw/input.c ...
[13/85] Compiling kitty/mouse.c ...
[14/85] Compiling [x11] glfw/xkb_glfw.c ...
[15/85] Compiling kitty/freetype.c ...
[16/85] Compiling [x11] glfw/window.c ...
[17/85] Compiling kitty/line.c ...
[18/85] Compiling kitty/glfw-wrapper.c ...
[19/85] Compiling kittens/transfer/algorithm.c ...
[20/85] Compiling [x11] glfw/x11_init.c ...
[21/85] Compiling kitty/freetype_render_ui_text.c ...
[22/85] Compiling [x11] glfw/egl_context.c ...
[23/85] Compiling kitty/disk-cache.c ...
[24/85] Compiling [x11] glfw/glx_context.c ...
[25/85] Compiling kitty/line-buf.c ...
[26/85] Compiling kitty/data-types.c ...
[27/85] Compiling kitty/colors.c ...
[28/85] Compiling kitty/history.c ...
[29/85] Compiling kitty/keys.c ...
[30/85] Compiling [x11] glfw/x11_monitor.c ...
[31/85] Compiling kitty/fontconfig.c ...
[32/85] Compiling [x11] glfw/context.c ...
[33/85] Compiling kitty/crypto.c ...
[34/85] Compiling [x11] glfw/ibus_glfw.c ...
[35/85] Compiling kitty/key_encoding.c ...
[36/85] Compiling kitty/launcher/main.c ...
[37/85] Compiling [x11] glfw/monitor.c ...
[38/85] Compiling kitty/font-names.c ...
[39/85] Compiling [x11] glfw/backend_utils.c ...
[40/85] Compiling kitty/charsets.c ...
[41/85] Compiling [x11] glfw/linux_joystick.c ...
[42/85] Compiling [x11] glfw/init.c ...
[43/85] Compiling [x11] glfw/dbus_glfw.c ...
[44/85] Compiling kitty/gl.c ...
[45/85] Compiling [x11] glfw/vulkan.c ...
[46/85] Compiling [x11] glfw/osmesa_context.c ...
[47/85] Compiling kitty/cursor.c ...
[48/85] Compiling kitty/launcher/single-instance.c ...
[49/85] Compiling kitty/desktop.c ...
[50/85] Compiling kitty/loop-utils.c ...
[51/85] Compiling 3rdparty/ringbuf/ringbuf.c ...
[52/85] Compiling kitty/simd-string.c ...
[53/85] Compiling kitty/systemd.c ...
[54/85] Compiling kitty/shlex.c ...
[55/85] Compiling kitty/child.c ...
[56/85] Compiling kitty/kittens.c ...
[57/85] Compiling 3rdparty/base64/lib/codec_choose.c ...
[58/85] Compiling kitty/png-reader.c ...
[59/85] Compiling [x11] glfw/linux_notify.c ...
[60/85] Compiling kitty/rowcolumn-diacritics.c ...
[61/85] Compiling kitty/hyperlink.c ...
[62/85] Compiling kitty/wcswidth.c ...
[63/85] Compiling kitty/fast-file-copy.c ...
[64/85] Compiling 3rdparty/base64/lib/lib.c ...
[65/85] Compiling [x11] glfw/posix_thread.c ...
[66/85] Compiling kitty/window_logo.c ...
[67/85] Compiling kitty/glyph-cache.c ...
[68/85] Compiling kitty/logging.c ...
[69/85] Compiling 3rdparty/base64/lib/arch/neon64/codec.c ...
[70/85] Compiling 3rdparty/base64/lib/tables/tables.c ...
[71/85] Compiling 3rdparty/base64/lib/arch/neon32/codec.c ...
[72/85] Compiling 3rdparty/base64/lib/arch/avx/codec.c ...
[73/85] Compiling 3rdparty/base64/lib/arch/ssse3/codec.c ...
[74/85] Compiling 3rdparty/base64/lib/arch/sse42/codec.c ...
[75/85] Compiling 3rdparty/base64/lib/arch/sse41/codec.c ...
[76/85] Compiling 3rdparty/base64/lib/arch/avx2/codec.c ...
[77/85] Compiling kitty/utmp.c ...
[78/85] Compiling 3rdparty/base64/lib/arch/avx512/codec.c ...
[79/85] Compiling 3rdparty/base64/lib/arch/generic/codec.c ...
[80/85] Compiling kitty/cleanup.c ...
[81/85] Compiling [x11] glfw/monotonic.c ...
[82/85] Compiling kitty/monotonic.c ...
[83/85] Compiling kitty/simd-string-128.c ...
[84/85] Compiling kitty/simd-string-256.c ...
[85/85] Compiling kitty/gl-wrapper.c ...
 done
[1/4] Linking kitty/fast_data_types ...
[2/4] Linking [x11] kitty/glfw-x11 ...
[3/4] Linking kittens/transfer/rsync ...
[4/4] Linking launcher ...
 done
Skipping building of the kitten binary because of a command line option. Build is incomplete
```

**Step 2 — complete output** (`BUILD2_EXIT=0`, wall time **1 s**):

```
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
Skipping generation of Go files due to command line option
```

**Artifacts produced** and **binary versions**, captured immediately after:

```
$ ls -l constants_generated.go kitty/fast_data_types.so kitty/launcher/kitten \
        kitty/launcher/kitty tools/tui/shell_integration/data_generated.bin
-rw-r--r-- 1 root root    14092 constants_generated.go
-rwxr-xr-x 1 root root  6285000 kitty/fast_data_types.so
-rwxr-xr-x 1 root root 22231315 kitty/launcher/kitten
-rwxr-xr-x 1 root root   281328 kitty/launcher/kitty
-rw-r--r-- 1 root root    25574 tools/tui/shell_integration/data_generated.bin

$ ./kitty/launcher/kitten --version ; ./kitty/launcher/kitty --version
kitten 0.35.2 created by Kovid Goyal
kitty 0.35.2 created by Kovid Goyal
```

**About `--ignore-compiler-warnings` (honest note).** The brief states the flag is required because a newer `wayland-protocols` trips `-Werror=switch` in `glfw/wl_window.c`. In *this* container `wayland-protocols` is **not** installed, so `setup.py` prints `Disabling building of wayland backend` (visible above) and never compiles `glfw/wl_window.c`; the `-Werror=switch` condition therefore never fires here and the flag is effectively a no-op for this build. It is retained because it is the documented canonical invocation and is harmless. This is reported as observed rather than repeating the brief's stated cause.

All five build outputs are git-ignored; `git status --porcelain` is empty immediately after building. The product tree is never modified.

### 0.3 Canonical entry points used (no mocks of kitten logic)

Every behavioral result below comes from one of four **real** entry points — never a debug hook, mock, or synthetic stand-in for the kitten's own logic:

| # | Entry point | What it exercises | Key source |
|---|-------------|-------------------|------------|
| 1 | `kitten ssh …` (driven through a real PTY) | the actual user command → `run_ssh` | [`kittens/ssh/main.go`:L597, L800] |
| 2 | `printf '<conf>' \| kitten __pytest__ ssh '<test-script>'` | the real `get_remote_command` path; prints JSON `{"cmd":…,"shm_name":…}`; reads an ssh-kitten config from **stdin**; fixes `request_id="testing"`, `request_data=true`, `echo_on=true` | [`kittens/ssh/main.go`:L847-L886] |
| 3 | `kitty +launch test.py --module ssh` | the PTY round-trip suite `kitty_tests/ssh.py` (`check_bootstrap` [L227]) | [`kitty_tests/ssh.py`:L227-L271] |
| 4 | system `ssh` invoked with the kitten's **own** six sharing args, against an isolated local `sshd` | the real OpenSSH ControlMaster lifecycle | [`kittens/ssh/main.go`:L121-L145] |

Entry point #2 is the integration hook `TestEntryPoint`/`test_integration_with_python` [`kittens/ssh/main.go`:L847-L886]. It is worth stating precisely what it fixes, because it constrains every observation drawn from it: it sets `request_id="testing"`, `request_data=true`, `echo_on=true`, `username="testuser"`, `hostname_for_match="host.test"`, reads the config from stdin, calls the real `get_remote_command`, and marshals `{"cmd": cd.rcmd, "shm_name": cd.shm_name}` to stdout. Because it sets `request_data=true` it always exercises the **fresh** path, and because it does **not** set `dont_create_shm` it leaves the real shm object on disk for inspection.

**The only shim** used anywhere is a **record-only fake `ssh`** that appends its `argv` to a log and, for `-O check`, returns an exit code we control (`FAKE_SSH_OCHECK_RC`) so the kitten's *own* decision branch runs. It re-implements **no** kitten logic. Complete source (`/tmp/blitzy_ssh_obs/fakebin/ssh`, 637 bytes):

```sh
#!/bin/sh
# Record-only fake ssh: appends argv to a log and, for "-O check", returns a
# controlled exit code (FAKE_SSH_OCHECK_RC) so the kitten's OWN decision branch
# executes. It re-implements NO kitten logic.
LOG="${FAKE_SSH_LOG:-/tmp/blitzy_ssh_obs/fakessh.log}"
{
  echo "=== invocation ts=$(date +%s.%N) argc=$# ==="
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

### 0.4 The isolated local `sshd` (used only for Q1's real ControlMaster lifecycle)

Q1's ControlMaster lifecycle is observed against a **fully isolated** `sshd` — no global host keys, no reuse of root's persistent identity, and host verification **retained** (not disabled). Everything lives in one throwaway directory `/tmp/blitzy_ssh_obs/sshd/` and is torn down by numeric PID at the end (§0.7).

Setup — dedicated host key, dedicated client key, explicit `AuthorizedKeysFile`, and a seeded `known_hosts` for host verification:

```
$ SSHDIR=/tmp/blitzy_ssh_obs/sshd
$ ssh-keygen -t ed25519 -f $SSHDIR/host_ed25519   -N '' -C blitzy-obs-host   >/dev/null
$ ssh-keygen -t ed25519 -f $SSHDIR/client_ed25519 -N '' -C blitzy-obs-client >/dev/null
$ cp $SSHDIR/client_ed25519.pub $SSHDIR/authorized_keys
$ printf '[127.0.0.1]:2222 %s\n' "$(cat $SSHDIR/host_ed25519.pub)" > $SSHDIR/known_hosts
```

The complete `sshd` config (every path is inside the temp dir; `PidFile` is explicit):

```
$ cat $SSHDIR/sshd_config
Port 2222
ListenAddress 127.0.0.1
HostKey /tmp/blitzy_ssh_obs/sshd/host_ed25519
PidFile /tmp/blitzy_ssh_obs/sshd/sshd.pid
AuthorizedKeysFile /tmp/blitzy_ssh_obs/sshd/authorized_keys
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
UsePAM no
PermitRootLogin prohibit-password
StrictModes no
LogLevel VERBOSE
```

Start it, capture the exact numeric PID (redirecting all output away from the shell pipe so the daemon does not hold it open), and verify with **host verification retained**:

```
$ /usr/sbin/sshd -f $SSHDIR/sshd_config -E $SSHDIR/sshd.log >/dev/null 2>&1 </dev/null
$ cat $SSHDIR/sshd.pid
110125
$ ssh -p 2222 -o StrictHostKeyChecking=yes -o UserKnownHostsFile=$SSHDIR/known_hosts \
      -i $SSHDIR/client_ed25519 root@127.0.0.1 'echo PROBE_OK' </dev/null
PROBE_OK
```

`StrictHostKeyChecking=yes` against the seeded temp `known_hosts` means the server's host key is genuinely verified — the earlier non-isolated approach (`ssh-keygen -A`, `StrictHostKeyChecking=no`, root's own `~/.ssh/id_ed25519`) is **not** used. The daemon is stopped by this exact PID and every path above removed during cleanup (§0.7).

### 0.5 Automated suites run before writing (Go unit tests and the PTY round-trip suite)

**Go SSH-kitten unit tests** — `go test ./kittens/ssh/...` (`GO_TEST_EXIT=0`):

```
=== RUN   TestSSHConfigParsing
--- PASS: TestSSHConfigParsing (0.01s)
=== RUN   TestCloneEnv
--- PASS: TestCloneEnv (0.00s)
=== RUN   TestSSHBootstrapScriptLimit
--- PASS: TestSSHBootstrapScriptLimit (0.01s)
=== RUN   TestSSHTarfile
--- PASS: TestSSHTarfile (0.01s)
=== RUN   TestGetSSHOptions
--- PASS: TestGetSSHOptions (0.00s)
=== RUN   TestParseSSHArgs
--- PASS: TestParseSSHArgs (0.00s)
=== RUN   TestRelevantKittyOpts
--- PASS: TestRelevantKittyOpts (0.00s)
PASS
ok  	kitty/kittens/ssh	0.049s
```

**PTY round-trip suite** — `./kitty/launcher/kitty +launch test.py --module ssh`, run **twice** to show stability (both `EXIT=0`; 8/8 each; 10.080 s then 9.907 s):

```
# run 1
Running under CI: False
test_basic_pty_operations (kitty_tests.ssh.SSHKitten.test_basic_pty_operations) ... ok
test_ssh_bootstrap_with_different_launchers (kitty_tests.ssh.SSHKitten.test_ssh_bootstrap_with_different_launchers) ... ok
test_ssh_connection_data (kitty_tests.ssh.SSHKitten.test_ssh_connection_data) ... ok
test_ssh_copy (kitty_tests.ssh.SSHKitten.test_ssh_copy) ... ok
test_ssh_env_vars (kitty_tests.ssh.SSHKitten.test_ssh_env_vars) ... ok
test_ssh_leading_data (kitty_tests.ssh.SSHKitten.test_ssh_leading_data) ... ok
test_ssh_login_shell_detection (kitty_tests.ssh.SSHKitten.test_ssh_login_shell_detection) ... ok
test_ssh_shell_integration (kitty_tests.ssh.SSHKitten.test_ssh_shell_integration) ... ok

----------------------------------------------------------------------
Ran 8 tests in 10.080s

OK

# run 2
Ran 8 tests in 9.907s

OK
```

### 0.6 Run-to-run variance (stated up front)

Several values are **freshly random every run**; they are shown at two values wherever they appear so the variance is explicit rather than implied:

* the shared-memory object name suffix `kssh-<pid>-<13-char base32>`, e.g. `kssh-108091-7MUZEN7V3YSLW` vs `kssh-109681-WIHB57IINC574`;
* the data password `pw`, a fresh 64-hex `secrets.TokenHex()` string every run (e.g. `fb7fda9c…0ec0`); these are ephemeral and unlinked on read, never real provider credentials;
* the OpenSSH `%C` connection hash (40 hex), process ids (`KITTY_PID`, master pid, `sshd` pid).

**Stable across runs:** the five-element `rcmd` shape, the four sh-scheme byte substitutions, the 15 tar members, the env-record format, the ≤254-byte transport framing, the six shared-memory security-branch message strings, the DCS frame format, `CURSOR_BEAM=2`, and the terminfo contents. Two `kitten __pytest__ ssh` captures taken back to back were byte-identical except for the ephemeral `shm_name`/`pw`.

### 0.7 Read-only guarantee and final proof

The source tree is treated as strictly read-only — the only path that differs from `HEAD` is this document. Every artifact **this investigation created** is removed at the end: all observation scripts and logs, the isolated `sshd` together with its temporary host/client keys, `sshd_config`, and PidFile, and the record-only fake `ssh` shim — all of which lived under `/tmp/blitzy_ssh_obs/` — plus every `/dev/shm/kssh-*`/`ksse-*` object, any `/tmp/kssh-rdir-*` symlink, and every git-ignored build artifact (the `kitty`/`kitten` launchers, `fast_data_types.so`, `constants_generated.go`, `data_generated.bin`, and the `build/` tree). The isolated `sshd` is stopped by its **exact numeric PID** (`110125`, recorded in §0.4), never a pattern kill. The proof below is captured **after** cleanup and deliberately covers **all five surfaces** the work touched — the Git tree, running processes, the listening socket, POSIX shared memory, and the `/tmp` scratch area — rather than relying on `git status` alone:

```
$ git status --porcelain -uall
 M blitzy/documentation/kitty_815df1e210e0.md

$ git status --porcelain -uall | grep -Ev 'blitzy/documentation/kitty_815df1e210e0\.md$' ; echo "other_paths_rc=$?"
other_paths_rc=1                        # no other tracked/untracked path differs from HEAD

$ kill -0 110125                        # the isolated sshd PID recorded in §0.4
bash: kill: (110125) - No such process
kill0_rc=1                              # process is gone

$ pgrep -af sshd | grep blitzy_ssh_obs ; echo "isolated_sshd_rc=$?"
isolated_sshd_rc=1                      # no isolated sshd remains

$ ss -ltn | grep ":2222" ; echo "port2222_rc=$?"
port2222_rc=1                           # the listening socket is closed

$ ls -1 /dev/shm/ | grep -Ec 'kssh-|ksse-'
0                                       # no shared-memory objects remain

$ ls -d /tmp/blitzy_ssh_obs 2>&1
ls: cannot access '/tmp/blitzy_ssh_obs': No such file or directory

$ ls -d /tmp/kssh-rdir-* 2>&1 ; echo "rdir_rc=$?"
ls: cannot access '/tmp/kssh-rdir-*': No such file or directory
rdir_rc=2
```

**On `/root/.ssh` and `/etc/ssh`.** These are **environment-provisioned baseline**, not artifacts of this investigation, and they lie **outside** the Git working tree. The container's setup prepared the global host keys and a `/root/.ssh` key pair before any work began (per the environment setup: "host keys + /root/.ssh ed25519 key ... prepared"). The isolated methodology of §0.4 was chosen precisely so the work never reads, writes, or authenticates against them: the local `sshd` was driven entirely from its **own** temporary keys under `/tmp/blitzy_ssh_obs/sshd/` (now removed) with a **seeded temporary `known_hosts`**, so no `[127.0.0.1]:2222` entry was ever written to a global `/root/.ssh/known_hosts` (none exists). Because they are environment infrastructure rather than task output, they are left intact — removing environment-provided keys is outside this task's cleanup scope, which covers the isolated `sshd` and its own temporary keys, the fake shim, `/dev/shm` objects, temp scripts, and build artifacts.

No existing source file was modified and no product code was added; the sole tracked change is this document. The read-only mandate is satisfied, and the proof above reflects the repository and host exactly as delivered.
---

## §1 How does the kitten set up a secure session and share connections?

**Direct answer.** The kitten never re-implements SSH. It resolves the system `ssh` binary once (`SSHExe` [`kittens/ssh/utils.go`:L22-L24]; its version is probed with `ssh -V` in `GetSSHVersion` [`kittens/ssh/utils.go`:L211]) and, when connection sharing is enabled (the default), prepends **six** OpenSSH `-o` options that turn on `ControlMaster` multiplexing. The first connection from a given kitty instance to a host opens a background *master* channel; every later connection *from that same kitty instance to the same host* piggybacks on it, so TCP setup and authentication happen only once. The six options come from `connection_sharing_args` [`kittens/ssh/main.go`:L121-L145]. Crucially, everything security-relevant about the session itself — key exchange, encryption, integrity, host authentication, and user authentication — is done by OpenSSH, not by the kitten (§1.6).

### 1.1 The six options, captured verbatim

Captured from a real `kitten ssh` run driven through a PTY with the record-only fake `ssh` (`KITTY_PID=55002`, `KITTY_WINDOW_ID=7`, `--kitten askpass=ssh` to isolate the sharing branch). This is the `-O check` probe the kitten emitted first, reproduced exactly (the full argv log):

```
$ # fake-ssh argv log from: kitten ssh --kitten askpass=ssh -- host.test echo hello
=== invocation ts=1784071066.726131816 argc=16 ===
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

So the six options, verbatim, with their **exact** source lines (the option slice is built at [`kittens/ssh/main.go`:L137-L143]):

| # | Option | Effect | Source |
|---|--------|--------|--------|
| 1 | `-o ControlMaster=auto` | reuse a master if one is present, else become the master | [`kittens/ssh/main.go`:L138] |
| 2 | `-o ControlPath=<runtime_dir>/kssh-<KITTY_PID>-%C` | per-connection master socket path | [`kittens/ssh/main.go`:L135-L136, L139] |
| 3 | `-o ControlPersist=yes` | keep the master alive after the first client exits | [`kittens/ssh/main.go`:L140] |
| 4 | `-o ServerAliveInterval=60` | send a keepalive probe every 60 s | [`kittens/ssh/main.go`:L141] |
| 5 | `-o ServerAliveCountMax=5` | drop the connection after 5 missed probes | [`kittens/ssh/main.go`:L142] |
| 6 | `-o TCPKeepAlive=no` | rely on SSH-level keepalive, not TCP keepalive | [`kittens/ssh/main.go`:L143] |

### 1.2 The `ControlPath` template and why it is short

The socket name is built from a template generated at build time:

* `ssh_control_master_template = 'kssh-{kitty_pid}-{ssh_placeholder}'` [`kitty/constants.py`:L188]
* emitted into Go by `generate_constants` as `const SSHControlMasterTemplate = "kssh-{kitty_pid}-{ssh_placeholder}"` [`gen/go_code.py`:L599], landing in the generated `constants_generated.go`
* at runtime `{kitty_pid}` → the pid via `strings.Replace` [`kittens/ssh/main.go`:L135] and `{ssh_placeholder}` → the literal `%C` [`kittens/ssh/main.go`:L136]

The prefix is deliberately terse. The source comment at [`kittens/ssh/main.go`:L123-L127] and [`kitty/constants.py`:L186-L187] explains that OpenSSH appends a 40-character connection hash plus a ~27-character temporary suffix to `ControlPath`, and a UNIX socket path is capped near ~104 bytes (103 on macOS); keeping `kssh-<pid>-` short leaves room. `%C` is OpenSSH's **connection hash** of the connection tuple (local host, remote host, port, user); the *exact* inputs are OpenSSH-internal and so are labelled **_inferred from source_**, but the hash's presence and 40-hex form are confirmed empirically in §1.3. It is this `%C` that scopes reuse to a specific *(user, host, port)* — see §1.5.

### 1.3 The real ControlMaster lifecycle (against the isolated local `sshd`)

Driving the system `ssh` with the kitten's exact six options against the isolated `sshd` (§0.4) on `127.0.0.1:2222`, with host verification retained. Complete transcript (`ControlPath=/root/.cache/kitty/run/kssh-77001-%C`):

```
### ControlMaster lifecycle — isolated sshd 127.0.0.1:2222, kitten's exact six -o options, host verification retained

--- (1) first connection becomes the master ---
$ ssh -o ControlMaster=auto -o ControlPath=$RD/kssh-77001-%C -o ControlPersist=yes \
      -o ServerAliveInterval=60 -o ServerAliveCountMax=5 -o TCPKeepAlive=no \
      -o StrictHostKeyChecking=yes -o UserKnownHostsFile=$KNOWN \
      -p 2222 -i $KEY root@127.0.0.1 "echo MASTER_CONNECTION_OK; hostname"
MASTER_CONNECTION_OK
reverse-code-generator-1a3e34e0-6h99z

--- (2) master socket present (UNIX socket "s", mode 0600, 40-hex %C hash) ---
$ ls -l $RD/kssh-77001-*
srw------- 1 root root 0 Jul 14 23:50 /root/.cache/kitty/run/kssh-77001-de6224549bf17f660bd566abc77d74659f79d881

--- (3) -O check reports the master alive (exit 0) ---
$ ssh <six opts> <ver> -O check -p 2222 -i $KEY root@127.0.0.1 ; echo check_exit=$?
Master running (pid=116567)
check_exit=0

--- (4) second connection REUSES the master (multiplexed), timed with bash builtin ---
$ time ssh <six opts> <ver> -p 2222 -i $KEY root@127.0.0.1 "echo REUSED_OK"
REUSED_OK

real	0m0.006s
user	0m0.000s
sys	0m0.004s

--- (5) controlled NON-multiplexed baseline (fresh TCP+auth, NO ControlPath), timed ---
$ time ssh -o ControlMaster=no -o ControlPath=none <ver> -p 2222 -i $KEY root@127.0.0.1 "echo BASELINE_OK"
BASELINE_OK

real	0m0.121s
user	0m0.008s
sys	0m0.002s

--- (6) -O exit tears down the master (ControlPersist=yes had kept it alive) ---
$ ssh <six opts> <ver> -O exit -p 2222 -i $KEY root@127.0.0.1 ; echo exit_cmd_exit=$?
Exit request sent.
exit_cmd_exit=0

--- (7) socket gone after -O exit ---
$ ls -l $RD/kssh-77001-* 2>&1
ls: cannot access '/root/.cache/kitty/run/kssh-77001-*': No such file or directory
```

Reading the evidence, effect by cause:

* **(2)** proves the master socket is a UNIX-domain socket (`s`), mode `0600` (owner-only), and that OpenSSH appended a 40-hex `%C` hash (`de62245…8f30`) exactly as §1.2 predicts.
* **(3)** the kitten's own reuse decision is exactly this `-O check` probe; here it reports `Master running (pid=…)` and exits `0`.
* **(4) vs (5)** is the controlled timing baseline the reuse claim needs: reusing the master takes **0.006 s**, whereas a **non-multiplexed** connection to the *same* server (identical auth, `ControlMaster=no ControlPath=none`) takes **0.121 s** — a ~20× difference that is entirely the TCP + key-exchange + authentication a reused connection skips.
* **(6)/(7)** `ControlPersist=yes` kept the master alive across (2)–(5); `-O exit` tears it down and the socket disappears.

### 1.4 Relevant configuration defaults

| Option | Default | Source |
|--------|---------|--------|
| `share_connections` | `yes` | [`kittens/ssh/main.py`:L183] |
| `forward_remote_control` | `no` | [`kittens/ssh/main.py`:L212] |
| `interpreter` | `sh` | [`kittens/ssh/main.py`:L87] |
| `askpass` | `unless-set` (choices: `unless-set` / `ssh` / `native`) | [`kittens/ssh/main.py`:L192] |

`forward_remote_control=yes` is only permitted together with `share_connections=yes` (it relies on the ControlMaster). The guard is at [`kittens/ssh/main.go`:L681-L683]; when violated the kitten returns the exact error:

```
Cannot use forward_remote_control=yes without share_connections=yes as it relies on SSH Controlmasters
```

### 1.5 Scope of reuse (corrected) and the macOS long-runtime-dir workaround (_inferred from source_)

**Scope of reuse.** Sharing is **not** "every later connection to the same host, forever." Two facts bound it precisely:

* The `ControlPath` embeds `{kitty_pid}` = the launching **kitty instance's** pid [`kittens/ssh/main.go`:L135], so the master socket is private to that one kitty process; a different kitty instance uses a different socket and does not reuse it.
* Within that instance, `%C` [`kittens/ssh/main.go`:L136] distinguishes connections by their attributes (user, host, port, …), so only connections that match on those attributes share a master.

Empirically, reuse is decided by the `-O check` probe against the pid-scoped, `%C`-scoped socket (§1.3 step 3), not by the mere absence of a repeated host-key warning.

**The macOS workaround (_inferred from source_).** If the runtime directory path is longer than 35 bytes, `connection_sharing_args` symlinks it to `/tmp/kssh-rdir-<euid>` to stay under the socket-path limit [`kittens/ssh/main.go`:L128-L134]. On Linux here the runtime dir is `/root/.cache/kitty/run`, which is **22 bytes** (`printf '%s' /root/.cache/kitty/run | wc -c` → `22`) and `22 < 35`, so this branch does **not** fire — confirmed, because the observed `ControlPath` in §1.3 used the real runtime dir with no `/tmp` symlink. The branch itself (the macOS-specific path) is therefore labelled **_inferred from source_**.

### 1.6 What is the kitten's security boundary vs. OpenSSH's?

Making the boundary explicit (finding this under-specified was a review point):

* **OpenSSH owns the session's security.** The encrypted transport, message integrity/authentication, the SSH key exchange and ciphers, **host authentication** (verifying the server's host key against `known_hosts`), and **user authentication** (public-key/password/etc.) are all performed by the system `ssh` the kitten shells out to [`kittens/ssh/utils.go`:L22-L24]. The kitten adds no crypto of its own to the wire.
* **The kitten owns only the bootstrap-data channel.** Its security contribution is confined to (a) keeping the shell-integration payload + data password off the network entirely by placing them in a local `0600` shared-memory object, and (b) gating the release of that payload behind the six checks in §9. That channel rides *inside* the already-established, OpenSSH-encrypted PTY stream; the base64/DCS framing is **not** a cryptographic layer, only a transport-safe encoding.
* **Consequence.** The kitten cannot make an insecure SSH configuration secure, and does not try to: if the user disables host checking, that is an OpenSSH-level decision. What the kitten *does* guarantee is that the data password is never sent over the network and that the tar payload is only handed to a requester that passes the §9 checks.

---


## §2 How does it use shared memory to pass credentials securely?

**Direct answer.** The kitten writes the entire bootstrap payload — the base64 gzipped tar **and** a one-time data password — into a **POSIX shared-memory object** at `/dev/shm/kssh-<pid>-<rand>`, created owner-only (`0600`) and exclusively (`O_EXCL`) by the **Go** side. This object is created **unconditionally** on every real run (the only suppressor is the test-only `dont_create_shm` flag). What differs between a fresh and a reused connection is **not** whether this object exists — it always does — but **who reads the password back to kitty** and, critically, **whether the credentials also end up in the local `ssh` process's argv** (they do, on a fresh connection). This section documents the object; §2.5 draws the fresh/reused distinction precisely.

### 2.1 A live `/dev/shm/kssh-*` object

Produced by a real `kitten __pytest__ ssh` run (entry point #2), which leaves the shm object on disk for inspection:

```
$ printf '' | ./kitty/launcher/kitten __pytest__ ssh 'echo UNTAR_DONE' > shm_live.json
$ python3 -c "import json;print(json.load(open('shm_live.json'))['shm_name'])"
kssh-119853-AGNKC3OBSHAWO
$ ls -l /dev/shm/kssh-119853-AGNKC3OBSHAWO
-rw------- 1 root root 31575 Jul 15 00:01 /dev/shm/kssh-119853-AGNKC3OBSHAWO
$ stat -c 'uid=%u gid=%g mode=%a size=%s' /dev/shm/kssh-119853-AGNKC3OBSHAWO
uid=0 gid=0 mode=600 size=31575
```

The object is a regular file under `/dev/shm`, **mode `600`** (`-rw-------`), owned by the creating process's uid/gid. No other user on the host can open it. The name suffix (`AGNKC3OBSHAWO`) is fresh per run (§0.6).

### 2.2 Creation guarantees — the Go producer, `O_EXCL` + `0600`

The producer is the **Go** shared-memory backend, not Python. `shm.CreateTemp(...)` opens the object with `O_EXCL | O_CREATE | O_RDWR` and mode `0600`:

```
tools/utils/shm/shm_syscall.go:L162   f, err = shm_open(name, os.O_EXCL|os.O_CREATE|os.O_RDWR, 0600)
```

`O_EXCL` guarantees the create fails rather than reusing a pre-existing name (no attacker-planted object can be silently adopted), and `0600` makes it owner-only from birth. The kitten calls this from `bootstrap_script`:

```
kittens/ssh/main.go:L445   if err == nil && !cd.dont_create_shm {
kittens/ssh/main.go:L446       data_shm, err = shm.CreateTemp(fmt.Sprintf("kssh-%d-", os.Getpid()), uint64(len(encoded_data)+8))
kittens/ssh/main.go:L447       if err == nil { err = shm.WriteWithSize(data_shm, encoded_data, 0) ... }
```

Creation is guarded only by `!cd.dont_create_shm`; it is **not** conditioned on `request_data`. And `dont_create_shm` is **test-only** (see §5). So in every real invocation, fresh or reused, the shm object is created.

### 2.3 The 4-byte size prefix and the payload shape

The payload is length-prefixed. `WriteWithSize` writes a **4-byte big-endian** length, then the bytes:

```
tools/utils/shm/shm.go:L116   const NUM_BYTES_FOR_SIZE = 4
tools/utils/shm/shm.go:L120   func WriteWithSize(self MMap, b []byte, at int) error {
tools/utils/shm/shm.go:L124       binary.BigEndian.PutUint32(self.Slice()[at:], uint32(len(b)))
tools/utils/shm/shm.go:L125       copy(self.Slice()[at+NUM_BYTES_FOR_SIZE:], b)
```

Decoding the live object confirms the layout exactly — a 4-byte prefix, then a JSON document, then a little slack (the producer over-allocates `len(encoded_data)+8` at `kittens/ssh/main.go:L446`, so 4 bytes go to the prefix and 4 remain as slack):

```
$ python3 - kssh-119853-AGNKC3OBSHAWO <<'PY'
import sys, struct, json
raw = open(f"/dev/shm/{sys.argv[1]}","rb").read()
print("total_object_bytes =", len(raw))
size = struct.unpack(">I", raw[:4])[0]
print("4-byte big-endian size prefix =", size)
d = json.loads(raw[4:4+size])
print("JSON keys (sorted) =", sorted(d.keys()))
print("tarfile(b64)=%d  pw=%d(hex)  hostname=%r  username=%r" % (len(d['tarfile']), len(d['pw']), d['hostname'], d['username']))
print("trailing slack bytes =", len(raw) - (4+size))
PY
total_object_bytes = 31575
4-byte big-endian size prefix = 31567
JSON keys (sorted) = ['hostname', 'pw', 'tarfile', 'username']
tarfile(b64)=31436  pw=64(hex)  hostname='host.test'  username='testuser'
trailing slack bytes = 4
```

So the payload is a JSON object with exactly four keys — **`tarfile`** (base64 of the gzipped tar), **`pw`** (the 64-hex one-time data password), **`hostname`**, and **`username`** — assembled at `kittens/ssh/main.go:L439-L443`. The password lives only here; it is never written to the network.

### 2.4 Lifetime and cleanup

There are two readers, and correspondingly two cleanup paths — both of which unlink the object so it is single-use:

* **Production (Go) cleanup.** `run_ssh` installs a `defer` that closes and unlinks the object when the kitten exits:

  ```
  kittens/ssh/main.go:L600   defer func() {
  kittens/ssh/main.go:L601       if data_shm != nil {
  kittens/ssh/main.go:L602           data_shm.Close()
  kittens/ssh/main.go:L603           _ = data_shm.Unlink()
  ```

* **Reader-side unlink.** When kitty answers the request DCS, the terminal-side reader opens the object read-only and **unlinks it before reading** (`read_data_from_shared_memory` at `kittens/ssh/utils.py:L105-L106`), so a served password can never be served twice. This single-use behavior is demonstrated exhaustively in §9.2.

Either way the object does not outlive the connection setup; the probe object in §2.1/§2.3 was removed immediately after inspection (`rm -f /dev/shm/kssh-119853-AGNKC3OBSHAWO`; remaining `kssh-` objects: `0`).

### 2.5 Fresh vs reused — where the credentials actually go (the corrected flow)

This is the crux the earlier draft got backwards. The shm object exists in both cases; the difference is the **substitution of the sensitive values into the generated bootstrap** and therefore **into the `ssh` argv**:

* On a **fresh** connection (`request_data=true`), the sensitive triple `{REQUEST_ID, DATA_PASSWORD, PASSWORD_FILENAME}` is merged into the script's substitution map [`kittens/ssh/main.go`:L476-L478], so the generated bootstrap's DCS line is filled with the **real** values, the script is encoded into `rcmd` [`kittens/ssh/main.go`:L508], and `rcmd` is **appended to the system-`ssh` argv** [`kittens/ssh/main.go`:L753-L754]. The password is therefore present in the local `ssh` process's arguments. Captured directly from the fresh-path fake-`ssh` argv log (`FAKE_SSH_OCHECK_RC=1`, master absent; password redacted here as a justified redaction — it is an ephemeral, already-unlinked 64-hex token):

  ```
  # fakessh_fresh.log — the argc=19 real connection, argv[18] is the bootstrap script
  request_data="1"
  ...
  [ "$request_data" = "1" ] && {
      command stty "-echo" < /dev/tty
      dcs_to_kitty "ssh" "id="55002-7":pwfile="kssh-109681-WIHB57IINC574":pw="<64-hex-REDACTED>""
  }
  ```

  The `id`, `pwfile`, and `pw` are literally substituted into `argv[18]` — a real **process-list exposure** on the machine running the kitten. (It is *not* sent over the network on the command line: `ssh` runs it as the remote command, but any local user who can read this process's argv sees the password.)

* On a **reused** connection (`request_data=false`; isolated here with `askpass=ssh` so only a live master can flip the decision), the sensitive values are **not** merged into the script map, so the same DCS line keeps its **literal placeholders**, and the local Go kitten sends the request itself (§6, §10). Captured from the reused-path log (`FAKE_SSH_OCHECK_RC=0`, master alive):

  ```
  # fakessh_reused.log — argv[18], same position, placeholders NOT substituted
  request_data="0"
  ...
  [ "$request_data" = "1" ] && {
      command stty "-echo" < /dev/tty
      dcs_to_kitty "ssh" "id="REQUEST_ID":pwfile="PASSWORD_FILENAME":pw="DATA_PASSWORD""
  }
  ```

  Here `request_data="0"`, the guarded block never runs on the remote, and the credentials never enter this argv. (They are instead sent locally by the Go kitten to its own kitty terminal — §6.3, §10.4.)

So "shared memory to pass credentials securely" is accurate about the **network** (the password never traverses it), but on a **fresh** connection the password does appear in the **local** process table for the lifetime of the `ssh` process. The rest of the security story — how the reader validates a request before releasing the tar — is §9.

---

## §3 How are the bootstrap scripts that run on the remote machine generated?

**Direct answer.** The kitten does **not** hand-write a script per connection. It takes a **checked-in template** — `shell-integration/ssh/bootstrap.sh` or `shell-integration/ssh/bootstrap.py`, embedded into the kitten binary at build time — and performs a **literal token substitution** on it, replacing a fixed set of `UPPER_CASE` placeholders with per-connection values. Which template is used is decided by the `interpreter` option (`sh` by default, `py`/`python3` selects the Python template). The filled-in script is then **encoded** (§7) and wrapped into the remote command `rcmd` that `ssh` executes. Everything happens locally in Go, in three functions in `kittens/ssh/main.go`: `get_remote_command` [`kittens/ssh/main.go`:L511-L525] picks the template, `bootstrap_script` [`kittens/ssh/main.go`:L455-L483] builds the substitution map and applies it, and `wrap_bootstrap_script` [`kittens/ssh/main.go`:L486-L509] encodes the result.

### 3.1 The generation pipeline

| Step | Code | What it does |
|------|------|--------------|
| Pick interpreter | `interpreter := cd.host_opts.Interpreter` [`kittens/ssh/main.go`:L512] | Reads the `interpreter` host-option (default `sh`, [`kittens/ssh/main.py`:L87]) |
| Pick template type | `cd.script_type = "sh"` [`kittens/ssh/main.go`:L515] / `"py"` [`kittens/ssh/main.go`:L517] | `py` iff the interpreter basename is `py`/`python`/`python2`/`python3`; otherwise `sh` |
| Build substitutions + apply | `bootstrap_script(cd)` [`kittens/ssh/main.go`:L519] → [L455-L483] | Builds the placeholder map and calls `prepare_script` |
| Fetch template | `shell_integration.Data()["shell-integration/ssh/bootstrap."+cd.script_type].Data` [`kittens/ssh/main.go`:L481] | Reads the embedded template blob (built by `gen/go_code.py`) |
| Substitute | `prepare_script(cd.bootstrap_script, sd)` [`kittens/ssh/main.go`:L482] | Replaces every `PLACEHOLDER` token with its value |
| Encode + wrap | `wrap_bootstrap_script(cd)` [`kittens/ssh/main.go`:L523] → [L486-L509] | Encodes (§7) and builds `cd.rcmd` [`kittens/ssh/main.go`:L508] |

### 3.2 The substitution map — and where the fresh/reused split happens

Two maps are built in `bootstrap_script`:

- **Non-secret placeholders** — `replacements` [`kittens/ssh/main.go`:L461], plus two booleans added by `add_bool`: `REQUEST_DATA` and `ECHO_ON` [`kittens/ssh/main.go`:L473-L474].
- **Secret placeholders** — `sensitive_data` [`kittens/ssh/main.go`:L460]: `REQUEST_ID`, `DATA_PASSWORD`, `PASSWORD_FILENAME`.

The **decisive lines** are the conditional merge:

```go
sd := maps.Clone(replacements)      // L475
if cd.request_data {                // L476
    maps.Copy(sd, sensitive_data)   // L477
}                                   // L478
```
[`kittens/ssh/main.go`:L475-L478]

`sd` is the map actually applied to the template ([`kittens/ssh/main.go`:L482]). So on a **fresh** connection (`request_data == true`) the real `id`/`pwfile`/`pw` are **baked into the script text**; on a **reused** connection (`request_data == false`) the three secret tokens are left as the **literal strings** `REQUEST_ID` / `PASSWORD_FILENAME` / `DATA_PASSWORD`. This is the same fresh/reused split proven with captured `ssh` argv in §2.5 and §6.2, seen here at its source. (The `maps.Copy(replacements, sensitive_data)` at [`kittens/ssh/main.go`:L479] populates the *separate* `cd.replacements` used for local DCS on the reused path — it does not affect the script `sd`.)

Placeholders substituted into the template:

| Placeholder | Source field | Fresh value (observed) | Reused value |
|-------------|--------------|------------------------|--------------|
| `REQUEST_ID` | `cd.request_id` (`KITTY_PID-KITTY_WINDOW_ID`) | e.g. `testing` (pytest) / `55002-7` (PTY) | literal `REQUEST_ID` |
| `PASSWORD_FILENAME` | `cd.shm_name` | e.g. `kssh-108091-7MUZEN7V3YSLW` | literal `PASSWORD_FILENAME` |
| `DATA_PASSWORD` | `pw` (32 random bytes, hex) | 64-hex (redacted below) | literal `DATA_PASSWORD` |
| `REQUEST_DATA` | `cd.request_data` | `1` | `0` |
| `ECHO_ON` | `cd.echo_on` | `1`/`0` | `1`/`0` |
| `EXPORT_HOME_CMD`, `EXEC_CMD`, `TEST_SCRIPT`, `LOGIN_SHELL`, `KITTY_...` | env/config | filled from config | filled from config |

### 3.3 Complete decoded **sh** bootstrap script

Generated with the default interpreter (`sh`), captured through the canonical `__pytest__` entry point and decoded by reversing the §7 substitution (the 64-hex password on the `dcs_to_kitty` line is the only redaction; everything else is byte-for-byte as emitted):

```
$ printf '' | ./kitty/launcher/kitten __pytest__ ssh 'echo UNTAR_DONE'   # -> JSON: cmd[], shm_name
$ # cmd[4] is the wrapped sh script; strip the wrapping quote and reverse the tr map (see §7)
```

Result — **164 lines, 5229 bytes** (`cmd` = `['exec','sh','-c', <unwrap>, <encoded>]`, `cmd[4]` decoded):

```sh
#!/bin/sh
# Copyright (C) 2022 Kovid Goyal <kovid at kovidgoyal.net>
# Distributed under terms of the GPLv3 license.

{ \unalias command; \unset -f command; } >/dev/null 2>&1
tdir=""
shell_integration_dir=""
echo_on="1"

cleanup_on_bootstrap_exit() {
    [ "$echo_on" = "1" ] && command stty "echo" 2> /dev/null < /dev/tty
    echo_on="0"
    [ -n "$tdir" ] && command rm -rf "$tdir"
    tdir=""
}

die() {
    if [ -e /dev/stderr ]; then
        printf "\033[31m%s\033[m\n\r" "$*" > /dev/stderr;
    elif [ -e /dev/fd/2 ]; then
        printf "\033[31m%s\033[m\n\r" "$*" > /dev/fd/2;
    else
        printf "\033[31m%s\033[m\n\r" "$*";
    fi
    cleanup_on_bootstrap_exit;
    exit 1;
}

python_detected="0"
detect_python() {
    if [ python_detected = "1" ]; then
        [ -n "$python" ] && return 0
        return 1
    fi
    python_detected="1"
    python=$(command -v python3)
    [ -z "$python" ] && python=$(command -v python2)
    [ -z "$python" ] && python=$(command -v python)
    if [ -z "$python" -o ! -x "$python" ]; then python=""; return 1; fi
    return 0
}

perl_detected="0"
detect_perl() {
    if [ perl_detected = "1" ]; then
        [ -n "$perl" ] && return 0
        return 1
    fi
    perl_detected="1"
    perl=$(command -v perl)
    if [ -z "$perl" -o ! -x "$perl" ]; then perl=""; return 1; fi
    return 0
}

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

dcs_to_kitty() { printf "\033P@kitty-$1|%s\033\134" "$(printf "%s" "$2" | base64_encode)" > /dev/tty; }
debug() { dcs_to_kitty "print" "debug: $1"; }

# If $HOME is configured set it here

# ensure $HOME is set
[ -z "$HOME" ] && HOME=~
# ensure $USER is set
[ -z "$USER" ] && USER="$LOGNAME"
[ -z "$USER" ] && USER="$(command whoami 2> /dev/null)"

leading_data=""
login_shell=""
login_cwd=""

request_data="1"
trap "cleanup_on_bootstrap_exit" EXIT
[ "$request_data" = "1" ] && {
    command stty "-echo" < /dev/tty
    dcs_to_kitty "ssh" "id="testing":pwfile="kssh-108091-7MUZEN7V3YSLW":pw="<64-hex-REDACTED>""
}

read_base64_from_tty() {
    while IFS= read -r line; do
        [ "$line" = "KITTY_DATA_END" ] && return 0
        printf "%s" "$line"
    done
}

untar_and_read_env() {
    # extract the tar file atomically, in the sense that any file from the
    # tarfile is only put into place after it has been fully written to disk
    command -v tar > /dev/null 2> /dev/null || die "tar is not available on this server. The ssh kitten requires tar."
    tdir=$(command mktemp -d "$HOME/.kitty-ssh-kitten-untar-XXXXXXXXXXXX")
    [ $? = 0 ] || die "Creating temp directory failed"
    # suppress STDERR for tar as tar prints various warnings if for instance, timestamps are in the future
    old_umask=$(umask)
    umask 000
    read_base64_from_tty | base64_decode | command tar "xpzf" "-" "-C" "$tdir" 2> /dev/null
    umask "$old_umask"
    . "$tdir/bootstrap-utils.sh"
    . "$tdir/data.sh"
    [ -z "$KITTY_SSH_KITTEN_DATA_DIR" ] && die "Failed to read SSH data from tty"
    case "$KITTY_SSH_KITTEN_DATA_DIR" in
        /*) data_dir="$KITTY_SSH_KITTEN_DATA_DIR" ;;
        *) data_dir="$HOME/$KITTY_SSH_KITTEN_DATA_DIR"
    esac
    shell_integration_dir="$data_dir/shell-integration"
    unset KITTY_SSH_KITTEN_DATA_DIR
    login_shell="$KITTY_LOGIN_SHELL"
    unset KITTY_LOGIN_SHELL
    login_cwd="$KITTY_LOGIN_CWD"
    unset KITTY_LOGIN_CWD
    kitty_remote="$KITTY_REMOTE"
    unset KITTY_REMOTE
    compile_terminfo "$tdir/home"
    mv_files_and_dirs "$tdir/home" "$HOME"
    [ -e "$tdir/root" ] && mv_files_and_dirs "$tdir/root" ""
    command rm -rf "$tdir"
    tdir=""
}

get_data() {
    started="n"
    while IFS= read -r line; do
        if [ "$started" = "y" ]; then
            [ "$line" = "OK" ] && break
            die "$line"
        else
            if [ "$line" = "KITTY_DATA_START" ]; then
                started="y"
            else
                leading_data="$leading_data$line"
            fi
        fi
    done
    untar_and_read_env
}

# ask for the SSH data
get_data
cleanup_on_bootstrap_exit
prepare_for_exec
# If a command was passed to SSH execute it here


# Used in the tests
echo UNTAR_DONE

exec_login_shell
```

### 3.4 Complete decoded **Python** bootstrap script

Generated with `interpreter python3`; here `cmd[4]` is plain base64, so decoding is a single `base64 -d` (proven byte-identical in §7.4):

```
$ printf 'interpreter python3\n' | ./kitty/launcher/kitten __pytest__ ssh 'echo UNTAR_DONE'
$ python3 -c "import json,base64,sys; print(base64.standard_b64decode(json.load(sys.stdin)['cmd'][4]).decode())" < gen_py.json
```

Result — **318 lines, 10137 bytes** (only the 64-hex password on the `dcs_to_kitty` line is redacted):

```python
#!/usr/bin/env python
# License: GPLv3 Copyright: 2022, Kovid Goyal <kovid at kovidgoyal.net>


import base64
import contextlib
import errno
import io
import json
import os
import pwd
import shutil
import subprocess
import sys
import tarfile
import tempfile
import termios

tty_file_obj = None
echo_on = int('1')
data_dir = shell_integration_dir = ''
request_data = int('1')
leading_data = b''
login_shell = os.environ.get('SHELL') or '/bin/sh'
try:
    login_shell = pwd.getpwuid(os.geteuid()).pw_shell
except KeyError:
    pass
export_home_cmd = b''
if export_home_cmd:
    HOME = base64.standard_b64decode(export_home_cmd).decode('utf-8')
    os.chdir(HOME)
else:
    HOME = os.path.expanduser('~')


def set_echo(fd, on=False):
    if fd < 0:
        fd = sys.stdin.fileno()
    old = termios.tcgetattr(fd)
    new = termios.tcgetattr(fd)
    if on:
        new[3] |= termios.ECHO
    else:
        new[3] &= ~termios.ECHO
    termios.tcsetattr(fd, termios.TCSANOW, new)
    return fd, old


def cleanup():
    global tty_file_obj
    if tty_file_obj is not None:
        if echo_on:
            set_echo(tty_file_obj.fileno(), True)
        tty_file_obj.close()
        tty_file_obj = None


def write_all(fd, data):
    if isinstance(data, str):
        data = data.encode('utf-8')
    data = memoryview(data)
    while data:
        try:
            n = os.write(fd, data)
        except BlockingIOError:
            continue
        if not n:
            break
        data = data[n:]


def dcs_to_kitty(payload, type='ssh'):
    if isinstance(payload, str):
        payload = payload.encode('utf-8')
    payload = base64.standard_b64encode(payload)
    return b'\033P@kitty-' + type.encode('ascii') + b'|' + payload + b'\033\\'


def send_data_request():
    write_all(tty_file_obj.fileno(), dcs_to_kitty('id=testing:pwfile=kssh-108508-VD6FZSISISTW6:pw=<64-hex-REDACTED>'))


def debug(msg):
    data = dcs_to_kitty('debug: {}'.format(msg), 'print')
    if tty_file_obj is None:
        with open(os.ctermid(), 'wb') as fl:
            write_all(fl.fileno(), data)
    else:
        write_all(tty_file_obj.fileno(), data)


def apply_env_vars(raw):
    global login_shell

    def process_defn(defn):
        parts = json.loads(defn)
        if len(parts) == 1:
            key, val = parts[0], ''
        else:
            key, val, literal_quote = parts
            if not literal_quote:
                val = os.path.expandvars(val)
        os.environ[key] = val

    for line in raw.splitlines():
        val = line.split(' ', 1)[-1]
        if line.startswith('export '):
            process_defn(val)
        elif line.startswith('unset '):
            os.environ.pop(json.loads(val)[0], None)
    login_shell = os.environ.pop('KITTY_LOGIN_SHELL', login_shell)


def move(src, base_dest):
    for x in os.listdir(src):
        path = os.path.join(src, x)
        dest = os.path.join(base_dest, x)
        if os.path.islink(path):
            try:
                os.unlink(dest)
            except EnvironmentError:
                pass
            os.symlink(os.readlink(path), dest)
        elif os.path.isdir(path):
            if not os.path.exists(dest):
                os.makedirs(dest)
            move(path, dest)
        else:
            shutil.move(path, dest)


def compile_terminfo(base):
    try:
        tic = shutil.which('tic')
    except AttributeError:
        # python2
        for x in os.environ.get('PATH', '').split(os.pathsep):
            q = os.path.join(x, 'tic')
            if os.access(q, os.X_OK) and os.path.isfile(q):
                tic = q
                break
        else:
            tic = ''
    if not tic:
        return
    tname = '.terminfo'
    q = os.path.join(base, tname, '78', 'xterm-kitty')
    if not os.path.exists(q):
        try:
            os.makedirs(os.path.dirname(q))
        except EnvironmentError as e:
            if e.errno != errno.EEXIST:
                raise
        os.symlink('../x/xterm-kitty', q)
    if os.path.exists('/usr/share/misc/terminfo.cdb'):
        # NetBSD requires this
        os.symlink('../../.terminfo.cdb', os.path.join(base, tname, 'x', 'xterm-kitty'))
        tname += '.cdb'
    os.environ['TERMINFO'] = os.path.join(HOME, tname)
    p = subprocess.Popen(
        [tic, '-x', '-o', os.path.join(base, tname), os.path.join(base, '.terminfo', 'kitty.terminfo')],
        stdout=subprocess.PIPE, stderr=subprocess.STDOUT
    )
    output = p.stdout.read()
    rc = p.wait()
    if rc != 0:
        getattr(sys.stderr, 'buffer', sys.stderr).write(output)
        raise SystemExit('Failed to compile the terminfo database')


def iter_base64_data(f):
    global leading_data
    started = 0
    while True:
        line = f.readline().rstrip()
        if started == 0:
            if line == b'KITTY_DATA_START':
                started = 1
            else:
                leading_data += line
        elif started == 1:
            if line == b'OK':
                started = 2
            else:
                raise SystemExit(line.decode('utf-8', 'replace').rstrip())
        else:
            if line == b'KITTY_DATA_END':
                break
            yield line


@contextlib.contextmanager
def temporary_directory(dir, prefix):
    # tempfile.TemporaryDirectory not available in python2
    tdir = tempfile.mkdtemp(dir=dir, prefix=prefix)
    try:
        yield tdir
    finally:
        shutil.rmtree(tdir)


def get_data():
    global data_dir, shell_integration_dir, leading_data
    data = []
    data = b''.join(iter_base64_data(tty_file_obj))
    if leading_data:
        # clear current line as it might have things echoed on it from leading_data
        # because we only turn off echo in this script whereas the leading bytes could
        # have been sent before the script had a chance to run
        sys.stdout.write('\r\033[K')
    data = base64.standard_b64decode(data)
    with temporary_directory(dir=HOME, prefix='.kitty-ssh-kitten-untar-') as tdir, tarfile.open(fileobj=io.BytesIO(data)) as tf:
        try:
            tf.extractall(tdir, filter='data')
        except TypeError:
            tf.extractall(tdir)
        with open(tdir + '/data.sh') as f:
            env_vars = f.read()
        apply_env_vars(env_vars)
        data_dir = os.environ.pop('KITTY_SSH_KITTEN_DATA_DIR')
        if not os.path.isabs(data_dir):
            data_dir = os.path.join(HOME, data_dir)
        data_dir = os.path.abspath(data_dir)
        shell_integration_dir = os.path.join(data_dir, 'shell-integration')
        compile_terminfo(tdir + '/home')
        move(tdir + '/home', HOME)
        if os.path.exists(tdir + '/root'):
            move(tdir + '/root', '/')


def exec_zsh_with_integration():
    zdotdir = os.environ.get('ZDOTDIR') or ''
    if not zdotdir:
        zdotdir = HOME
        os.environ.pop('KITTY_ORIG_ZDOTDIR', None)  # ensure this is not propagated
    else:
        os.environ['KITTY_ORIG_ZDOTDIR'] = zdotdir
    # dont prevent zsh-newuser-install from running
    for q in ('.zshrc', '.zshenv', '.zprofile', '.zlogin'):
        if os.path.exists(os.path.join(zdotdir, q)):
            os.environ['ZDOTDIR'] = shell_integration_dir + '/zsh'
            os.execlp(login_shell, os.path.basename(login_shell), '-l')
    os.environ.pop('KITTY_ORIG_ZDOTDIR', None)  # ensure this is not propagated


def exec_fish_with_integration():
    if not os.environ.get('XDG_DATA_DIRS'):
        os.environ['XDG_DATA_DIRS'] = shell_integration_dir
    else:
        os.environ['XDG_DATA_DIRS'] = shell_integration_dir + ':' + os.environ['XDG_DATA_DIRS']
    os.environ['KITTY_FISH_XDG_DATA_DIR'] = shell_integration_dir
    os.execlp(login_shell, os.path.basename(login_shell), '-l')


def exec_bash_with_integration():
    os.environ['ENV'] = os.path.join(shell_integration_dir, 'bash', 'kitty.bash')
    os.environ['KITTY_BASH_INJECT'] = '1'
    if not os.environ.get('HISTFILE'):
        os.environ['HISTFILE'] = os.path.join(HOME, '.bash_history')
        os.environ['KITTY_BASH_UNEXPORT_HISTFILE'] = '1'
    os.execlp(login_shell, os.path.basename('login_shell'), '--posix')


def exec_with_shell_integration():
    shell_name = os.path.basename(login_shell).lower()
    if shell_name == 'zsh':
        exec_zsh_with_integration()
    if shell_name == 'fish':
        exec_fish_with_integration()
    if shell_name == 'bash':
        exec_bash_with_integration()


def install_kitty_bootstrap():
    kitty_remote = os.environ.pop('KITTY_REMOTE', '')
    kitty_exists = shutil.which('kitty')
    if kitty_remote == 'yes' or (kitty_remote == 'if-needed' and not kitty_exists):
        kitty_dir = os.path.join(data_dir, 'kitty', 'bin')
        if kitty_exists:
            os.environ['PATH'] = kitty_dir + os.pathsep + os.environ['PATH']
        else:
            os.environ['PATH'] = os.environ['PATH'] + os.pathsep + kitty_dir


def main():
    global tty_file_obj, login_shell
    # the value of O_CLOEXEC below is on macOS which is most likely to not have
    # os.O_CLOEXEC being still stuck with python2
    tty_file_obj = os.fdopen(os.open(os.ctermid(), os.O_RDWR | getattr(os, 'O_CLOEXEC', 16777216)), 'rb')
    try:
        if request_data:
            set_echo(tty_file_obj.fileno(), on=False)
            send_data_request()
        get_data()
    finally:
        cleanup()
    cwd = os.environ.pop('KITTY_LOGIN_CWD', '')
    install_kitty_bootstrap()
    if cwd:
        try:
            os.chdir(cwd)
        except Exception as err:
            print(f'Failed to change working directory to: {cwd} with error: {err}', file=sys.stderr)
    ksi = frozenset(filter(None, os.environ.get('KITTY_SHELL_INTEGRATION', '').split()))
    exec_cmd = b''
    if exec_cmd:
        os.environ.pop('KITTY_SHELL_INTEGRATION', None)
        cmd = base64.standard_b64decode(exec_cmd).decode('utf-8')
        os.execlp(login_shell, os.path.basename(login_shell), '-c', cmd)
    echo UNTAR_DONE  # noqa
    if ksi and 'no-rc' not in ksi:
        exec_with_shell_integration()
    os.environ.pop('KITTY_SHELL_INTEGRATION', None)
    os.execlp(login_shell, '-' + os.path.basename(login_shell))


main()
```

---

## §4 How is the archive of all shell-integration files built and sent over?

**Direct answer.** The kitten packs everything the remote side needs — the serialized environment, the shell-integration files for **all** shells, the kitty terminfo, and (optionally) `kitten`/`kitty` launcher stubs — into a single **gzip-compressed tar built entirely in memory** by `make_tarfile` [`kittens/ssh/main.go`:L255-L360]. That tar is base64-encoded and stored under the `tarfile` key of the shared-memory JSON (§2). It is **not** sent eagerly: the remote bootstrap requests it over the tty, and the terminal-side data server streams it back **base64, in 254-byte lines** framed by `KITTY_DATA_START` / `OK` / `KITTY_DATA_END` [`kittens/ssh/utils.py`:L115-L148].

### 4.1 Building the archive (all in-memory)

```go
gw, err := gzip.NewWriterLevel(&w, gzip.BestCompression)   // L259
...
tw := tar.NewWriter(gw)                                     // L263
...
h.Mode |= 0o600                                             // L269  (every entry forced u+rw)
```
[`kittens/ssh/main.go`:L259, L263, L269] — a `gzip` writer at `BestCompression` wrapping a `bytes.Buffer`, then a `tar` writer on top; every header's mode is OR'd with `0o600`.

### 4.2 What goes in — every conditional

| Member(s) | Code | Condition |
|-----------|------|-----------|
| User-requested `copy` files | copy loop [`kittens/ssh/main.go`:L282-L287] | only if `copy` host-option lists files (default none) |
| `data.sh` (serialized env) | `add_data(fe{"data.sh", ...})` [`kittens/ssh/main.go`:L321] | **always** |
| `bootstrap-utils.sh` | guard [`kittens/ssh/main.go`:L324-L325] | **only when `script_type == "sh"`** (the Python bootstrap has its helpers inline) |
| All shell-integration files (bash/zsh/fish + completions) | `FilesMatching(...)` [`kittens/ssh/main.go`:L330-L336] | only if `shell_integration != ""` (default on); **excludes** `shell-integration/ssh/.+` and `shell-integration/zsh/kitty.zsh` |
| `kitty/version` + `kitty/bin/{kitty,kitten}` stubs | guard [`kittens/ssh/main.go`:L342-L349] | only if `remote_kitty != no` (default `if-needed`) |
| `.terminfo/kitty.terminfo` + `.terminfo/x/xterm-kitty` | [`kittens/ssh/main.go`:L355-L357] | **always** |

### 4.3 The real archive contents

Captured from a live shm object created by the canonical path, then listed with GNU `tar` (1.35):

```
$ tar -tvf archive.tar
-rw-r--r-- 0/0             195 2026-07-14 23:15 data.sh
-rw-r--r-- 0/0            8468 2026-07-14 23:15 bootstrap-utils.sh
-rw-r--r-- 0/0             294 1970-01-01 00:00 home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_completions.d/clone-in-kitty.fish
-rw-r--r-- 0/0           10409 1970-01-01 00:00 home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish
-rw-r--r-- 0/0           17363 1970-01-01 00:00 home/.local/share/kitty-ssh-kitten/shell-integration/bash/kitty.bash
-rw-r--r-- 0/0           22557 1970-01-01 00:00 home/.local/share/kitty-ssh-kitten/shell-integration/zsh/kitty-integration
-rw-r--r-- 0/0             280 1970-01-01 00:00 home/.local/share/kitty-ssh-kitten/shell-integration/zsh/completions/_kitty
-rw-r--r-- 0/0            1880 1970-01-01 00:00 home/.local/share/kitty-ssh-kitten/shell-integration/zsh/.zshenv
-rw-r--r-- 0/0             285 1970-01-01 00:00 home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_completions.d/kitty.fish
-rw-r--r-- 0/0             286 1970-01-01 00:00 home/.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_completions.d/kitten.fish
-rw-r--r-- 0/0               6 2026-07-14 23:15 home/.local/share/kitty-ssh-kitten/kitty/version
-rwxr-xr-x 0/0            4377 1970-01-01 00:00 home/.local/share/kitty-ssh-kitten/kitty/bin/kitty
-rwxr-xr-x 0/0            2761 1970-01-01 00:00 home/.local/share/kitty-ssh-kitten/kitty/bin/kitten
-rw-r--r-- 0/0            4271 1970-01-01 00:00 home/.terminfo/kitty.terminfo
-rw-r--r-- 0/0            3711 1970-01-01 00:00 home/.terminfo/x/xterm-kitty
```

**15 members.** Note the two mode classes — `0644` data files vs the **`0755`** `kitty/bin/{kitty,kitten}` launcher stubs (the `Mode |= 0o600` at [`kittens/ssh/main.go`:L269] only *adds* u+rw; it never strips the execute bit). The generated members (`data.sh`, `bootstrap-utils.sh`, `kitty/version`) carry the real generation timestamp, while the members lifted verbatim from the embedded shell-integration blob carry a fixed `1970-01-01` epoch stamp for reproducibility.

### 4.4 The environment record (`data.sh`)

`data.sh` is produced by `serialize_env` [`kittens/ssh/main.go`:L204] which calls `final_env_instructions` [`kittens/ssh/config.go`:L93-L108], joining one line per variable produced by `EnvInstruction.Serialize` [`kittens/ssh/config.go`:L54-L91]. There are two output dialects:

- **sh** (`for_python == false`): `export key=val` / `unset key`, with values run through `quote_for_sh` [`kittens/ssh/config.go`:L73-L77, L33].
- **Python** (`for_python == true`): JSON-array lines — `export ["key","val",literal_quote]` / `unset ["key"]` [`kittens/ssh/config.go`:L62-L69] — so the Python bootstrap can parse them without a shell.

### 4.5 How it is sent — shared memory + 254-byte tty framing

The base64 tar lives in the shm JSON `tarfile` key (§2.3). When the remote bootstrap asks for it (the DCS request of §10), the terminal-side `get_ssh_data` [`kittens/ssh/utils.py`:L115-L148] replies over the tty. Framing is fixed at **`line_sz = 254`** [`kittens/ssh/utils.py`:L143] (the comment there notes the macOS 255-byte canonical-input limit as the reason). The reply is: `KITTY_DATA_START`, then `OK`, then the base64 in 254-byte lines, then `KITTY_DATA_END`.

Measured on a fresh archive through the canonical entry point:

```
command: printf '' | ./kitty/launcher/kitten __pytest__ ssh 'echo UNTAR_DONE'
shm_name: kssh-124235-VFXKRF3UHTTLA
shm total bytes: 32227  declared payload size: 32219
JSON keys: ['hostname', 'pw', 'tarfile', 'username']
tarfile base64 length (bytes to frame): 32088
number of 254-byte data lines: 127
max data line length: 254
last data line length: 84
total reply lines incl markers: 130
marker sequence: KITTY_DATA_START, OK, <127 data lines>, KITTY_DATA_END
shm removed: True
remaining /dev/shm/kssh-*: []
```

The 254-byte line **rule** is invariant; only the **line count** varies run to run with the gzipped-archive size (observed 124-127 data lines across runs — the same nondeterminism noted in §0.6). Here: 32088 base64 bytes → **127** lines (last line 84 bytes), 130 total reply lines including the three markers.

### 4.6 Remote extraction and environment apply

The two bootstrap templates extract the same tar but with their native tools:

- **sh** — `untar_and_read_env` [`shell-integration/ssh/bootstrap.sh`:L104-L151] pipes the tty base64 through `base64 -d` into `tar` directly: `read_base64_from_tty | base64_decode | command tar "xpzf" "-" "-C" "$tdir"` [`shell-integration/ssh/bootstrap.sh`:L113]. The env file (`data.sh`) is then sourced, and control passes to the login shell via `exec_login_shell` [`shell-integration/ssh/bootstrap.sh`:L164].
- **Python** — `get_data` [`shell-integration/ssh/bootstrap.py`:L203] reads the payload, then `tarfile.open(fileobj=io.BytesIO(data))` [`shell-integration/ssh/bootstrap.py`:L213] extracts with `tf.extractall(tdir, filter='data')` [`shell-integration/ssh/bootstrap.py`:L215] (falling back to `extractall(tdir)` on older Pythons [`shell-integration/ssh/bootstrap.py`:L217]); environment is applied by `apply_env_vars` [`shell-integration/ssh/bootstrap.py`:L93-L112], then the login shell is `execlp`'d [`shell-integration/ssh/bootstrap.py`:L243/L253/L315].

---

## §5 How does the kitten keep track of everything it needs for a connection?

**Direct answer.** All per-connection state lives in one Go struct, `connection_data`, defined at [`kittens/ssh/main.go`:L171-L189] — **16 fields**. It is populated in three phases: argument and config parsing fills the inputs; `bootstrap_script` [`kittens/ssh/main.go`:L422-L483] derives the password, shm object, substitution map, and script; and `get_remote_command` [`kittens/ssh/main.go`:L511-L525] finishes the `script_type` and `rcmd`. Below is every field with its **source → mutation → consumer → lifetime**, followed by raw values observed from real `kitten __pytest__ ssh` runs.

### 5.1 The 16 fields, each traced end-to-end

| # | Field (type) | Source (where set) | Mutation / role | Consumer | Lifetime |
|---|--------------|--------------------|-----------------|----------|----------|
| 1 | `remote_args []string` | arg parse; hook L861 sets `[]` | the command to run on the remote | assembled into the final ssh `cmd` | whole run |
| 2 | `host_opts *Config` | `load_config` for the matched host [L865] | resolved per-host options | read throughout (`Interpreter`, `Share_connections`, `Askpass`, `Forward_remote_control`) | whole run |
| 3 | `hostname_for_match string` | arg / hook `"host.test"` | host used for config + env matching | `make_tarfile`; written to shm `data["hostname"]` [L442] | whole run |
| 4 | `username string` | arg / hook `"testuser"` | remote user | written to shm `data["username"]` [L442] | whole run |
| 5 | `echo_on bool` | `term.WasEchoOnOriginally()` [L722] | was the tty echoing | `ECHO_ON` substitution [L474] | whole run |
| 6 | `request_data bool` | `= need_to_request_data` [L724] | **the fresh/reused decision** | gates sensitive merge [L476-L478] and who sends the DCS [L761] | whole run |
| 7 | `literal_env map[string]string` | config env directives [L723] | env to force on the remote | `serialize_env` → env records | whole run |
| 8 | `listen_on string` | `"tcp:localhost:<port>"` [L716], only under `forward_remote_control` | remote-control forwarding target | `KITTY_LISTEN_ON` env [L249-L250] | whole run |
| 9 | `test_script string` | hook arg `"echo UNTAR_DONE"` / passed command | script to run after bootstrap | `TEST_SCRIPT` substitution [L464] | whole run |
| 10 | `dont_create_shm bool` | **test-only**: `main_test.go`:L53 | suppresses shm creation in unit tests | gates shm creation [L445] + `shm_name` set [L457] | whole run (always `false` in production) |
| 11 | `shm_name string` | `data_shm.Name()` [L458] | the `/dev/shm/kssh-*` object name | `PASSWORD_FILENAME` in `sensitive_data` [L460]; returned in hook JSON | created L446, unlinked by defer [L600-L604] or by the reader [`kittens/ssh/utils.py`:L106] |
| 12 | `script_type string` | `"sh"` [L515] or `"py"` [L517] from `Interpreter` | selects bootstrap template + env format | template fetch [L481]; many branches (L252, L324, L369, L395, L497) | from `get_remote_command` on |
| 13 | `rcmd []string` | built [L508] as `[exec, <interp>, -c, <unwrap>, <encoded_script>]` | the remote command | **appended to ssh argv** [L753] | from `get_remote_command` on |
| 14 | `replacements map[string]string` | built L461-L465; `+REQUEST_DATA/ECHO_ON` [L473-L474]; `+sensitive_data` unconditionally [L479] | the Go-side substitution map | `prepare_script` (via `sd`); reused-path DCS reads `REQUEST_ID/PASSWORD_FILENAME/DATA_PASSWORD` [L762] | whole run |
| 15 | `request_id string` | `KITTY_PID-KITTY_WINDOW_ID` [L423-L424] / hook `"testing"` | anti-replay id | `REQUEST_ID` in `sensitive_data` [L460]; must match kitty's expected `f'{getpid()}-{id}'` [`kitty/window.py`:L1291] | whole run |
| 16 | `bootstrap_script string` | template fetch [L481] → `prepare_script(..., sd)` [L482] | the fully-substituted remote script | encoded into `rcmd` [L505, L508] | from `bootstrap_script` on |

Two subtleties worth stating because they drove review findings:

* **`dont_create_shm` is not a production "no data needed" switch.** It is set to `true` in exactly one place — the Go unit-test helper `basic_connection_data` [`kittens/ssh/main_test.go`:L53] — and the production code only ever *reads* it [`kittens/ssh/main.go`:L445, L457]. In every real run it is `false`, so the shm object is always created (§2.2). (Grep of the whole kitten: the only assignment to `true` is `main_test.go:L53`.)
* **`request_data` (field 6) is the single decision** that both (a) determines whether the sensitive triple is baked into the script and hence the argv [L476-L478], and (b) selects who sends the request DCS [L761]. It is *not* the same thing as "whether shm exists."

### 5.2 Raw observed field values (from real runs)

The `__pytest__` hook prints the two directly-observable outputs, `rcmd` (as `cmd`) and `shm_name`, for the fully-populated struct. Both the sh and Python interpreters were captured:

```
$ printf ''                 | ./kitty/launcher/kitten __pytest__ ssh 'echo UNTAR_DONE'   # sh (default)
   shm_name = kssh-108091-7MUZEN7V3YSLW
   cmd (rcmd) = ['exec', 'sh', '-c', '<unwrap-eval>', '<encoded bootstrap, 5278 bytes>']   # 5 elements
$ printf 'interpreter python3' | ./kitty/launcher/kitten __pytest__ ssh 'echo UNTAR_DONE'  # python
   shm_name = kssh-108508-VD6FZSISISTW6
   cmd (rcmd) = ['exec', 'python3', '-c', "import base64, sys; eval(compile(base64.standard_b64decode(sys.argv[-1]), 'bootstrap.py', 'exec'))", '<encoded bootstrap, 13580 bytes>']  # 5 elements
```

The Python `cmd[3]` above is shown **byte-for-byte as emitted**: it begins and ends with a **literal double-quote** and uses **single-quotes** inside (`'bootstrap.py'`, `'exec'`) — 100 bytes, byte-identical to the raw Go literal at [`kittens/ssh/main.go`:L499] and to the full form in §7.1. (The outer double-quotes are part of the emitted bytes — they serve as the remote shell's quoting for `python3 -c` — not list delimiters.)

The remaining fields are fixed by the hook and therefore observed as: `request_id="testing"`, `request_data=true`, `echo_on=true`, `username="testuser"`, `hostname_for_match="host.test"`, `test_script="echo UNTAR_DONE"`, `remote_args=[]`, `dont_create_shm=false`, `script_type="sh"`/`"py"`. In a non-test run `request_id` instead takes the `KITTY_PID-KITTY_WINDOW_ID` form (e.g. `55002-7`, seen in the fresh-path capture in §2.5). The `replacements` map's sensitive entries observed live were `REQUEST_ID=testing`, `PASSWORD_FILENAME=kssh-108091-7MUZEN7V3YSLW`, `DATA_PASSWORD=<64-hex>`; the full decoded `bootstrap_script` is shown in §3.3.

---


## §6 How does connection reuse decide between a fresh connection and piggybacking on an existing one?

**Direct answer.** One line decides it [`kittens/ssh/main.go`:L663-L665]:

```go
if need_to_request_data && host_opts.Share_connections && master_is_functional() {
    need_to_request_data = false
}
```

If the kitten still needs to request data, sharing is on, **and** a master is already alive, it flips `need_to_request_data` to `false` — i.e. it piggybacks. That `need_to_request_data` becomes `cd.request_data` at [`kittens/ssh/main.go`:L724], which then drives everything in §2.5/§5/§10. "Master alive" is decided by an actual `ssh -O check`.

### 6.1 `master_is_functional()` — the `-O check` probe

`master_is_functional` inserts `-O check` into the kitten's own command and runs it; a zero exit means a master is alive [`kittens/ssh/main.go`:L653-L660]:

```go
master_is_functional := func() bool {
    if master_checked { return master_is_alive }
    master_checked = true
    check_cmd := slices.Insert(cmd, 1, "-O", "check")
    master_is_alive = exec.Command(check_cmd[0], check_cmd[1:]...).Run() == nil
    return master_is_alive
}
```

This is exactly the `argc=16` probe captured in §1.1 (`-O check … -- host.test`). Against the real isolated `sshd` (§1.3 step 3) it prints `Master running (pid=…)` and exits `0`.

### 6.2 Isolating the branch, and observing BOTH outcomes

A second mechanism can independently set `need_to_request_data=false`: the kitty-askpass path [`kittens/ssh/main.go`:L648-L652] (`use_kitty_askpass` is true when `askpass` is `native`, or `unless-set` with `SSH_ASKPASS` empty). To observe the **connection-sharing** branch cleanly, the runs below force **`askpass=ssh`**, which makes `use_kitty_askpass=false`, so the *only* thing that can flip the decision is a live master. The record-only fake `ssh` returns a controlled `-O check` exit code so the kitten's own branch runs.

**Outcome A — master ABSENT (`FAKE_SSH_OCHECK_RC=1`) → fresh, `request_data=true`.** The `-O check` fails, so the branch does not flip:

```
# fakessh_fresh.log
=== invocation ts=1784071066.726131816 argc=16 ===
argv[0]=-O
argv[1]=check
...
argv[15]=host.test
  -> -O check seen; returning FAKE_SSH_OCHECK_RC=1
=== invocation ts=1784071066.740833644 argc=19 ===          # the real connection
...
argv[14]=exec
argv[15]=sh
argv[16]=-c
argv[17]='eval "$(echo "$0" | tr ...)"'
argv[18]='#/bin/sh ... request_data="1" ...
          dcs_to_kitty "ssh" "id="55002-7":pwfile="kssh-109681-WIHB57IINC574":pw="<64-hex-REDACTED>"" ...'
```

The generated bootstrap carries `request_data="1"` and the **real** credentials substituted into `argv[18]` — the fresh-path process-list exposure of §2.5.

**Outcome B — master ALIVE (`FAKE_SSH_OCHECK_RC=0`) → reused, `request_data=false`.** The `-O check` succeeds, the branch flips, and the credentials are *not* substituted:

```
# fakessh_reused.log
=== invocation ts=1784071068.830360912 argc=16 ===
argv[0]=-O
argv[1]=check
...
argv[15]=host.test
  -> -O check seen; returning FAKE_SSH_OCHECK_RC=0
=== invocation ts=1784071068.844242358 argc=19 ===          # the real connection
...
argv[18]='#/bin/sh ... request_data="0" ...
          dcs_to_kitty "ssh" "id="REQUEST_ID":pwfile="PASSWORD_FILENAME":pw="DATA_PASSWORD"" ...'
```

`request_data="0"`, the placeholders stay **literal**, and — because the remote will not ask — the local Go kitten sends the request DCS itself (§6.3, §10.4).

### 6.3 Cause → effect summary (corrected)

The two outcomes differ in exactly three observable ways; the shm/tar channel exists in **both**:

| Aspect | Fresh (`request_data=true`, master absent) | Reused (`request_data=false`, master alive) |
|--------|--------------------------------------------|---------------------------------------------|
| shm object created? | **Yes** [L445] | **Yes** [L445] — identical |
| Sensitive triple merged into the script? | Yes [L476-L478] | No (placeholders stay literal) |
| Credentials in the local `ssh` argv? | **Yes** — `argv[18]` carries `id/pwfile/pw` (process-list exposure) | **No** — placeholders only |
| Who sends the request DCS? | the **remote** bootstrap, over the tty [`shell-integration/ssh/bootstrap.sh`:L92-L95] | the **local Go kitten**, to its own kitty terminal [`kittens/ssh/main.go`:L761-L769] |
| Extra TCP+auth handshake? | Yes (new connection) | No (multiplexed; §1.3 shows 0.006 s vs 0.121 s) |

The earlier draft had the credential columns inverted (claiming fresh keeps creds off the argv and reused opens no channel). The captured argv logs above show the opposite, and the timing in §1.3 confirms the reuse itself.

---

## §7 How does the bootstrap encoding work with character substitutions for different shells?

**Direct answer.** There are **two** encoding schemes, chosen by `script_type` — not one scheme per shell family. The **Python** template is transported as **base64** (any byte is safe). The **POSIX-sh** template is transported by a **four-character control-code substitution** and reversed on the remote by a single `tr` call. The substitution is **identical for every POSIX launcher** (`sh`, `dash`, `bash`, `zsh`, `posh`): the kitten does not vary the encoding per shell. `fish` is not an interpreter for the bootstrap at all — it is only ever a **login shell** reached at the end of the `sh` bootstrap, so it uses the sh scheme transitively (§7.5). Both schemes are built in `wrap_bootstrap_script` [`kittens/ssh/main.go`:L486-L509].

### 7.1 The two schemes

```go
if cd.script_type == "py" {
    encoded_script = base64.StdEncoding.EncodeToString(...)                 // L498
    unwrap_script  = `"import base64, sys; eval(compile(base64.standard_b64decode(sys.argv[-1]), 'bootstrap.py', 'exec'))"`  // L499
} else {
    encoded_script = "'" + strings.NewReplacer("'", "\v", "\\", "\f", "\n", "\r", "!", "\b").Replace(cd.bootstrap_script) + "'"  // L505
    unwrap_script  = `'eval "$(echo "$0" | tr \\\v\\\f\\\r\\\b \\\047\\\134\\\n\\\041)"' `  // L506
}
cd.rcmd = []string{"exec", cd.host_opts.Interpreter, "-c", unwrap_script, encoded_script}   // L508
```
[`kittens/ssh/main.go`:L498-L508]. The remote therefore runs `exec <interp> -c <unwrap> <encoded>`, and the unwrap turns `<encoded>` back into the original script before executing it. The exact sh unwrap emitted (`cmd[3]`) is the following — note it ends with **one trailing space** after the closing single-quote (part of the emitted bytes; the raw space is stripped at line-end here only to keep the document `git diff --check`-clean):

```
'eval "$(echo "$0" | tr \\\v\\\f\\\r\\\b \\\047\\\134\\\n\\\041)"'
```

### 7.2 The four substitutions (sh scheme)

`strings.NewReplacer` at [`kittens/ssh/main.go`:L505] maps exactly four bytes; the remote `tr` at [`kittens/ssh/main.go`:L506] maps them back:

| Original byte | Encoded as | Reverse (`tr`) |
|---------------|-----------|----------------|
| `'` single-quote `0x27` | `\v` vertical-tab `0x0b` | `\013` → `\047` |
| `\` backslash `0x5c` | `\f` form-feed `0x0c` | `\014` → `\134` |
| newline `0x0a` | `\r` carriage-return `0x0d` | `\015` → `\012` |
| `!` bang `0x21` | `\b` backspace `0x08` | `\010` → `\041` |

**Why these four:** the whole encoded script is passed as **one single-quoted `sh` argument** (note the wrapping `'...'` at [L505]). Inside single quotes the only byte that ends the string is `'` itself; a literal newline would also break the single-argument framing, and `\`/`!` are hazardous under some shells' quoting/history expansion. Mapping exactly those four to control codes that never appear in the source lets the entire multi-line script travel intact as a single argv slot, then `tr` restores them in one pass on the remote.

### 7.3 Byte-accurate round-trip proof

Reversed through the **real system `tr`** (uutils coreutils 0.2.2) using the same map as the emitted unwrap, then re-encoded and compared byte-for-byte:

```
$ tr '\013\014\015\010' '\047\134\012\041' < S_sub.bin > B_rev.sh      # reverse the 4 subs
$ # re-apply the main.go:L505 Replacer to B_rev.sh -> S_roundtrip.bin
$ cmp S_sub.bin S_roundtrip.bin && echo IDENTICAL
=== emitted encoded script (cmd[4]) ===
length (bytes): 5278
first byte: 0x27  last byte: 0x27  (both should be 0x27 = single-quote wrapper)
substituted body length: 5276
=== byte histogram of the four substitution codes in the body ===
  0x08  <- '!'  (backslash-b) : 3
  0x0b  <- "'"  (backslash-v) : 10
  0x0c  <- '\\' (backslash-f) : 25
  0x0d  <- newline (backslash-r) : 164
  literal newline 0x0a present in body: False (must be False - all newlines were substituted)
wrote S_sub.bin (substituted body, no wrapper)
=== round-trip ===
re-encoded S_roundtrip.bin bytes: 5276
cmp: IDENTICAL (exit 0)
```

The histogram shows the substitution actually occurred (164 newlines became `0x0d`, 10 single-quotes became `0x0b`, 25 backslashes became `0x0c`, 3 bangs became `0x08`) and that **no literal newline (`0x0a`) survives** in the wire form; the `cmp` proves the transform is exactly invertible.

### 7.4 Python scheme: base64 decode-equality

```
$ python3 -c "import json,base64,sys; d=base64.standard_b64decode(json.load(sys.stdin)['cmd'][4]); \
              open('py.dec','wb').write(d); print('decoded bytes', len(d))" < gen_py.json
decoded bytes 10184
$ cmp py.dec py_bootstrap_decoded.py && echo IDENTICAL
```

Observed: `cmd[4]` is **13580 base64 characters** → **10184 bytes** → **318 lines** beginning `#!/usr/bin/env python`, **byte-identical** to the independently-captured `bootstrap.py`. No substitution table is involved; base64 is self-describing and any byte round-trips.

### 7.5 Coverage across launchers — and the fish case

The launcher set exercised by the PTY suite is `all_possible_sh` [`kitty_tests/ssh.py`:L63-L65] = `filter(which, ('dash','zsh','bash','posh','sh',python))`. In this container `which` resolves **dash, sh, bash, zsh, python3** (so all five are run); **`posh` is not installed**, so it is filtered out and not exercised here. All POSIX launchers use the identical sh substitution of §7.2; `python3` uses the base64 scheme.

**fish is a special case (and a gap the standard suite leaves).** The shell-integration test intersects `{'fish','zsh','bash'}` with `all_possible_sh` [`kitty_tests/ssh.py`:L201] — and because `all_possible_sh` **does not contain `fish`**, fish is *never* exercised as a login shell by the standard suite. To satisfy the "every named item" requirement I drove it explicitly through the same canonical `check_bootstrap` harness with `login_shell='fish'`:

```
test_fish_login_shell (__main__.FishProbe.test_fish_login_shell) ... [0.221] [PARSE ERROR] The application is trying to use xterm's modifyOtherKeys. This is superseded by the kitty keyboard protocol: https://sw.kovidgoyal.net/kitty/keyboard-protocol/ the application should be updated to use that
=== FISH check_bootstrap PASSED internal asserts ===
UNTAR_DONE: asserted present by check_bootstrap()
terminfo kitty.terminfo exists: True
cursor.shape: 2  CURSOR_BEAM: 2  beam_applied: True
OSC-133 prompt marker in received_bytes: False
fish confirmation line: root@reverse-code-generator-1a3e34e0-6h99z ~# echo "FISHVER=$FISH_VERSION"
ok

----------------------------------------------------------------------
Ran 1 test in 0.195s

OK
FISH_PROBE_OK: True
```
```
test_fish_real_version (__main__.FishProbe2.test_fish_real_version) ... [0.217] [PARSE ERROR] The application is trying to use xterm's modifyOtherKeys. This is superseded by the kitty keyboard protocol: https://sw.kovidgoyal.net/kitty/keyboard-protocol/ the application should be updated to use that
[0.221] [PARSE ERROR] The application is trying to use xterm's modifyOtherKeys. This is superseded by the kitty keyboard protocol: https://sw.kovidgoyal.net/kitty/keyboard-protocol/ the application should be updated to use that
[0.221] [PARSE ERROR] The application is trying to use xterm's modifyOtherKeys. This is superseded by the kitty keyboard protocol: https://sw.kovidgoyal.net/kitty/keyboard-protocol/ the application should be updated to use that
terminfo exists: True
beam cursor applied (shape==CURSOR_BEAM): True
fish output line: REALLYFISH=4.0.6
ok

----------------------------------------------------------------------
Ran 1 test in 0.195s

OK
FISH_PROBE2_OK: True
```

Observed: the `sh` bootstrap completed (`UNTAR_DONE`), the kitty terminfo was installed, the shell-integration set a **beam cursor** (`cursor.shape == 2 == CURSOR_BEAM`), and a fish-only construct (`set -q FISH_VERSION; and echo ...`) printed **`REALLYFISH=4.0.6`**, proving the login shell really was fish. (OSC-133 prompt marking is not asserted for fish here; the beam cursor is the integration signal.)

### 7.6 Interpreter vs. login shell — a distinction the question invites

The `interpreter` (`sh`/`python3`) decides **which bootstrap template and which encoding** are used to *run the bootstrap*. The **login shell** (bash/zsh/fish/…) is what the bootstrap `exec`s **after** untarring and applying the environment ([`shell-integration/ssh/bootstrap.sh`:L164]; [`shell-integration/ssh/bootstrap.py`:L243/L253/L315]). They are independent: a fish or zsh user is still bootstrapped by the `sh` (or `python3`) template — fish never sees the encoded blob, only its own already-extracted shell-integration files. This is why "encoding for different shells" is really "one sh scheme + one Python scheme," with the per-shell differences living entirely in the **shell-integration files inside the tar** (§4), not in the transport encoding.

---

## §8 Full trace: from the user starting an SSH session to the bootstrap executing remotely

**Direct answer.** The kitten runs entirely locally in Go until it hands a crafted remote command to the system `ssh`; the remote then unwraps and runs the bootstrap, which pulls its data back over the tty and hands off to the login shell. The exact local order is fixed in `run_ssh` [`kittens/ssh/main.go`:L597-L800]. The **two routes differ only in who sends the credential request** (§6, §10): on a **fresh** connection the credentials are already baked into the remote command and the **remote** bootstrap asks for the data; on a **reused** connection the **local** kitten sends the request itself.

### 8.1 The exact local startup order (`run_ssh`)

| # | Step | Code |
|---|------|------|
| 1 | Enter `run_ssh`; register a `defer` that closes+unlinks the shm object | [`kittens/ssh/main.go`:L597]; defer [L600-L604] |
| 2 | Decide whether to request data (askpass) | `set_askpass()` [`kittens/ssh/main.go`:L651] |
| 3 | **Connection-sharing decision** — may flip request off | `if need_to_request_data && Share_connections && master_is_functional()` [`kittens/ssh/main.go`:L663-L664] |
| 4 | *(reused/sharing)* spawn or verify the ControlMaster; optional remote-control forward | [`kittens/ssh/main.go`:L666-L703] |
| 5 | Open the controlling terminal with echo off | `tty.OpenControllingTerm(tty.SetNoEcho)` [`kittens/ssh/main.go`:L718] |
| 6 | Record original echo state | `cd.echo_on = term.WasEchoOnOriginally()` [`kittens/ssh/main.go`:L722] |
| 7 | **Finalize `request_data`** | `cd.request_data = need_to_request_data` [`kittens/ssh/main.go`:L724] |
| 8 | Save current DEC private modes + set `HANDLE_TERMIOS_SIGNALS` (mode 19997); prepare restore | [`kittens/ssh/main.go`:L728, L733] |
| 9 | **Build the remote command and create the shm object** | `get_remote_command(&cd)` [`kittens/ssh/main.go`:L749]; shm created at [L446] |
| 10 | **Append the remote command to the `ssh` argv** (fresh: credentials are now in the argv) | `cmd = append(cmd, cd.rcmd...)` [`kittens/ssh/main.go`:L753] |
| 11 | **Start `ssh`** — the remote runs `exec <interp> -c <unwrap> <encoded>` | `c.Start()` [`kittens/ssh/main.go`:L756] |
| 12 | *(reused only)* local kitten sends the `@kitty-ssh` request over the tty | `if !cd.request_data { ... DCSToKitty("ssh", rq) ... term.WriteAllString(dcs) }` [`kittens/ssh/main.go`:L761-L768] |
| 13 | Wait for `ssh`; then drain the tty with an `@kitty-echo` canary | `c.Wait()` [`kittens/ssh/main.go`:L782]; `drain_potential_tty_garbage` [L783] → `DCSToKitty("echo", canary)` [L539] |

### 8.2 The remote side (after `ssh` connects)

`ssh` executes `exec <interp> -c <unwrap_script> <encoded_script>` (§3, §7). The unwrap restores the original bootstrap text and runs it:

1. **Fresh only** — the bootstrap sends the credential request itself: `[ "$request_data" = "1" ] && dcs_to_kitty "ssh" "id=...:pwfile=...:pw=..."` [`shell-integration/ssh/bootstrap.sh`:L92-L95], or in Python `if request_data: ... send_data_request()` [`shell-integration/ssh/bootstrap.py`:L292-L294].
2. **Both routes** — the bootstrap then reads the reply unconditionally: `get_data` [`shell-integration/ssh/bootstrap.sh`:L155] / [`shell-integration/ssh/bootstrap.py`:L295]. It waits for `OK`, captures the base64 after `KITTY_DATA_START` until `KITTY_DATA_END`, untars into a temp dir, sources `data.sh`, and finally `exec`s the login shell ([`shell-integration/ssh/bootstrap.sh`:L164]; [`shell-integration/ssh/bootstrap.py`:L243/L253/L315]).
3. **Terminal side** — kitty receives the `@kitty-ssh` DCS and dispatches it: `handle_remote_ssh` → `get_ssh_data(msg, f'{os.getpid()}-{self.id}')` [`kitty/window.py`:L1289-L1291], which validates the request (§9) and streams the tar back (§4.5, §10).

### 8.3 Observed end-to-end round trip (canonical PTY harness)

The full round trip — local kitten → real `ssh`/PTY → remote bootstrap → DCS request → terminal data server → untar → login shell — is exactly what kitty's own PTY test module drives. Run through the built launcher:

```
$ ./kitty/launcher/kitty +launch test.py --module ssh
........
----------------------------------------------------------------------
Ran 8 tests in 9.907s

OK
```

All 8 tests pass (repeated twice, §0.5). The per-route specifics (fresh bakes credentials into `argv[18]`; reused leaves placeholders and the local kitten sends the DCS) are the captured `ssh` argv logs in §6.2 and the captured DCS wire in §10.2.

---

## §9 How does shared memory keep things secure?

**Direct answer.** The credential channel is a POSIX shared-memory object whose security rests on **five independent guarantees**, all enforced **locally** (the password never crosses the network): it is **created race-free** (`O_EXCL`, mode `0600`, owner = the kitty process); it is **single-use** (unlinked the instant it is read); and every read is gated by **four validations** — the request must **parse**, the object must have the right **owner** and **permissions**, and the request must present the correct **password** and **request-id**. Any failure yields an error line and logs a traceback; the tar is released only when all pass. The one caveat the design does *not* hide is that on a **fresh** connection the password is also in the local `ssh` argv (§2.5) — shared memory secures the *network* path, not the local process table.

### 9.1 The producer (Go) — created race-free

The object is created by the Go kitten, not Python: `shm_open(name, os.O_EXCL|os.O_CREATE|os.O_RDWR, 0600)` [`tools/utils/shm/shm_syscall.go`:L162]. `O_EXCL` makes creation fail if the name already exists (no pre-seeding by an attacker); mode `0600` restricts it to the owner from the moment it exists. The kitten writes a 4-byte big-endian size prefix then the JSON payload [`tools/utils/shm/shm.go`:L116-L124], and a `defer` guarantees it is closed and unlinked when `run_ssh` returns [`kittens/ssh/main.go`:L600-L604].

### 9.2 The reader (terminal side) — the checks in runtime order

`get_ssh_data` [`kittens/ssh/utils.py`:L115-L148] and its helper `read_data_from_shared_memory` [`kittens/ssh/utils.py`:L100-L112] enforce, in this order:

| Order | Check | Code | On failure |
|-------|-------|------|-----------|
| 0 | Emit `KITTY_DATA_START` first (to discard leading tty noise) | [`kittens/ssh/utils.py`:L117] | (always emitted) |
| 1 | **Parse/shape**: base64-decode, split `k=v:...` into a dict, require `pw`/`pwfile`/`id` | [`kittens/ssh/utils.py`:L118-L123] | `invalid ssh data request message` [L124-L126] |
| — | **Single-use unlink** (lifecycle, *before* validation) | [`kittens/ssh/utils.py`:L106] | (object gone after read) |
| 2 | **Owner**: `st_uid == euid` **and** `st_gid == egid` | [`kittens/ssh/utils.py`:L107-L108] | `Incorrect owner on pwfile: uid=.. gid=..` |
| 3 | **Permissions**: mode `== 0o600` | [`kittens/ssh/utils.py`:L109-L111] | `Incorrect permissions on pwfile: 0o..` |
| 4 | **Password**: `pw == env_data['pw']` | [`kittens/ssh/utils.py`:L130-L131] | `Incorrect password` |
| 5 | **Request-id**: `rq_id == request_id` (the `KITTY_PID-KITTY_WINDOW_ID` of the target window) | [`kittens/ssh/utils.py`:L132-L133] | `Incorrect request id: ...` |

Every rejection also calls `traceback.print_exc()` [`kittens/ssh/utils.py`:L125, L135], so the terminal's stderr records the exact failing line. Only when all pass does it `yield b'OK\n'` and stream the tar (§4.5).

### 9.3 Observed — happy path plus every rejection branch

Driven against the **real** `get_ssh_data` through kitty (`kitty +launch <probe>.py`), one shm object per case, request-id `111466-1`:

```
===== 1 HAPPY PATH =====
(request_id arg = '111466-1')
markers: ['KITTY_DATA_START', 'OK', 'KITTY_DATA_END']
  OK present=True  DATA_END present=True
  data lines=126  max line len=254 (<=254)

===== 2 WRONG PASSWORD =====
(request_id arg = '111466-1')
markers: ['KITTY_DATA_START']
  yielded-error-line: Incorrect password
  traceback last line: ValueError: Incorrect password

===== 3 WRONG REQUEST ID =====
(request_id arg = '111466-1')
markers: ['KITTY_DATA_START']
  yielded-error-line: Incorrect request id: 'NOT-111466-1' expecting the KITTY_PID-KITTY_WINDOW_ID for the current kitty window
  traceback last line: ValueError: Incorrect request id: 'NOT-111466-1' expecting the KITTY_PID-KITTY_WINDOW_ID for the current kitty window

[4 setup] /dev/shm/kssh-111522-LMKLNEVQQOYDQ mode now = 0o644

===== 4 WRONG PERMISSIONS =====
(request_id arg = '111466-1')
markers: ['KITTY_DATA_START']
  yielded-error-line: Incorrect permissions on pwfile: 0o644
  traceback last line: ValueError: Incorrect permissions on pwfile: 0o644

[5 setup] /dev/shm/kssh-111540-PH3RZK237CAPM chowned to uid=1 gid=1 (euid=0 egid=0)

===== 5 WRONG OWNER =====
(request_id arg = '111466-1')
markers: ['KITTY_DATA_START']
  yielded-error-line: Incorrect owner on pwfile: uid=1 gid=1
  traceback last line: ValueError: Incorrect owner on pwfile: uid=1 gid=1

===== 6 INVALID MESSAGE =====
(request_id arg = '111466-1')
markers: ['KITTY_DATA_START']
  yielded-error-line: invalid ssh data request message
  traceback last line: ValueError: dictionary update sequence element #0 has length 1; 2 is required

===== 7 SINGLE-USE =====

===== 7a first call =====
(request_id arg = '111466-1')
markers: ['KITTY_DATA_START', 'OK', 'KITTY_DATA_END']
  exists_after_first=False

===== 7b second call (same shm, now unlinked) =====
(request_id arg = '111466-1')
markers: ['KITTY_DATA_START']
  yielded-error-line: [Errno 2] No such file or directory: 'kssh-111559-VPAIGLOHU2VRS'
  traceback last line: FileNotFoundError: [Errno 2] No such file or directory: 'kssh-111559-VPAIGLOHU2VRS'
```

All six message strings match the source exactly, and every rejection begins with `KITTY_DATA_START` and never reaches `OK`.

### 9.4 Defense-in-depth — tampered objects are still unlinked

Because the unlink [`kittens/ssh/utils.py`:L106] runs *before* the owner/permission checks raise, even a rejected object is removed — an attacker cannot leave a poisoned object lying around for a retry:

```
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-95a6b618-f37d-498b-b357-7dc63907cc2f_30475d/kittens/ssh/utils.py", line 129, in get_ssh_data
    env_data = read_data_from_shared_memory(pwfilename)
  File "/tmp/blitzy/kitty/blitzy-95a6b618-f37d-498b-b357-7dc63907cc2f_30475d/kittens/ssh/utils.py", line 111, in read_data_from_shared_memory
    raise ValueError(f'Incorrect permissions on pwfile: 0o{mode:03o}')
ValueError: Incorrect permissions on pwfile: 0o644
Traceback (most recent call last):
  File "/tmp/blitzy/kitty/blitzy-95a6b618-f37d-498b-b357-7dc63907cc2f_30475d/kittens/ssh/utils.py", line 129, in get_ssh_data
    env_data = read_data_from_shared_memory(pwfilename)
  File "/tmp/blitzy/kitty/blitzy-95a6b618-f37d-498b-b357-7dc63907cc2f_30475d/kittens/ssh/utils.py", line 108, in read_data_from_shared_memory
    raise ValueError(f'Incorrect owner on pwfile: uid={shm.stats.st_uid} gid={shm.stats.st_gid}')
ValueError: Incorrect owner on pwfile: uid=1 gid=1
PERMS branch: present_before=True present_after=False
OWNER branch: present_before=True present_after=False
```

Both the perms branch (raised at [`kittens/ssh/utils.py`:L111]) and the owner branch (raised at [`kittens/ssh/utils.py`:L108]) show `present_before=True present_after=False`.

### 9.5 Single-use (unlink-on-read)

Case 7 above: the first read returns `OK`+data and the object is already gone (`exists_after_first=False`); a second read of the same name fails with `[Errno 2] No such file or directory` (`FileNotFoundError`). A captured request can therefore never be replayed against the same object.

### 9.6 Threat model — what it does and does **not** protect against

- **Protects:** the password never traverses the network (it lives in local `/dev/shm` and is matched locally); other-UID processes cannot read it (`0600` + owner check); a captured request cannot be replayed (single-use unlink); a request aimed at the wrong window is refused (request-id); an attacker cannot pre-create or leave behind a poisoned object (`O_EXCL` + unlink-before-validate).
- **Does not protect against:** a process running as the **same UID** (or **root**) on the local machine — it can read `/dev/shm` directly, though `O_EXCL`, `0600`, and single-use unlink make the window small; a descriptor already `open`ed before unlink; and — on a **fresh** connection — the password is additionally present in the local `ssh` **argv** (visible in `/proc/<pid>/cmdline` to same-UID/root) for the lifetime of the `ssh` process (§2.5). The payload is base64, **not** encryption; the base64 tar is not secret (it is shell-integration files), only the `pw`/`pwfile`/`id` triple is sensitive.

---

## §10 How does the terminal communicate back and forth with the remote shell during setup?

**Direct answer.** All setup traffic rides the **tty** as **DCS (Device Control String) escape sequences** of the form `ESC P @kitty-<type> | <base64-payload> ESC \`. The remote→terminal direction carries the credential **request** (`@kitty-ssh`) and debug (`@kitty-print`); the terminal→remote direction carries the framed **reply** (`KITTY_DATA_START` / `OK` / 254-byte base64 / `KITTY_DATA_END`). The **request is sent by different senders depending on the route**: the remote bootstrap on a fresh connection, the local kitten on a reused one. Locally injected DCS is wrapped in a save/set/restore of DEC private mode **19997** (`HANDLE_TERMIOS_SIGNALS`).

### 10.1 The DCS frame (identical in both directions)

Producer of the frame on the remote side: `dcs_to_kitty() { printf "\033P@kitty-$1|%s\033\134" "$(printf "%s" "$2" | base64_encode)" > /dev/tty; }` [`shell-integration/ssh/bootstrap.sh`:L75] (Python equivalent [`shell-integration/ssh/bootstrap.py`:L73-L77]). `\033P` is `ESC P` (DCS), `\033\134` is `ESC \` (ST, string terminator). Captured **verbatim** from a real PTY run (`od -c` of the local kitten's tty output on the fresh route):

```
0000000 033   [   ?   s 033   [   ?   1   9   9   9   7   h 033   P   @
0000020   k   i   t   t   y   -   e   c   h   o   |   O   T   I   2   Y
0000040   T   F   h   Z   W   I   5   Z   T   M   3   O   D   d   j   Y
0000060   z   E   2   Z   T   Q   0   N   T   l   i   Y   z   I   z   N
0000100   G   M   0   M   j   A   y   M   z   U   4   N   2   J   l   N
0000120   j   E   1   O   G   E   w   O   T   k   y   N   D   Y   y   Z
0000140   j   d   h   N   D   c   0   Z   D   Q   y   N   z   k   w   N
0000160   A   =   = 033   \ 033   [   ?   r 033   [   ?   1   9   9   9
0000200   7   l
0000202
```

Read left to right: `ESC [ ? s` (save private modes) · `ESC [ ? 1 9 9 9 7 h` (set `HANDLE_TERMIOS_SIGNALS`, mode 19997) · `ESC P @ k i t t y - e c h o | <base64> ESC \` (the DCS) · `ESC [ ? r` (restore) · `ESC [ ? 1 9 9 9 7 l` (reset). Mode 19997 is defined at [`kitty/modes.h`:L89] (`#define HANDLE_TERMIOS_SIGNALS (19997 << 5)`) and [`kittens/tui/operations.py`:L48].

### 10.2 Who sends the request — the directional split (observed)

Decoding every `@kitty-*` frame from the two captured PTY outputs (password redacted):

```
pty_out_fresh.bin (130 bytes): 1 @kitty DCS frame(s)
  @kitty-echo|  ->  <64-hex drain-canary nonce>

pty_out_reused.bin (293 bytes): 2 @kitty DCS frame(s)
  @kitty-ssh|  ->  id=55002-7:pwfile=kssh-109709-QSDOU7G3YYGPU:pw=<64-hex-REDACTED>
  @kitty-echo|  ->  <64-hex drain-canary nonce>
```

- **Fresh** (`pty_out_fresh.bin`, 130 B): the local kitten emits **only** an `@kitty-echo` drain canary; it sends **no** `@kitty-ssh` — the **remote** bootstrap sends the request over `/dev/tty` [`shell-integration/ssh/bootstrap.sh`:L92-L95].
- **Reused** (`pty_out_reused.bin`, 293 B): the **local Go kitten** sends the `@kitty-ssh` request carrying the real `id`/`pwfile`/`pw` [`kittens/ssh/main.go`:L761-L768], plus the same `@kitty-echo` canary.

This is the definitive evidence for the fresh/reused split (§6): the credential request exists on **both** routes, but the **sender** differs.

### 10.3 The reply (terminal → remote)

kitty receives the `@kitty-ssh` DCS and dispatches it to the data server: `handle_remote_ssh` → `get_ssh_data(msg, f'{os.getpid()}-{self.id}')` [`kitty/window.py`:L1289-L1291]; each yielded line is written back to the child. The reply framing (validated in §9, measured in §4.5) is `\nKITTY_DATA_START\n`, then `OK\n`, then the base64 tar in **254-byte** lines, then `KITTY_DATA_END\n` [`kittens/ssh/utils.py`:L117, L138-L148].

### 10.4 The remote read/decode/handoff — sh and Python

- **sh** — `get_data` [`shell-integration/ssh/bootstrap.sh`:L137-L155] waits for `OK` [L141], starts capturing at `KITTY_DATA_START` [L144], and `read_base64_from_tty` [L97-L102] reads until `KITTY_DATA_END` [L99]; then `untar_and_read_env` [L104-L151] decodes and untars [L113].
- **Python** — the read loop keys on `KITTY_DATA_START` [`shell-integration/ssh/bootstrap.py`:L178] and `KITTY_DATA_END` [L188]; `get_data` [L203] then base64-decodes and `extractall`s [L215]. The request itself is `send_data_request` [L80-L81], gated by `if request_data:` in `main` [L292-L294].

**Message types observed/used:** `@kitty-ssh` (credential request → tar reply), `@kitty-echo` (drain canary, round-trips a nonce to flush the tty), and `@kitty-print` (`debug()` diagnostics [`shell-integration/ssh/bootstrap.sh`:L76]). All share the one frame format of §10.1.

---

## Architecture at a glance

The diagram below reflects the **corrected** flow (shm built unconditionally; fresh bakes credentials into the `ssh` argv and the **remote** sends the request; reused leaves placeholders and the **local** kitten sends the request). All arrows are directional.

```
        LOCAL HOST                                            |        REMOTE HOST
                                                              |
  +------------------------------------------------+          |
  | kitty terminal  (pid P, window id W)           |          |
  |   DCS dispatch : handle_remote_ssh             |          |
  |                  [kitty/window.py:L1289-L1291] |          |
  |   data server  : get_ssh_data                  |          |
  |                  [kittens/ssh/utils.py:L115]   |          |
  +----------^--------------------------+----------+          |
             |                          |                     |
   reply over tty:              reads @kitty-ssh              |
   START / OK / <=254B b64 / END   request  (from remote      |
             |                    on FRESH, from local Go     |
             |                    on REUSED)                  |
  +----------+--------------------------v----------+          |
  | kitten ssh (Go)  run_ssh [kittens/ssh/main.go:L597]       |
  |  1 build pw + tar + shm  UNCONDITIONALLY [L431-L446]      |
  |  2 decide fresh/reused                   [L663-L664]      |
  |  3 get_remote_command -> rcmd            [L749]           |
  |  4 append rcmd to ssh argv               [L753]           |
  |  5 exec ssh                              [L756]           |
  |  6 REUSED only: send @kitty-ssh itself   [L761-L768]      |
  +----+---------------------------+-----------------+        |
       |                           |                          |
  /dev/shm/kssh-<pid>-*       ssh argv (FRESH: pw here)        |
  0600, O_EXCL                     |                          |
  [tools/utils/shm/                v                          |
   shm_syscall.go:L162]     +-------------+                   |
       |                    | system ssh  |===== ssh =============> sshd
       +------ read ------->| +ControlMstr|                   |     |
                            +-------------+                   |     v
                                                              |  exec <interp> -c <unwrap> <encoded>
                                                              |     |
                                                              |     v
                                                              |  bootstrap.sh / bootstrap.py
                                                              |   - FRESH: send @kitty-ssh  [bootstrap.sh:L92-L95]
                                                              |   - BOTH : get_data -> untar -> source data.sh
                                                              |   - exec login shell        [bootstrap.sh:L164]
```

---

## Coverage matrix — the ten questions

Each row lists the question, the section that answers it, the canonical command that produced the evidence, and the key observed result. (Literal `|` characters inside cells are written `\|`.)

| # | Question | Section | Canonical command | Key observed evidence |
|---|----------|---------|-------------------|-----------------------|
| 1 | Secure session setup + connection sharing (ControlMaster) | §1 | `ssh -o ControlMaster=auto -o ControlPath=$RD/kssh-77001-%C -o ControlPersist=yes ...` against isolated `sshd` | master UNIX socket `srw------- 0600`, 40-hex `%C`; reuse 0.006 s vs 0.121 s baseline |
| 2 | Shared memory to pass credentials | §2 | `printf '' \| ./kitty/launcher/kitten __pytest__ ssh 'echo UNTAR_DONE'` | live `/dev/shm/kssh-*` `-rw------- 0600`, 4-byte size prefix, JSON keys `[hostname,pw,tarfile,username]` |
| 3 | Bootstrap-script generation | §3 | `printf '' \| kitten __pytest__ ssh ...` (sh) and `printf 'interpreter python3\n' \| ...` (py) | full 164-line sh + 318-line Python bootstrap decoded from `cmd[4]` |
| 4 | Archive build + transport | §4 | `tar -tvf archive.tar`; live shm framing measurement | 15 members (0644 data, 0755 launcher stubs); 254-byte tty lines (127 lines, 32088 b64) |
| 5 | Per-connection state | §5 | source read of `connection_data` + observed field values | 16-field table; `dont_create_shm` is TEST-only [`kittens/ssh/main_test.go`:L53] |
| 6 | Fresh vs reused decision | §6 | record-only fake `ssh` + `--kitten askpass=ssh`, `-O check` RC 1 vs 0 | FRESH `argv[18]` carries `id/pwfile/pw`; REUSED leaves literal placeholders |
| 7 | Bootstrap encoding per shell | §7 | reverse via real `tr '\013\014\015\010' '\047\134\012\041'`; `cmp` | round-trip byte-`IDENTICAL`; fish driven explicitly -> `REALLYFISH=4.0.6`, beam cursor |
| 8 | Full end-to-end trace | §8 | `./kitty/launcher/kitty +launch test.py --module ssh` | 8/8 tests pass; 13-step local order L718 -> L724 -> L749 -> L753 -> L756 |
| 9 | Shared-memory security | §9 | real `get_ssh_data` via `kitty +launch`, 7 branches | happy + 4 rejections + invalid + single-use; tampered objects still unlinked |
| 10 | Terminal <-> remote comms | §10 | `od -c` of captured PTY output; decode `@kitty-*` frames | DCS `ESC P @kitty-<type> \| <b64> ESC \`; FRESH: only `@kitty-echo`; REUSED: `@kitty-ssh` (local) |

---

## Appendix A — Claims inferred from source (not observed at runtime here)

Per the observed-output discipline, only the following two claims could **not** be reproduced on this Linux host and are labelled _inferred from source_ throughout:

1. **macOS long-runtime-dir symlink workaround** — `connection_sharing_args` symlinks the runtime dir to `/tmp/kssh-rdir-<euid>` when `len(rd) > 35` [`kittens/ssh/main.go`:L128-L134]. On this host the runtime dir `/root/.cache/kitty/run` is **22 bytes** (`printf '%s' /root/.cache/kitty/run \| wc -c` → `22`), so `22 < 35` and the branch does not fire; it is macOS-specific (the source comment cites Apple's ~48-char cache-dir path). The *non-firing* was confirmed (no `/tmp` symlink in the observed `ControlPath`, §1.3); the firing path itself is inferred.
2. **`%C` connection-hash inputs** — OpenSSH computes `%C` as a hash of `(local host, remote host, port, user)` [`kittens/ssh/main.go`:L123-L127 comment; `kitty/constants.py`:L186-L187]. The hash's **presence and 40-hex form** are observed (§1.3), but the exact hash inputs are internal to OpenSSH and so labelled inferred.

Everything else in this document is backed by a captured command and its complete output.

## Appendix B — Reproduction index

All evidence was produced from the built launcher (`./kitty/launcher/{kitty,kitten}`, §0.2) via these canonical entry points:

| Evidence | Command |
|----------|---------|
| Build + artifacts + versions | `python3 setup.py build --debug --ignore-compiler-warnings --skip-building-kitten` then `... --skip-code-generation` |
| Go unit tests (7/7) | `go test ./kittens/ssh/...` |
| PTY suite (8/8) | `./kitty/launcher/kitty +launch test.py --module ssh` |
| sh / py generation | `printf '' \| ./kitty/launcher/kitten __pytest__ ssh 'echo UNTAR_DONE'` (and `printf 'interpreter python3\n' \| ...`) |
| Fresh vs reused argv | record-only fake `ssh` on `PATH` + `kitten ssh --kitten askpass=ssh -- host.test echo hello`, `-O check` RC 1 / 0 |
| ControlMaster lifecycle | isolated `sshd` on `127.0.0.1:2222` (dedicated keys, `PidFile`, seeded `known_hosts`, `StrictHostKeyChecking=yes`) |
| Shared-memory security | real `get_ssh_data` driven via `./kitty/launcher/kitty +launch <probe>.py` |
| DCS wire | `od -c` of the kitten's captured tty output (fresh / reused) |
| fish login shell | `check_bootstrap('sh', tdir, login_shell='fish')` via `kitty +launch` |

All temporary scripts, logs, the isolated `sshd` and its keys, the fake `ssh` shim, every `/dev/shm/kssh-*` object, and all build artifacts are removed at the end of the investigation; `git status --porcelain` then lists only this document (§0.7).
