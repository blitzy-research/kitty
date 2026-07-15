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

**The credential flow hinges on one decision — `request_data` — which is *independent* of whether the transport is fresh or multiplexed.** The rest of this document keeps **two independent dimensions** separate:

- **Transport state:** *fresh* (no live master — a new TCP connection + authentication, and, with sharing on, a new `ControlMaster` socket) versus *multiplexed* (piggyback on a live, already-authenticated master). Decided by an `-O check` probe [`kittens/ssh/main.go`:L663-L665].
- **Data-request mode (`request_data`):** this alone controls **(a)** whether the sensitive triple `{REQUEST_ID, PASSWORD_FILENAME, DATA_PASSWORD}` is baked into the generated bootstrap [`kittens/ssh/main.go`:L475-L478], encoded into the remote command [`kittens/ssh/main.go`:L508], and **appended to the local `ssh` argv** [`kittens/ssh/main.go`:L753-L754] — a real process-list exposure when `true` (§2.5, §6) — and **(b)** who sends the request DCS: the **remote** bootstrap when `true`, the **local Go kitten** when `false` [`kittens/ssh/main.go`:L761-L769]. The shm/tar object is built, and the tar transfer happens, **either way**.

These two dimensions do **not** move together. `request_data` is set to `false` by **either** independent cause: the **kitty-askpass** path — the default `askpass=unless-set` with a new-enough OpenSSH clears it in `set_askpass()` [`kittens/ssh/main.go`:L648-L652], *before* any master check — **or** a **live master** [`kittens/ssh/main.go`:L663-L665]. So a **fresh** transport routinely runs with `request_data=false` (observed below as `sh_default_askpass_fresh_transport`: fresh socket, `request_data=0`, local sender, literal placeholders, and **no** `-O check` needed). To study the connection-sharing branch in isolation, this document forces `askpass=ssh` so that **only** a live master can flip `request_data`. Ten specific questions follow, each with a runtime demonstration.

---

## §0 Methodology & Build

### 0.1 Toolchain actually observed

All work was performed in the task's Linux container (`uname -srm` → `Linux 6.6.122+ x86_64`). The exact tool versions present were captured with:

```
$ go version; python3 --version; ssh -V; zsh --version; fish --version; \
  bash --version | head -1; tar --version | head -1; \
  base64 --version | head -1; tr --version | head -1; \
  readlink -f /bin/sh; dpkg-query -W -f='${Package} ${Version}\n' dash
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
/usr/bin/dash
dash 0.5.12-12ubuntu2
```

The last two lines matter because the kitten's default `interpreter` is the bare name `sh` [`kittens/ssh/main.py`:L87], and on this system `/bin/sh` is a symlink to **dash** — confirmed both statically (`readlink -f /bin/sh` → `/usr/bin/dash`) and at runtime (`/bin/sh -c 'ls -l /proc/$$/exe'` → `/proc/<pid>/exe -> /usr/bin/dash`). So every "sh" bootstrap in this document is actually executed by **dash 0.5.12-12ubuntu2**; the other named shells present are **bash 5.2.37**, **zsh 5.9**, and **fish 4.0.6**.

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

Entry point #2 is the integration hook `TestEntryPoint`/`test_integration_with_python` [`kittens/ssh/main.go`:L847-L886]. It is worth stating precisely what it fixes, because it constrains every observation drawn from it: it sets `request_id="testing"`, `request_data=true`, `echo_on=true`, `username="testuser"`, `hostname_for_match="host.test"`, reads the config from stdin, calls the real `get_remote_command`, and marshals `{"cmd": cd.rcmd, "shm_name": cd.shm_name}` to stdout. Because it sets `request_data=true` it always exercises the **credential-baking (`request_data=true`)** path — regardless of transport state, which this generate-only hook never establishes — and because it does **not** set `dont_create_shm` it leaves the real shm object on disk for inspection.

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

The source tree is treated as strictly read-only — the only path that differs from `HEAD` is this document. Every artifact **this investigation created** is removed at the end: all observation scripts and logs, the isolated `sshd` together with its temporary host/client keys, `sshd_config`, and PidFile, and the record-only fake `ssh` shim — all of which lived under `/tmp/blitzy_ssh_obs/` — plus every `/dev/shm/kssh-*`/`ksse-*` object, any `/tmp/kssh-rdir-*` symlink, and every git-ignored build artifact (the `kitty`/`kitten` launchers, `fast_data_types.so`, `constants_generated.go`, `data_generated.bin`, and the `build/` tree). The isolated `sshd` is stopped by its **exact numeric PID** (`110125`, recorded in §0.4), never a pattern kill. The proof below is captured **after cleanup and the final commit** and deliberately covers **all five surfaces** the work touched — the Git tree, running processes, the listening socket, POSIX shared memory, and the `/tmp` scratch area — rather than relying on `git status` alone:

