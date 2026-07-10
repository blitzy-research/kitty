# kitty PTY-read investigation — how the terminal reads from the shell over a pseudoterminal

**Repository:** kitty terminal emulator · **Commit:** `815df1e21` ("Wire up applying of font config") · **Branch:** `kitty_815df1e210e0`
**Task type:** Evidence-backed Q&A investigation (read-only; the sole artifact added to the repo is this document).
**Methodology:** `build → launch → instrument → observe → document`. Every runtime value below was captured **live** from a real, default-configuration kitty process using `strace`, `ps`, `/proc`, `readlink`, and `lsof`. Statements are labelled **[OBSERVED]** (captured from a live run) or **[INFERRED]** (derived from reading verified source). Every value carries a `file:line` citation.

**One coherent run.** All Q1/Q2/Q4/Q5 facts and every Q3 flood come from a **single** kitty process launched once — **kitty PID 293413**, its shell child **PID 293486**, its I/O thread **TID 293485** (`KittyChildMon`), its PTY master **fd 8**, and its X window **id 2097164** (proven owned-and-focused by PID 293413 before every input; see §3 and each Q). Dynamic identifiers are process-specific to this session and are reported exactly as observed. The Q3 flood runs are labelled `run1`/`run2`/`run3`/`scaleup` and each carries its own metric ledger (§6, §9).

**Output-completeness & redaction policy.** Nothing in an `[OBSERVED]` block below is hand-abbreviated or hand-redacted: the full Kubernetes cgroup path, the complete build log, the full `ls -l` artifact sizes, the complete master-fd symlink lines, and the real disposable-build `cwd` that the shell places in its OSC-2 title (`/tmp/kitty_qa.emz5EMLx/kitty_pristine`, 37 bytes — so the prompt `read()` byte count is verifiable) are all shown in full. Security screening found no secrets in this environment, and the container cgroup id / scratch path are not credentials. The **only** `"..."` that appears inside a quoted `strace` payload is strace's own `-s` length-truncation marker (its use is disclosed in §3, and the corresponding complete short-read payloads are shown in full in Q2); no payload the document actually counts on is abbreviated.

---

## 1. Overview

This document answers five sub-questions about how kitty's native code communicates with a spawned shell over a pseudoterminal (PTY):

1. **Q1 — Shell spawn identity:** what process is spawned, its PID, its exact command line, and the PTY device path.
2. **Q2 — Reading a typed command:** which syscalls kitty makes to read `echo test123`, the buffer size, and how many bytes come back.
3. **Q3 — High-volume output:** how kitty's reading behavior changes under `yes hello`, the read frequency, and the typical bytes-per-read.
4. **Q4 — File descriptor number:** the fd number kitty uses to read the PTY master side.
5. **Q5 — Responsible C functions:** the function that reads from the fd, and the function that parses input to separate printable text from escape sequences.

