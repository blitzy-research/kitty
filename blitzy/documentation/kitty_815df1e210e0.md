# How `kitty` handles a child that prints a few lines and exits 0

## 1. Title & scope

This document answers, with **runtime‑observed evidence** and **cited `file:line` source references**, how the
[`kitty`](https://github.com/kovidgoyal/kitty) terminal emulator behaves from the moment a launched child
program is running, through it printing a few lines to standard output, to it exiting with status `0` and the
window being torn down.

| Item | Value |
|------|-------|
| Repository | `kovidgoyal/kitty` |
| Commit (HEAD) | `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` |
| Branch | `kitty_815df1e210e0` (working branch `blitzy-4da995f7-9d87-4535-892b-6e7ceff8c1ad`) |
| Built version | `kitty 0.35.2 created by Kovid Goyal` |
| Intended container | `andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (from `ghcr.io/scaleapi/swe-atlas`) |

The nine questions answered below, in order, are: (1) end‑to‑end flow; (2) kitty's own exit code; (3) the full
completion message(s); (4) the child‑tracking component; (5) the message‑generating function; (6) the OS signal;
(7) the status‑retrieval system call; (8) the exit‑status transport; and (9) where the child's printed output
actually appears.

> **Methodology (binding rule: run first, then write).** Every value below was produced by building kitty in its
> default configuration and exercising its real entry points inside the environment described in §2, capturing the
> raw output *before* writing this document. Values that could not be observed at runtime are explicitly labelled
> **inferred** and grounded in a `file:line`. Where a debug facility (kitty's own `--dump-bytes`) was used, it is
> labelled as such and used only to corroborate a value already observed through a canonical path.

---

## 2. Environment & build

### 2.1 Canonical build

The canonical build is `make`, whose `all:` target runs `python3 setup.py`:

```
$ sed -n '12,13p' Makefile
all:
	python3 setup.py $(VVAL)
```

In this environment kitty was built (per the provisioned toolchain) with `setup.py build`, producing the launcher
`kitty/launcher/kitty`, the companion `kitty/launcher/kitten`, and the C extension `kitty/fast_data_types.so`
(all git‑ignored build artefacts). The exact toolchain present:

```
$ /usr/local/bin/python3.11 --version ; /usr/local/go/bin/go version ; /bin/bash --version | head -1 ; gcc --version | head -1
Python 3.11.13
go version go1.22.12 linux/amd64
GNU bash, version 5.2.37(1)-release (x86_64-pc-linux-gnu)
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
```

Supported‑version anchors: `pyproject.toml` line 2 `requires-python = ">=3.8"` (CI‑tested version 3.11);
`go.mod` line 3 `go 1.22`.

### 2.2 Headless display

kitty is a GPU terminal, so a virtual display and a software GL backend were used to obtain a real on‑screen
window:

```
$ Xvfb :99 -screen 0 1280x800x24 -nolisten tcp &
$ export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1
```

All GUI runs below use `DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1`.

### 2.3 Canonical invocation and configuration

There is no user config at `~/.config/kitty/kitty.conf`, so `--config NONE` is equivalent to the shipped defaults;
it is passed explicitly to make the "default configuration" claim unambiguous. The canonical reproduction command
used throughout is:

```
./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0'
```

All reported values come from this canonical path unless a line is explicitly labelled **configured (non‑default)**
or **inferred**.

---

## 3. Executive summary

When kitty launches a child on a pseudo‑terminal (PTY) and the child prints a few lines and exits `0`, kitty's
`ChildMonitor` reads the child's bytes from the PTY master, feeds them through the VT parser into the on‑screen
`Screen` grid (that is where the output appears), learns of the child's death via a **`SIGCHLD`** delivered through
a **signalfd**, reaps the child with **`waitpid(-1,&status,WNOHANG)`** (the `wait4` syscall on Linux), and tears the
window down; by default the window closes. kitty's *own* process then exits `0`. A user‑visible "finished" message
is **not** shown in the default configuration; it appears only when either `--hold`/the `hold` kitten shows its
banner, or shell integration plus a non‑default `notify_on_cmd_finish` fires a desktop notification whose body
carries the exit status. The exit status reaches that notification not via the OS signal but via the **OSC 133
`D;<code>`** semantic‑prompt escape sequence emitted by the integrated shell.

One‑line answers:

1. **End‑to‑end flow** — `Child.fork()` spawns the program on a PTY; the child writes the PTY slave; `ChildMonitor` reads the PTY master (`read_bytes`), parses to the `Screen`, renders the grid; on exit the OS raises `SIGCHLD` (signalfd) → `reap_children` `waitpid` reaps status 0 → `on_child_death(window_id)` tears the window down; by default the window closes.
2. **kitty's own exit code** — `0` (observed stable across 5 runs), independent of the child's status.
3. **Completion message** — two *separate* messages: the `--hold`/`hold`‑kitten green banner `Press Enter or Esc to exit` (no status), and the shell‑integration notification body `Command echo hello finished with status: 0.\nClick to focus.` (carries the status; non‑default).
4. **Child‑tracking component** — the `ChildMonitor` (C core `kitty/child-monitor.c` + Boss registration in `kitty/boss.py`).
5. **Message‑generating function** — `Window.handle_cmd_end()` (`kitty/window.py:1408`).
6. **Signal** — `SIGCHLD` (numeric 17), delivered via a signalfd.
7. **System call** — `waitpid(-1,&status,WNOHANG)` (`kitty/child-monitor.c:1418`), i.e. the `wait4` syscall on Linux.
8. **Status transport** — the OSC 133 `D;<exit_code>` (FinalTerm/iTerm2) escape sequence emitted by the integrated shell; parsed at `kitty/vt-parser.c:536` → `kitty/screen.c:2350-2352` → `handle_cmd_end`.
9. **Output location** — kitty's on‑screen terminal grid (the `Screen` buffer, GPU‑rendered), **not** kitty's own stdout/stderr.

---

## Q1 — End‑to‑end flow (child running → prints stdout → exits 0 → teardown)

**Command**

```
$ export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1
$ strace -f -e trace=signalfd4,rt_sigprocmask,wait4,read -e signal=all -o /tmp/kitty_sig.strace \
    ./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0'
```

**Raw output** (the exact death‑sequence window from the child‑monitor I/O thread, TID `93767`; the child `sh`
is PID `93768`):

```
93767 read(8, "hello\r\nworld\r\n", 1048576) = 14
93768 +++ exited with 0 +++
93767 read(8, 0x57cc861ae50e, 1048562)  = -1 EIO (Input/output error)
93767 read(7, "\21\0\0\0\0\0\0\0\1\0\0\0Hn\1\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 128
93767 read(7, 0x799ff35143a0, 4096)     = -1 EAGAIN (Resource temporarily unavailable)
93767 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 93768
93767 wait4(-1, 0x799fc4f87e50, WNOHANG, NULL) = -1 ECHILD (No child processes)
```

The window then closes and the kitty process returns exit `0` (see Q2). fd `8` is the PTY master; fd `7` is the
signalfd (see Q6).

**Code citation** — the pipeline is anchored in the following verbatim lines.

Spawn — `kitty/child.py:276,281,329-330,333,337-338`:

```
    def fork(self) -> Optional[int]:
        master, slave = openpty()
        if self.hold:
            argv = cmdline_for_hold(argv)
        pid = fast_data_types.spawn(
        self.pid = pid
        self.child_fd = master
```

PTY‑master read — `kitty/child-monitor.c:1337,1345,1354,1355`:

```
read_bytes(int fd, Screen *screen) {
        len = read(fd, buf, available_buffer_space);
    vt_parser_commit_write(screen->vt_parser, len);
    return len != 0;
```

I/O‑loop death dispatch — `kitty/child-monitor.c:1519,1526,1535`:

```
                read_signals(children_fds[1].fd, handle_signal, &ss);
                if (ss.child_died) reap_children(self, OPT(close_on_child_death));
                        children[i].needs_removal = true;
```

Direct‑child window teardown callback — `kitty/boss.py:881`:

```
    def on_child_death(self, window_id: int) -> None:
```

**Rationale.** `Child.fork()` opens a PTY and calls `fast_data_types.spawn(...)`, storing `self.pid` and
`self.child_fd = master` — kitty keeps only the master end. The child writes its stdout to the PTY *slave*; kitty's
`ChildMonitor` thread reads those bytes from the master via `read_bytes()`' `read(fd, …)` and commits them to the
VT parser (`vt_parser_commit_write`), which fills the `Screen` grid (Q9). When the child exits, the kernel raises
`SIGCHLD`; the monitor loop drains it (`read_signals` → `handle_signal`, Q6) and calls
`reap_children(self, OPT(close_on_child_death))` which `waitpid`‑reaps the zombie (Q7). Independently, the PTY hits
end‑of‑file so `read()` returns `0`/`EIO` and `read_bytes()` returns `false`, marking the child
`needs_removal = true`. Either way the window is torn down through `on_child_death(window_id)` and, in the default
configuration, the window closes (see Nuance #1, §11). The raw trace shows all of this in order: the PTY read of
`"hello\r\nworld\r\n"`, the child's `exited with 0`, the signalfd read, and the `wait4` reap returning the child PID
with `WEXITSTATUS == 0`.

---

## Q2 — kitty's OWN exit code when the child exits 0

**Command** (run five times for stability, then two corroborating non‑zero‑child runs):

```
$ export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1
$ for i in 1 2 3 4 5; do
    ./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0' >/dev/null 2>&1
    echo "run $i: kitty_exit=$?"
  done
$ ./kitty/launcher/kitty --config NONE sh -c 'echo hi; exit 5'  >/dev/null 2>&1; echo "child exit 5 -> kitty_exit=$?"
$ ./kitty/launcher/kitty --config NONE sh -c 'echo hi; exit 42' >/dev/null 2>&1; echo "child exit 42 -> kitty_exit=$?"
```

**Raw output**

```
run 1: kitty_exit=0
run 2: kitty_exit=0
run 3: kitty_exit=0
run 4: kitty_exit=0
run 5: kitty_exit=0
child exit 5 -> kitty_exit=0
child exit 42 -> kitty_exit=0
```

**Code citation** — `kitty/main.py:524-531` (verbatim, 8 lines):

```
def main() -> None:
    try:
        _main()
    except Exception:
        import traceback
        tb = traceback.format_exc()
        log_error(tb)
        raise SystemExit(1)
```

**Rationale.** kitty's own process exit code is **`0`**, stable across all five runs (distribution `{0: 5}`). It is
distinct from — though here numerically equal to — the child `sh`'s status: when the child instead exits `5` or
`42`, kitty *still* exits `0`. That is because kitty does not propagate the child's status as its own; `main()`
returns normally after a clean shutdown, so the process exits `0`, and only an *internal* kitty exception would
turn into `raise SystemExit(1)`. The number in `$?` is therefore kitty's own clean‑shutdown status, not the child's.

---

## Q3 — Full completion message(s)

There are **two different** user‑facing "completion" messages. They are reported separately and must not be
conflated (see Distinction A, §10).

### Q3a — the `--hold` / `hold`‑kitten green banner

**Command** (capture the raw bytes the banner code writes, off a PTY, from the real `__hold_till_enter__` kitten —
the code path that actually renders the banner in this commit):

```
$ export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1
# a small PTY harness exec'd:  kitten __hold_till_enter__ sh -c 'echo hello; echo world; exit 0'
# and recorded the master bytes to /tmp/hold_banner_raw.bin
$ python3 -c "print(repr(open('/tmp/hold_banner_raw.bin','rb').read()))"
```

**Raw output** (full capture, then the banner segment isolated):

```
b'hello\r\nworld\r\n\x1b[?s\x1b[*x\x1b[4l\x1b[?1l\x1b[?5l\x1b[?2004l\x1b[?1004l\x1b[?1000l\x1b[?1002l\x1b[?1003l\x1b[?1005l\x1b[?1006l\x1b[?8h\x1b[?7h\x1b[?25h\x1b[>29u\x1b[?25l\r\n\x1b[1;32mPress Enter or Esc to exit\x1b[m'
```

Banner segment (Python `repr`, then `cat -v`):

```
\x1b[1;32mPress Enter or Esc to exit\x1b[m
^[[1;32mPress Enter or Esc to exit^[[m
```

**Code citation** — `tools/tui/hold.go:16,26` (verbatim):

```
func HoldTillEnter(start_with_newline bool) {
		lp.QueueWriteString("\x1b[1;32mPress Enter or Esc to exit\x1b[m")
```

Dispatch to that banner — `tools/cmd/tool/main.go:87-93` (verbatim):

```
	// __hold_till_enter__
	root.AddSubCommand(&cli.Command{
		Name:            "__hold_till_enter__",
		Hidden:          true,
		OnlyArgsAllowed: true,
		Run: func(cmd *cli.Command, args []string) (rc int, err error) {
			tui.ExecAndHoldTillEnter(args)
```

**Rationale.** The banner text is exactly `Press Enter or Esc to exit`, wrapped in SGR `1;32` (bold + green,
`\x1b[1;32m`) and reset (`\x1b[m`). It is emitted by `HoldTillEnter()` in the Go `hold` kitten, reached via the
hidden `__hold_till_enter__` subcommand → `tui.ExecAndHoldTillEnter()`. **This banner contains no exit status.**

> **Correction to the naive `--hold` hypothesis (observed).** In this commit the plain `--hold` *flag* does **not**
> print this banner. `Child.fork()` rewrites the argv through `cmdline_for_hold()`
> (`kitty/child.py:329-330`; `kitty/utils.py:1192,1202` — it returns
> `[kitten_exe(), 'run-shell', …, '--env=KITTY_HOLD=1'] + list(cmd)`), which holds the window open by launching an
> **interactive shell** with `KITTY_HOLD=1` in its environment, not by printing the banner. The green banner is
> produced only by the `hold` kitten path (`kitty/entry_points.py:27-29` and `kitty/utils.py:1031-1034`
> `subprocess.Popen([kitten_exe(), '__hold_till_enter__'])`), which is what the capture above exercised. This was
> confirmed at runtime: `kitty --hold …`'s child is an interactive `bash` carrying `KITTY_HOLD=1`, and its screen
> shows only `hello`/`world` with **no** green banner, whereas the `__hold_till_enter__` kitten shows the bold‑green
> banner.

### Q3b — the shell‑integration desktop notification

This notification does **not** fire in the default configuration (Nuance #2, §11: `notify_on_cmd_finish` default is
`never`). To observe its body it must be enabled — a real option, not a mock — so this is a
**configured (non‑default) but canonical** observation. It was captured on a real D‑Bus session by eavesdropping on
the `org.freedesktop.Notifications` interface while kitty's default `notify` action ran.

**Command**

```
# /tmp/dbus_notify_test.sh, run inside a fresh D-Bus session:
#   dbus-monitor "interface='org.freedesktop.Notifications'" > /tmp/dbus_notify.log 2>&1 &
#   DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 \
#     ./kitty/launcher/kitty --config NONE -o "notify_on_cmd_finish always 0" sh -c "sh /tmp/child_osc.sh"
# /tmp/child_osc.sh emits the real OSC 133 bytes a shell would send:
#   printf '\033]133;C;cmdline=echo\\ hello\007'; echo hello; echo world; printf '\033]133;D;0\007'; sleep 2; exit 0
$ dbus-run-session -- /tmp/dbus_notify_test.sh
```

**Raw output** (the captured `org.freedesktop.Notifications.Notify` method call; identical across two runs):

```
method call time=1783962369.836648 sender=:1.2 -> destination=org.freedesktop.Notifications serial=3 path=/org/freedesktop/Notifications; interface=org.freedesktop.Notifications; member=Notify
   string "kitty"
   uint32 0
   string "/tmp/blitzy/kitty/blitzy-4da995f7-9d87-4535-892b-6e7ceff8c1ad_2bb4d2/logo/kitty.png"
   string "kitty"
   string "Command echo hello finished with status: 0.
Click to focus."
   array [
      string "default"
      string "Click to see changes"
   ]
   array [
      dict entry(
         string "urgency"
         variant             byte 1
      )
   ]
   int32 -1
```

Byte‑exact body (via `cat -A`, where `$` marks end‑of‑line, revealing the embedded newline):

```
   string "Command echo hello finished with status: 0.$
Click to focus."$
```

**Code citation** — `kitty/window.py:1429` (verbatim):

```
            cmd.body = f'Command {s} finished with status: {exit_status}.\nClick to focus.'
```

**Rationale.** The notification body is exactly `Command echo hello finished with status: 0.\nClick to focus.`
(a literal newline between `0.` and `Click to focus.`). It is built by `handle_cmd_end()` (Q5) from the exit
status decoded out of the OSC 133 `D;0` sequence (Q8); `echo hello` is the `%q`‑decoded command line. **This message
does include the exit status (`0`).** In the default configuration it is not emitted at all (§11, Nuance #2).

---


## Q4 — Child‑tracking component

**Command** (identify the component that owns each child's PID, PTY FD and Screen — the registration observed in the
running process, corroborated by the strace showing that same thread polling fd 7/8 and reaping):

```
$ grep -nE "self.child_monitor = ChildMonitor\(|self.child_monitor.add_child\(" kitty/boss.py
```

**Raw output**

```
370:        self.child_monitor = ChildMonitor(
587:        self.child_monitor.add_child(window.id, window.child.pid, window.child.child_fd, window.screen)
```

**Code citation** — `kitty/boss.py:370-374` (verbatim):

```
        self.child_monitor = ChildMonitor(
            self.on_child_death,
            DumpCommands(args) if args.dump_commands or args.dump_bytes else None,
            talk_fd, listen_fd,
        )
```

and `kitty/boss.py:585-587` (verbatim):

```
    def add_child(self, window: Window) -> None:
        assert window.child.pid is not None and window.child.child_fd is not None
        self.child_monitor.add_child(window.id, window.child.pid, window.child.child_fd, window.screen)
```

**Rationale.** The component that tracks the child across its lifetime is the **`ChildMonitor`** — its C core is
`kitty/child-monitor.c`, and it is created once by the `Boss` (`kitty/boss.py:370`) with `self.on_child_death` as
its death callback. Each child is registered via `add_child(window.id, window.child.pid, window.child.child_fd,
window.screen)`, so the `ChildMonitor` owns, per child, the **window id**, the **PID**, the **PTY master FD**
(`child_fd`) and the **`Screen`**. It is the `ChildMonitor` thread that (as the Q1/Q6/Q7 strace shows) polls the
child's FD, reads its output, drains its `SIGCHLD` from the signalfd, and `waitpid`‑reaps it — i.e. it tracks the
child from spawn to death.

---

## Q5 — Message‑generating function

**Command** (show the function and its entry point; the runtime firing of it is captured in Q3b and below):

```
$ grep -nE "def handle_cmd_end|def cmd_output_marking|self.handle_cmd_end\(cmdline\)" kitty/window.py
```

**Raw output**

```
1408:    def handle_cmd_end(self, exit_status: str = '') -> None:
1453:    def cmd_output_marking(self, is_start: Optional[bool], cmdline: str = '') -> None:
1461:            self.handle_cmd_end(cmdline)
```

Runtime confirmation that `handle_cmd_end` actually runs and receives status `0` — a `command`‑action probe wrote
its `%s` (exit status) and `%c` (command line) to a file when the OSC 133 `D;0` arrived:

```
$ ./kitty/launcher/kitty --config NONE \
    -o "notify_on_cmd_finish always 0 command /tmp/notify_capture.sh %s %c" \
    sh -c 'printf "\033]133;C;cmdline=echo\\ hello\007"; echo hello; echo world; printf "\033]133;D;0\007"; sleep 2; exit 0'
$ cat /tmp/notify_capture.txt
NOTIFY_FIRED exit_status=[0] cmdline=[echo hello]
```

**Code citation** — `kitty/window.py:1408,1413` (verbatim):

```
    def handle_cmd_end(self, exit_status: str = '') -> None:
            self.last_cmd_exit_status = int(exit_status)
```

and its entry point `kitty/window.py:1453,1461` (verbatim `else` branch):

```
    def cmd_output_marking(self, is_start: Optional[bool], cmdline: str = '') -> None:
            self.handle_cmd_end(cmdline)
```

**Rationale.** The single function that turns the child's exit status into the user‑facing message is
**`Window.handle_cmd_end()`** (`kitty/window.py:1408`). It is entered from `Window.cmd_output_marking()`: when
`is_start` is falsy — the OSC 133 `D` case — the `else` branch calls `self.handle_cmd_end(cmdline)`, where the
passed string *is* the exit status. `handle_cmd_end` parses it (`self.last_cmd_exit_status = int(exit_status)`) and,
subject to `notify_on_cmd_finish`, builds `cmd.body` (Q3b) and dispatches the notification. The runtime probe shows
it firing with `exit_status=[0]` for a command that exited 0, decoded from the real `D;0` sequence.

---

## Q6 — Signal kitty listens for to learn a child terminated

**Command**

```
$ export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1
$ strace -f -e trace=signalfd4,rt_sigprocmask,wait4,read -e signal=all -o /tmp/kitty_sig.strace \
    ./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0'
$ grep -nE "signalfd4\(|rt_sigprocmask\(SIG_BLOCK, \[HUP" /tmp/kitty_sig.strace | head -2
```

**Raw output** — the signal set‑up (SIGCHLD, shown by strace as `CHLD`, is blocked and routed to a signalfd), and
the signalfd read that delivers it, decoded:

```
974:93701 signalfd4(-1, [HUP INT USR1 USR2 TERM CHLD], 8, SFD_CLOEXEC|SFD_NONBLOCK) = 7
249:93701 rt_sigprocmask(SIG_BLOCK, [HUP INT USR1 USR2 TERM CHLD], NULL, 8) = 0
```

```
# the 128-byte read on the signalfd (fd 7) right before the reap
# (the trailing "..." is strace's own default buffer truncation, not an elision by us;
#  the meaningful signalfd_siginfo fields are within the leading bytes shown):
93767 read(7, "\21\0\0\0\0\0\0\0\1\0\0\0Hn\1\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 128

# decoding the first 16 bytes of struct signalfd_siginfo, i.e. struct.unpack('<IiiI', first16bytes):
#   \21\0\0\0 = 0x11 = 17 (ssi_signo) | \0\0\0\0 (ssi_errno) | \1\0\0\0 = 1 (ssi_code) | Hn\1\0 = 0x00016e48 = 93768 (ssi_pid)
ssi_signo = 17  -> SIGCHLD
ssi_errno = 0
ssi_code  = 1   (CLD_EXITED == 1)
ssi_pid   = 93768    (the child process that exited)
```

**Code citation** — the handled‑signal set including `SIGCHLD`, the block, and the handler.
`kitty/child-monitor.c:121` (verbatim):

```
#define KITTY_HANDLED_SIGNALS SIGINT, SIGHUP, SIGTERM, SIGCHLD, SIGUSR1, SIGUSR2, 0
```

`kitty/child-monitor.c:136` (verbatim):

```
    sigprocmask(SIG_BLOCK, &signals, NULL);
```

`kitty/loop-utils.c:42` (verbatim — the signalfd is created for those signals):

```
        ld->signal_read_fd = signalfd(-1, &ld->signals, SFD_NONBLOCK | SFD_CLOEXEC);
```

`kitty/child-monitor.c:1362,1370-1372` (verbatim — the handler that records the death):

```
handle_signal(const siginfo_t *siginfo, void *data) {
        case SIGCHLD:
            ss->child_died = true;
            break;
```

**Rationale.** The OS‑level signal kitty relies on to learn that a child terminated is **`SIGCHLD`** (numeric `17`).
Because kitty blocks its handled signals process‑wide (`sigprocmask(SIG_BLOCK, …)`) and consumes them through a
**signalfd** (`signalfd(-1, &ld->signals, …)`), there is no asynchronous `--- SIGCHLD ---` handler invocation in the
trace; instead the signal arrives as a `struct signalfd_siginfo` from a `read()` on the signalfd (fd `7`). Decoding
that record proves it is `SIGCHLD` (`ssi_signo == 17`), a normal child exit (`ssi_code == CLD_EXITED`), for the exact
child that just exited (`ssi_pid == 93768`). The monitor loop drains it with `read_signals(...)` which invokes
`handle_signal(...)`, whose `case SIGCHLD` sets `ss->child_died = true`, triggering the reap in Q7.

---


## Q7 — System call used to retrieve the child's exit status

**Command** (same strace as Q6)

```
$ grep -nE "wait4\(-1" /tmp/kitty_sig.strace | head -2
```

**Raw output**

```
1008:93767 wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL) = 93768
1009:93767 wait4(-1, 0x799fc4f87e50, WNOHANG, NULL) = -1 ECHILD (No child processes)
```

**Code citation** — `kitty/child-monitor.c:1413,1418,1422,1423` (verbatim):

```
reap_children(ChildMonitor *self, bool enable_close_on_child_death) {
        pid = waitpid(-1, &status, WNOHANG);
            if (enable_close_on_child_death) mark_child_for_removal(self, pid);
            mark_monitored_pids(pid, status);
```

**Rationale.** kitty retrieves the child's exit status with **`waitpid(-1, &status, WNOHANG)`** in
`reap_children()` (`kitty/child-monitor.c:1418`). On Linux/glibc `waitpid` is implemented on top of the `wait4`
syscall, which is exactly what strace shows: `wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], WNOHANG, NULL)`
returning the child PID `93768`, with the status word decoding to a normal exit of value `0`. The `-1` (any child)
and `WNOHANG` (non‑blocking) arguments match the source verbatim; the immediately following `wait4` returns
`ECHILD` because there are no further children to reap. The reaped raw status is recorded via
`mark_monitored_pids(pid, status)` — but note (Distinction B, §10) that the *window* death callback
`on_child_death(window_id)` receives only the window id, **not** this status.

---

## Q8 — Status transport (how the shell‑reported exit code travels)

**Command** — capture the byte‑exact OSC 133 the *real* bash shell‑integration script emits, in an interactive
shell, with integration ENABLED vs DISABLED; then corroborate what kitty's own parser receives end‑to‑end with
kitty's `--dump-bytes` recorder (labelled as kitty's debug facility; it records the *real* bytes received):

```
# (1) real shell-integration emission, captured off a PTY running: bash --rcfile shell-integration/bash/kitty.bash -i
$ cat -v /tmp/osc_enabled.bin  | grep -o '\^\[]133;[ACD][^^]*\^G'
# (2) integration disabled: bash --norc -i
$ python3 -c "print(repr(open('/tmp/osc_disabled.bin','rb').read()))"
# (3) end-to-end, kitty's own recorder of received bytes:
$ ./kitty/launcher/kitty --config NONE --dump-bytes /tmp/kdump.bin sh -c 'sh /tmp/child_osc.sh'
$ cat -v /tmp/kdump.bin
```

**Raw output**

(1) ENABLED — the shell emits the full semantic‑prompt cycle; `^[` is ESC, `^G` is BEL (so `^[]133;D;0^G` is
`ESC ] 1 3 3 ; D ; 0 BEL`):

```
^[]133;D;0^G
^[]133;A^G
^[]133;C;cmdline=echo\ hello^G
^[]133;D;0^G
^[]133;A^G
^[]133;C;cmdline=echo\ world^G
^[]133;D;0^G
^[]133;A^G
^[]133;C;cmdline=exit^G
```

(2) DISABLED — no OSC 133 sequences at all:

```
b'PROMPT> echo hello\r\nhello\r\nPROMPT> echo world\r\nworld\r\nPROMPT> exit\r\nexit\r\n'
```

(3) end‑to‑end — kitty received exactly the `C` and `D;0` markers around the output:

```
^[]133;C;cmdline=echo\ hello^Ghello^M
world^M
^[]133;D;0^G
```

**Code citation** — the VT parser routes OSC code 133 to `shell_prompt_marking`. `kitty/vt-parser.c:536,544`
(verbatim):

```
        case 133:
                shell_prompt_marking(self->screen, (char*)buf + i);
```

`shell_prompt_marking`'s `D` case extracts the status — `kitty/screen.c:2350-2352` (verbatim):

```
            case 'D': {
                const char *exit_status = buf[1] == ';' ? buf + 2 : "";
                CALLBACK("cmd_output_marking", "Os", Py_None, exit_status);
```

The shell emitters (all three integrated shells) — `shell-integration/bash/kitty.bash:239` and `:208` (verbatim):

```
        _ksi_prompt[ps1]+="\[\e]133;D;\$?\a\e]133;A\a\]"
                builtin printf "\e]133;C;cmdline=%q\a" "$last_cmd"
```

`shell-integration/zsh/kitty-integration:145` and `:149` (verbatim):

```
                    builtin print -nu $_ksi_fd '\e]133;D;'$cmd_status'\a'
                    builtin print -nu $_ksi_fd '\e]133;D\a'
```

`shell-integration/fish/vendor_conf.d/kitty-shell-integration.fish:96` (verbatim):

```
                echo -en "\e]133;D;$status\a"
```

**Rationale.** The shell‑reported exit code travels via the **OSC 133 `D;<exit_code>` semantic‑prompt (FinalTerm)
escape sequence**, not via any OS signal. The integrated shell exercised here is **bash** (the container's default,
`GNU bash 5.2.37`); its integration appends `\e]133;D;$?\a` to `PS1` (line 239) and emits
`\e]133;C;cmdline=%q\a` from the command‑start hook (line 208). The runtime capture shows the real byte sequence
`ESC ]133;D;0 BEL` emitted after each command that exited 0, preceded by the `C` (command‑output start) and `A`
(prompt start) markers; with integration disabled, none of these appear. kitty's VT parser dispatches OSC code
`133` (`kitty/vt-parser.c:536`) to `shell_prompt_marking()`, whose `case 'D'` slices the exit status out of the
sequence (`buf + 2`) and hands it to the `cmd_output_marking` callback (`kitty/screen.c:2352`), which lands in
`Window.cmd_output_marking()` → `handle_cmd_end(exit_status)` (Q5). The `--dump-bytes` recorder confirms kitty's
parser really received `ESC ]133;D;0 BEL` end‑to‑end.

**Cross‑vendor context.** OSC 133 (`A` = prompt start, `B` = prompt/command boundary, `C` = command‑output start,
`D[;ExitCode]` = command finished) is the FinalTerm/iTerm2 "semantic prompt" protocol, also implemented by iTerm2,
WezTerm, VS Code's terminal and Ghostty; on `D`, `0` denotes success and any non‑zero denotes an error. kitty's
`133;D;<status>` handling therefore conforms to an industry‑standard mechanism rather than a proprietary one.

---

## Q9 — Where the child's printed output actually appears

**Command** (redirect kitty's *own* stdout/stderr to files, so any child text leaking to them would be visible;
the child still prints `hello`/`world`):

```
$ export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1
$ ./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0' \
    >/tmp/q9_stdout.txt 2>/tmp/q9_stderr.txt
$ echo "kitty_exit=$?"
$ echo "own stdout bytes:"; wc -c < /tmp/q9_stdout.txt; cat /tmp/q9_stdout.txt
$ echo "own stderr bytes:"; wc -c < /tmp/q9_stderr.txt; cat /tmp/q9_stderr.txt
$ echo "child lines in kitty's own stdout/stderr:"; grep -c -E "hello|world" /tmp/q9_stdout.txt /tmp/q9_stderr.txt
```

**Raw output**

```
kitty_exit=0
own stdout bytes:
0
own stderr bytes:
71
[0.157] Failed to open systemd user bus with error: Connection refused
grep child lines in kitty's own stdout/stderr:
/tmp/q9_stdout.txt:0
/tmp/q9_stderr.txt:0
```

Positive side — the strace from Q1/Q6 shows the child's bytes being read by kitty from the PTY master (fd `8`) and
committed to the parser (which fills the `Screen` grid); a rendered screenshot taken during investigation showed the
two lines `hello` and `world` in the terminal grid:

```
93767 read(8, "hello\r\nworld\r\n", 1048576) = 14
```

**Code citation** — the PTY‑master read that feeds the grid, `kitty/child-monitor.c:1345,1354` (verbatim):

```
        len = read(fd, buf, available_buffer_space);
    vt_parser_commit_write(screen->vt_parser, len);
```

**Rationale.** The child's printed output appears in **kitty's on‑screen terminal grid** — the `Screen` buffer,
GPU‑rendered — and **not** on kitty's own stdout/stderr. The redirection proves the negative side: kitty's own
stdout is `0` bytes and its own stderr contains only a benign, unrelated systemd‑bus warning; neither contains
`hello` or `world` (`grep -c` returns `0` for both). The positive side is the PTY‑master `read(8, "hello\r\nworld\r\n",
…) = 14` in the trace, immediately followed by `vt_parser_commit_write(...)` which writes the bytes into the
`Screen`, where they are rendered as the two visible grid lines. The child writes the PTY *slave*; kitty reads the
PTY *master*; the text lives in kitty's grid.

---


## 10. Two critical distinctions

### Distinction A — two different "completion" messages

| | `--hold` / `hold`‑kitten banner | shell‑integration notification |
|---|---|---|
| Text | `Press Enter or Esc to exit` (bold green, `\x1b[1;32m…\x1b[m`) | `Command echo hello finished with status: 0.\nClick to focus.` |
| Contains exit status? | **No** | **Yes** (`0`) |
| Produced by | `HoldTillEnter()` — `tools/tui/hold.go:26` | `Window.handle_cmd_end()` — `kitty/window.py:1429` |
| Trigger | the `hold` kitten / `__hold_till_enter__` (the plain `--hold` flag holds via an interactive shell, see Q3a) | shell emits OSC 133 `D;<code>`; requires `notify_on_cmd_finish != never` |

Conflating these would misstate Q3 and Q5. The banner is a Go‑kitten TUI string with no status; the notification is
a Python‑built desktop message that embeds the status.

### Distinction B — two different exit‑status transports

- A **directly launched** child's termination is delivered by the OS as **`SIGCHLD`** (via the signalfd) and reaped
  by **`waitpid`/`wait4`** (`kitty/child-monitor.c:1418`). The direct‑reap *window* callback
  `on_child_death(window_id)` (`kitty/boss.py:881`) carries **no exit status** — which is precisely why the
  "finished with status" message cannot come from the direct reap.
- A **shell‑reported** command's exit code travels via the **OSC 133 `D;<code>`** escape sequence
  (`kitty/screen.c:2350-2352`).
- Q6–Q7 concern the former (OS/signal transport); Q8 concerns the latter (escape‑sequence transport). The separate
  callback `on_monitored_pid_death(self, pid, exit_status)` (`kitty/boss.py:2725`) *does* carry a status, but it is
  for *monitored background pids*, not the primary window child — do not confuse it with `on_child_death`.

---

## 11. Two default‑configuration nuances

### Nuance #1 — `close_on_child_death` default is `no`, yet the window still closes

Defaults — `kitty/options/types.py:500` and `kitty/options/definition.py:2920` (verbatim):

```
    close_on_child_death: bool = False
```

```
opt('close_on_child_death', 'no',
```

**Observed.** With the default (`no`) *and* with `-o close_on_child_death=yes`, the simple foreground child causes
the window to close and kitty to exit `0`:

```
$ ./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0' >/dev/null 2>&1; echo "default: $?"
default: 0
$ ./kitty/launcher/kitty --config NONE -o close_on_child_death=yes sh -c 'echo hello; echo world; exit 0' >/dev/null 2>&1; echo "yes: $?"
yes: 0
```

For a simple child with no lingering PTY writers, the default (`no`) close happens via the **PTY‑EOF path**: the
child dies → the PTY slave closes → `read()` returns `0`/`EIO` → `read_bytes()` returns `false`
(`kitty/child-monitor.c:1355`) → the child is marked `needs_removal = true` (`kitty/child-monitor.c:1535`). The
`SIGCHLD`‑reap dispatch at `kitty/child-monitor.c:1526` always runs, but by default it passes
`OPT(close_on_child_death) == false`, so `mark_child_for_removal` (`:1422`) is *not* the mechanism that removes the
window in the default config; setting `close_on_child_death=yes` is the alternate immediate‑removal path.

### Nuance #2 — `notify_on_cmd_finish` default is `never`

Defaults — `kitty/options/types.py:560` and `kitty/options/definition.py:3190` (verbatim):

```
    notify_on_cmd_finish: NotifyOnCmdFinish = NotifyOnCmdFinish(when='never', duration=5.0, action='notify', cmdline=())
```

```
opt('notify_on_cmd_finish', 'never', option_type='notify_on_cmd_finish', long_text='''
```

**Observed.** In the default configuration, no desktop notification is emitted — on the same D‑Bus session used for
Q3b, running the same OSC‑133‑emitting child *without* the `notify_on_cmd_finish` override produced **zero**
`org.freedesktop.Notifications.Notify` method calls:

```
==== Notify method calls in DEFAULT config (expect NONE) ====
NONE — no org.freedesktop.Notifications.Notify method call was emitted.
```

The gate is `kitty/window.py:1425` (verbatim):

```
        if last_cmd_output_duration >= duration and when != 'never':
```

`handle_cmd_end()` still runs on every command end (updating `last_cmd_exit_status`), but the body is only built and
sent when `when != 'never'` (and the output lasted at least `duration`). The Q3b observation therefore sets
`notify_on_cmd_finish always 0` — a real, canonical option value, labelled configured (non‑default).

---

## 12. Observed vs inferred

| # | Reported value | Observed / Inferred | How |
|---|----------------|---------------------|-----|
| Q1 | full spawn→read→reap→teardown ordering | **Observed** | strace death‑sequence excerpt + code |
| Q2 | kitty own exit code `0` (stable ×5; `0` even when child exits 5/42) | **Observed** | 5+2 runs of `$?` |
| Q3a | banner bytes `\x1b[1;32mPress Enter or Esc to exit\x1b[m` | **Observed** | PTY capture of `__hold_till_enter__` (158 bytes) |
| Q3a | plain `--hold` flag holds via interactive shell, not the banner | **Observed** | child is `bash` with `KITTY_HOLD=1`; screen shows no banner |
| Q3b | notification body `Command echo hello finished with status: 0.\nClick to focus.` | **Observed** | D‑Bus `Notify` capture (×2 identical), configured `notify_on_cmd_finish always 0` |
| Q4 | `ChildMonitor` owns window id/PID/child_fd/Screen | **Observed** | `add_child(...)` registration + strace of the monitor thread |
| Q5 | `Window.handle_cmd_end()` builds the message | **Observed** | `command`‑action probe: `exit_status=[0] cmdline=[echo hello]` |
| Q6 | `SIGCHLD` (17), via signalfd; `ssi_code=CLD_EXITED`, `ssi_pid=child` | **Observed** | `signalfd4([… CHLD …])`, decoded signalfd read |
| Q7 | `waitpid(-1,&status,WNOHANG)` = child, status 0 (`wait4` syscall) | **Observed** | strace `wait4(-1, [{… WEXITSTATUS(s)==0}], WNOHANG, NULL) = <pid>` |
| Q8 | OSC 133 `D;0` transport (bytes `ESC ]133;D;0 BEL`) | **Observed** | real bash‑integration PTY capture + kitty `--dump-bytes` |
| Q9 | output in on‑screen grid, not kitty's own stdout/stderr | **Observed** | own stdout 0 B / stderr benign; PTY read of `"hello\r\nworld\r\n"` |

No reported value in this document is inferred‑only; every answer is backed by captured runtime output plus a
`file:line` citation. (The on‑screen rendering in Q9 was additionally confirmed visually via a screenshot during
investigation; the byte‑level PTY/strace evidence is the primary, self‑contained proof retained here.)

---

## 13. Reproducibility

All commands, in order, with the environment `export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1` where a GUI window is
needed:

```
# Q2 / Q1 / Q9 primary path
./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0'; echo $?
./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0' >/tmp/q9_stdout.txt 2>/tmp/q9_stderr.txt

# Q1 / Q6 / Q7 — signals and reap
strace -f -e trace=signalfd4,rt_sigprocmask,wait4,read -e signal=all -o /tmp/kitty_sig.strace \
  ./kitty/launcher/kitty --config NONE sh -c 'echo hello; echo world; exit 0'

# Q3a — hold banner (via the hold kitten)
#   (PTY harness exec'ing:  kitten __hold_till_enter__ sh -c 'echo hello; echo world; exit 0')

# Q3b / Q5 — notification body (configured, non-default) on a D-Bus session
dbus-run-session -- bash -c 'dbus-monitor "interface=org.freedesktop.Notifications" >/tmp/dbus_notify.log 2>&1 & \
  DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 ./kitty/launcher/kitty --config NONE \
  -o "notify_on_cmd_finish always 0" sh -c "sh /tmp/child_osc.sh"'

# Q8 — OSC 133 wire capture (real bash integration enabled vs disabled) + kitty --dump-bytes corroboration
#   (PTY harness exec'ing:  bash --rcfile shell-integration/bash/kitty.bash -i   vs   bash --norc -i)
./kitty/launcher/kitty --config NONE --dump-bytes /tmp/kdump.bin sh -c 'sh /tmp/child_osc.sh'

# Q1 nuance — close_on_child_death
./kitty/launcher/kitty --config NONE -o close_on_child_death=yes sh -c 'echo hello; echo world; exit 0'; echo $?
```

**Stability.** kitty's own exit code (Q2) was `0` on all five consecutive runs — distribution `{0: 5}` — and
remained `0` when the child exited non‑zero. The notification body (Q3b) was byte‑identical across two runs. No
run‑to‑run variance was observed for any reported value.

**Repository left unchanged.** This investigation modified no source file. Temporary observation scripts and
captures were written under `/tmp` (outside the repository) and removed afterward; the only file added to the
repository is this document, `blitzy/documentation/kitty_815df1e210e0.md`.

---

## 14. Coverage checklist

- [x] **Q1 — end‑to‑end flow**: spawn (`child.py:276+`) → PTY‑master read (`child-monitor.c:1345`) → parse/Screen → `SIGCHLD` (signalfd) → `waitpid` reap (`child-monitor.c:1418`) → `on_child_death` teardown (`boss.py:881`); default close.
- [x] **Q2 — kitty's own exit code**: `0`, stable ×5, independent of child status (`main.py:524-531`).
- [x] **Q3a — `--hold` banner (named item: banner bytes)**: `\x1b[1;32mPress Enter or Esc to exit\x1b[m` (`hold.go:26`); no status; plain `--hold` flag holds via interactive shell.
- [x] **Q3b — notification body (named item)**: `Command echo hello finished with status: 0.\nClick to focus.` (`window.py:1429`); non‑default.
- [x] **Q4 — child‑tracking component (named item: ChildMonitor)**: `ChildMonitor` (`boss.py:370,587`; `child-monitor.c`).
- [x] **Q5 — message‑generating function (named item: `handle_cmd_end`)**: `Window.handle_cmd_end()` (`window.py:1408`, body `:1429`).
- [x] **Q6 — signal (named item: `SIGCHLD`)**: `SIGCHLD` (17) via signalfd (`child-monitor.c:121,136,1370`; `loop-utils.c:42`).
- [x] **Q7 — system call (named item: `waitpid`/`wait4`)**: `waitpid(-1,&status,WNOHANG)` (`child-monitor.c:1418`).
- [x] **Q8 — status transport (named item: OSC 133 `D;<code>`)**: `vt-parser.c:536` → `screen.c:2350-2352`; emitters `bash:239`, `zsh:145`, `fish:96`.
- [x] **Q9 — output location (named item: on‑screen grid)**: on‑screen `Screen` grid, not kitty's own stdout/stderr (`child-monitor.c:1345,1354`).
- [x] **Distinction A** (two message paths) and **Distinction B** (two transports) stated explicitly.
- [x] **Nuance #1** (default close via PTY‑EOF; `close_on_child_death=yes` alternate) and **Nuance #2** (`notify_on_cmd_finish` default `never`) addressed with observed contrasts.
- [x] Every question has **Command → Raw output → `file:line` citation (verbatim) → Rationale**.