```
$ git status --porcelain --untracked-files=all
                                        # empty: the deliverable is committed; nothing else differs

$ git diff 815df1e21..HEAD --name-status
A	blitzy/documentation/kitty_815df1e210e0.md
                                        # the sole difference from the source branch is this added document

$ kill -0 110125                        # the isolated sshd PID recorded in §0.4
bash: kill: (110125) - No such process
kill0_rc=1                              # process is gone

$ pgrep -af sshd | grep blitzy_ssh_obs ; echo "isolated_sshd_rc=$?"
isolated_sshd_rc=1                      # no isolated sshd remains

# iproute2 (`ss`) is not installed in this container (`command -v ss` → empty; `ss` → rc 127),
# so the listening socket is probed with a real Python connect_ex instead:
$ python3 -c 'import socket; s=socket.socket(); s.settimeout(1); \
    rc=s.connect_ex(("127.0.0.1",2222)); s.close(); print("port2222_connect_ex=%d"%rc)'
port2222_connect_ex=111                 # ECONNREFUSED: no listener remains on 2222

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

Captured from a real `kitten ssh` run driven through a PTY with the record-only fake `ssh` (`KITTY_PID=55002`, `KITTY_WINDOW_ID=7`, `--kitten askpass=ssh` to isolate the sharing branch). One detail to state up front: the kitten's *very first* `ssh` invocation is not the one shown here. It first runs a deterministic **options-discovery probe** — the `ssh` binary with **no arguments at all** (`argc=0`) — so that `SSHOptions` can parse OpenSSH's own usage text and learn which flags take a value [`kittens/ssh/utils.go`:L40, consumed by `GetSSHCLI` L91 during `ParseSSHArgs` L134]; that probe carries **none** of the sharing options. The dump below is therefore the **first *options-bearing* invocation** — the `-O check` master probe (`argc=16`) — reproduced exactly (the full argv log):

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

Driving the system `ssh` with the kitten's exact six options against the isolated `sshd` (§0.4) on `127.0.0.1:2222`, with host verification retained. The six sharing options and the two retained host-verification options are captured once as the shell variables `$OPTS` and `$VER` in step (0) below (so every subsequent command is directly executable); `$RD`, `$KEY`, and `$KNOWN` are likewise defined there and are consistent with `$SSHDIR` from §0.4. Complete transcript (`ControlPath=/root/.cache/kitty/run/kssh-77001-%C`):

```
### ControlMaster lifecycle — isolated sshd 127.0.0.1:2222, kitten's exact six -o options, host verification retained

--- (0) reusable variables: the kitten's exact six sharing -o options + retained host verification ---
$ SSHDIR=/tmp/blitzy_ssh_obs/sshd
$ RD=/root/.cache/kitty/run
$ KEY=$SSHDIR/client_ed25519
$ KNOWN=$SSHDIR/known_hosts
$ OPTS="-o ControlMaster=auto -o ControlPath=$RD/kssh-77001-%C -o ControlPersist=yes -o ServerAliveInterval=60 -o ServerAliveCountMax=5 -o TCPKeepAlive=no"
$ VER="-o StrictHostKeyChecking=yes -o UserKnownHostsFile=$KNOWN"

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
$ ssh $OPTS $VER -O check -p 2222 -i $KEY root@127.0.0.1 ; echo check_exit=$?
Master running (pid=116567)
check_exit=0

--- (4) second connection REUSES the master (multiplexed), timed with bash builtin ---
$ time ssh $OPTS $VER -p 2222 -i $KEY root@127.0.0.1 "echo REUSED_OK"
REUSED_OK

real	0m0.006s
user	0m0.000s
sys	0m0.004s

--- (5) controlled NON-multiplexed baseline (fresh TCP+auth, NO ControlPath), timed ---
$ time ssh -o ControlMaster=no -o ControlPath=none $VER -p 2222 -i $KEY root@127.0.0.1 "echo BASELINE_OK"
BASELINE_OK

real	0m0.121s
user	0m0.008s
sys	0m0.002s

--- (6) -O exit tears down the master (ControlPersist=yes had kept it alive) ---
$ ssh $OPTS $VER -O exit -p 2222 -i $KEY root@127.0.0.1 ; echo exit_cmd_exit=$?
Exit request sent.
exit_cmd_exit=0