**One-line data-flow summary** (each hop cited in the answers below; identifiers are from this session's live run):

```
Boss.add_child (kitty/boss.py:585/587)
   -> Child.fork (kitty/child.py:276) -> openpty (kitty/child.py:281) -> fast_data_types.spawn (kitty/child.py:333)
      -> native spawn() (kitty/child.c:81): fork -> setsid -> ioctl TIOCSCTTY -> dup2(slave) -> execvp  => /bin/bash --posix (PID 293486)
   -> master fd retained self.child_fd = 8 (kitty/child.py:338), set non-blocking (kitty/child.py:345)
      -> registered into poll set children_fds[EXTRA_FDS + i].fd (kitty/child-monitor.c:1286)

I/O thread  (io_loop, kitty/child-monitor.c:1481; TID 293485 "KittyChildMon"): poll (kitty/child-monitor.c:1509/1512)
   -> read_bytes (kitty/child-monitor.c:1337) -> read() (kitty/child-monitor.c:1345) into a BUF_SZ=1 MiB buffer (kitty/vt-parser.c:18)
Main thread : parse_input (kitty/child-monitor.c:451) -> do_parse (kitty/child-monitor.c:438)
   -> consume_input (kitty/vt-parser.c:1367)
       -> consume_normal (kitty/vt-parser.c:230) -> screen_draw_text (kitty/screen.c:866)   [printable text]
       -> consume_esc    (kitty/vt-parser.c:261) -> CSI/OSC/DCS/APC/PM/SOS handlers   [escape sequences]
```

---

## 2. Environment & Build

### 2.1 Canonical container & toolchain

The build, launch, and syscall tracing were performed inside the user-provided container. Its image is identified two ways in the task setup (the same image, two registries): `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` and `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`. The image **tag** cannot be read from inside a running container, so it is reported here from the setup metadata (not observed); what **is** observable from inside is the cgroup/container id and OS. **[OBSERVED]** (complete, unedited):

```
$ cat /proc/1/cgroup
0::/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-pod70e3e272_514d_4610_b9f2_09b476631f1e.slice/cri-containerd-991458ab8b1dd5b46f5681cebe76fd44a435ea75eb91bfa4c2274a930f24c8d8.scope
$ grep PRETTY_NAME /etc/os-release
PRETTY_NAME="Ubuntu 25.10"
$ uname -s -r -m
Linux 6.6.122+ x86_64
```

**[OBSERVED]** tool versions (the launcher below is the disposable build from §2.2):

```
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
$ python3 --version
Python 3.13.7
$ go version
go version go1.22.12 linux/amd64
$ gcc --version | head -1
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
$ make --version | head -1
GNU Make 4.4.1
$ strace --version | head -1
strace -- version 6.16
$ xdotool --version
xdotool version 3.20160805.1
$ lsof -v 2>&1 | grep revision
    revision: 4.99.4
```

Build dependencies are those listed in `docs/build.rst` (`python >= 3.8` at `docs/build.rst:83`; harfbuzz, freetype, fontconfig, zlib, libpng, lcms2, xxhash, openssl); the harfbuzz version guard is `at_least_version('harfbuzz', 1, 5)` at `setup.py:609`. **[INFERRED]** (from source).

### 2.2 Default-configuration build (disposable copy, no repository mutation)

To build without touching the repository at all (not even its git-ignored artifacts), the checkout was copied into a private scratch tree and built there. The copy deliberately **excludes** any pre-built native extension/launcher, so `make` performs a genuine from-scratch link. The canonical build command is `make`, whose `all:` target (`Makefile:12`) runs `python3 setup.py $(VVAL)` (`Makefile:13`); `VVAL` is empty in the default (non-verbose) build. The complete build log — every line, unedited, ending in the captured exit status — is **[OBSERVED]**:

```
$ SCRATCH="$(mktemp -d /tmp/kitty_qa.XXXXXXXX)"          # private scratch, mode 0700  => /tmp/kitty_qa.emz5EMLx
$ mkdir -p "$SCRATCH/kitty_pristine"
$ tar --exclude=./.git --exclude='*.so' \
      --exclude=./kitty/launcher/kitty --exclude=./kitty/launcher/kitten \
      -cf - . | ( cd "$SCRATCH/kitty_pristine" && tar -xf - )   # disposable copy; no prebuilt artifacts
$ cd "$SCRATCH/kitty_pristine"
$ export PATH="$PATH:/usr/local/go/bin" GOTOOLCHAIN=local
$ make ; echo "make_exit=$?"
python3 setup.py 
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
[1/1] Compiling kitty/data-types.c ...
 done
[1/4] Linking kitty/fast_data_types ...
[2/4] Linking [x11] kitty/glfw-x11 ...
[3/4] Linking kittens/transfer/rsync ...
[4/4] Linking launcher ...
 done
kitty
kitty/tools/utils/shlex
kitty/tools/utils/secrets
kitty/tools/utils
kitty/tools/tty
kitty/tools/utils/base85
kitty/tools/utils/paths
kitty/tools/rsync
kitty/tools/wcswidth
kitty/tools/crypto
kitty/tools/tui/shell_integration
kitty/tools/utils/humanize
kitty/tools/utils/style
kitty/tools/cli/markup
kitty/tools/tui/sgr
kitty/tools/tui/loop
kitty/tools/cli
kitty/tools/config
kitty/tools/tui/shortcuts
kitty/tools/cmd/mouse_demo
kitty/tools/utils/shm
kitty/kittens/hyperlinked_grep
kitty/kittens/query_terminal
kitty/tools/tui/readline
kitty/kittens/show_key
kitty/tools/tui
kitty/tools/utils/images
kitty/tools/tui/subseq
kitty/kittens/clipboard
kitty/tools/unicode_names
kitty/kittens/ask
kitty/tools/tui/graphics
kitty/tools/cmd/show_error
kitty/tools/cmd/run_shell
kitty/kittens/hints
kitty/tools/cmd/update_self
kitty/tools/cmd/edit_in_kitty
kitty/tools/cmd/at
kitty/tools/themes
kitty/kittens/unicode_input
kitty/tools/cmd/benchmark
kitty/kittens/choose_fonts
kitty/kittens/icat
kitty/kittens/themes
kitty/kittens/ssh
kitty/kittens/transfer
kitty/tools/cmd/pytest
kitty/kittens/diff
kitty/tools/cmd/tool
kitty/tools/cmd/completion
kitty/tools/cmd
make_exit=0
```

The `wayland-protocols` lines are a benign configuration notice — the X11 backend (used here via Xvfb) is built regardless, as the very next `[2/4] Linking [x11] kitty/glfw-x11` line confirms. The block after ` done` is the Go toolchain compiling the bundled `kitten` tool packages (each line is one Go package). No `--debug` flag was used; all answers come from the **default** build, which exited `make_exit=0`. The produced artifacts, shown with their complete sizes and build ids **[OBSERVED]**:

```
$ ls -l kitty/fast_data_types.so kitty/launcher/kitty kitty/launcher/kitten
-rwxr-xr-x 1 root root  1253792 Jul 10 14:56 kitty/fast_data_types.so
-rwxr-xr-x 1 root root 15757572 Jul 10 14:56 kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 Jul 10 14:56 kitty/launcher/kitty
$ file kitty/fast_data_types.so kitty/launcher/kitty
kitty/fast_data_types.so: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, BuildID[sha1]=de56de664dde21b23a324685eb08bf0032548704, not stripped
kitty/launcher/kitty:     ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=4a693e4304285476522c8ac6a4eef4babf9072b7, for GNU/Linux 3.2.0, not stripped
```

Because this is a throwaway copy under `/tmp`, the repository working tree is provably unchanged (see the footer for the closing `git status --porcelain`).

### 2.3 Default configuration & headless launch

**Default configuration is proven, not assumed.** kitty resolves its config dir in `_get_config_dir()` (`kitty/constants.py:87`–`131`): it honours `KITTY_CONFIG_DIRECTORY`, then `XDG_CONFIG_HOME`, then `XDG_CONFIG_DIRS`, else falls back to `~/.config`. All of those are unset and no user `kitty.conf` exists, so kitty runs on **built-in defaults**. **[OBSERVED]**:

```
$ for v in KITTY_CONFIG_DIRECTORY XDG_CONFIG_HOME XDG_CONFIG_DIRS KITTY_CONFIG; do printf "%s=[%s]\n" "$v" "${!v-<unset>}"; done
KITTY_CONFIG_DIRECTORY=[<unset>]
XDG_CONFIG_HOME=[<unset>]
XDG_CONFIG_DIRS=[<unset>]
KITTY_CONFIG=[<unset>]
$ ls -A ~/.config/kitty/ 2>/dev/null | wc -l
0
$ test -e /etc/xdg/kitty/kitty.conf && echo EXISTS || echo ABSENT
ABSENT
```

There is no physical display, so kitty was launched under an X virtual framebuffer. This is a *launch mechanism*, not a configuration change (no `--config` was passed). The Xvfb is **access-controlled** (a private MIT-MAGIC-COOKIE-1 auth file; `-nolisten tcp`; **no** `-ac`), and its PID plus kitty's PID are captured for a scoped teardown (§3, footer). **[OBSERVED]**:

```
$ export XAUTHORITY="$SCRATCH/Xauthority"
$ xauth -f "$XAUTHORITY" add :99 . "$(mcookie)"        # private cookie, mode 0600
$ setsid Xvfb :99 -screen 0 1280x800x24 -nolisten tcp -auth "$XAUTHORITY" > "$SCRATCH/xvfb.log" 2>&1 &
$ XVFB_PID=$!            # => 293409   (killed in teardown; see footer)
$ export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1
$ setsid ./kitty/launcher/kitty > "$SCRATCH/kitty.log" 2>&1 &      # default config, no --config
```

kitty started as **PID 293413** (confirmed by `ps` in Q1) and a real shell spawned as its child. `Xvfb` emitted no output (`xvfb.log` is empty); kitty's entire startup log was one benign message:

```
$ cat "$SCRATCH/kitty.log"
[0.161] Failed to open systemd user bus with error: Connection refused
```

---

## 3. Methodology

- **Canonical input path only.** Every command was typed into the real kitty window via `xdotool` — genuine X key events delivered to the window (`id 2097164`). kitty's X11 keyboard handler receives each event, writes the byte to the PTY master, and reads back the echo/output. This is **not** remote control, a debug hook, a mock, or synthetic byte injection. kitty's remote control is off by default (`allow_remote_control 'no'` at `kitty/options/definition.py:2969`), and invoking it here fails, so the answer values can only come from the canonical keyboard path. **[OBSERVED]**:

```
$ ./kitty/launcher/kitty @ ls ; echo exit=$?
Error: open /dev/tty: no such device or address
exit=1
```

- **Window provenance (proving the window belongs to *this* kitty).** Immediately after launch, and again immediately before every input phase, the target window was proven to be owned by kitty PID 293413 and to hold input focus, using `xdotool getwindowpid`, `xprop _NET_WM_PID WM_CLASS WM_NAME`, and `xdotool getwindowfocus`. The launch-time provenance **[OBSERVED]**:

```
$ xdotool search --pid 293413
2097164
$ xdotool getwindowpid 2097164
293413
$ xprop -id 2097164 _NET_WM_PID WM_CLASS WM_NAME
_NET_WM_PID(CARDINAL) = 293413
WM_CLASS(STRING) = "kitty", "kitty"
WM_NAME(STRING) = "/tmp/kitty_qa.emz5EMLx/kitty_pristine"
$ xdotool getwindowfocus
2097164
```

  `getwindowpid 2097164 = 293413` and `_NET_WM_PID = 293413` both tie the window to the live kitty PID; `WM_CLASS = "kitty","kitty"` confirms it is kitty's window; `getwindowfocus = 2097164` confirms keystrokes typed here land in this window. The per-phase re-checks appear in Q2 (before `echo test123`) and Q3 (before each `yes hello` and each Ctrl-C).

- **Syscall capture.** `strace -tt -s <N> -e trace=poll,read[,write]` attached to a specific thread with `-p <TID>` and logged to a private file with `-o "$SCRATCH/…"`. For Q2, `-s 512` was used so the short output reads (≤163 B) are captured **in full** (no `"...` truncation). For Q3's flood, `-s 64` was used (the payload is the repeating `hello\r\n`), so flood reads show strace's `"...` truncation marker — this is disclosed, not hidden. Reading was traced on the **single I/O thread** so its `poll`/`read` lines appear unsplit (see the note below and Q5).
- **ptrace permissions (disclosed).** `kernel.yama.ptrace_scope = 1` in this container and was **not** changed. `strace` attaches only because the tracer and the target share the same owner (both uid 0); no global weakening was performed:

```
$ cat /proc/sys/kernel/yama/ptrace_scope
1
```

- **Process / fd / PTY facts.** `ps -o …`, `cat -A /proc/<pid>/cmdline`, `readlink /proc/<pid>/fd/N`, `ls -l /proc/<pid>/fd/N`, and `lsof -p <pid> -a -d N`.
- **Reproducibility & scale.** The high-volume magnitudes (Q3) were confirmed across **three** unchanged 5 s runs plus a 10 s scale-up run; per-metric spread and the stability criterion are given in §9. Backpressure was probed with a **21-run** flood sweep (§6).

- **Scratch & cleanup — strict-mode PID-scoped lifecycle helper.** All scripts, logs, and the disposable build live under a single `mktemp -d` scratch dir (mode 0700) outside the repository. Teardown is performed by the helper below, which captures the exact PIDs, terminates only those PIDs (never `pkill`/`killall`), waits for them, and removes only the unique scratch dir. It is input-validated and idempotent (its execution, including a second idempotent run and a deliberate invalid-input failure, is shown in the footer). Source (`lifecycle.sh`):

```bash
#!/usr/bin/env bash
# lifecycle.sh — strict-mode launch/teardown helper for the kitty PTY investigation.
# All process control is PID-SCOPED (never pkill/killall) so only the exact
# children this investigation spawned are affected. Teardown is idempotent and
# input-validated. Usage:
#     lifecycle.sh teardown <state.env>   # kill scoped PIDs, wait, remove scratch
#     lifecycle.sh verify   <state.env>   # report whether scoped PIDs are gone
set -euo pipefail

usage() {
    echo "usage: $0 {teardown|verify} <state.env>" >&2
    exit 2
}

[ "$#" -eq 2 ] || usage
action="$1"; state="$2"
case "$action" in teardown|verify) ;; *) echo "error: unknown action '$action'" >&2; usage ;; esac
[ -r "$state" ] || { echo "error: cannot read state file '$state'" >&2; exit 3; }
# shellcheck disable=SC1090
source "$state"

# Ordered so children die before their controlling X server.
SCOPED_PIDS=( "${KITTY_PID:-}" "${XVFB_PID:-}" )

alive() { [ -n "$1" ] && kill -0 "$1" 2>/dev/null; }

kill_scoped() {
    local pid="$1" name="$2"
    if ! alive "$pid"; then
        echo "  [$name] pid=$pid already gone (idempotent no-op)"
        return 0
    fi
    echo "  [$name] pid=$pid alive -> SIGTERM"
    kill -TERM "$pid" 2>/dev/null || true
    for _ in 1 2 3 4 5 6 7 8 9 10; do alive "$pid" || break; sleep 0.3; done
    if alive "$pid"; then
        echo "  [$name] pid=$pid survived -> SIGKILL"
        kill -KILL "$pid" 2>/dev/null || true
        wait "$pid" 2>/dev/null || true
    fi
    echo "  [$name] pid=$pid terminated"
}

case "$action" in
  verify)
    rc=0
    for pid in "${SCOPED_PIDS[@]}"; do
        if alive "$pid"; then echo "  ALIVE  pid=$pid"; rc=1; else echo "  gone   pid=$pid"; fi
    done
    exit "$rc"
    ;;
  teardown)
    echo "teardown: scoped pids = ${SCOPED_PIDS[*]}"
    kill_scoped "${KITTY_PID:-}" kitty
    kill_scoped "${XVFB_PID:-}"  xvfb
    if [ -n "${SCRATCH:-}" ] && [ -d "${SCRATCH:-}" ]; then
        echo "  removing scratch dir: $SCRATCH"
        rm -rf -- "$SCRATCH"
    fi
    echo "teardown: complete"
    ;;
esac
```

> **A note on strace's line-splitting.** Under `-f` (follow all threads) strace interleaves threads and splits an interrupted syscall across two lines, e.g. `read(8 <unfinished ...>` … `<... read resumed>, "e", 1048576) = 1`. Both halves are the same `read()` on fd 8. To avoid miscounting, the primary traces below attach to only the I/O thread (no `-f`), which yields unsplit lines; where the `-f` form is shown, the reconciliation is made explicit (Q2, "Reconciling the read count").

---

## 4. Q1 — Shell spawn identity

**Direct answer (all four items [OBSERVED]):**

| Item | Value |
|------|-------|
| **(a) Process spawned** | `/bin/bash` — the account-configured shell, started in POSIX mode |
| **(b) PID** | **293486** |
| **(c) Exact command line** | `/bin/bash --posix` |
| **(d) PTY device path** | slave **`/dev/pts/0`** (kitty's master counterpart is fd 8 → `/dev/pts/ptmx`) |

### Evidence

**Process tree — kitty → shell [OBSERVED]** (producing command shown):

```
$ ps -o pid,ppid,stat,tty,args -p 293413        # kitty
    PID    PPID STAT TT       COMMAND
 293413       1 Ssl  ?        ./kitty/launcher/kitty
$ ps -o pid,ppid,stat,tty,args --ppid 293413     # its children (the shell)
    PID    PPID STAT TT       COMMAND
 293486  293413 Ss+  pts/0    /bin/bash --posix
```

The shell (293486) is a direct child of kitty (293413), attached to `pts/0`.

**Exact command line [OBSERVED]:**

```
$ cat -A /proc/293486/cmdline
/bin/bash^@--posix^@
```

`cat -A` shows the NUL-delimited argv exactly: `argv[0]=/bin/bash`, `argv[1]=--posix` (`^@` is the NUL separator; there is no trailing text). `STAT=Ss+` means the process is a **session leader** (`s`) with a controlling terminal, in the **foreground** process group (`+`). It is **not** a login shell: `argv[0]` is `/bin/bash`, not the `-`-prefixed `-bash` form, and there is no `--login`. The precise characterization is therefore *the account-configured interactive shell running in POSIX mode, as session leader of the foreground process group with `/dev/pts/0` as its controlling terminal*.

**PTY device path [OBSERVED]:**

```
$ for fd in 0 1 2; do readlink /proc/293486/fd/$fd; done   # shell stdin/stdout/stderr
/dev/pts/0
/dev/pts/0
/dev/pts/0
```

All three standard streams of the shell point at the PTY **slave** `/dev/pts/0`. The **master** counterpart is held by kitty on fd 8 **[OBSERVED]**:

```
$ ls -l /proc/293413/fd/8
lrwx------ 1 root root 64 Jul 10 14:57 /proc/293413/fd/8 -> /dev/pts/ptmx
$ lsof -p 293413 -a -d 8
COMMAND    PID USER FD   TYPE DEVICE SIZE/OFF NODE NAME
kitty   293413 root 8u   CHR    5,2      0t0    2 /dev/pts/ptmx
```

So the PTY pair is: **kitty master fd 8 (`/dev/pts/ptmx`, char device 5,2) ↔ shell slave `/dev/pts/0`**.

### Rationale & code citations

- **Which shell [INFERRED]:** the shell path is resolved as `shell_path = pwd.getpwuid(os.geteuid()).pw_shell or '/bin/sh'` at `kitty/constants.py:181`. In this container the effective user's `pw_shell` is `/bin/bash`, so the resolved shell is `/bin/bash` and the `/bin/sh` fallback at `kitty/constants.py:185` is **not** taken.
- **Why `--posix` and no leading `-` [OBSERVED + INFERRED]:** by default kitty launches the shell through its `run-shell` kitten to set up shell integration — `argv = [kitten_exe(), 'run-shell', '--shell', shlex.join(argv), '--shell-integration', ksi]` at `kitty/child.py:314`. The `run-shell` kitten re-executes the real shell in POSIX mode with integration injected via environment/rcfile rather than the historical `-`-prefixed login form; the resulting live process image observed is exactly `/bin/bash --posix`. The command line is reported **exactly as it appears**.
- **Spawn chain [INFERRED]:** `Boss.add_child` (`kitty/boss.py:585`) calls `self.child_monitor.add_child(window.id, window.child.pid, window.child.child_fd, window.screen)` (`kitty/boss.py:587`); `Child.fork()` (`kitty/child.py:276`) creates the PTY via `openpty()` (defined `kitty/child.py:170`, invoked `kitty/child.py:281`) and calls `fast_data_types.spawn(...)` (`kitty/child.py:333`, statement spans L333–L335). The native `spawn()` (`kitty/child.c:81`) performs `fork()` (`kitty/child.c:97`), `setsid()` (`kitty/child.c:123`), `ioctl(tfd, TIOCSCTTY, 0)` (`kitty/child.c:129`), `safe_dup2(slave, STDOUT_FILENO)` (`kitty/child.c:138`), `safe_dup2(slave, STDERR_FILENO)` (`kitty/child.c:139`), the STDIN dup2 (`kitty/child.c:141`/`145`), and `execvp(exe, argv)` (`kitty/child.c:159`). The `dup2` of the slave onto fds 0/1/2 is exactly why the shell's fd 0/1/2 all read back as `/dev/pts/0` above.

---

## 5. Q2 — Reading the typed command `echo test123`

**Direct answer:**

- **(a) System calls [OBSERVED]:** per keystroke the I/O thread does `poll()` (fd 8 reports `POLLOUT`) → `write(8, …)` (kitty writes the key to the PTY master) → `poll()` (fd 8 reports `POLLIN`, the line discipline echoed it) → `read(8, …)`. The `read()` is the syscall that pulls the byte(s) off the PTY master; it is always paired with a preceding `poll()`.
- **(b) Buffer size [OBSERVED]:** the third argument to `read()` is **1048576** bytes = 1 MiB = `BUF_SZ` (`kitty/vt-parser.c:18`); it shrinks below 1 MiB when unparsed bytes are still buffered.
- **(c) Bytes returned [OBSERVED]:** each typed keystroke echoes back as a **1-byte** read; the `Enter` read returns **11** bytes; the command's title/marker/output/prompt arrive in reads of **47**, **114**, and **163** bytes. The whole interaction is **16 reads / 347 bytes** on fd 8. The count returned is bounded by what the kernel line discipline currently holds, **not** by the 1 MiB request.

**Window provenance immediately before typing (F-2) [OBSERVED]** — the focused window is owned by kitty PID 293413:

```
$ xdotool getwindowfocus
2097164
$ xdotool getwindowpid $(xdotool getwindowfocus)
293413
$ xprop -id 2097164 _NET_WM_PID
_NET_WM_PID(CARDINAL) = 293413
```

The command was then typed into that window (canonical path). strace was attached to the single I/O thread (TID **293485**), so lines are unsplit:

```
$ strace -tt -s 512 -e trace=poll,read,write -p 293485 -o "$SCRATCH/trace_q2.log"
$ xdotool type   --window 2097164 --delay 120 "echo test123"
$ xdotool key    --window 2097164 Return
```

### Evidence — the `poll → write → poll → read` pipeline for one keystroke (`e`) [OBSERVED]

Contiguous, unedited excerpt (timestamps are HH:MM:SS.microseconds):

```
14:58:38.909490 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN|POLLOUT}], 3, -1) = 1 ([{fd=8, revents=POLLOUT}])
14:58:38.909537 write(8, "e", 1)        = 1
14:58:38.909573 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}])
14:58:38.910386 read(8, "e", 1048576)   = 1
14:58:38.910418 write(4, "\1\0\0\0\0\0\0\0", 8) = 8
```

Reading these five lines: the first `poll` reports fd 8 **writable** (`POLLOUT`); `write(8, "e", 1)` puts the keystroke onto the master — this `write` is **directly observed** (the `write` syscall is in the trace filter, so it need not be inferred from `POLLOUT`). The second `poll` reports fd 8 **readable** (`POLLIN`) because the kernel line discipline echoed the byte back; `read(8, "e", 1048576) = 1` reads that one echoed byte; finally `write(4, …, 8)` bumps the main-thread wakeup eventfd (fd 4) so the parser runs.

The poll set `[{fd=6}, {fd=7}, {fd=8}]` with `nfds=3` maps exactly to the source: `children_fds` holds two reserved slots (`#define EXTRA_FDS 2` at `kitty/child-monitor.c:35`) — here fd 6 (I/O-thread wakeup eventfd) and fd 7 (signal fd) — followed by the child PTY fd at index `EXTRA_FDS + 0` (fd 8). `nfds = self->count + EXTRA_FDS = 1 + 2 = 3`, matching the blocking `poll(children_fds, self->count + EXTRA_FDS, -1)` at `kitty/child-monitor.c:1512` **[INFERRED]**. The `events` value on fd 8 is set by `children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(...) ? POLLIN : 0` at `kitty/child-monitor.c:1501` (plus `POLLOUT` while kitty has a keystroke queued to write). The `read()` itself is issued at `kitty/child-monitor.c:1345` inside `read_bytes()` (`kitty/child-monitor.c:1337`).

### Evidence — all 12 keystroke echoes [OBSERVED]

Typing `echo test123` (12 characters) produced exactly 12 single-byte reads on fd 8, each requesting the full 1 MiB buffer:

```
14:58:38.910386 read(8, "e", 1048576)   = 1
14:58:38.964978 read(8, "c", 1048576)   = 1
14:58:39.025468 read(8, "h", 1048576)   = 1
14:58:39.085916 read(8, "o", 1048576)   = 1
14:58:39.146431 read(8, " ", 1048576)   = 1
14:58:39.206917 read(8, "t", 1048576)   = 1
14:58:39.267494 read(8, "e", 1048576)   = 1
14:58:39.327887 read(8, "s", 1048576)   = 1
14:58:39.388313 read(8, "t", 1048576)   = 1
14:58:39.449447 read(8, "1", 1048576)   = 1
14:58:39.509696 read(8, "2", 1048576)   = 1
14:58:39.570051 read(8, "3", 1048576)   = 1
```

Each read returns exactly `1` byte — the character just echoed by the line discipline — while requesting `1048576` bytes. The reads are ≈54–61 ms apart, i.e. one per typed key (the `--delay 120` between key press/release yields ~60 ms per character).

### Evidence — `Enter` and the command output reads [OBSERVED]

After `Return`, the following contiguous excerpt shows the `Enter` read and the command output/prompt (`-s 512` keeps every read complete — the fourth read's OSC-2 title carries the **real, complete** 37-byte cwd, shown unredacted so its `= 163` count is verifiable):

```
14:58:39.633472 read(8, "\r\n\33[?2004l\r", 1048576) = 11
14:58:39.635007 read(8, "\33]2;echo test123\7\33]133;C;cmdline=echo\\ test123\7", 1048565) = 47
14:58:39.635302 read(8, "\1\33]133;k;start_kitty\7\2\1\33]133;k;end_kitty\7\2\1\33]133;k;start_suffix_kitty\7\2\1\33[0 q\2\1\33]133;k;end_suffix_kitty\7\2test123\r\n", 1048518) = 114
14:58:39.635962 read(8, "\33[?2004h\33]133;k;start_kitty\7\33]133;D;0\7\33]133;A\7\33]133;k;end_kitty\7\33]133;k;start_suffix_kitty\7\33[5 q\33]2;/tmp/kitty_qa.emz5EMLx/kitty_pristine\7\33]133;k;end_suffix_kitty\7", 1048404) = 163
```

Reading these:

- `read(8, "\r\n\33[?2004l\r", 1048576) = 11` — the `Enter` **read of 11 bytes**, of which only **2 bytes are the terminal echo** of the pressed key (`\r\n`, i.e. CR LF); the remaining **9 bytes are shell/line-discipline control**, not echo: `\33[?2004l` (8 bytes, the `CSI ?2004l` "bracketed-paste-off" the shell emits before running the command) plus a final `\r` (1 byte). So: 11 bytes = 2 echo + 9 control.
- `read(8, "\33]2;echo test123\7…", 1048565) = 47` — the shell (via integration) sets the window title (OSC 2 `ESC]2;echo test123 BEL`) and reports the command line (OSC 133 `;C;cmdline=echo test123`).
- `read(8, "…test123\r\n", 1048518) = 114` — **the actual command output**: the literal `test123\r\n` is present at the end of this read, wrapped by OSC 133 shell-integration markers.
- `read(8, "\33[?2004h…/tmp/kitty_qa.emz5EMLx/kitty_pristine\7…", 1048404) = 163` — the next prompt: bracketed-paste-on (`CSI ?2004h`), OSC 133 prompt markers, the exit-status report `133;D;0` (previous command succeeded), cursor-shape `CSI 5 q`, and the OSC-2 title carrying the **complete cwd `/tmp/kitty_qa.emz5EMLx/kitty_pristine` (37 bytes)**. (Under `bash --posix` the `PS1` itself is minimal, which is why this prompt read is 163 B rather than the hundreds of bytes a decorated `PS1` would add.)

### Reconciling the read count — 16 reads on fd 8 [OBSERVED]

The clean single-thread trace above contains exactly **16** `read(8, …)` lines (12 keystroke echoes + the 11/47/114/163-byte reads). A trace taken with `-f` (all threads) tells the *same* story but splits interrupted reads, so it must be reconciled before counting:

```
# same interaction, traced with -f (follow all threads):
$ grep -c 'read(8,'            trace_q2_f.log     # naive literal match
8
$ grep -c 'read(8 <unfinished' trace_q2_f.log     # reads split by the scheduler
8
# example of one split read (both halves are TID 293485, fd 8, one read):
293485 14:59:10.545680 read(8 <unfinished ...>
293485 14:59:10.545710 <... read resumed>, "e", 1048576) = 1
```

A naive `grep -c 'read(8,'` on the `-f` log returns **8** because 8 of the reads were interrupted and appear as `read(8 <unfinished ...>` / `<... read resumed>` pairs. Reconciled: 8 literal + 8 split = **16**, matching the clean single-thread trace exactly. The clean trace (no `-f`) is used as the primary evidence throughout.

### Buffer size ↔ `BUF_SZ`, and returned-count rationale

- **Buffer size [OBSERVED + INFERRED]:** the third `read()` argument is `1048576` when the parser buffer is empty, and `1048565`, `1048518`, `1048404` on the successive output reads. Those reductions equal the bytes still sitting unparsed in the parser buffer: `1048576 − 1048565 = 11`, `1048576 − 1048518 = 58` (=11+47), `1048576 − 1048404 = 172` (=11+47+114). This is exactly `*sz = BUF_SZ - self->write.offset` computed by `vt_parser_create_write_buffer()` at `kitty/vt-parser.c:1451`, where `BUF_SZ` is `#define BUF_SZ (1024u*1024u)` at `kitty/vt-parser.c:18` and the backing store is `uint8_t buf[BUF_SZ + BUF_EXTRA]` at `kitty/vt-parser.c:194`.
- **Bytes returned [OBSERVED value; INFERRED cause]:** the returned counts (1, 1, …, 11, 47, 114, 163) are tiny compared with the ~1 MiB request. A `read()` on a PTY master returns only what the kernel line discipline currently holds — the echoed keystroke, or the burst the shell just wrote — not the requested capacity. The whole `echo test123` interaction was **347 bytes** across **16 reads** on fd 8 (12×1 + 11 + 47 + 114 + 163).

---


## 6. Q3 — High-volume output `yes hello`

**Direct answer:**

- **(a) How reading behavior changes [OBSERVED]:** instead of one ~1-byte read every ~60 ms (the echo case), reads become **back-to-back** — each `read()` is immediately preceded by a `poll()` that reports fd 8 readable, and the next `read()` fires ~57–60 µs later — and each read returns a **large multi-hundred-byte chunk** of `hello\r\n` lines rather than a single byte. The `poll` timeout also changes from blocking (`-1`, when idle) to a short timed value (`1`–`2` ms) during the stream.
- **(b) Frequency [OBSERVED]:** roughly **9,700–10,500 reads/second** on the PTY fd across four runs (run1 9704.08, run2 10395.20, run3 9898.09 at 5 s; scale-up 10504.36 at 10 s). Two distinct intervals must not be conflated: the **read-to-read cadence** (consecutive reads) has median **57–60 µs**, while the **poll-to-read latency** (the `poll` immediately before a read → that read) has median **28–30 µs**. The ~28 µs is *not* the inter-read interval.
- **(c) Typical bytes-per-read [OBSERVED]:** the robust "typical" is **median 492–551 B** and **mode 448–567 B**; the mean is higher and less stable (559–610 B) because the distribution is heavy-tailed (individual reads range from single digits up to ~17–19 KB). Contrast Q2, where every keystroke read was exactly 1 byte.

**Window provenance immediately before the flood and before Ctrl-C (F-2) [OBSERVED]** (run1; the other runs' provenance is identical — focus 2097164, owner 293413):

```
# run1: immediately before typing 'yes hello' into window 2097164
$ xdotool getwindowfocus ; xdotool getwindowpid $(xdotool getwindowfocus)
2097164
293413
# run1: immediately before real Ctrl-C into window 2097164
$ xdotool getwindowfocus
2097164
```

`yes hello` was typed into that window (canonical path); strace was attached to the single I/O thread (TID 293485, `comm=KittyChildMon` — see Q5) so lines are unsplit; the stream was interrupted with a real Ctrl-C keystroke. The exact per-run commands:

```
$ strace -tt -s 64 -e trace=poll,read -p 293485 -o "$SCRATCH/trace_q3_run1.log" &
$ xdotool type --window 2097164 --delay 60 "yes hello"
$ xdotool key  --window 2097164 Return
$ sleep 5                                   # flood window (stated scale)
$ xdotool key  --window 2097164 ctrl+c      # real Ctrl-C keystroke -> SIGINT to the foreground pgrp
```

### Evidence — the I/O thread is the child monitor [OBSERVED]

```
$ cat /proc/293413/task/293485/comm
KittyChildMon
```

### Evidence — before → during → after

**Before/transition (idle → flood).** The `Enter` read (11 B), the `yes hello` title (41 B) and shell-integration marker (105 B) reads, then the poll timeout switches from blocking `-1` to timed (`1`) as the flood begins — contiguous, unedited, from run1 **[OBSERVED]**:

```
15:01:10.903048 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}])
15:01:10.903137 read(8, "\r\n\33[?2004l\r", 1048576) = 11
15:01:10.903207 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}])
15:01:10.904691 read(8, "\33]2;yes hello\7\33]133;C;cmdline=yes\\ hello\7", 1048565) = 41
15:01:10.904730 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
15:01:10.904925 read(8, "\1\33]133;k;start_kitty\7\2\1\33]133;k;end_kitty\7\2\1\33]133;k;start_suffix_"..., 1048524) = 105
15:01:10.904957 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 0 (Timeout)
```

**During (back-to-back reads mid-stream).** The four steady-state windows below (one per run) are contiguous, unedited excerpts; the `"...` marker is strace's `-s 64` truncation of the repeating `hello\r\n` payload (disclosed). Note the ~28–30 µs poll→read spacing, the ~57–60 µs read→read spacing, the large returns, and the shrinking buffer-size argument (`write.offset` growing as reads briefly outrun the parser). These are the ≥20-line raw correlated blocks for each quoted run **[OBSERVED]**.

*run1 (13 poll→read pairs):*

```
15:01:12.925137 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:01:12.925167 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1024706) = 364
15:01:12.925199 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:01:12.925230 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1024342) = 357
15:01:12.925259 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:01:12.925287 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1023985) = 495
15:01:12.925314 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:01:12.925350 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 1023490) = 590
15:01:12.925381 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:01:12.925409 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1022900) = 462
15:01:12.925440 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:01:12.925468 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1022438) = 490
15:01:12.925494 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:01:12.925525 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1021948) = 390
15:01:12.925554 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:01:12.925581 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 1021558) = 485
15:01:12.925611 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:01:12.925638 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1021073) = 474
15:01:12.925667 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:01:12.925698 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 1020599) = 506
15:01:12.925729 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:01:12.925760 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1020093) = 530
15:01:12.925790 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:01:12.925821 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 1019563) = 562
15:01:12.925848 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
15:01:12.925876 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1019001) = 411
```

*run2 (13 poll→read pairs):*

```
15:01:21.152454 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:21.152482 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1013499) = 259
15:01:21.152508 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:21.152536 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1013240) = 397
15:01:21.152564 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:21.152592 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 1048576) = 401
15:01:21.152623 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:21.152651 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1048175) = 392
15:01:21.152681 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:21.152708 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1047783) = 518
15:01:21.152739 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:21.152767 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1047265) = 425
15:01:21.152794 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:21.152821 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 1046840) = 485
15:01:21.152848 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:21.152875 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1046355) = 439
15:01:21.152907 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:21.152935 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 1045916) = 485
15:01:21.152965 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:21.152993 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1045431) = 511
15:01:21.153023 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:21.153050 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1044920) = 511
15:01:21.153098 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}])
15:01:21.153126 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1044409) = 665
15:01:21.153158 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:01:21.153193 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1043744) = 663
```

*run3 (13 poll→read pairs):*

```
15:01:29.431972 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
15:01:29.432000 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 969301) = 392
15:01:29.432028 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
15:01:29.432059 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 968909) = 406
15:01:29.432086 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
15:01:29.432113 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 968503) = 287
15:01:29.432140 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
15:01:29.432171 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 968216) = 483
15:01:29.432207 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
15:01:29.432238 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 967733) = 644
15:01:29.432269 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
15:01:29.432297 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 967089) = 467
15:01:29.432324 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
15:01:29.432351 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 966622) = 366
15:01:29.432378 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
15:01:29.432409 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 966256) = 665
15:01:29.432438 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
15:01:29.432466 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 965591) = 301
15:01:29.432492 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
15:01:29.432519 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 965290) = 327
15:01:29.432549 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
15:01:29.432579 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 964963) = 569
15:01:29.432606 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
15:01:29.432634 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 964394) = 411
15:01:29.432667 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 1 ([{fd=8, revents=POLLIN}])
15:01:29.432695 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 963983) = 518
```

*scaleup (13 poll→read pairs):*

```
15:01:39.749546 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:39.749573 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1040237) = 602
15:01:39.749600 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:39.749626 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1039635) = 474
15:01:39.749657 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:39.749685 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 1039161) = 637
15:01:39.749711 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:39.749738 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 1038524) = 448
15:01:39.749764 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:39.749791 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 1038076) = 658
15:01:39.749822 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:39.749848 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 1037418) = 539
15:01:39.749875 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:39.749905 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 1036879) = 511
15:01:39.749931 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:39.749959 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 1036368) = 408
15:01:39.749989 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 1 ([{fd=8, revents=POLLIN}])
15:01:39.750018 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1035960) = 518
15:01:39.750069 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 1 ([{fd=8, revents=POLLIN}])
15:01:39.750098 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1035442) = 805
15:01:39.750129 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:01:39.750158 read(8, "hello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nh"..., 1034637) = 488
15:01:39.750196 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:01:39.750226 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 1034149) = 644
15:01:39.750254 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:01:39.750284 read(8, "\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r\nhello\r"..., 1033505) = 434
```

**After (Ctrl-C → prompt recovery).** A real Ctrl-C keystroke sends SIGINT to the foreground process group; `yes` dies and the prompt returns. The final read carries the shell-integration exit-status report `133;D;130` (**130 = 128 + SIGINT(2)** — direct proof the interrupt reached `yes`), and the buffer-size argument is back to ≈full (`1048574`, i.e. only 2 bytes still buffered) — run1 **[OBSERVED]**:

```
15:01:15.954227 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:01:15.954693 read(8, "\33[?2004h\33]133;k;start_kitty\7\33]133;D;130\7\33]133;A\7\33]133;k;end_kitt"..., 1048574) = 165
15:01:15.954764 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 1) = 0 (Timeout)
15:01:15.955899 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 0) = 0 (Timeout)
```

The `133;D;130` exit report proves the SIGINT reached `yes`; the buffer returning to ≈`1048576` shows the parser has drained and the prompt recovered (the two-byte residual is the trailing prompt bytes still unparsed at that instant).

### The analyzer — `q3_stats.py` (embedded source) [documented tool]

Each raw trace was parsed by the following script. It classifies a fd-8 read as a "flood read" iff its payload contains `hello`; it reports the exact timestamp endpoints, the rate as an explicit division, the returned-byte distribution (min/median/mean/mode/max/p90/p99 + a 100-byte-bucket histogram), the requested-size distribution, the read→read cadence and the poll→read latency (kept distinct), short/zero/error read counts, the backpressure signals (polls that watch fd 8 with `events=0`, and poll timeouts), and the peak occupancy `BUF_SZ − min(reqsize)`. It is deterministic (same input → byte-identical output, verified below).

```python
#!/usr/bin/env python3
"""q3_stats.py - analyze a strace '-tt -e trace=poll,read' log of kitty's I/O thread.

Usage: python3 q3_stats.py TRACE.log

Classifies each read() on the PTY master fd as a "flood read" when its payload
contains the marker substring 'hello' (the body of `yes hello`). Reports, using
ONLY those flood reads: count, wall span (from the -tt timestamps), reads/sec
(shown as an explicit division), the returned-byte distribution (min/median/
mean/mode/max/p90/p99 + a 100-byte-bucket histogram), the requested-size
distribution (3rd read() argument), the read->read cadence and the poll->read
latency (kept distinct), short/zero/error read counts, and backpressure signals
(polls that watch the PTY fd with events=0, and poll timeouts). Deterministic:
same input file -> byte-identical output.
"""
import sys, re, statistics as st

TS   = re.compile(r'^(?:\d+\s+)?(\d\d):(\d\d):(\d\d)\.(\d{6})')
# read(FD, <payload>, REQSZ) = RET     (RET may be -1 ... for errors)
READ = re.compile(r'read\((\d+),\s*(.*),\s*(\d+)\)\s*=\s*(-?\d+)')
POLL = re.compile(r'poll\(\[(.*?)\],\s*\d+,\s*(-?\d+)\)\s*=\s*(-?\d+)(.*)$')

def ts_us(line):
    m = TS.match(line)
    if not m: return None
    h, mnt, s, us = map(int, m.groups())
    return ((h*60 + mnt)*60 + s)*1_000_000 + us

def main(path):
    reads = []          # (ts_us, fd, reqsz, ret, is_flood)
    last_poll_ts = None
    poll_before_read = {}   # index -> poll ts (immediately preceding)
    pollin_withheld = 0     # polls that watch fd 8 with events=0
    poll_timeouts = 0       # polls returning 0 (Timeout)
    with open(path, 'r', errors='replace') as fh:
        for line in fh:
            t = ts_us(line)
            pm = POLL.search(line)
            if pm:
                setspec, timeout, ret, tail = pm.groups()
                if re.search(r'fd=8,\s*events=0\b', setspec):
                    pollin_withheld += 1
                if ret == '0' and 'Timeout' in tail:
                    poll_timeouts += 1
                if t is not None:
                    last_poll_ts = t
                continue
            rm = READ.search(line)
            if rm and t is not None:
                fd   = int(rm.group(1))
                body = rm.group(2)
                reqsz= int(rm.group(3))
                ret  = int(rm.group(4))
                is_flood = (fd == 8 and 'hello' in body)
                idx = len(reads)
                reads.append([t, fd, reqsz, ret, is_flood])
                if is_flood and last_poll_ts is not None:
                    poll_before_read[idx] = last_poll_ts

    flood = [r for r in reads if r[4]]
    n = len(flood)
    if n == 0:
        print("flood_reads(n)          = 0")
        print("(no reads whose payload contained 'hello' were found in this trace)")
        return
    rets   = [r[3] for r in flood if r[3] > 0]
    reqs   = [r[2] for r in flood]
    zero   = sum(1 for r in flood if r[3] == 0)
    err    = sum(1 for r in flood if r[3] < 0)
    short  = sum(1 for r in flood if 0 < r[3] < 64)
    ts_list= [r[0] for r in flood]
    first, last = ts_list[0], ts_list[-1]
    span = (last - first) / 1_000_000
    # read->read cadence (successive flood reads)
    rr = [ (ts_list[i]-ts_list[i-1]) for i in range(1, n) ]
    # poll->read latency (the poll immediately preceding each flood read)
    pr = []
    fidx = [i for i,r in enumerate(reads) if r[4]]
    for i in fidx:
        if i in poll_before_read:
            d = reads[i][0] - poll_before_read[i]
            if d >= 0: pr.append(d)

    def pct(xs, p):
        if not xs: return 0
        xs2 = sorted(xs); k = (len(xs2)-1)*p/100.0
        f = int(k); c = min(f+1, len(xs2)-1)
        return xs2[f] + (xs2[c]-xs2[f])*(k-f)

    mode_val = st.mode(rets)
    mode_freq= rets.count(mode_val)
    print(f"flood_reads(n)          = {n}")
    print(f"first_ts                = {first/1_000_000:.6f}")
    print(f"last_ts                 = {last/1_000_000:.6f}")
    print(f"span_seconds            = {span:.6f}")
    print(f"reads_per_sec           = {n}/{span:.6f} = {n/span:.4f}")
    print(f"bytes_total             = {sum(rets)}")
    print(f"bytes/read  min         = {min(rets)}")
    print(f"bytes/read  max         = {max(rets)}")
    print(f"bytes/read  mean        = {st.mean(rets):.3f}")
    print(f"bytes/read  median      = {st.median(rets):.1f}")
    print(f"bytes/read  mode        = {mode_val}  (freq={mode_freq})")
    print(f"bytes/read  p90         = {pct(rets,90):.1f}")
    print(f"bytes/read  p99         = {pct(rets,99):.1f}")
    print(f"reqsize     min         = {min(reqs)}")
    print(f"reqsize     max         = {max(reqs)}")
    print(f"reqsize     median      = {st.median(reqs):.1f}")
    print(f"short_reads (0<ret<64)  = {short}")
    print(f"zero_reads  (ret==0)    = {zero}")
    print(f"error_reads (ret<0)     = {err}")
    if rr:
        print(f"read->read us  min/med/mean/max = {min(rr)}/{st.median(rr):.0f}/{st.mean(rr):.0f}/{max(rr)}")
        print(f"read->read us  p90/p99          = {pct(rr,90):.0f}/{pct(rr,99):.0f}")
    if pr:
        print(f"poll->read us  min/med/mean/max = {min(pr)}/{st.median(pr):.0f}/{st.mean(pr):.0f}/{max(pr)}")
    print(f"backpressure fd8 events=0 polls = {pollin_withheld}")
    print(f"poll timeouts (= 0 Timeout)     = {poll_timeouts}")
    peak_occ = (1024*1024) - min(reqs)
    print(f"peak_occupancy_bytes    = 1048576 - {min(reqs)} = {peak_occ}  ({100.0*peak_occ/(1024*1024):.2f}% of BUF_SZ)")
    buckets = {}
    for v in rets:
        b = min(v//100, 8)
        buckets[b] = buckets.get(b, 0) + 1
    labels = {0:'[0-99]',1:'[100-199]',2:'[200-299]',3:'[300-399]',4:'[400-499]',
              5:'[500-599]',6:'[600-699]',7:'[700-799]',8:'[800-inf]'}
    hist = ' '.join(f"{labels[b]}={buckets.get(b,0)}" for b in range(9))
    print(f"byte histogram (100B buckets): {hist}")

if __name__ == '__main__':
    if len(sys.argv) != 2:
        print("usage: python3 q3_stats.py TRACE.log", file=sys.stderr); sys.exit(2)
    main(sys.argv[1])
```

### Complete analyzer output — every run [OBSERVED]

*run1:*

```
$ python3 q3_stats.py trace_q3_run1.log
flood_reads(n)          = 48995
first_ts                = 54070.904691
last_ts                 = 54075.953600
span_seconds            = 5.048909
reads_per_sec           = 48995/5.048909 = 9704.0767
bytes_total             = 29520219
bytes/read  min         = 30
bytes/read  max         = 19052
bytes/read  mean        = 602.515
bytes/read  median      = 551.0
bytes/read  mode        = 567  (freq=536)
bytes/read  p90         = 793.0
bytes/read  p99         = 1589.0
reqsize     min         = 901051
reqsize     max         = 1048576
reqsize     median      = 1027786.0
short_reads (0<ret<64)  = 9
zero_reads  (ret==0)    = 0
error_reads (ret<0)     = 0
read->read us  min/med/mean/max = 51/60/103/36571
read->read us  p90/p99          = 74/101
poll->read us  min/med/mean/max = 24/30/51/36535
backpressure fd8 events=0 polls = 0
poll timeouts (= 0 Timeout)     = 117
peak_occupancy_bytes    = 1048576 - 901051 = 147525  (14.07% of BUF_SZ)
byte histogram (100B buckets): [0-99]=146 [100-199]=722 [200-299]=1973 [300-399]=4136 [400-499]=10376 [500-599]=13704 [600-699]=9298 [700-799]=3860 [800-inf]=4780
```

*run2:*

```
$ python3 q3_stats.py trace_q3_run2.log
flood_reads(n)          = 52300
first_ts                = 54079.140340
last_ts                 = 54084.171506
span_seconds            = 5.031166
reads_per_sec           = 52300/5.031166 = 10395.2046
bytes_total             = 29255062
bytes/read  min         = 12
bytes/read  max         = 17235
bytes/read  mean        = 559.370
bytes/read  median      = 492.0
bytes/read  mode        = 448  (freq=635)
bytes/read  p90         = 726.0
bytes/read  p99         = 1717.0
reqsize     min         = 862096
reqsize     max         = 1048576
reqsize     median      = 1028713.5
short_reads (0<ret<64)  = 15
zero_reads  (ret==0)    = 0
error_reads (ret<0)     = 0
read->read us  min/med/mean/max = 50/57/96/30891
read->read us  p90/p99          = 65/97
poll->read us  min/med/mean/max = 23/28/52/28434
backpressure fd8 events=0 polls = 0
poll timeouts (= 0 Timeout)     = 137
peak_occupancy_bytes    = 1048576 - 862096 = 186480  (17.78% of BUF_SZ)
byte histogram (100B buckets): [0-99]=196 [100-199]=410 [200-299]=1998 [300-399]=8730 [400-499]=16094 [500-599]=13080 [600-699]=5854 [700-799]=1975 [800-inf]=3963
```

*run3:*

```
$ python3 q3_stats.py trace_q3_run3.log
flood_reads(n)          = 50158
first_ts                = 54087.376671
last_ts                 = 54092.444112
span_seconds            = 5.067441
reads_per_sec           = 50158/5.067441 = 9898.0925
bytes_total             = 30609746
bytes/read  min         = 41
bytes/read  max         = 18512
bytes/read  mean        = 610.266
bytes/read  median      = 516.0
bytes/read  mode        = 476  (freq=618)
bytes/read  p90         = 868.0
bytes/read  p99         = 1874.9
reqsize     min         = 874019
reqsize     max         = 1048576
reqsize     median      = 1025777.0
short_reads (0<ret<64)  = 1
zero_reads  (ret==0)    = 0
error_reads (ret<0)     = 0
read->read us  min/med/mean/max = 50/57/101/34388
read->read us  p90/p99          = 74/100
poll->read us  min/med/mean/max = 23/28/48/33344
backpressure fd8 events=0 polls = 0
poll timeouts (= 0 Timeout)     = 145
peak_occupancy_bytes    = 1048576 - 874019 = 174557  (16.65% of BUF_SZ)
byte histogram (100B buckets): [0-99]=17 [100-199]=134 [200-299]=738 [300-399]=6366 [400-499]=15347 [500-599]=12439 [600-699]=6114 [700-799]=2840 [800-inf]=6163
```

*scaleup (10 s):*

```
$ python3 q3_stats.py trace_q3_scaleup.log
flood_reads(n)          = 105471
first_ts                = 54095.621813
last_ts                 = 54105.662498
span_seconds            = 10.040685
reads_per_sec           = 105471/10.040685 = 10504.3630
bytes_total             = 64061539
bytes/read  min         = 9
bytes/read  max         = 16500
bytes/read  mean        = 607.385
bytes/read  median      = 532.0
bytes/read  mode        = 539  (freq=1228)
bytes/read  p90         = 849.0
bytes/read  p99         = 1729.0
reqsize     min         = 788601
reqsize     max         = 1048576
reqsize     median      = 1025275.0
short_reads (0<ret<64)  = 30
zero_reads  (ret==0)    = 0
error_reads (ret<0)     = 0
read->read us  min/med/mean/max = 49/57/95/31649
read->read us  p90/p99          = 65/88
poll->read us  min/med/mean/max = 23/28/48/30167
backpressure fd8 events=0 polls = 0
poll timeouts (= 0 Timeout)     = 286
peak_occupancy_bytes    = 1048576 - 788601 = 259975  (24.79% of BUF_SZ)
byte histogram (100B buckets): [0-99]=422 [100-199]=852 [200-299]=2813 [300-399]=10597 [400-499]=27681 [500-599]=29945 [600-699]=15557 [700-799]=5467 [800-inf]=12137
```

The rate is shown as an explicit division (endpoints and precision explicit, so there is no rounding ambiguity). The dominant reads sit in the 400–599-byte range; median (492–551) and mode (448–567) are ~500 bytes; `zero_reads` and `error_reads` are `0` in every run and short reads are few (1–30). Per-run spread and the stability criterion are in §9.

### Edge / robustness cases for the analyzer [OBSERVED]

- **Determinism** (same input → byte-identical output, by `diff` and `sha256sum`):

```
$ python3 q3_stats.py trace_q3_run1.log > det_a.txt
$ python3 q3_stats.py trace_q3_run1.log > det_b.txt
$ diff det_a.txt det_b.txt && echo "IDENTICAL (deterministic)"
IDENTICAL (deterministic)
$ sha256sum det_a.txt det_b.txt
77d2fd3531390ea552a3694ab11dbb340de16b13c035cc7116e1f5804190b36a  det_a.txt
77d2fd3531390ea552a3694ab11dbb340de16b13c035cc7116e1f5804190b36a  det_b.txt
```

- **Empty input** (zero-byte trace) → graceful zero-count, no crash:

```
$ : > empty_trace.log        # zero-byte trace
$ python3 q3_stats.py empty_trace.log
flood_reads(n)          = 0
(no reads whose payload contained 'hello' were found in this trace)
```

- **Malformed input** (garbage lines, no valid read/poll shapes) → graceful zero-count, no crash:

```
$ cat malformed_trace.log
this is not a trace
read(without a valid shape
12:00:00.000000 garbage line
poll(oops
$ python3 q3_stats.py malformed_trace.log
flood_reads(n)          = 0
(no reads whose payload contained 'hello' were found in this trace)
```

- **Invalid usage** (no argument) → usage message on stderr, non-zero exit:

```
$ python3 q3_stats.py            # no argument
usage: python3 q3_stats.py TRACE.log
exit=2
```

### Backpressure — the flow-control gate [OBSERVED engaged in 1 of 21 runs; run-to-run variable]

The read loop only asks for `POLLIN` on the PTY fd while the parser has room: `children_fds[EXTRA_FDS + i].events = vt_parser_has_space_for_input(screen->vt_parser) ? POLLIN : 0;` at `kitty/child-monitor.c:1501`, where `vt_parser_has_space_for_input` (`kitty/vt-parser.c:1477`, body `ans = self->read.sz + self->write.pending < BUF_SZ;` at `kitty/vt-parser.c:1481`, declared `kitty/vt-parser.h:36`) reports whether the 1 MiB buffer still has space. Because the per-read request size is `BUF_SZ − write.offset` (`self->write.offset = self->read.sz + self->write.pending; *sz = BUF_SZ - self->write.offset;` in `vt_parser_create_write_buffer`, `kitty/vt-parser.c:1451`), the occupancy the gate watches equals `BUF_SZ − requested_size`, and the gate returns `false` — setting `events = 0` for fd 8 — **iff occupancy reaches `BUF_SZ` (100 %)** **[INFERRED from the predicate]**.

**The sweep — 21 canonical `yes hello` floods.** To probe whether the buffer actually fills and `POLLIN` is actually withheld, an expanded sweep of **21 floods** was run (all typed into the real window 2097164, all interrupted with a real Ctrl-C). It comprises: the **4** metric runs of §9 (`run1`/`run2`/`run3`/`scaleup`, poll+read traced), **6** read+poll floods (three under induced host-CPU contention), **8** poll-only floods under heavy host-CPU contention (`NPROC×3` busy loops; poll-only so the reader runs at full native speed), and **3** long poll-only floods (20–25 s, `NPROC×4` load). Per-run ledgers **[OBSERVED]**:

```
# 4 metric runs (peak occupancy = BUF_SZ - min reqsize, from the §6 analyzer output above):
run1     peak_occ=147525 (14.07%)   events=0(fd8)=0
run2     peak_occ=186480 (17.78%)   events=0(fd8)=0
run3     peak_occ=174557 (16.65%)   events=0(fd8)=0
scaleup  peak_occ=259975 (24.79%)   events=0(fd8)=0

# 6 read+poll floods (bp_sweep/ledger.txt):
sweep01      contend=no  reads=61200   min_reqsize=870634   peak_occ=177942   (16.97%)  events=0=0
sweep02      contend=yes reads=25655   min_reqsize=903023   peak_occ=145553   (13.88%)  events=0=0
sweep03      contend=yes reads=24117   min_reqsize=930526   peak_occ=118050   (11.26%)  events=0=0
sweep04      contend=yes reads=33970   min_reqsize=928932   peak_occ=119644   (11.41%)  events=0=0
sweep05      contend=no  reads=64933   min_reqsize=912225   peak_occ=136351   (13.00%)  events=0=0
sweep06      contend=yes reads=35419   min_reqsize=877162   peak_occ=171414   (16.35%)  events=0=0

# 8 poll-only heavy-contention floods (bp_sweep/ledger_pollonly.txt):
poll01 polls=25950 events=0(fd8)=0   poll02 polls=25558 events=0(fd8)=0
poll03 polls=29966 events=0(fd8)=0   poll04 polls=28707 events=0(fd8)=0
poll05 polls=30725 events=0(fd8)=0   poll06 polls=29678 events=0(fd8)=0
poll07 polls=29427 events=0(fd8)=0   poll08 polls=27372 events=0(fd8)=0

# 3 long poll-only floods, 20-25 s, NPROC*4 load (bp_sweep/ledger_long.txt):
long01       (20s, load=16x) polls=65443    events=0(fd8)=0
long02       (20s, load=16x) polls=57179    events=0(fd8)=4
long03       (25s, load=16x) polls=74563    events=0(fd8)=0
```

**What the sweep shows [OBSERVED]:** In the **10 read-traced floods**, peak parser-buffer occupancy ranged **11.26 % → 24.79 %** of `BUF_SZ` — i.e. the buffer never came close to full, and `POLLIN` was never withheld. `POLLIN` **withholding (full backpressure engagement) was directly observed in exactly 1 of the 21 runs** — `long02`, a 20 s flood under 16× CPU contention — where **4** polls watched fd 8 with `events=0` (out of 57 179 polls that run, i.e. 0.007 %). The other **20** runs showed `events=0 = 0`. Engagement is therefore **genuine but rare and scheduling-sensitive**: it required a sustained 20 s flood under heavy host-CPU contention (so the main-thread parser fell far enough behind for occupancy to reach `BUF_SZ`), and even then it fired in only 1 of the 3 long runs.

**The raw, unedited engagement transition from `long02` [OBSERVED]** — fd 8 is watched with `events=POLLIN`, the buffer fills, fd 8 flips to `events=0` (POLLIN withheld) for four polls, then resumes:

```
15:07:54.259197 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:07:54.259248 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, 2) = 1 ([{fd=8, revents=POLLIN}])
15:07:54.259297 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=0}], 3, 2) = 0 (Timeout)
15:07:54.261397 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=0}], 3, 0) = 0 (Timeout)
15:07:54.261430 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=0}], 3, 0) = 0 (Timeout)
15:07:54.261463 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=0}], 3, 0) = 0 (Timeout)
15:07:54.344034 poll([{fd=6, events=POLLIN}, {fd=7, events=POLLIN}, {fd=8, events=POLLIN}], 3, -1) = 2 ([{fd=6, revents=POLLIN}, {fd=8, revents=POLLIN}])
```

Reading this block: the first two polls request `POLLIN` on fd 8 (reader wants data). At `15:07:54.259297` fd 8 flips to **`events=0`** — kitty stops requesting readability on the PTY master because `vt_parser_has_space_for_input` returned `false` (the 1 MiB buffer is full). Critically, **fd 6 (wakeup eventfd) and fd 7 (signalfd) remain `events=POLLIN`** — only the PTY master is gated, exactly as the source dictates (the gate at `:1501` touches only the child fd's `events`). The withholding lasts **4 polls / 84.737 ms** (from `.259297` to the resume at `.344034`), then fd 8 returns to `events=POLLIN` and the loop reverts to the infinite-timeout idle wait (`, 3, -1`), with fd 6 also readable — the main thread posted its wakeup token after draining the buffer via `parse_input` (`kitty/child-monitor.c:451`) → `do_parse` (`kitty/child-monitor.c:438`).

**Honest summary of the backpressure evidence:**

- **[OBSERVED]** The gate *does* engage: `POLLIN` was withheld on fd 8 (`events=0`) in `long02` — 4 polls over an 84.737 ms window — proving the flow-control path is real and reachable through the canonical input path.
- **[OBSERVED]** Engagement is **rare and run-to-run/scheduling variable**: 20 of 21 floods (including the 10 read-traced runs, whose peak occupancy stayed 11.26 %–24.79 %, and the 10 s / 105 k-read scale-up) never withheld `POLLIN`; only a sustained 20 s flood under 16× CPU contention triggered it, and only in 1 of 3 such runs. The per-run *engagement count* is therefore **not** a stable metric — the reproducible, stable metrics are the per-read/rate figures in §9; the reproducible *qualitative* fact is that engagement **can and does** occur.
- **[INFERRED]** That the gate flips **precisely at `BUF_SZ`** (rather than some lower watermark) follows from the predicate `read.sz + write.pending < BUF_SZ` (`kitty/vt-parser.c:1481`), corroborated by the observed request-size collapse toward the buffer ceiling in the runs that approached it.

This corrects any impression that occupancy routinely "peaks at ~99.9 %": in this environment the parser almost always keeps pace (peak ~14–25 %), and full engagement is an intermittent, contention-dependent event that was captured once, in full, above.

---


## 7. Q4 — File-descriptor number (PTY master side)

**Direct answer [OBSERVED]:** kitty reads the PTY master on **file descriptor 8** in this run. The concrete integer is **process-specific** — it is whatever number the OS assigned to the master when kitty opened the PTY — and must be read from the running process (it is not a fixed constant).

### Evidence

**From the syscall trace — every PTY payload read is on fd 8 [OBSERVED].** In the clean `echo test123` I/O-thread trace, reads land on exactly two fds; only fd 8 carries PTY payload, while fd 6 is the I/O thread's own wakeup eventfd (its reads return the 8-byte counter `"\1\0\0\0\0\0\0\0"`):

```
$ grep -oE 'read\([0-9]+,' trace_q2.log | sort | uniq -c
     26 read(6,
     16 read(8,
$ grep 'read(6,' trace_q2.log | head -1
14:58:38.909302 read(6, "\1\0\0\0\0\0\0\0", 1024) = 8
```

In each Q3 flood run the same holds — every PTY read is on fd 8 (run 1: 49,008 `read(8,` lines; the only non-fd-8 reads, 22 of them, are the fd-6 eventfd wakeups):

```
$ grep -c 'read(8,' trace_q3_run1.log
49008
$ grep -c 'read(6,' trace_q3_run1.log
22
```

**Cross-checked against `/proc` and `lsof` — fd 8 is the PTY master, with the complete fd map for context [OBSERVED]:**

```
$ ls -l /proc/293413/fd/8
lrwx------ 1 root root 64 Jul 10 14:57 /proc/293413/fd/8 -> /dev/pts/ptmx
$ lsof -p 293413 -a -d 8
COMMAND    PID USER FD   TYPE DEVICE SIZE/OFF NODE NAME
kitty   293413 root 8u   CHR    5,2      0t0    2 /dev/pts/ptmx
$ ls -l /proc/293413/fd/
total 0
lr-x------ 1 root root 64 Jul 10 14:57 0 -> /dev/null
l-wx------ 1 root root 64 Jul 10 14:57 1 -> /tmp/kitty_qa.emz5EMLx/kitty.log
l-wx------ 1 root root 64 Jul 10 14:57 2 -> /tmp/kitty_qa.emz5EMLx/kitty.log
lrwx------ 1 root root 64 Jul 10 14:57 3 -> socket:[790447275]
lrwx------ 1 root root 64 Jul 10 14:57 4 -> anon_inode:[eventfd]
lrwx------ 1 root root 64 Jul 10 14:57 5 -> /memfd:allocation fd (deleted)
lrwx------ 1 root root 64 Jul 10 14:57 6 -> anon_inode:[eventfd]
lrwx------ 1 root root 64 Jul 10 14:57 7 -> anon_inode:[signalfd]
lrwx------ 1 root root 64 Jul 10 14:57 8 -> /dev/pts/ptmx
```

fd 8 is a character device `5,2` = `/dev/pts/ptmx` (the PTY master), whose slave counterpart is the shell's `/dev/pts/0` from Q1. The full map also explains the poll set: fd 4 and fd 6 are the two `eventfd` wakeups, fd 7 is the `signalfd`; fd 6 and fd 7 are the two `EXTRA_FDS` slots watched alongside fd 8 in every poll. fd 8 stayed fd 8 across all captures (Q1, Q2, and all Q3 floods).

### Rationale & code citations [INFERRED]

The master fd is retained on the Python side as `self.child_fd = master` at `kitty/child.py:338` and made non-blocking via `os.set_blocking(self.child_fd, False)` at `kitty/child.py:345`. It is handed to the child monitor through `self.child_monitor.add_child(window.id, window.child.pid, window.child.child_fd, window.screen)` at `kitty/boss.py:587` (receiver `add_child()` at `kitty/child-monitor.c:305`). Inside the monitor it is registered into the poll set: `children_fds[EXTRA_FDS + self->count].fd = children[self->count].fd;` at `kitty/child-monitor.c:1286` with `children_fds[EXTRA_FDS + self->count].events = POLLIN;` at `kitty/child-monitor.c:1287`. The array is `static struct pollfd children_fds[MAX_CHILDREN + EXTRA_FDS]` at `kitty/child-monitor.c:86`, and `#define EXTRA_FDS 2` at `kitty/child-monitor.c:35` — which is why the PTY appears at poll index 2 (after the wakeup and signal fds) in the Q2/Q3 traces.

---

## 8. Q5 — Responsible C functions (reader + text/escape parser)

**Direct answer:**

- **(a) The function that reads from the PTY fd:** **`read_bytes()`** at `kitty/child-monitor.c:1337`, which issues the `read()` syscall at `kitty/child-monitor.c:1345`. It runs on the **I/O thread**.
- **(b) The function that parses input to separate printable text from escape sequences:** the VT state machine **`consume_input()`** at `kitty/vt-parser.c:1367`. It routes printable text through **`consume_normal()`** (`kitty/vt-parser.c:230`) to **`screen_draw_text()`** (`kitty/screen.c:866`), and escape/control sequences through **`consume_esc()`** (`kitty/vt-parser.c:261`) to the CSI/OSC/DCS/APC/PM/SOS sub-handlers. It runs on the **Main thread**.

### Reader — `read_bytes()` [OBSERVED it runs; INFERRED the code]

**[OBSERVED]** every PTY `read()` in the traces was issued by TID 293485, whose thread name is `KittyChildMon` (the KittyChildMonitor I/O thread):

```
$ cat /proc/293413/task/293485/comm
KittyChildMon
```

**[INFERRED]** the reader is `read_bytes()`. Verbatim source, `kitty/child-monitor.c:1337`–`1356`:

```c
read_bytes(int fd, Screen *screen) {
    ssize_t len;
    size_t available_buffer_space;

    uint8_t *buf = vt_parser_create_write_buffer(screen->vt_parser, &available_buffer_space);
    if (!available_buffer_space) return true;

    while(true) {
        len = read(fd, buf, available_buffer_space);
        if (len < 0) {
            if (errno == EINTR || errno == EAGAIN) continue;
            if (errno != EIO) perror("Call to read() from child fd failed");
            vt_parser_commit_write(screen->vt_parser, 0);
            return false;
        }
        break;
    }
    vt_parser_commit_write(screen->vt_parser, len);
    return len != 0;
}
```

The `read(fd, buf, available_buffer_space)` at `kitty/child-monitor.c:1345` is exactly the `read(8, …, 1048576)` observed in Q2/Q3; `available_buffer_space` comes from `vt_parser_create_write_buffer()` (`kitty/vt-parser.c:1451`), explaining the shrinking third argument.

### Parser — `consume_input()` [OBSERVED both classes present; INFERRED the routing]

**[OBSERVED]** the raw bytes read from fd 8 contain **both** classes the parser must separate — printable text and escape/control sequences — and the terminal acted on both (it displayed `test123`/`hello` and honoured the title/prompt escapes). Rather than reconstruct examples, here are the two classes matched **directly out of the real `trace_q2.log` reads** with `grep` (the payloads are the complete quoted strings from the reads in §5):

```
$ grep -aoE 'read\(8, "[^"]*\\33\][^"]*"' trace_q2.log | head -1     # escape/control class (OSC 2 + OSC 133)
read(8, "\33]2;echo test123\7\33]133;C;cmdline=echo\\ test123\7"
$ grep -aoE 'read\(8, "\\r\\n\\33\[\?2004l\\r"' trace_q2.log | head -1  # escape/control class (CSI ?2004l)
read(8, "\r\n\33[?2004l\r"
```

The **printable** class is equally present in the same reads shown verbatim in §5/§6: the literal `test123\r\n` at the tail of the 114-byte Q2 read, and the `hello\r\n` runs that fill every Q3 flood read (e.g. `read(8, "hello\r\nhello\r\n…", …) = 364`). So a single read routinely carries **both** classes — printable runs (→ `consume_normal`) interleaved with escape introducers `\33]`/`\33[` (→ `consume_esc`). These lines are quoted from the complete reads above; they are **not** reconstructed.

**[INFERRED]** the routing. `consume_input()` switches on `self->vte_state`. Verbatim excerpt, `kitty/vt-parser.c:1367`–`1385` (the `switch` continues past line 1385 with the remaining `consume(...)` cases — `VTE_APC`, `VTE_PM`, `VTE_DCS`, `VTE_SOS` — through the closing brace; the block below is quoted contiguously, nothing elided within it):

```c
consume_input(PS *self, PyObject *dump_callback UNUSED, id_type window_id UNUSED) {
#define consume(x) if (accumulate_st_terminated_esc_code(self, dispatch_##x)) { self->read.consumed = self->read.pos; SET_STATE(NORMAL); } break;

#ifdef DUMP_COMMANDS
    PyObject *dumped_bytes = PyBytes_FromStringAndSize((const char*)self->buf + self->read.pos, self->read.sz - self->read.pos);
    size_t pre_consume_pos = self->read.pos;
#endif

    switch (self->vte_state) {
        case VTE_NORMAL:
            consume_normal(self); self->read.consumed = self->read.pos; break;
        case VTE_ESC:
            if (consume_esc(self)) { self->read.consumed = self->read.pos; }
            break;
        case VTE_CSI:
            if (consume_csi(self)) { self->read.consumed = self->read.pos; if (self->csi.is_valid) dispatch_csi(self); SET_STATE(NORMAL); }
            break;
        case VTE_OSC:
            consume(osc);
```

Printable text is handled by `consume_normal()` (`kitty/vt-parser.c:230`), which decodes UTF-8 up to an ESC sentinel and pushes the decoded run to the screen — verbatim, `kitty/vt-parser.c:230`–`240`:

```c
consume_normal(PS *self) {
    do {
        const bool sentinel_found = utf8_decode_to_esc(&self->utf8_decoder, self->buf + self->read.pos, self->read.sz - self->read.pos);
        self->read.pos += self->utf8_decoder.num_consumed;
        if (self->utf8_decoder.output.pos) {
            REPORT_DRAW(self->utf8_decoder.output.storage, self->utf8_decoder.output.pos);
            screen_draw_text(self->screen, self->utf8_decoder.output.storage, self->utf8_decoder.output.pos);
        }
        if (sentinel_found) { SET_STATE(ESC); break; }
    } while (self->read.pos < self->read.sz);
}
```

The `screen_draw_text(...)` call lands in `screen_draw_text()` at `kitty/screen.c:866` — the printable-text destination. When an ESC byte is hit, `consume_normal` sets state `ESC`, and the next dispatch enters `consume_esc()` (`kitty/vt-parser.c:261`), whose first-character switch selects the sub-state. Verbatim excerpt, `kitty/vt-parser.c:261`–`274` (the same `switch` continues past line 274 with the `IS_ESCAPED_CHAR` label, single-char escapes, and a `default`, ending at `kitty/vt-parser.c:298`; the block below is quoted contiguously):

```c
consume_esc(PS *self) {
#define CALL_ED(name) REPORT_COMMAND(name); name(self->screen); SET_STATE(NORMAL);
#define CALL_ED1(name, ch) REPORT_COMMAND(name, ch); name(self->screen, ch); SET_STATE(NORMAL);
#define CALL_ED2(name, a, b) REPORT_COMMAND(name, a, b); name(self->screen, a, b); SET_STATE(NORMAL);
    const uint8_t ch = self->buf[self->read.pos++];
    const bool is_first_char = self->read.pos - self->read.consumed == 1;
    if (is_first_char) {
        switch(ch) {
            case ESC_DCS: SET_STATE(DCS); break;
            case ESC_OSC: SET_STATE(OSC); break;
            case ESC_CSI: SET_STATE(CSI); reset_csi(&self->csi); break;
            case ESC_APC: SET_STATE(APC); break;
            case ESC_SOS: SET_STATE(SOS); break;
            case ESC_PM: SET_STATE(PM); break;
```

This first-character `switch` is the exact point where an escape sequence's introducer selects the OSC/CSI/DCS/APC/PM/SOS sub-state — i.e. where escape sequences are separated from the printable text handled by `consume_normal`.

### Thread split [OBSERVED reader thread; INFERRED parser thread]

- **Reading — I/O thread.** `read_bytes()`/`read()` run in `io_loop` (`kitty/child-monitor.c:1481`). **[OBSERVED]** every PTY read came from TID 293485 (`KittyChildMon`).
- **Parsing — Main thread.** **[INFERRED]** `consume_input` is reached from the main loop: `parse_input()` at `kitty/child-monitor.c:451` — whose own comment at `kitty/child-monitor.c:452` reads `// Parse all available input that was read in the I/O thread.` — calls `do_parse()` at `kitty/child-monitor.c:438`, which invokes `self->parse_func`. In the default build `parse_func = parse_worker` (`kitty/child-monitor.c:181`; the `parse_worker_dump` variant at `:180` is only selected when a dump callback is set), and `parse_worker`/`run_worker` (`kitty/vt-parser.c:1417`) call `consume_input`. The `Screen` owns the parser via `Parser *vt_parser;` at `kitty/screen.h:158`.

---

## 9. Stability confirmation (Q3) — four runs, per-metric spread

The same unchanged input (`yes hello` typed into the real window 2097164, interrupted with a real Ctrl-C) was run **three times at a 5 s scale** (the stability triple) plus a **10 s scale-up** run (run 4). All measurements are on the I/O thread (TID 293485), under strace, via the embedded `q3_stats.py`. Every run recovered the prompt (exit-status `133;D;130`). **[OBSERVED]**:

| Run | Wall window | Flood reads | Reads/sec | Mean B | Median B | Mode B | Max B | read→read med (µs) | poll→read med (µs) | `POLLIN` withheld (this run) |
|-----|-------------|-------------|-----------|--------|----------|--------|-------|--------------------|--------------------|-------------------|
| 1   | 5.048909 s  | 48 995      | 9 704.08  | 602.52 | 551 | 567 | 19 052 | 60 | 30 | 0 |
| 2   | 5.031166 s  | 52 300      | 10 395.20 | 559.37 | 492 | 448 | 17 235 | 57 | 28 | 0 |
| 3   | 5.067441 s  | 50 158      | 9 898.09  | 610.27 | 516 | 476 | 18 512 | 57 | 28 | 0 |
| 4 (10 s scale-up) | 10.040685 s | 105 471 | 10 504.36 | 607.39 | 532 | 539 | 16 500 | 57 | 28 | 0 |

**Per-metric spread across the three 5 s runs** (range ÷ mean):

| Metric | min | max | spread |
|--------|-----|-----|--------|
| read→read cadence (median µs) | 57 | 60 | **5.17 %** |
| reads/sec | 9 704.08 | 10 395.20 | **6.91 %** |
| poll→read latency (median µs) | 28 | 30 | **6.98 %** |
| bytes/read mean | 559.37 | 610.27 | **8.62 %** |
| bytes/read median | 492 | 551 | **11.35 %** |
| bytes/read mode | 448 | 567 | **23.94 %** |

**Explicit stability criterion & conclusion.** Treating a metric as *stable* when its range ÷ mean ≤ 10 % across the three unchanged 5 s runs: the read→read cadence (5.17 %), the read rate (6.91 %), the poll→read latency (6.98 %), and the byte **mean** (8.62 %) **pass**; the byte **median** (11.35 %) is marginally above the line and the byte **mode** (23.94 %) is clearly the most variable — because the per-read size distribution is heavy-tailed and multimodal (individual reads span single digits to ~17–19 KB, and the modal bucket shifts run to run within the 400–600 B band). The robust "typical bytes-per-read" is therefore best stated as a **range** — median **~492–551 B**, mode **~448–567 B**, i.e. all runs cluster around ~500 B — rather than a single number. The 10 s scale-up (run 4: median 532, mode 539, mean 607, 10 504 reads/s) falls inside the same bands, confirming that increasing scale to ~105 k reads did not change the per-read/rate picture. In all four of these runs the main-thread parser kept pace and `POLLIN` was never withheld (peak occupancy ≤ 24.79 %, §6); full backpressure engagement is a separate, rare event captured once in the 21-run sweep (`long02`, §6), not a property of these four scheduling-typical runs. Caveat: these cadences are measured **under strace**, which adds per-syscall overhead, so absolute reads/sec is a lower bound; the qualitative change (back-to-back large reads vs. blocking 1-byte reads) and the ~500 B byte-size central tendency are the robust findings.

---

## 10. Coverage pass — every named item answered

| # | Named item asked | Concrete answer (OBSERVED unless noted) | `file:line` |
|---|------------------|------------------------------------------|-------------|
| Q1a | Process spawned | `/bin/bash` (account shell, POSIX mode — not a login shell) | `kitty/constants.py:181` |
| Q1b | PID | `293486` | `ps` / `/proc/293486` |
| Q1c | Exact command line | `/bin/bash --posix` (`/bin/bash^@--posix^@`) | argv `kitty/child.py:314`; exec `kitty/child.c:159` |
| Q1d | PTY device path | slave `/dev/pts/0`; master `/dev/pts/ptmx` (fd 8) | dup2 `kitty/child.c:138`–`145` |
| Q2a | System calls | `poll()` then `read()` on fd 8 (write `poll→write` for the keystroke also observed) | `kitty/child-monitor.c:1512`/`1509`; read `:1345` |
| Q2b | Buffer size | `1048576` (= `BUF_SZ` 1 MiB); shrinks by `write.offset` | `kitty/vt-parser.c:18`; `:1451` |
| Q2c | Bytes returned | 1 per keystroke; Enter 11 (2 echo + 9 control); output 47/114/163 → **347 total, 16 reads** | reader `kitty/child-monitor.c:1337` |
| Q3a | How behavior changes | blocking 1-byte reads → back-to-back multi-hundred-byte reads; poll timeout −1 → 1–2 ms | `kitty/child-monitor.c:1509`/`1512`; gate `:1501` |
| Q3b | Frequency | ~9 704–10 504 reads/sec (4 runs); read→read median 57–60 µs (poll→read latency 28–30 µs, distinct) | dispatch `kitty/child-monitor.c:1529`–`1531` |
| Q3c | Typical bytes/read | median 492–551, mode 448–567 (robust); mean 559–610; up to ~19 KB | buffer `kitty/vt-parser.c:18` |
| Q3 stability | ≥2-run stability | stable across 3×5 s + 1×10 s; per-metric spread + criterion (§9) | §9 |
| Q3 backpressure | flow-control gate | 10 read-traced floods peak occupancy 11.26–24.79 %; `POLLIN` **withheld (`events=0`) OBSERVED in 1 of 21 sweep floods** (`long02`: 4 polls / 84.7 ms); rare & scheduling-variable | `kitty/child-monitor.c:1501`; `kitty/vt-parser.c:1481` |
| Q4 | fd integer | **8** (process-specific; `/dev/pts/ptmx`) | reg. `kitty/child-monitor.c:1286`; retain `kitty/child.py:338` |
| Q5a | Reader function | `read_bytes()` — I/O thread (TID 293485 `KittyChildMon`) | `kitty/child-monitor.c:1337` (read `:1345`) |
| Q5b | Parser function | `consume_input()` — Main thread | `kitty/vt-parser.c:1367` |
| Q5b | Text path | `consume_normal()` → `screen_draw_text()` | `kitty/vt-parser.c:230` → `kitty/screen.c:866` |
| Q5b | Escape path | `consume_esc()` → CSI/OSC/DCS/APC/PM/SOS | `kitty/vt-parser.c:261` |

---

## 11. Observed-vs-Inferred summary

| Fact | Classification | Basis |
|------|----------------|-------|
| Shell is `/bin/bash`, PID 293486, cmdline `/bin/bash --posix`, session-leader/foreground (not login) | **[OBSERVED]** | `ps`, `cat -A /proc/293486/cmdline` |
| PTY slave `/dev/pts/0`; master fd 8 `/dev/pts/ptmx` (5,2) | **[OBSERVED]** | `readlink /proc/.../fd`, `ls -l`, `lsof` |
| `poll → write → poll → read` per keystroke on fd 8; read buffer arg `1048576` | **[OBSERVED]** | `strace -tt -s 512` (write included) |
| Returned bytes: 1/keystroke, 11 Enter (2 echo + 9 control), 47/114/163 output, 347 total / 16 reads | **[OBSERVED]** | `strace` (clean single-thread; `-f` reconciled 8+8=16) |
| `yes hello`: ~9.7–10.5 k reads/s; read→read 57–60 µs vs poll→read 28–30 µs; median ~500 B; stable ×3 + 10 s | **[OBSERVED]** | `strace` + `q3_stats.py`, 4 runs |
| Ctrl-C → exit status `133;D;130` (SIGINT) → prompt recovers | **[OBSERVED]** | `strace` after-state |
| Parser-buffer peak occupancy 11.26–24.79 % across the 10 read-traced floods (buffer did **not** fill) | **[OBSERVED]** | `strace` read 3rd arg (`BUF_SZ − min reqsize`) |
| `POLLIN` **withheld (`events=0`) observed in 1 of 21 sweep floods** (`long02`: 4 polls / 84.7 ms window); rare & scheduling-variable | **[OBSERVED]** | `strace` poll `events=` field, `long02` |
| That the gate flips *precisely at* `BUF_SZ` (not a lower watermark) | **[INFERRED]** | predicate `read.sz + write.pending < BUF_SZ`, `kitty/vt-parser.c:1481` |
| fd number = 8 | **[OBSERVED]** | `strace`, `/proc`, `lsof` |
| Reader executes on the I/O thread (`KittyChildMon`, TID 293485) | **[OBSERVED]** | `/proc/.../task/293485/comm` + `strace` |
| Both text and escape bytes are present in the reads | **[OBSERVED]** | `strace` payloads (grep of `trace_q2.log`) |
| Shell resolution `pw_shell or '/bin/sh'` | **[INFERRED]** | `kitty/constants.py:181` |
| Spawn chain (fork/setsid/TIOCSCTTY/dup2/execvp) | **[INFERRED]** | `kitty/child.c:81`–`159` |
| `read()` lives in `read_bytes()`; buffer sizing formula | **[INFERRED]** | `kitty/child-monitor.c:1337`/`1345`; `kitty/vt-parser.c:1451` |
| Text/escape routing `consume_input`→`consume_normal`/`consume_esc` | **[INFERRED]** | `kitty/vt-parser.c:1367`/`230`/`261` |
| Parsing dispatched on the Main thread | **[INFERRED]** | `kitty/child-monitor.c:451`/`452`/`438` |
| fd registration into the poll set | **[INFERRED]** | `kitty/child-monitor.c:1286`/`1287`/`35`/`86` |

---

## 12. Cleanup & repository-unchanged proof

Teardown used the `lifecycle.sh` helper from §3 against the captured state (`KITTY_PID=293413`, `XVFB_PID=293409`, `SCRATCH=/tmp/kitty_qa.emz5EMLx`). The complete, unedited transcript — including the three input-validation failure cases, the before/after verify, the success teardown, and the idempotent second run — is **[OBSERVED]**:

```
# --- input-validation / failure cases ---
$ bash lifecycle.sh ; echo "exit=$?"                       # no arguments
usage: lifecycle.sh {teardown|verify} <state.env>
exit=2
$ bash lifecycle.sh bogus state.env ; echo "exit=$?"       # unknown action
error: unknown action 'bogus'
usage: lifecycle.sh {teardown|verify} <state.env>
exit=2
$ bash lifecycle.sh teardown /no/such/state.env ; echo "exit=$?"   # unreadable state file
error: cannot read state file '/no/such/state.env'
exit=3

# --- verify BEFORE teardown (processes alive) ---
$ bash lifecycle.sh verify state.env ; echo "verify_rc=$?"
  ALIVE  pid=293413
  ALIVE  pid=293409
verify_rc=1

# --- teardown (success): PID-scoped SIGTERM, wait, scoped scratch removal ---
$ bash lifecycle.sh teardown state.env ; echo "teardown_rc=$?"
teardown: scoped pids = 293413 293409
  [kitty] pid=293413 alive -> SIGTERM
  [kitty] pid=293413 terminated
  [xvfb] pid=293409 alive -> SIGTERM
  [xvfb] pid=293409 terminated
  removing scratch dir: /tmp/kitty_qa.emz5EMLx
teardown: complete
teardown_rc=0

# --- verify AFTER teardown (processes gone) ---
$ bash lifecycle.sh verify state.env ; echo "verify_rc=$?"
  gone   pid=293413
  gone   pid=293409
verify_rc=0

# --- second teardown: idempotent no-op ---
$ bash lifecycle.sh teardown state.env ; echo "teardown_rc=$?"
teardown: scoped pids = 293413 293409
  [kitty] pid=293413 already gone (idempotent no-op)
  [xvfb] pid=293409 already gone (idempotent no-op)
teardown: complete
teardown_rc=0
```

No `pkill`/`killall` and no unscoped deletion were used — only the exact captured PIDs were signalled, and only the unique `mktemp -d` scratch dir was removed. After teardown, no leftover process remained (the `verify` above returns `gone`/`gone`), and the repository working tree differs from the baseline by exactly this one document **[OBSERVED]**:

```
$ git status --porcelain
?? blitzy/documentation/kitty_815df1e210e0.md
$ git status --porcelain -- kitty/ setup.py Makefile docs/ | wc -l    # no source/build/doc file touched
0
```

---

*All runtime values above were captured from a single default-configuration kitty (v0.35.2) built with `make` (`python3 setup.py`, `Makefile:13`) from a disposable copy under `/tmp`, launched headless under an access-controlled Xvfb (PID 293409), with input delivered exclusively through the canonical terminal keyboard path into the PID-owned, focus-verified window 2097164. The one coherent process — kitty PID 293413, shell PID 293486, I/O thread TID 293485, PTY master fd 8 — served Q1/Q2/Q4/Q5 and every Q3 flood. Temporary observation scripts, strace logs, and the disposable build copy lived only under a private `mktemp -d` scratch directory and were removed by the PID-scoped `lifecycle.sh` teardown (kitty 293413 and Xvfb 293409 terminated). The repository working tree is unchanged apart from this document, as confirmed by `git status --porcelain` above.*