--- (7) socket gone after -O exit ---
$ ls -l $RD/kssh-77001-* 2>&1
ls: cannot access '/root/.cache/kitty/run/kssh-77001-*': No such file or directory
```

Reading the evidence, effect by cause:

* **(2)** proves the master socket is a UNIX-domain socket (`s`), mode `0600` (owner-only), and that OpenSSH appended a 40-hex `%C` hash (`de62245…d881`, the abbreviation of the full `de6224549bf17f660bd566abc77d74659f79d881` shown on the `ls -l` line above) exactly as §1.2 predicts.
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
* **The kitten owns only the bootstrap-data channel.** Its security contribution is confined to **local** staging and access control: it places the shell-integration payload + data password in an owner-only (`0600`) shared-memory object created race-free (`O_EXCL`) and made **single-use** (unlinked on read), and it gates release of that payload behind the six checks in §9. The payload does **not** stay off the network — the tar reply always crosses the connection to the remote (which needs it), and on the request_data route the credential triple crosses too (§2.5). Whatever crosses rides *inside* the already-established, OpenSSH-encrypted PTY stream; the base64/DCS framing is **not** a cryptographic layer, only a transport-safe encoding, and confidentiality/integrity on the wire are provided by OpenSSH, not by the kitten.
* **Consequence.** The kitten cannot make an insecure SSH configuration secure, and does not try to: if the user disables host checking, that is an OpenSSH-level decision. What the kitten *does* guarantee is **local**: the tar and data password are staged in an owner-only, single-use object and released only to a requester that passes the §9 checks. Confidentiality and integrity of whatever then travels the wire — the tar reply always, and the credential request on the request_data route — are provided by the OpenSSH-encrypted transport.

---


## §2 How does it use shared memory to pass credentials securely?

**Direct answer.** The kitten writes the entire bootstrap payload — the base64 gzipped tar **and** a one-time data password — into a **POSIX shared-memory object** at `/dev/shm/kssh-<pid>-<rand>`, created owner-only (`0600`) and exclusively (`O_EXCL`) by the **Go** side. This object is created **unconditionally** on every real run (the only suppressor is the test-only `dont_create_shm` flag). What differs between the two credential routes is **not** whether this object exists — it always does — but **who reads the password back to kitty** and, critically, **whether the credentials also end up in the local `ssh` process's argv** (they do when `request_data=true`). That switch is `request_data`, which is independent of transport state (fresh vs reused) — see §6.2 Outcome C. This section documents the object; §2.5 draws the `request_data` distinction precisely.

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

So the payload is a JSON object with exactly four keys — **`tarfile`** (base64 of the gzipped tar), **`pw`** (the 64-hex one-time data password), **`hostname`**, and **`username`** — assembled at `kittens/ssh/main.go:L439-L443`. The password is staged **only** in this local shared-memory object; but on the request_data route the same value is additionally baked into the remote command and therefore traverses the OpenSSH-encrypted connection to the remote and back in the request (§2.5) — it is OpenSSH, not the shm object, that protects it on the wire.

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

### 2.5 The `request_data` route — where the credentials actually go (the corrected flow)

This is the crux the earlier draft got backwards. The shm object exists in both cases; the difference is gated on **`request_data`** (not transport state) — the **substitution of the sensitive values into the generated bootstrap** and therefore **into the `ssh` argv**:

* When **`request_data=true`** (observed here on a fresh connection, isolated via `askpass=ssh` with the master absent), the sensitive triple `{REQUEST_ID, DATA_PASSWORD, PASSWORD_FILENAME}` is merged into the script's substitution map [`kittens/ssh/main.go`:L476-L478], so the generated bootstrap's DCS line is filled with the **real** values, the script is encoded into `rcmd` [`kittens/ssh/main.go`:L508], and `rcmd` is **appended to the system-`ssh` argv** [`kittens/ssh/main.go`:L753-L754]. The password is therefore present in the local `ssh` process's arguments. Captured directly from the `request_data=true` fake-`ssh` argv log (`FAKE_SSH_OCHECK_RC=1`, master absent; password redacted here as a justified redaction — it is an ephemeral, already-unlinked 64-hex token):

  ```
  # fakessh_fresh.log — the argc=19 real connection, argv[18] is the bootstrap script
  request_data="1"
  ...
  [ "$request_data" = "1" ] && {
      command stty "-echo" < /dev/tty
      dcs_to_kitty "ssh" "id="55002-7":pwfile="kssh-109681-WIHB57IINC574":pw="<64-hex-REDACTED>""
  }
  ```

  The `id`, `pwfile`, and `pw` are literally substituted into `argv[18]` — a real **process-list exposure** on the machine running the kitten, **and** `ssh` transmits that remote command to the server, so the triple also **traverses the connection** (inside the OpenSSH-encrypted channel) and the remote echoes the password back to the terminal in its `@kitty-ssh` request. Two exposures, then: any local user who can read this process's argv sees the password, and on the wire the triple is protected by OpenSSH's encryption — **not** by the shm object.

* When **`request_data=false`** (isolated here with `askpass=ssh` so that *only* a live master can flip the decision — observed on a reused connection; but note the default kitty-askpass path reaches this same state on a **fresh** connection, §6.2 Outcome C), the sensitive values are **not** merged into the script map, so the same DCS line keeps its **literal placeholders**, and the local Go kitten sends the request itself (§6, §10). Captured from the `request_data=false` log (`FAKE_SSH_OCHECK_RC=0`, master alive):

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

So "shared memory to pass credentials securely" is really about **local** staging and access control — an owner-only, single-use object read and validated locally — **not** about keeping data off the network. The tar reply always crosses the connection, and on the `request_data=true` route the credential triple crosses too; what protects those bytes on the wire is **OpenSSH's encryption**, while what the shm object adds is that the payload is released only to a validated local requester (§9) and only once. When `request_data=true` the password additionally appears in the **local** process table for the lifetime of the `ssh` process. The rest of the security story — how the reader validates a request before releasing the tar — is §9.

---

## §3 How are the bootstrap scripts that run on the remote machine generated?

**Direct answer.** The kitten does **not** hand-write a script per connection. It takes a **checked-in template** — `shell-integration/ssh/bootstrap.sh` or `shell-integration/ssh/bootstrap.py`, embedded into the kitten binary at build time — and performs a **literal token substitution** on it, replacing a fixed set of `UPPER_CASE` placeholders with per-connection values. Which template is used is decided by the `interpreter` option (`sh` by default; an interpreter whose lowercased basename **contains** `python` — e.g. `python3` — selects the Python template). The filled-in script is then **encoded** (§7) and wrapped into the remote command `rcmd` that `ssh` executes. Everything happens locally in Go, in three functions in `kittens/ssh/main.go`: `get_remote_command` [`kittens/ssh/main.go`:L511-L525] picks the template, `bootstrap_script` [`kittens/ssh/main.go`:L455-L483] builds the substitution map and applies it, and `wrap_bootstrap_script` [`kittens/ssh/main.go`:L486-L509] encodes the result.

### 3.1 The generation pipeline

| Step | Code | What it does |
|------|------|--------------|
| Pick interpreter | `interpreter := cd.host_opts.Interpreter` [`kittens/ssh/main.go`:L512] | Reads the `interpreter` host-option (default `sh`, [`kittens/ssh/main.py`:L87]) |
| Pick template type | `is_python := strings.Contains(q, "python")` [`kittens/ssh/main.go`:L514], then `cd.script_type = "sh"` [`kittens/ssh/main.go`:L515] / `"py"` [`kittens/ssh/main.go`:L517] | `script_type = "py"` iff `strings.ToLower(path.Base(interpreter))` **contains** the substring `python` (the deciding line is L514); otherwise `sh`. It is an open substring test, not a fixed list — values that select `py` include `python`, `python3`, `cpython`, `mypython-wrapper`, `PYTHON3`, `/opt/foo/python3.11`; note `py` alone selects `sh` because it does not contain `python` |
| Build substitutions + apply | `bootstrap_script(cd)` [`kittens/ssh/main.go`:L519] → [L455-L483] | Builds the placeholder map and calls `prepare_script` |
| Fetch template | `shell_integration.Data()["shell-integration/ssh/bootstrap."+cd.script_type].Data` [`kittens/ssh/main.go`:L481] | Reads the embedded template blob (built by `gen/go_code.py`) |
| Substitute | `prepare_script(cd.bootstrap_script, sd)` [`kittens/ssh/main.go`:L482] | Replaces every `PLACEHOLDER` token with its value |
| Encode + wrap | `wrap_bootstrap_script(cd)` [`kittens/ssh/main.go`:L523] → [L486-L509] | Encodes (§7) and builds `cd.rcmd` [`kittens/ssh/main.go`:L508] |

### 3.2 The substitution map — and where the `request_data` split happens

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

`sd` is the map actually applied to the template ([`kittens/ssh/main.go`:L482]). The merge is gated on **`cd.request_data`**, *not* on transport state: when `request_data == true` the real `id`/`pwfile`/`pw` are **baked into the script text**; when `request_data == false` the three secret tokens are left as the **literal strings** `REQUEST_ID` / `PASSWORD_FILENAME` / `DATA_PASSWORD`. This is the same `request_data` split proven with captured `ssh` argv in §2.5 and §6.2, seen here at its source. Because `request_data` can be `false` on a **fresh** transport (the default kitty-askpass path — §6.2 Outcome C), the literal-placeholder branch here is **not** synonymous with "reused connection." (The `maps.Copy(replacements, sensitive_data)` at [`kittens/ssh/main.go`:L479] populates the *separate* `cd.replacements` used for the local DCS the kitten sends when `request_data == false` — it does not affect the script `sd`.)

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
| 6 | `request_data bool` | `= need_to_request_data` [L724] | **the credential-request decision** (independent of transport state — §6.2) | gates sensitive merge [L476-L478] and who sends the DCS [L761] | whole run |
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

### 6.2 Isolating the branch, and observing all three outcomes

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

**Outcome C — DEFAULT `askpass` (`unless-set`), master ABSENT → a *fresh* transport that nonetheless has `request_data=0`.** This is the case the fresh/reused binary misses. With no `askpass=ssh` override, `use_kitty_askpass` is true, so `set_askpass()` clears `need_to_request_data` at [`kittens/ssh/main.go`:L648-L652] — *before* the master branch — and the `-O check` probe is **never reached** (the `need_to_request_data &&` short-circuit at [`kittens/ssh/main.go`:L663]). The transport is still fresh (a new `ControlMaster=auto` connection), but `request_data=0`, the placeholders stay literal, and the **local** kitten sends the request. Captured from the default run (`harness_fakessh.py default`, invoked as `kitten ssh root@localhost echo UNTAR_DONE` — **no** overrides):

```
request_id = 507065-1
local @kitty-ssh payload (sent by the LOCAL kitten) = id=507065-1:pwfile=kssh-507066-EZ6KF6GLGHNJC:pw=<64-hex-REDACTED>
fake-ssh -O check invocations: IS_O_CHECK=True count = 0        # the -O check is never reached
# the real connection still carries the six sharing options (fresh master would be created):
CALL argc=19: -o ControlMaster=auto -o ControlPath=/root/.cache/kitty/run/kssh-507065-%C -o ControlPersist=yes \
              -o ServerAliveInterval=60 -o ServerAliveCountMax=5 -o TCPKeepAlive=no -- root@localhost exec sh -c <unwrap> <encoded>
argv[18] (remote bootstrap): request_data="0"
                             dcs_to_kitty "ssh" "id="REQUEST_ID":pwfile="PASSWORD_FILENAME":pw="DATA_PASSWORD""   # LITERAL — not substituted
```

So a **fresh** transport runs with `request_data=0`, proving the two dimensions are independent: a live master is only **one** of the two ways `request_data` becomes false; the default kitty-askpass path is the other, and it does so on a fresh connection **without ever probing the master** (see §8, §10.2).

### 6.3 Cause → effect summary (corrected)

The observable behaviour is keyed on **`request_data`**, *not* on transport state; the shm/tar channel exists in **both** columns. The columns below therefore track `request_data` (the variable actually consulted at [`kittens/ssh/main.go`:L475-L478, L508, L761-L769]); the transport-state row records which transport was observed in each isolated run, and the note beneath explains why they are independent:

| Aspect | `request_data=true` (observed: Outcome A — askpass=ssh, master absent) | `request_data=false` (observed: Outcome B — askpass=ssh, master alive; **and** Outcome C — default askpass, master absent) |
|--------|-----------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| shm object created? | **Yes** [L445] | **Yes** [L445] — identical |
| Sensitive triple merged into the script? | Yes [L476-L478] | No (placeholders stay literal) |
| Credentials in the local `ssh` argv? | **Yes** — `argv[18]` carries `id/pwfile/pw` (process-list exposure) | **No** — placeholders only |
| Who sends the request DCS? | the **remote** bootstrap, over the tty [`shell-integration/ssh/bootstrap.sh`:L92-L95] | the **local Go kitten**, to its own kitty terminal [`kittens/ssh/main.go`:L761-L769] |
| Transport observed in the isolated run | fresh (new connection) | Outcome B: reused/multiplexed (§1.3 shows 0.006 s vs 0.121 s); Outcome C: **fresh** (new connection, yet still `request_data=false`) |

**Independence of the two dimensions.** The `request_data=false` column above is reached by **two independent causes**, and only one of them is "master alive": the **kitty-askpass** path clears `need_to_request_data` at [`kittens/ssh/main.go`:L648-L652] *before* the master is probed, so **Outcome C** lands in the right-hand column on a **fresh** transport with **zero** `-O check` calls (§6.2). Transport state (fresh vs multiplexed) is decided separately by `-O check` at [`kittens/ssh/main.go`:L663-L665]. Do **not** read the header labels as "fresh ⇔ request_data=true / reused ⇔ request_data=false": that binary is exactly the conflation Outcome C disproves. The earlier draft additionally had the credential columns inverted (claiming fresh keeps creds off the argv and a reused connection opens no channel); the captured argv logs above show the opposite, and the timing in §1.3 confirms the reuse itself.

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

Reversed through the **real system `tr`** (uutils coreutils 0.2.2) using the same map as the emitted unwrap, then **re-encoded through the real `tr` with the inverse map** (the exact byte-for-byte equivalent of the `main.go`:L505 `strings.NewReplacer`), and compared byte-for-byte. Both directions are executable commands (no synthetic re-encode step):

```
$ tr '\013\014\015\010' '\047\134\012\041' < S_sub.bin > B_rev.sh        # reverse the 4 subs (the remote's tr map)
$ tr '\047\134\012\041' '\013\014\015\010' < B_rev.sh  > S_roundtrip.bin # re-apply main.go:L505 subs (inverse map, same real tr)
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

The histogram shows the substitution actually occurred (164 newlines became `0x0d`, 10 single-quotes became `0x0b`, 25 backslashes became `0x0c`, 3 bangs became `0x08`) and that **no literal newline (`0x0a`) survives** in the wire form; because the re-encode is performed by an explicit `tr '\047\134\012\041' '\013\014\015\010'` (the inverse of the remote's reverse map), the `cmp` returning `IDENTICAL` proves the transform is exactly invertible end-to-end, not merely that the reverse was self-consistent.

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

**Direct answer.** The kitten runs entirely locally in Go until it hands a crafted remote command to the system `ssh`; the remote then unwraps and runs the bootstrap, which pulls its data back over the tty and hands off to the login shell. The exact local order is fixed in `run_ssh` [`kittens/ssh/main.go`:L597-L800]. The **two routes differ only in who sends the credential request** (§6, §10), and that split is governed by **`request_data`**, *not* by transport state: when `request_data=true` the credentials are already baked into the remote command and the **remote** bootstrap asks for the data; when `request_data=false` the **local** kitten sends the request itself. Because `request_data` can be false on a *fresh* transport (the default kitty-askpass path — §6.2 Outcome C), "who sends the request" is decided by `request_data`, not by whether a master exists.

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
| 10 | **Append the remote command to the `ssh` argv** (when `request_data=true` the credentials are now in the argv; when `false` the argv carries literal placeholders) | `cmd = append(cmd, cd.rcmd...)` [`kittens/ssh/main.go`:L753] |
| 11 | **Start `ssh`** — the remote runs `exec <interp> -c <unwrap> <encoded>` | `c.Start()` [`kittens/ssh/main.go`:L756] |
| 12 | *(only when `request_data=false`)* local kitten sends the `@kitty-ssh` request over the tty | `if !cd.request_data { ... DCSToKitty("ssh", rq) ... term.WriteAllString(dcs) }` [`kittens/ssh/main.go`:L761-L768] |
| 13 | Wait for `ssh`; then drain the tty with an `@kitty-echo` canary | `c.Wait()` [`kittens/ssh/main.go`:L782]; `drain_potential_tty_garbage` [L783] → `DCSToKitty("echo", canary)` [L539] |

### 8.2 The remote side (after `ssh` connects)

`ssh` executes `exec <interp> -c <unwrap_script> <encoded_script>` (§3, §7). The unwrap restores the original bootstrap text and runs it:

1. **Only when `request_data=1`** — the bootstrap sends the credential request itself: `[ "$request_data" = "1" ] && dcs_to_kitty "ssh" "id=...:pwfile=...:pw=..."` [`shell-integration/ssh/bootstrap.sh`:L92-L95], or in Python `if request_data: ... send_data_request()` [`shell-integration/ssh/bootstrap.py`:L292-L294]. (The remote's `request_data` is the literal `0`/`1` baked into `argv[18]`; on the default kitty-askpass fresh path it is `0`, so the remote stays silent and the **local** kitten sends the DCS — §6.2 Outcome C.)
2. **Both routes** — the bootstrap then reads the reply unconditionally: `get_data` [`shell-integration/ssh/bootstrap.sh`:L155] / [`shell-integration/ssh/bootstrap.py`:L295]. It waits for `OK`, captures the base64 after `KITTY_DATA_START` until `KITTY_DATA_END`, untars into a temp dir, sources `data.sh`, and finally `exec`s the login shell ([`shell-integration/ssh/bootstrap.sh`:L164]; [`shell-integration/ssh/bootstrap.py`:L243/L253/L315]).
3. **Terminal side** — kitty receives the `@kitty-ssh` DCS and dispatches it: `handle_remote_ssh` → `get_ssh_data(msg, f'{os.getpid()}-{self.id}')` [`kitty/window.py`:L1289-L1291], which validates the request (§9) and streams the tar back (§4.5, §10).

### 8.3 Observed bootstrap execution — canonical PTY suite (LOCAL, no `ssh`/sshd)

kitty's own PTY test module exercises the bootstrap **execution and data round trip locally** — it does **not** invoke the system `ssh`, stand up an sshd, or open a network socket. `check_bootstrap` in [`kitty_tests/ssh.py`:L227-L272] generates the remote command with `subprocess.run([kitten_exe(), '__pytest__', 'ssh', test_script], ...)` [L241] (the `__pytest__` hook hardcodes `request_data:true`) and then runs that command through a **local** PTY, `self.create_pty([launcher, '-c', ' '.join(self.rdata['cmd'])], ...)` [L251] — where `launcher` is one of `sh`/`dash`/`bash`/`zsh`/`python`. There is no `/usr/bin/ssh`, no sshd, and no TCP connection in this path; the "remote" and "terminal" sides are the same local process pair joined by the PTY. Run through the built launcher:

```
$ ./kitty/launcher/kitty +launch test.py --module ssh
........
----------------------------------------------------------------------
Ran 8 tests in 9.907s

OK
```

All 8 tests pass (repeated twice, §0.5). This suite is the authoritative check that the *bootstrap script itself* (DCS request framing, untar, `data.sh` sourcing, login-shell handoff) works for every launcher; because it is local, it does **not** demonstrate that the payload traverses a network. The genuine over-the-wire round trip is captured separately in §8.4.

### 8.4 Observed end-to-end round trip over a real `ssh` connection (`/usr/bin/ssh` → local sshd)

To observe the payload actually crossing a network connection, a separate harness drives the real system `ssh` against an isolated local sshd (§0.4). The harness imports the real terminal-side `get_ssh_data` [`kittens/ssh/utils.py`:L115-L148], forks a PTY child that `exec`s the built `kitten ssh` against `/usr/bin/ssh`, detects the remote's `@kitty-ssh` DCS request, writes the reply back over the tty, and reads `/proc/net/tcp` to confirm an established TCP endpoint (port `0x08AE` = 2222). Invocation and captured output:

```
$ ./kitty/launcher/kitten ssh -p 2222 -o StrictHostKeyChecking=no \
    -o UserKnownHostsFile=/tmp/kssh_qa_work/sshd/known_hosts -o IdentitiesOnly=yes \
    -i /tmp/kssh_qa_work/sshd/client_ed25519 -o BatchMode=yes -o RequestTTY=force \
    --kitten askpass=ssh --kitten share_connections=no \
    --kitten env=HOME=/tmp/kssh_qa_work/remote_home root@localhost echo UNTAR_DONE

request_id: 509290-1
GOT UNTAR_DONE marker: True
answered remote DCS request: True
num remote @kitty-ssh DCS requests: 1
remote DCS request payload (decoded): id=509290-1:pwfile=kssh-509291-YNCFR4PCG6BP2:pw=<64-hex-REDACTED>
established TCP at first request: ['127.0.0.1:56860 -> 127.0.0.1:2222 ESTABLISHED',
                                   '127.0.0.1:2222 -> 127.0.0.1:56860 ESTABLISHED']
reply size bytes: 31721
reply KITTY_DATA_START: True
reply OK\n: True
reply KITTY_DATA_END: True
reply has ESC-P DCS prefix (expect False): False
reply has '@kitty-' (expect False): False
```

The connection reaches `UNTAR_DONE`, an established TCP endpoint (`127.0.0.1:56860 ↔ 127.0.0.1:2222`) is confirmed in both directions, and the remote side untars **13 files** into the isolated `HOME`:

```
$ find /tmp/kssh_qa_work/remote_home -type f | sort
.local/share/kitty-ssh-kitten/kitty/bin/kitten
.local/share/kitty-ssh-kitten/kitty/bin/kitty
.local/share/kitty-ssh-kitten/kitty/version
.local/share/kitty-ssh-kitten/shell-integration/bash/kitty.bash
.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_completions.d/clone-in-kitty.fish
.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_completions.d/kitten.fish
.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_completions.d/kitty.fish
.local/share/kitty-ssh-kitten/shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish
.local/share/kitty-ssh-kitten/shell-integration/zsh/.zshenv
.local/share/kitty-ssh-kitten/shell-integration/zsh/completions/_kitty
.local/share/kitty-ssh-kitten/shell-integration/zsh/kitty-integration
.terminfo/kitty.terminfo
.terminfo/x/xterm-kitty
```

This is the genuine local-kitten → `/usr/bin/ssh` → TCP → sshd → remote-bootstrap → DCS-request → terminal-data-server → untar → login-shell round trip. The per-route specifics — when `request_data=true` the credentials are baked into `argv[18]`; when `request_data=false` the argv keeps literal placeholders and the local kitten sends the DCS — are the captured `ssh` argv logs in §6.2 (all three outcomes) and the captured DCS wire in §10.2. Note the run above uses `--kitten askpass=ssh` to isolate the `request_data=true` route; the reply-framing lines (`ESC-P` prefix and `@kitty-` both **False**) additionally re-confirm the §10.3 asymmetry over a real connection.

---

## §9 How does shared memory keep things secure?

**Direct answer.** The credential channel is a POSIX shared-memory object whose security rests on layered, **locally**-enforced guarantees (the object itself never leaves the machine — it is read and validated locally): it is **created race-free** (`O_EXCL`, mode `0600`, owner = the kitty process) and is **single-use** (unlinked the instant it is read); and every read is gated by **five validations** — the request must **parse**; the object must have the correct **owner** and, as a separate check, the correct **permissions** (`0o600`); and the request must present the correct **password** and the correct **request-id**. (Those five validations, together with the always-emitted `KITTY_DATA_START` marker that leads every reply, are the six ordered checks tabulated in §9.2 — the "six checks" referred to in §1.6.) Any failure yields an error line and logs a traceback; the tar is released only when all pass. Two things this does **not** cover, by design: the **network** — on the request_data route the credential triple is baked into the remote command and travels the OpenSSH-encrypted connection, and the tar reply always does, so wire confidentiality/integrity are **OpenSSH's** job, not the shm object's (§2.5); and the **local process table** — on a **fresh** connection the password is also in the local `ssh` argv (§2.5). Shared memory secures **local object access and single-use release**, nothing more.

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

- **Protects:** local access to the staged object — other-UID processes cannot read it (`0600` + owner check); a captured request cannot be replayed (single-use unlink); a request aimed at the wrong window is refused (request-id); an attacker cannot pre-create or leave behind a poisoned object (`O_EXCL` + unlink-before-validate).
- **Does not protect against:** the **network** — the shm object does **not** keep the payload off the wire: the tar reply always crosses the connection, and on the request_data route the credential triple crosses too, so wire confidentiality/integrity are provided by **OpenSSH's** encrypted transport, not by the shm object (§2.5); a process running as the **same UID** (or **root**) on the local machine — it can read `/dev/shm` directly, though `O_EXCL`, `0600`, and single-use unlink make the window small; a descriptor already `open`ed before unlink; and — on a **fresh** connection — the password is additionally present in the local `ssh` **argv** (visible in `/proc/<pid>/cmdline` to same-UID/root) for the lifetime of the `ssh` process (§2.5). The payload is base64, **not** encryption; the base64 tar is not secret (it is shell-integration files), only the `pw`/`pwfile`/`id` triple is sensitive.

---

## §10 How does the terminal communicate back and forth with the remote shell during setup?

**Direct answer.** Setup traffic uses **two different framings, one per direction — they are not the same**. The **request** (the credential request `@kitty-ssh` and debug `@kitty-print`) rides the **tty** as **DCS (Device Control String) escape sequences** of the form `ESC P @kitty-<type> | <base64-payload> ESC \`. The **reply** (terminal→remote) is **not** DCS at all: it is **ordinary newline-framed text** written to the remote's tty — `KITTY_DATA_START` / `OK` / ≤254-byte base64 lines / `KITTY_DATA_END`, carrying **zero** `ESC P`/`@kitty-` prefixes (captured in §10.3 and byte-checked below). The **request is sent by different senders depending on the route**: the remote bootstrap on a fresh connection, the local kitten on a reused one. Locally injected DCS is wrapped in a save/set/restore of DEC private mode **19997** (`HANDLE_TERMIOS_SIGNALS`).

### 10.1 The DCS request frame (the reply uses different, non-DCS framing — §10.3)

Producer of the frame on the remote side: `dcs_to_kitty() { printf "\033P@kitty-$1|%s\033\134" "$(printf "%s" "$2" | base64_encode)" > /dev/tty; }` [`shell-integration/ssh/bootstrap.sh`:L75] (Python equivalent [`shell-integration/ssh/bootstrap.py`:L73-L77]). `\033P` is `ESC P` (DCS), `\033\134` is `ESC \` (ST, string terminator). Captured **verbatim** from a real PTY run (`od -c` of the local kitten's tty output — here the `request_data=true` capture, which shows the always-emitted `@kitty-echo` drain canary; the frame format is identical for `@kitty-ssh`):

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

### 10.2 Who sends the request — the directional split is governed by `request_data` (observed)

The **sender** of the `@kitty-ssh` request is decided by `request_data`, *not* by transport state (§6.2): when `request_data=true` the **remote** bootstrap sends it; when `request_data=false` the **local** Go kitten sends it. Decoding every `@kitty-*` frame from two captured PTY outputs — the first with `request_data=true` (isolated via `askpass=ssh`, master absent — Outcome A), the second with `request_data=false` (isolated via `askpass=ssh`, master alive — Outcome B) — with password redacted:

```
pty_out_reqdata_true.bin (130 bytes): 1 @kitty DCS frame(s)
  @kitty-echo|  ->  <64-hex drain-canary nonce>

pty_out_reqdata_false.bin (293 bytes): 2 @kitty DCS frame(s)
  @kitty-ssh|  ->  id=55002-7:pwfile=kssh-109709-QSDOU7G3YYGPU:pw=<64-hex-REDACTED>
  @kitty-echo|  ->  <64-hex drain-canary nonce>
```

- **`request_data=true`** (`pty_out_reqdata_true.bin`, 130 B): the local kitten emits **only** an `@kitty-echo` drain canary; it sends **no** `@kitty-ssh` — the **remote** bootstrap sends the request over `/dev/tty` [`shell-integration/ssh/bootstrap.sh`:L92-L95].
- **`request_data=false`** (`pty_out_reqdata_false.bin`, 293 B): the **local Go kitten** sends the `@kitty-ssh` request carrying the real `id`/`pwfile`/`pw` [`kittens/ssh/main.go`:L761-L768], plus the same `@kitty-echo` canary.

This is the definitive evidence for the **directional** split (§6): the credential request exists on **both** routes, but the **sender** differs — and the switch is `request_data`. Because `request_data` can be false on a **fresh** transport (the default kitty-askpass path — §6.2 Outcome C, where the **local** kitten sends the DCS even though no master exists), the sender is **not** a proxy for "fresh vs reused." The two isolated captures above happen to pair `request_data=true` with a fresh transport and `request_data=false` with a reused one only because `askpass=ssh` was used to hold the askpass dimension fixed; Outcome C breaks that pairing.

### 10.3 The reply (terminal → remote)

kitty receives the `@kitty-ssh` DCS and dispatches it to the data server: `handle_remote_ssh` → `get_ssh_data(msg, f'{os.getpid()}-{self.id}')` [`kitty/window.py`:L1289-L1291]; each yielded line is written back to the child. The reply framing (validated in §9, measured in §4.5) is `\nKITTY_DATA_START\n`, then `OK\n`, then the base64 tar in **254-byte** lines, then `KITTY_DATA_END\n` [`kittens/ssh/utils.py`:L117, L138-L148].

### 10.4 The remote read/decode/handoff — sh and Python

- **sh** — `get_data` [`shell-integration/ssh/bootstrap.sh`:L137-L155] waits for `OK` [L141], starts capturing at `KITTY_DATA_START` [L144], and `read_base64_from_tty` [L97-L102] reads until `KITTY_DATA_END` [L99]; then `untar_and_read_env` [L104-L151] decodes and untars [L113].
- **Python** — the read loop keys on `KITTY_DATA_START` [`shell-integration/ssh/bootstrap.py`:L178] and `KITTY_DATA_END` [L188]; `get_data` [L203] then base64-decodes and `extractall`s [L215]. The request itself is `send_data_request` [L80-L81], gated by `if request_data:` in `main` [L292-L294].

**Message types observed/used:** `@kitty-ssh` (credential request → tar reply), `@kitty-echo` (drain canary, round-trips a nonce to flush the tty), and `@kitty-print` (`debug()` diagnostics [`shell-integration/ssh/bootstrap.sh`:L76]). All share the one frame format of §10.1.

---

## Architecture at a glance

The diagram below reflects the **corrected** flow, keyed on **`request_data`** rather than transport state (shm built unconditionally; when `request_data=true` the credentials are baked into the `ssh` argv and the **remote** sends the request; when `request_data=false` the argv keeps placeholders and the **local** kitten sends the request). `request_data` and transport state (fresh vs multiplexed) are **independent** — the default kitty-askpass path yields `request_data=false` on a *fresh* transport (§6.2 Outcome C). All arrows are directional. Note in particular the two **distinct local channels**: the `/dev/shm/kssh-*` credential object is read by the **terminal-side** `get_ssh_data` [`kittens/ssh/utils.py`:L100-L148] — which then replies over the tty — whereas `system ssh` only carries the `argv` and multiplexes the transport and **never opens the shared-memory object** (verified at runtime in §9.3, and confirmed by source: the only Go-side shm reads, `main.go`:L72 and `askpass.go`:L76, are unrelated features).

```
        LOCAL HOST                                          |        REMOTE HOST
                                                            |
     +-----------------------------------------------+      |
     | kitty terminal  (pid P, window id W)          |      |
     |   DCS dispatch : handle_remote_ssh            |      |
     |                  [kitty/window.py:L1289-L1291]|      |
 +-->|   data server  : get_ssh_data                 |      |
 |   |                  [kittens/ssh/utils.py:L115]  |      |
 |   +----------^-------------------------+----------+      |
 |              |                         |                 |
 |    reply over tty:             reads @kitty-ssh          |
 |    START/OK/<=254B b64/END     request (from remote when |
 |              |                 request_data=1; from local |
 |              |                 Go kitten when =0)          |
 |   +----------+-------------------------v----------+      |
 |   | kitten ssh (Go)  run_ssh [kittens/ssh/main.go:L597]  |
 |   |  1 build pw + tar + shm  UNCONDITIONALLY [L431-L446] |
 |   |  2 set request_data (askpass L648-652;              |
 |   |                      master   L663-L664)            |
 |   |  3 get_remote_command -> rcmd            [L749]      |
 |   |  4 append rcmd to ssh argv               [L753]      |
 |   |  5 exec ssh                              [L756]      |
 |   |  6 request_data=0 only: send @kitty-ssh  [L761-L768] |
 |   +----+--------------------------+----------------+     |
 |        | create (write)           | ssh argv (req_data=1:|
 |        |                          |            pw baked) |
 |        v                          v                      |
 |   /dev/shm/kssh-<pid>-*     +-------------+              |
 |   0600, O_EXCL              | system ssh  |==== ssh =======> sshd
 |   [tools/utils/shm/         | +ControlMstr|              |     |
 |    shm_syscall.go:L162]     +-------------+              |     v
 |        |                                                 |  exec <interp> -c <unwrap> <encoded>
 +--------+  read by get_ssh_data (terminal side, up the    |     |
            tty; system ssh never opens the shm object)     |     v
            [kittens/ssh/utils.py:L100-L148]                |  bootstrap.sh / bootstrap.py
                                                            |   - req_data=1: send @kitty-ssh [bootstrap.sh:L92-L95]
                                                            |   - BOTH : get_data -> untar -> source data.sh
                                                            |   - exec login shell       [bootstrap.sh:L164]
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
| 6 | Fresh vs reused decision (+ independent `request_data`) | §6 | record-only fake `ssh` + `--kitten askpass=ssh`, `-O check` RC 1 vs 0; **and** default-askpass run (`kitten ssh ... echo UNTAR_DONE`, 0 `-O check`) | `request_data=1` → `argv[18]` carries `id/pwfile/pw`; `request_data=0` → literal placeholders. Outcome C: default askpass gives `request_data=0` on a **fresh** transport (0 `-O check`) — the two dimensions are independent |
| 7 | Bootstrap encoding per shell | §7 | reverse via real `tr '\013\014\015\010' '\047\134\012\041'`; `cmp` | round-trip byte-`IDENTICAL`; fish driven explicitly -> `REALLYFISH=4.0.6`, beam cursor |
| 8 | Full end-to-end trace | §8 | LOCAL bootstrap suite `./kitty/launcher/kitty +launch test.py --module ssh` (§8.3); **and** real `/usr/bin/ssh`→sshd round trip (§8.4) | 8/8 local tests pass; 13-step local order L718 -> L724 -> L749 -> L753 -> L756; real ssh: established TCP `127.0.0.1:56860 ↔ :2222`, reached `UNTAR_DONE`, 13 files untarred remotely |
| 9 | Shared-memory security | §9 | real `get_ssh_data` via `kitty +launch`, 7 branches | happy + 4 rejections + invalid + single-use; tampered objects still unlinked |
| 10 | Terminal <-> remote comms | §10 | `od -c` of captured PTY output; decode `@kitty-*` frames | request is DCS `ESC P @kitty-<type> \| <b64> ESC \`, reply is newline-framed (zero DCS, §10.3); sender keyed on `request_data`: `=1` → only `@kitty-echo` locally (remote sent request); `=0` → `@kitty-ssh` from local kitten |

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
| LOCAL bootstrap suite (8/8) — no `ssh`/sshd (§8.3) | `./kitty/launcher/kitty +launch test.py --module ssh` |
| Real `ssh`→sshd round trip (§8.4) | `./kitty/launcher/kitten ssh -p 2222 ... --kitten askpass=ssh --kitten share_connections=no --kitten env=HOME=<tmp> root@localhost echo UNTAR_DONE` against the isolated `sshd`, terminal side driven by real `get_ssh_data` |
| sh / py generation | `printf '' \| ./kitty/launcher/kitten __pytest__ ssh 'echo UNTAR_DONE'` (and `printf 'interpreter python3\n' \| ...`) |
| `request_data` argv split (+ Outcome C independence) | record-only fake `ssh` on `PATH` + `kitten ssh --kitten askpass=ssh -- host.test echo hello` (`-O check` RC 1 / 0); **and** default-askpass `kitten ssh ... echo UNTAR_DONE` (0 `-O check`, fresh transport, `request_data=0`) |
| ControlMaster lifecycle | isolated `sshd` on `127.0.0.1:2222` (dedicated keys, `PidFile`, seeded `known_hosts`, `StrictHostKeyChecking=yes`) |
| Shared-memory security | real `get_ssh_data` driven via `./kitty/launcher/kitty +launch <probe>.py` |
| DCS wire | `od -c` of the kitten's captured tty output (`request_data=1` / `request_data=0` captures) |
| fish login shell | `check_bootstrap('sh', tdir, login_shell='fish')` via `kitty +launch` |

All temporary scripts, logs, the isolated `sshd` and its keys, the fake `ssh` shim, every `/dev/shm/kssh-*` object, and all build artifacts are removed at the end of the investigation; `git status --porcelain` is then **empty** (this document is committed) and `git diff 815df1e21..HEAD --name-status` lists exactly one added path — this document (`A blitzy/documentation/kitty_815df1e210e0.md`, §0.7).
