
# kitty TTY File-Transfer Protocol — End-to-End Trace & Delta-Efficiency Proof

> **Onboarding answer document.** This is an evidence-backed, *runtime-observed* trace of
> kitty's TTY file-transfer protocol — the machinery behind the `transfer` kitten that is
> made available over SSH by the `ssh` kitten. Every behavioral claim below is backed by
> the **complete, unedited output** of a command that was actually run against a
> **from-source build** of kitty, and every structural claim carries a `file:line` citation
> pinned to the repository HEAD.
>
> - **Repository:** kitty terminal emulator
> - **Branch:** `kitty_815df1e210e0`
> - **Pinned HEAD:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
> - **Build:** `kitty 0.35.2` (built from source in this environment — see the *Environment & Methodology* section)

---

## Table of contents

1. [Environment & methodology (read this first)](#0-environment--methodology-read-this-first)
2. [Q1 — End-to-end trace of file data](#q1--end-to-end-trace-of-file-data)
3. [Q2 — Handshake & the exact escape sequences](#q2--handshake--the-exact-escape-sequences)
4. [Q3 — Rsync-style delta transfer & the data structures](#q3--rsync-style-delta-transfer--the-data-structures)
5. [Q4 — Chunk encoding & demultiplexing](#q4--chunk-encoding--demultiplexing)
6. [Q5 — Resumption behavior](#q5--resumption-behavior)
7. [Q6 — Empirical delta efficiency (measured, ≥2 runs)](#q6--empirical-delta-efficiency-measured-2-runs)
8. [Coverage pass — every condition the questions imply](#coverage-pass--every-condition-the-questions-imply)
9. [Appendix A — verified citation quick-reference](#appendix-a--verified-citation-quick-reference)
10. [Appendix B — read-only guarantee (git status proof)](#appendix-b--read-only-guarantee-git-status-proof)

---

## 0. Environment & methodology (read this first)

### 0.1 What was built and how

The rules governing this document require that the answer be written **from what was
observed by running the code**, not from reading alone, and that the software be built and
run in its **default, canonical configuration** with the exact commands stated. Here they
are.

**Canonical build command.** kitty is built from source by running `setup.py` with **no
action argument** — the `build` action is the default. `setup.py` declares
`action: str = 'build'` on its `Options` dataclass (`setup.py:175`), wires that value in as
the argparse default (`setup.py:1829`, `default=Options.action`), and dispatches it with
`if args.action == 'build':` (`setup.py:2115`). So the canonical, default-configuration
command a normal user runs is:

```
$ python setup.py
```

**Exact invocation used in this (headless CI) container** — equivalent to the canonical
command, with the `build` action written out explicitly and `CI=true` exported so the
build is non-interactive:

```
$ CI=true python setup.py build
```

`CI=true` and the explicit `build` word do not change *what* is built (the action already
defaults to `build`); they only make the run non-interactive. `python` here is CPython
3.13.7 (`/usr/bin/python`). Complete, unedited tail of that build (re-run incrementally to
confirm it succeeds; exit status `0`):

```
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

The `wayland-protocols` message is a **non-fatal** notice — it only disables the Wayland
GUI backend (the X11 backend is used instead) and has no bearing on the file-transfer
subsystem. The build exits `0` and produces the launchers:

```
$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
$ ./kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal
```

Toolchain (all supplied by the canonical image): Go 1.22.12 (`go.mod:3` pins `go 1.22`),
CPython 3.13.7 (`pyproject.toml:2` requires `>=3.8`), gcc 15.2.0. The XXH3 hash family that
the delta engine depends on comes from the system `xxhash` headers, `#include <xxhash.h>`
at `kittens/transfer/algorithm.c:13`.

Environment variables used for every run (they configure the test/runtime, none are
secrets): `CI=true`, `LANG=en_US.UTF-8`, `LC_ALL=en_US.UTF-8`, and
`TMPDIR=/tmp/kitty_test_tmp` (a non-setgid scratch dir; `/tmp` itself is setgid here, which
would break the handler's directory-mode assertions).

### 0.2 How the protocol was exercised — the real entry point, and an honest disclosure

The **canonical** entry point for a transfer is the `ssh` kitten placing the `transfer`
kitten on the remote and the user running `kitten transfer` there. The `ssh` kitten
documents this delivery via its `remote_kitty` option (`kittens/ssh/main.py:164`), whose
help text says it copies "the :doc:`transfer file kitten </kittens/transfer>` to transfer
files" (`kittens/ssh/main.py:167`).

**Two things were *not* exercised, and both are disclosed up front:**

1. **No GPU window.** A full windowed GPU terminal cannot be launched in this headless
   container.
2. **No live SSH-server round-trip.** There is **no `sshd` in this container** (only the
   `/usr/bin/ssh` *client* is present), so a real `kitten ssh <host>` → remote
   `kitten transfer` hop over an SSH server **could not be exercised**. Any value that would
   depend on that SSH hop is therefore **non-canonical / not observed here** and is labeled
   as such wherever it appears.

To exercise the protocol through its **real code**, the transfers below were driven through
the repository's own pseudo-terminal harness `kitty_tests/file_transmission.py`. This is
**not a synthetic stand-in** for the protocol logic:

- The harness runs the **real `kitten transfer` binary** (`kitty/launcher/kitten`, resolved
  via `kitty.constants.kitten_exe`) as the child of a real pseudo-terminal — class
  `TransferPTY` at `kitty_tests/file_transmission.py:173`.
- The bytes the kitten writes are read from the pty master and fed to the **real C VT
  parser** (`parse_bytes` → `kitty/vt-parser.c`), which dispatches the file-transfer OSC to
  the **real terminal-side handler** `kitty/file_transmission.py` — class
  `PtyFileTransmission(FileTransmission)` at `kitty_tests/file_transmission.py:160`.
- Files are written to a real filesystem.

**Exactly which layers are canonical (exercised byte-for-byte) vs. not:**

| Layer | Status in this document |
|-------|-------------------------|
| Client serializer (real `kitten transfer` binary → `FileTransmissionCommand.Serialize`) | **Canonical — exercised** |
| TTY byte stream (pty master/slave) | **Canonical — exercised** |
| C VT parser demultiplexing `OSC 5113` (`kitty/vt-parser.c:547`) | **Canonical — exercised** |
| Terminal-side handler (`kitty/file_transmission.py`) + disk write | **Canonical — exercised** |
| GPU window rendering / confirmation dialog | **Not exercised** (headless) — disclosed |
| SSH transport hop (`kitten ssh` → remote `kitten transfer` over `sshd`) | **Not exercised** (no `sshd`) — **non-canonical** |

**One claim is explicitly labeled `INFERRED`, not observed:** that the `OSC 5113` frames
travelling over a local pty would be **byte-identical** over an SSH connection. This follows
from SSH being a transparent 8-bit byte pipe and from the `ssh` kitten merely *delivering*
the `transfer` kitten to the remote (`kittens/ssh/main.py:167`) rather than transforming its
bytes — but because no `sshd` round-trip was run, it is an inference from the code and
protocol, **not** a runtime observation. Every *other* value in this document was produced
by the real local path above.

All observation scripts and sample files were created under `/tmp` (outside the repository
tree). The read-only guarantee is proven in [Appendix B](#appendix-b--read-only-guarantee-git-status-proof).

### 0.3 The baseline test suite passes

Before capturing anything, the in-repo suites were run to confirm the handler and engine
work end-to-end:

```
$ CI=true LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 TMPDIR=/tmp/kitty_test_tmp \
    ./kitty/launcher/kitty +launch test.py --module file_transmission
Intrinsics: has_avx2=True has_sse4_2=True
test_file_get (kitty_tests.file_transmission.TestFileTransmission.test_file_get) ... ok
test_parse_ftc (kitty_tests.file_transmission.TestFileTransmission.test_parse_ftc) ... ok
test_rsync_hashers (kitty_tests.file_transmission.TestFileTransmission.test_rsync_hashers) ... ok
test_rsync_roundtrip (kitty_tests.file_transmission.TestFileTransmission.test_rsync_roundtrip) ... ok
test_transfer_receive (kitty_tests.file_transmission.TestFileTransmission.test_transfer_receive) ... ok
test_transfer_send (kitty_tests.file_transmission.TestFileTransmission.test_transfer_send) ... ok

----------------------------------------------------------------------
Ran 6 tests in 1.226s

OK
```

```
$ go test -count=1 ./tools/rsync/...
ok  	kitty/tools/rsync	0.010s
```

---

## Q1 — End-to-end trace of file data

### Direct answer

For an upload (a `send` session — the default `--direction=download`, i.e. the kitten sends
its files to the terminal), file data travels this exact path, naming the specific
function/struct that does each step:

1. **`kittens/transfer/main.py`** — Python entry point / CLI parsing for `kitten transfer`.
2. **`SendManager.start_transfer()`** (`kittens/transfer/send.go:363`) emits the opening
   `action=send` command that begins the session.
3. **`SendHandler.send_payload()`** (`kittens/transfer/send.go:646`) wraps *every* command
   by writing `manager.prefix` + payload + `manager.suffix` to the TTY loop — that is, it
   frames each command as an `OSC 5113` escape code.
4. The bytes travel over the **TTY / SSH** byte stream.
5. The terminal's VT parser dispatches the code at
   **`kitty/vt-parser.c:547`** (`case FILE_TRANSFER_CODE:` → `DISPATCH_OSC(file_transmission)`),
   handing the payload to the file-transmission subsystem instead of the screen.
6. The Python handler **`kitty/file_transmission.py`** validates the session, acknowledges,
   and writes the bytes to **disk** (via a `DestFile` / `PatchFile`).
7. Status/PROGRESS acknowledgements flow **back** to the client. For this `send` session the
   client's terminal loop routes them through the `OnEscapeCode` callback installed inside
   **`send_loop`** at **`kittens/transfer/send.go:1223-1232`** (`ftc_code :=
   strconv.Itoa(kitty.FileTransferCode)` … `return handler.on_file_transfer_response(ftc)`).
   *(The receive/download-of-files direction has the structurally identical callback in its
   own loop at `kittens/transfer/receive.go:1120-1121`; see [Q4](#q4--chunk-encoding--demultiplexing).)*

### Runtime evidence

**Exact command** (driven through the pty harness described in §0.2, which forks this real
binary as the pty child):

```
$ kitten transfer /tmp/kitty_test_tmp/q1_lbbv4tdr/src /tmp/kitty_test_tmp/q1_lbbv4tdr/dest
    (default --direction=download -> a 'send' session)
```

A 28-byte file `hello, kitty file transfer!\n` was sent. The driver captured (a) the exact
bytes the client emitted on the pty, (b) each command after the real C parser demultiplexed
it, and (c) each response the real handler wrote back. Complete, unedited output:

```
=== (a) RAW BYTES emitted by real kitten client (pty.received_bytes) ===
total bytes from client: 1274
repr: b'Scanning files\xe2\x80\xa6\r\nFound 1 files and directories, requesting transfer permission\xe2\x80\xa6\r\n\x1b[?s\x1b[*x\x1b[4l\x1b[?1l\x1b[?5l\x1b[?2004l\x1b[?1004l\x1b[?1000l\x1b[?1002l\x1b[?1003l\x1b[?1005l\x1b[?1006l\x1b[?8h\x1b[?7h\x1b[?25h\x1b[>29u\x1b[?25l\x1b]5113;id=180c22586;ac=send\x1b\\\x1b[32mPermission granted for this transfer\x1b[39m\n\r\x1b]5113;id=180c22586;n=L3RtcC9raXR0eV90ZXN0X3RtcC9xMV9sYmJ2NHRkci9kZXN0;mod=1783487607447484512;ac=file;fid=1;prm=420\x1b\\\x1b]5113;id=180c22586;ac=end_data;fid=1;d=aGVsbG8sIGtpdHR5IGZpbGUgdHJhbnNmZXIhCg\x1b\\\x1b[?7l\xe2\xa0\x8b Transferring metadata...\n\rFile data transfer has not yet started\n\r\x1b[?7h\x1b[?2026h\x1b[2A\r\x1b[J\x1b[?7l\xe2\xa0\x8b Transferring metadata...\n\rFile data transfer has not yet started\n\r\x1b[?7h\x1b[?2026l\x1b[?2026h\x1b[2A\r\x1b[J\x1b[?7l\x1b[32m\xe2\x9c\x94\x1b[39m /tmp/kitty_test_tmp/q1_lbbv4tdr/src                                    28\x1b[2mB\x1b[222m\x1b[93m @ \x1b[39m69\x1b[2mkB/s\x1b[222m \x1b[32m\xf0\x9f\xac\x8b\x1b[24b\x1b[39m \x1b[32m  <1 sec\x1b[39m\n\r\n\r\xe2\xa0\x8b Total                                                                  28\x1b[2mB\x1b[222m\x1b[93m @ \x1b[39m63\x1b[2mkB/s\x1b[222m \x1b[32m\xf0\x9f\xac\x8b\x1b[24b\x1b[39m \x1b[32m  <1 sec\x1b[39m\n\r\x1b[?7h\x1b[?2026l\x1b]5113;id=180c22586;ac=finish\x1b\\\x1b[?2026h\x1b[2A\r\x1b[J\x1b[?7l\xe2\x94\x80\x1b[119b\n\r\x1b[32m\xe2\x9c\x94\x1b[39m Total                                                                  28\x1b[2mB\x1b[222m\x1b[93m @ \x1b[39m46\x1b[2mkB/s\x1b[222m \x1b[32m\xf0\x9f\xac\x8b\x1b[24b\x1b[39m \x1b[32m  <1 sec\x1b[39m\n\r\x1b[?7h\x1b[?2026l\x1b[?25h\x1b[<u\x1b7\x1b[?r\x1b8'

=== (b) ORDERED CLIENT->TERMINAL COMMANDS (demuxed by the real C parser) ===
[0] id=180c22586;ac=send
[1] id=180c22586;n=L3RtcC9raXR0eV90ZXN0X3RtcC9xMV9sYmJ2NHRkci9kZXN0;mod=1783487607447484512;ac=file;fid=1;prm=420
[2] id=180c22586;ac=end_data;fid=1;d=aGVsbG8sIGtpdHR5IGZpbGUgdHJhbnNmZXIhCg
[3] id=180c22586;ac=finish

=== (c) ORDERED TERMINAL->CLIENT RESPONSES (real handler) ===
[0] action='status' id='180c22586' status='OK'
[1] action='status' id='180c22586' file_id='1' name='/tmp/kitty_test_tmp/q1_lbbv4tdr/dest' status='STARTED'
[2] action='status' id='180c22586' file_id='1' size=28 name='/tmp/kitty_test_tmp/q1_lbbv4tdr/dest' status='OK'

=== (d) DISK TERMINUS ===
dest bytes: 28 match src: True
dest content repr: b'hello, kitty file transfer!\n'

=== (e) base64/value decodes verified against emitted bytes ===
  n=   -> b'/tmp/kitty_test_tmp/q1_lbbv4tdr/dest'
  mod=1783487607447484512 (ns since epoch)
  prm=420 -> octal 0o644
  d=   -> b'hello, kitty file transfer!\n'
  response status decoded -> 'OK'
  response status decoded -> 'STARTED'
  response status decoded -> 'OK'
```

### Why this is the whole path

The ordered command list **is** the trace, and it matches the narrated chain step for step:

- `[0] ac=send` is produced by `SendManager.start_transfer()`
  (`kittens/transfer/send.go:363-364`, which returns
  `FileTransmissionCommand{Action: Action_send, Bypass: self.bypass}.Serialize()`).
- Each command is framed and written by `SendHandler.send_payload()`
  (`kittens/transfer/send.go:646-649`).
- The three responses (`OK` → `STARTED` → `OK`) are produced by the **real handler** after
  the C parser routed the commands to it (`kitty/vt-parser.c:547`), and are delivered back
  to the client through the `send_loop` `OnEscapeCode` callback
  (`kittens/transfer/send.go:1223-1232`).
- `[2] ac=end_data` carries the file's bytes under `d=` and marks the last chunk; the
  handler then reports `status=OK` with `size=28`, and the **file on disk is byte-identical
  to the source** — the terminus of the trace.

The send side advances a per-file state machine `WAITING_FOR_START` (`send.go:38`) →
`TRANSMITTING` (`send.go:40`) → `FINISHED` (`send.go:41`), and a session permission state
`SEND_WAITING_FOR_PERMISSION` (`send.go:280`) that becomes `SEND_PERMISSION_GRANTED`
(`send.go:800`) on a `status=OK` reply or `SEND_PERMISSION_DENIED` (`send.go:802`) otherwise
— which is exactly why response `[0]` (`status=OK`) must arrive before any `data`/`end_data`
command is sent.

---

## Q2 — Handshake & the exact escape sequences

### Direct answer

The transfer kitten initiates the handshake by serializing a single **`action=send`**
command and transmitting it wrapped in an **`OSC 5113`** escape frame. Concretely,
`SendManager.start_transfer()` (`kittens/transfer/send.go:363`) returns

```go
return FileTransmissionCommand{Action: Action_send, Bypass: self.bypass}.Serialize()
```

and `SendHandler.send_payload()` (`kittens/transfer/send.go:646`) writes it to the TTY
sandwiched between a prefix and suffix that form the escape code. The exact bytes on the
wire are:

```
ESC ] 5 1 1 3 ; id=<request_id> ; ac=send   ESC \
0x1b 0x5d 5113 ';'                            0x1b 0x5c
└── OSC introducer ──┘                        └── ST ──┘
```

- **`OSC`** (Operating System Command introducer) = the two bytes `0x1b 0x5d` (`ESC ]`).
- **`5113`** = `FILE_TRANSFER_CODE`.
- **`ST`** (String Terminator) = the two bytes `0x1b 0x5c` (`ESC \`).

The overall encoding is `<OSC> 5113 ; key=value ; key=value … <ST>`, specified in
`docs/file-transfer-protocol.rst` under *"Encoding of transfer commands as escape codes"*
(section heading at line 540).

### Where the frame is constructed

The prefix and suffix are assembled once per session and then reused for every command:

```go
// kittens/transfer/send.go:384-385
self.prefix = fmt.Sprintf("\x1b]%d;id=%s;", kitty.FileTransferCode, self.request_id)
self.suffix = "\x1b\\"
```

and written around each payload:

```go
// kittens/transfer/send.go:646-649
func (self *SendHandler) send_payload(payload string) loop.IdType {
	self.lp.QueueWriteString(self.manager.prefix)
	self.lp.QueueWriteString(payload)
	return self.lp.QueueWriteString(self.manager.suffix)
}
```

The constant `5113` is defined once in C and mirrored into the other two languages:

```c
// kitty/control-codes.h:233
#define FILE_TRANSFER_CODE 5113
```

```c
// kitty/data-types.c:596
PyModule_AddIntMacro(m, FILE_TRANSFER_CODE);   // exposes it to Python
```

```python
# gen/go_code.py:597  (generates the Go mirror)
const FileTransferCode int = {FILE_TRANSFER_CODE}
```

### Runtime evidence (byte-exact)

**Exact command:**

```
$ kitten transfer /tmp/kitty_test_tmp/q2_7o283_k7/src /tmp/kitty_test_tmp/q2_7o283_k7/dest
```

The opening frame emitted by the **real kitten** was captured and dumped as a Python
`repr()` and as the full hex of every byte. Complete, unedited output:

```
=== BYTE-EXACT OPENING action=send FRAME (first OSC 5113 frame on the wire) ===
repr : b'\x1b]5113;id=180d5da25;ac=send\x1b\\'
full hex: 1b 5d 35 31 31 33 3b 69 64 3d 31 38 30 64 35 64 61 32 35 3b 61 63 3d 73 65 6e 64 1b 5c
first 7 bytes (hex): 1b 5d 35 31 31 33 3b  ->  ESC ]  5  1  1  3  ;
last 2 bytes  (hex): 1b 5c                      ->  ESC  \   (ST)
verify begins ESC ] 5 1 1 3 ; : True
verify ends   1b 5c                 : True

=== ALL OSC 5113 FRAMES ON THE WIRE (count=4) ===
[0] b'\x1b]5113;id=180d5da25;ac=send\x1b\\'
[1] b'\x1b]5113;id=180d5da25;mod=1783487607577485912;ac=file;fid=1;n=L3RtcC9raXR0eV90ZXN0X3RtcC9xMl83bzI4M19rNy9kZXN0;prm=420\x1b\\'
[2] b'\x1b]5113;id=180d5da25;ac=end_data;d=eA;fid=1\x1b\\'
[3] b'\x1b]5113;id=180d5da25;ac=finish\x1b\\'
```

This literally begins `1b 5d 35 31 31 33 3b` (`ESC ] 5 1 1 3 ;`) and ends `1b 5c` (`ESC \`),
confirming the `OSC 5113 … ST` form byte-for-byte. (This run used a 1-byte file `x`; the
payload `d=eA` base64-decodes to `b'x'`.)

### The key schema (field → wire short-key)

Every command is a `FileTransmissionCommand` (struct at `kittens/transfer/ftc.go:120`).
On the wire the field names are abbreviated to reduce overhead; the struct's JSON tags give
the exact mapping (verified against the source), and the protocol keys table in
`docs/file-transfer-protocol.rst` (rows at lines 565–574) documents each value's type:

| Struct field  | Wire key | Type / notes                                                             |
|---------------|----------|--------------------------------------------------------------------------|
| `Action`      | `ac`     | enum: `send, file, data, end_data, receive, cancel, status, finish`      |
| `Compression` | `zip`    | `none` \| `zlib` (RFC 1950 deflate)                                        |
| `Ftype`       | `ft`     | `regular` \| `directory` \| `symlink` \| `link`                           |
| `Ttype`       | `tt`     | `simple` \| `rsync`                                                       |
| `Quiet`       | `q`      | `0` verbose \| `1` errors only \| `2` silent                              |
| `Id`          | `id`     | session id — `safe_string` (spec `:565`)                                 |
| `File_id`     | `fid`    | per-file id — `safe_string` (spec `:566`)                                |
| `Bypass`      | `pw`     | **see divergence note below**                                            |
| `Name`        | `n`      | `base64_string` (spec `:572`)                                            |
| `Status`      | `st`     | `base64_string` (spec `:573`)                                            |
| `Parent`      | `pr`     | parent file id — `safe_string` (spec `:574`)                            |
| `Mtime`       | `mod`    | integer; ns since epoch                                                  |
| `Permissions` | `prm`    | integer; UNIX mode bits                                                  |
| `Size`        | `sz`     | integer; default `-1`                                                    |
| `Data`        | `d`      | `base64_bytes`; the file chunk, ≤4096 bytes                              |

> **Spec-vs-implementation divergence for `Bypass` (`pw`) — documented, with both
> citations.** The wire spec lists `pw` as a **`safe_string`**: `docs/file-transfer-protocol.rst:567`
> reads `bypass            pw       safe_string    hash of the bypass password and the session id`.
> **However, both implementations base64-encode it.** In Go, the struct field is tagged for
> base64 — `Bypass      string        json:"pw,omitempty" encoding:"base64"`
> (`kittens/transfer/ftc.go:129`) — and the serializer honors that tag by emitting
> `base64.RawStdEncoding.EncodeToString(...)` for a `reflect.String` whose `encoding` tag is
> `"base64"` (`kittens/transfer/ftc.go:174-183`). In Python, the field carries
> `metadata={'base64': True, 'sname': 'pw'}` (`kitty/file_transmission.py:260`). So although
> the spec's type column says `safe_string`, the value that actually travels for `pw` is
> **base64-encoded** by both the Go kitten and the Python handler. (`id`/`fid` have no such
> tag and are serialized as plain `safe_string`, matching the spec.)

Cross-checking against the captured commands from Q1 confirms the rest of the schema is
exactly what the wire uses — e.g. `ac=send`, `n=<b64>;mod=<ns>;ac=file;fid=1;prm=420`,
`ac=end_data;fid=1;d=<b64>`, and `ac=finish`.

---

## Q3 — Rsync-style delta transfer & the data structures

### Direct answer

kitty implements the classic rsync algorithm: the side that **already has a copy** of the
file computes a **signature** — a list of fixed-size block hashes — and the side that has
the **new** version computes a **delta** that references the unchanged blocks by index and
inlines only the changed bytes.

The data structures that track file signatures and block differences are the rsync-binding
types declared in **`kittens/transfer/rsync.pyi`**:

- **`Hasher`** (`kittens/transfer/rsync.pyi:9`) — a hash object (`xxh3-64` or `xxh3-128`)
  used for block-identity and whole-file integrity hashing.
- **`Patcher`** (`kittens/transfer/rsync.pyi:24`) — holds the receiver's side: it produces
  the **signature** of the file it already has and applies incoming deltas; it exposes
  `total_data_in_delta` (`kittens/transfer/rsync.pyi:35`), the count of literal bytes that
  actually travelled in a delta being applied.
- **`Differ`** (`kittens/transfer/rsync.pyi:38`) — holds the sender's side: fed the peer's
  signature, its `next_op` method (`kittens/transfer/rsync.pyi:42`) walks the new file with
  the rolling weak hash + strong hash and emits the delta operations.
- **`parse_ftc`** (`kittens/transfer/rsync.pyi:45`) — parses the on-wire command form.

> **Accuracy note (a documented citation trap):** the AAP *body* mis-cited these as
> `total_data_in_delta` @ L30-32 and `next_op` @ L38-40. The **observed** lines in
> `kittens/transfer/rsync.pyi` at this HEAD are **`total_data_in_delta` @ L35** and
> **`next_op` @ L42** (with `RsyncError`@6, `Hasher`@9, `Patcher`@24, `Differ`@38,
> `parse_ftc`@45). This document uses the observed lines.

The Go engine mirrors these types in `tools/rsync/api.go`: `Api` (`:47`), `Differ` (`:56`),
`Patcher` (`:61`), with `CreateSignatureIterator` (`:195`) and `CreateDelta` (`:230`); the C
implementation is `kittens/transfer/algorithm.c`, which uses XXH3 via
`#include <xxhash.h>` (`kittens/transfer/algorithm.c:13`).

### The wire formats (authoritative: `docs/file-transfer-protocol.rst:409`)

All integers are little-endian.

**Signature — 12-byte header**, then one record per block:

```
struct signature_header {   // 12 bytes
    uint16 version;
    uint16 checksum_type;    // 0 -> XXH3-128 whole-file integrity checksum
    uint16 strong_hash_type; // 0 -> XXH3-64 block-identity hash
    uint16 weak_hash_type;   // 0 -> rsync rolling checksum (the "weak" hash)
    uint32 block_size;       // usually ~ sqrt(file size)
}
struct block_signature {     // 20 bytes each
    uint64 index;            // block position = index * block_size
    uint32 weak_hash;
    uint64 strong_hash;
}
```

**Delta — a stream of typed operations:**

| Op           | type | payload                                   | meaning                            |
|--------------|:----:|-------------------------------------------|------------------------------------|
| `Block`      | `0`  | `uint64` block index                      | copy that block from the old file  |
| `Data`       | `1`  | `uint32` size + `size` raw bytes          | write these literal bytes          |
| `Hash`       | `2`  | `uint16` size + checksum                  | verify reconstructed output        |
| `BlockRange` | `3`  | `uint64` start index + `uint32` N         | copy start + N further blocks      |

The three hash families are named in the spec: the **weak** hash is the rsync **rolling
checksum**; the **strong** (block-identity) hash is **XXH3-64**; the whole-file integrity
checksum is **XXH3-128**. (Op wire sizes, confirmed against `tools/rsync/algorithm.go`:
`OpBlock`=9 bytes, `OpData`=5+len, `OpHash`=3+len, `OpBlockRange`=13 bytes.)

### Runtime evidence — a real signature and a real delta

**Exact command / driver** (uses the in-repo rsync engine — the same one the kitten uses —
imported as `kittens.transfer.rsync`; run via `./kitty/launcher/kitty +launch <script>`):

```
base = bytes(bytearray((i // 32) & 0xff for i in range(1024)))   # 1024 bytes, 32 blocks
target = bytearray(base); target[458:463] = b'XYZ!!'             # edit 5 bytes at offset 458
#   Patcher(expected_size=1024).signature_of_file(base)  -> signature bytes
#   Differ() fed that signature over target                -> delta bytes
```

Complete, unedited output (raw bytes dumped and decoded):

```
=== SIGNATURE: 12-byte header (hexdump) ===
00 00 00 00 00 00 00 00 20 00 00 00
decoded header:
  version          = 0
  checksum_type    = 0   -> XXH3-128 (whole-file integrity)
  strong_hash_type = 0   -> XXH3-64  (block identity)
  weak_hash_type   = 0   -> rsync rolling checksum (weak)
  block_size       = 32  (0x20)  == floor(sqrt(1024))=32
total signature length = 652 bytes  == 12 (header) + 32 records * 20 bytes

=== FIRST BLOCK SIGNATURE RECORD (20 bytes) ===
00 00 00 00 00 00 00 00 00 00 00 00 9d c9 71 90 1c 27 57 a0
  index       (uint64) = 0
  weak_hash   (uint32) = 0x00000000  = 0   (4 bytes)
  strong_hash (uint64) = 0xa057271c9071c99d = 11553746372678240669 (8 bytes)

=== DELTA (82 bytes) — operation histogram and decoded ops ===
Block(type=0)    x   0
Data(type=1)     x   1
    Data size=32  payload=b'\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0eXYZ!!\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e\x0e'
Hash(type=2)     x   1
    Hash size=16  checksum=099e5b664c55976121c8b168fcc78659   (XXH3-128 = 16 bytes = 128 bits)
BlockRange(type=3) x   2
literal bytes inlined as Data: 32 of 1024 target bytes
Patcher.total_data_in_delta (rsync.pyi:35) = 0
```

Every decoded field matches the spec: the header is all zeros except `block_size = 32`
(= √1024); each block record is 20 bytes (8 + 4 + 8), giving a total signature of
`12 + 32×20 = 652` bytes; the strong hash occupies 8 bytes and the weak hash 4 bytes; and
the trailing `Hash` op carries a 16-byte (128-bit) XXH3-128 checksum. Critically, of the
1024 target bytes, **only the single 32-byte block that contains the edit travelled as
literal `Data`** (`b'\x0e'×10 + b'XYZ!!' + b'\x0e'×17`), while the other 31 unchanged blocks
were expressed as **two `BlockRange` references** (the runs before and after the edited
block). That is the delta mechanism in miniature.

> **Note on `total_data_in_delta = 0` here.** `Patcher.total_data_in_delta`
> (`kittens/transfer/rsync.pyi:35`) counts literal bytes on the **apply/receive** side; it is
> populated while a `Patcher` *applies* an incoming delta. In this capture the `Patcher` was
> used only to compute the *signature* (the `Differ` produced the delta), so the counter is
> `0`. [Q6](#q6--empirical-delta-efficiency-measured-2-runs) shows it non-zero (`2000`) when a
> delta is actually applied.

### Cross-engine confirmation

The hash families were confirmed directly (the same values as `test_rsync_hashers`), and the
whole signature/delta round-trip against the Go engine's tests:

```
Hasher('xxh3-64')  of b'abcd' -> hexdigest 6497a96f53a89890
Hasher('xxh3-128') of b'abcd' -> hexdigest 8d6b60383dfa90c21be79eecd1b1353d

$ go test -v ./tools/rsync/...
=== RUN   TestRsyncRoundtrip
--- PASS: TestRsyncRoundtrip (0.00s)
=== RUN   TestRsyncHashers
--- PASS: TestRsyncHashers (0.00s)
PASS
ok  	kitty/tools/rsync	0.010s
```

---

## Q4 — Chunk encoding & demultiplexing

### Direct answer

**Encoding:** file bytes are carried in the `data` command under the **`d=`** key as
**base64** (`base64_bytes`), in chunks of **≤4096 raw bytes each**. The final chunk is sent
as an `end_data` command rather than `data`.

**Demultiplexing** happens on both ends:

- **Terminal side** — the C VT parser recognizes the `5113` OSC code and routes the payload
  to the file-transmission subsystem *instead of drawing it to the screen*. This is the
  precise point of separation:
  ```c
  // kitty/vt-parser.c:547
  case FILE_TRANSFER_CODE:
      START_DISPATCH
      DISPATCH_OSC(file_transmission);
      END_DISPATCH
  ```
- **Client side** — the kitten's terminal loop installs an `OnEscapeCode` callback that
  filters on the file-transfer code so protocol responses are never confused with the
  child program's ordinary output. For the receive/download-of-files direction this is at
  `kittens/transfer/receive.go:1120-1121` (the send session's structurally identical
  callback is at `kittens/transfer/send.go:1223-1232`, cited in [Q1](#q1--end-to-end-trace-of-file-data)).
  The complete, unedited receive-side callback:
  ```go
  // kittens/transfer/receive.go:1120 onward
  ftc_code := strconv.Itoa(kitty.FileTransferCode)
  lp.OnEscapeCode = func(et loop.EscapeCodeType, payload []byte) error {
      if et == loop.OSC {
          if idx := bytes.IndexByte(payload, ';'); idx > 0 {
              if utils.UnsafeBytesToString(payload[:idx]) == ftc_code {
                  ftc, err := NewFileTransmissionCommand(utils.UnsafeBytesToString(payload[idx+1:]))
                  if err != nil {
                      return fmt.Errorf("Received invalid FileTransmissionCommand from terminal with error: %w", err)
                  }
                  return handler.on_file_transfer_response(ftc)
              }
          }
      }
      return nil
  }
  ```
  Read step by step: the callback fires only for `loop.OSC` escape codes; it finds the
  first `;`, and only if the code portion before it equals `ftc_code` (`"5113"`) does it
  parse the remainder as a `FileTransmissionCommand` and forward it to
  `on_file_transfer_response`. Any other escape code (ordinary program output) falls through
  to `return nil` and is ignored by the transfer logic. Reassembly of the chunks into the
  destination file happens in **`remote_file.write_data()`**
  (`kittens/transfer/receive.go:200`).

### Runtime evidence — chunk encoding

**Exact command:**

```
$ kitten transfer --compress=never /tmp/kitty_test_tmp/q4_w0nqba_1/src /tmp/kitty_test_tmp/q4_w0nqba_1/dest
```

A 10,000-byte file was sent with compression disabled (`--compress=never`) so the raw
chunking is visible. Complete, unedited output:

```
=== CHUNK ENCODING (10000-byte file, --compress=never) ===
data cmd: ac=data      d(base64)=5462 chars -> decoded 4096 raw bytes
data cmd: ac=data      d(base64)=5462 chars -> decoded 4096 raw bytes
data cmd: ac=end_data  d(base64)=2411 chars -> decoded 1808 raw bytes

chunks: 3 ; max raw bytes in any chunk: 4096  (spec limit: <=4096)  OK
sum of decoded chunk sizes: 4096 + 4096 + 1808 = 10000  == file size 10000  OK
last chunk carries ac=end_data: True
base64 is UNPADDED (4096 raw -> 5462 chars, no '=' padding on wire): True
first 16 decoded bytes: 00 07 0e 15 1c 23 2a 31 38 3f 46 4d 54 5b 62 69
first 16 source  bytes: 00 07 0e 15 1c 23 2a 31 38 3f 46 4d 54 5b 62 69  (match True)
full reassembled == source: True
```

The three chunks are 4096 + 4096 + 1808 = 10000 bytes, so no chunk exceeds the 4096-byte
limit, the final chunk is the `end_data` command, and the decoded payload matches the
source bytes exactly. Note base64 expands 4096 raw bytes to 5462 wire characters
(≈ 4/3 overhead), unpadded.

### Runtime evidence — demultiplexing

The same run captured the raw byte stream from the client and, separately, the final screen
buffer produced by the real C parser. Complete, unedited output:

```
=== DEMULTIPLEXING PROOF ===
raw bytes emitted by client                     : 14647
count of OSC 5113 frames in raw stream           : 6
first data chunk base64 (first 40 chars)         : AAcOFRwjKjE4P0ZNVFtiaXB3foWMk5qhqK+2vcTL
  present in RAW byte stream                     : True
  present in SCREEN buffer (what user sees)      : False   <-- KEY RESULT

screen buffer (what the user would see), non-empty lines:
  'Scanning files…'
  'Found 1 files and directories, requesting transfer permission…'
  'Permission granted for this transfer'
  '✔ /tmp/kitty_test_tmp/q4_w0nqba_1/src                                  10kB @ 6.8MB/s 🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋   <1 sec'
  '────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────'
  '✔ Total                                                                10kB @ 5.8MB/s 🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋🬋   <1 sec'
```

This is the demultiplexing made visible: the payload's base64
(`AAcOFRwjKjE4P0ZNVFtiaXB3foWMk5qhqK+2vcTL…`) is present in the raw stream (six `OSC 5113`
frames — `send`, `file`, `data`, `data`, `end_data`, `finish`) but **absent from the screen
buffer** — the C parser at `kitty/vt-parser.c:547` diverted every `5113` frame to the
handler rather than rendering it, so the file data never touches the screen. Only the
progress UI (which the kitten draws with ordinary SGR/CSI sequences) is rendered.

---

## Q5 — Resumption behavior

### Direct answer

An interrupted transfer resumes rather than restarts because **the existing (partial)
on-disk file is reused as the rsync delta base**. There is **no separate journal or
checkpoint file** — the "resumption metadata" *is* the existing file's content plus the
signature computed from it on the fly.

The mechanism, terminal-side (`kitty/file_transmission.py`):

- The destination file object stats any pre-existing file when it is created:
  ```python
  # kitty/file_transmission.py:449-451
  try:
      self.existing_stat: Optional[os.stat_result] = os.stat(self.name, follow_symlinks=False)
  except OSError:
      self.existing_stat = None
  ```
- It opens that existing file as the patch target, seeding it with the existing size:
  ```python
  # kitty/file_transmission.py:470
  self.actual_file = PatchFile(self.name, self.existing_stat.st_size if self.existing_stat is not None else 0)
  ```
- When the transmission type is `rsync`, the handler transmits a **signature of the existing
  content** back to the sender via `transmit_rsync_signature`
  (defined at `kitty/file_transmission.py:1081`, invoked at `:1038`, `:1090`, `:1129`), so
  the sender only needs to return a delta.

Receive-side (`kittens/transfer/receive.go`): the `remote_file` struct (`:125`) carries the
resumption fields `expect_diff` (`:127`) and `patcher` (`:128`). On a resuming transfer they
are set up so the incoming delta is applied against the existing file:

```go
// kittens/transfer/receive.go:422-425
f.expect_diff = true
f.patcher = rsync.NewPatcher(f.expected_size)
output := sigwriter{q: queue_write, file_id: f.file_id, prefix: self.prefix, suffix: self.suffix}
s_it := f.patcher.CreateSignatureIterator(fsf, &output)   // signature of what is already on disk
```

### Runtime evidence — before / during / after (the transitional states)

**(A) Interrupt — the exact command and how it was killed.** A 4,000,000-byte random file
transfer was started and its **real `kitten transfer` child process was killed with
`SIGKILL`** once the handler had written between 1 MB and 3 MB to disk (the driver flushes
and `fsync`s the partial `DestFile`, then issues `os.kill(child_pid, signal.SIGKILL)`):

```
interrupt command: kitten transfer /tmp/kitty_test_tmp/q5_xuqszn56/src /tmp/kitty_test_tmp/q5_xuqszn56/dest
BEFORE-STATE (partial left on disk):
  partial file exists : True
  partial file size   : 1003515 of 4000000
  partial prefix matches source prefix : True
```

**(B) Resume — the exact command, with the complete observed frames.** The same transfer
was restarted with `--transmit-deltas` (rsync mode) against that 1,003,515-byte partial.
Complete, unedited output:

```
resume command: kitten transfer --transmit-deltas /tmp/kitty_test_tmp/q5_xuqszn56/src /tmp/kitty_test_tmp/q5_xuqszn56/dest
DURING-STATE (observed):
  STARTED response (terminal->client), complete:
     {'action': 'status', 'ttype': 'rsync', 'id': '18118d76c', 'file_id': '1', 'size': 1003515, 'name': '/tmp/kitty_test_tmp/q5_xuqszn56/dest', 'status': 'STARTED'}
  transmit_rsync_signature call count  : 6
  existing_stat.st_size seen by handler: [1003515] (== partial size 1003515)
  signature DATA frames emitted terminal->client: 6
  delta DATA commands sent client->terminal      : 734
  client->terminal delta DATA-frame wire bytes   : 4028520
AFTER-STATE:
  dest size : 4000000 ; dest == source : True
```

**What the frames prove.** The `STARTED` response reports `ttype='rsync'` and
`size=1003515` — i.e. the handler adopted the **existing 1,003,515-byte partial** as the
transfer's starting size, not zero. The real `transmit_rsync_signature`
(`kitty/file_transmission.py:1081`) was invoked **6 times**, and the size it saw via
`existing_stat.st_size` was exactly the partial size (`1003515`) — this is the handler
computing a **signature of the bytes already on disk** and sending them back
(6 signature DATA frames terminal→client). The client then returned a delta
(734 DATA commands), and the reconstructed destination is byte-identical to the source
(`dest == source : True`). All three transitional states — **before** (partial present on
disk), **during** (handler stats the partial and signs it), **after** (transfer completes,
reusing the partial as the delta base) — were **observed**, not inferred.

> **Honest nuance (observed):** the delta DATA-frame wire bytes here (4,028,520) are
> **larger** than the file itself, because this file is **random / incompressible** — the
> matched ~1 MB prefix is referenced cheaply as blocks, but the remaining ~3 MB of unique
> content must travel as base64 literals (≈ 4/3 expansion). Q5 is about **resumption
> correctness** (the partial *is* reused as the delta base — no journal, no restart from
> zero), *not* about byte savings; [Q6](#q6--empirical-delta-efficiency-measured-2-runs)
> isolates the efficiency question with a small edit to an otherwise-unchanged file.

The direct-answer claim ("the existing file is the resumption metadata; no journal") is
grounded in `existing_stat` (`file_transmission.py:450`) + `PatchFile`
(`file_transmission.py:470`) + `transmit_rsync_signature` (`file_transmission.py:1081`) and
the receive-side `expect_diff`/`patcher` (`receive.go:127-128`, set up at
`receive.go:422-425`).

---

## Q6 — Empirical delta efficiency (measured, ≥2 runs)

### Direct answer

**The second transfer sends dramatically less data than the first.** Measured as the
**actual bytes on the TTY wire** (the `OSC 5113` frames, including their framing and the
base64 expansion of every data payload), the second (delta) transfer is
**≈ 1795× smaller** than the first (full) transfer: **5,373,074 bytes vs 2,993 bytes**,
stable across 3 runs. The reduction happens because rsync block matching replaces the
unchanged regions with cheap `Block`/`BlockRange` references and transmits only the changed
bytes as `Data` operations. The matching is driven by the rsync rolling **weak** hash plus
the **XXH3-64 strong** hash inside `Differ.next_op` (`kittens/transfer/rsync.pyi:42`), and
the literal bytes that actually travel are counted by `Patcher.total_data_in_delta`
(`kittens/transfer/rsync.pyi:35`).

### Three distinct metrics — do not conflate them

This question is easy to get subtly wrong by reporting the *file* size or the *rsync engine's
literal-byte* count as if it were the wire cost. They are three different numbers, reported
separately here:

| Metric | Transfer #1 (full) | Transfer #2 (delta) | What it measures |
|--------|--------------------:|--------------------:|------------------|
| **(a) Raw file bytes** | 4,000,000 | 4,000,000 | the file on disk (identical both times) |
| **(b) Actual TTY wire bytes** | **5,373,074** | **2,993** | `OSC 5113` frame bytes incl. framing + base64 (**the real answer**) |
| **(c) Rsync literal `Data` bytes** | n/a (simple) | 2,000 | `Patcher.total_data_in_delta` — literal bytes inside the delta stream |

The **(b)** row is the honest answer to "how much less data does the second transfer send
over the terminal": **5,373,074 → 2,993 bytes**. The full transfer's wire cost (5,373,074)
is *larger* than the 4,000,000-byte file precisely because base64 inflates every data
payload by ≈ 4/3 and each ≤4096-byte chunk is wrapped in an `OSC 5113 … ST` frame
(**980 data frames**, base64 payload 5,335,634 chars). The delta transfer carries the
changed region in **one** data frame (base64 payload 2,755 chars). The **(c)** number
(2,000) is the rsync engine's own count of literal bytes *inside* the delta stream — smaller
than the 2,993 wire bytes because it excludes base64 expansion and op/frame headers.

### The experiment (the user's operative example)

1. Create a **4,000,000-byte** random file under `/tmp` (large enough that delta savings
   dominate round-trip overhead; the docs warn deltas can be *slower* for tiny files —
   `docs/kittens/transfer.rst:77`, *"Delta transfers"*).
2. Transfer it once and measure the actual TTY wire bytes.
3. Modify a **small portion** in place (overwrite **68 bytes** with fresh random data at
   offset 2,000,000; the evidence's per-run "*N*-byte edit" reports the number of bytes that
   actually *differ* — 67 or 68 across runs — since a random overwrite byte can coincidentally
   equal the original).
4. Transfer again with `--transmit-deltas` (`-x`; `docs/kittens/transfer.rst:81`) and
   measure the actual TTY wire bytes.
5. Repeat the whole thing and confirm stability. `block_size` = floor(√4,000,000) = 2000.

### Runtime evidence — measured, stable across 3 runs

**Exact command** (each transfer, driven through the pty harness as in §0.2):

```
transfer #1 (full):  kitten transfer                   <4MB-src> <dest>
transfer #2 (delta): kitten transfer --transmit-deltas <4MB-src> <dest>   # after the 68-byte edit
```

Complete, unedited output:

```
=== Q6: DELTA EFFICIENCY — actual TTY wire bytes vs raw file bytes vs rsync literal Data bytes ===
file size = 4000000 bytes; in-place edit = 68 bytes at offset 2000000; block_size = floor(sqrt(4000000)) = 2000

RUN 1:
  RAW FILE BYTES ...................... first=4000000  second(after 67-byte edit)=4000000
  ACTUAL TTY WIRE BYTES (OSC 5113 frames, incl. framing+base64):
     transfer #1 (full)  =    5373074 B  across 980 data frames (base64 payload 5335634 chars)
     transfer #2 (delta) =       2993 B  across 1 data frames (base64 payload 2755 chars)
     reduction in actual TTY wire bytes = 1795.2x
  total raw client stream (incl. progress UI) : #1=5381967  #2=4133
  RSYNC LITERAL DATA BYTES (engine, v1->v2): delta_stream=2050 B  literal Data=2000 B  total_data_in_delta=2000  reconstruct_ok=True

RUN 2:
  RAW FILE BYTES ...................... first=4000000  second(after 68-byte edit)=4000000
  ACTUAL TTY WIRE BYTES (OSC 5113 frames, incl. framing+base64):
     transfer #1 (full)  =    5373074 B  across 980 data frames (base64 payload 5335634 chars)
     transfer #2 (delta) =       2993 B  across 1 data frames (base64 payload 2755 chars)
     reduction in actual TTY wire bytes = 1795.2x
  total raw client stream (incl. progress UI) : #1=5377405  #2=4133
  RSYNC LITERAL DATA BYTES (engine, v1->v2): delta_stream=2050 B  literal Data=2000 B  total_data_in_delta=2000  reconstruct_ok=True

RUN 3:
  RAW FILE BYTES ...................... first=4000000  second(after 68-byte edit)=4000000
  ACTUAL TTY WIRE BYTES (OSC 5113 frames, incl. framing+base64):
     transfer #1 (full)  =    5373074 B  across 980 data frames (base64 payload 5335634 chars)
     transfer #2 (delta) =       2993 B  across 1 data frames (base64 payload 2755 chars)
     reduction in actual TTY wire bytes = 1795.2x
  total raw client stream (incl. progress UI) : #1=5377397  #2=4133
  RSYNC LITERAL DATA BYTES (engine, v1->v2): delta_stream=2050 B  literal Data=2000 B  total_data_in_delta=2000  reconstruct_ok=True

=== STABILITY across 3 runs ===
actual TTY wire bytes, transfer #1 (full)  : [5373074, 5373074, 5373074]
actual TTY wire bytes, transfer #2 (delta) : [2993, 2993, 2993]
rsync literal Data bytes (total_data_in_delta): [2000, 2000, 2000]
reduction (wire #1 / wire #2)              : ['1795.2x', '1795.2x', '1795.2x']
```

The measured wire bytes are **identical across all three runs** (5,373,074 and 2,993), so
the value is stable. The only slightly variable number is the *total raw client stream*
including the progress-UI redraws (`#1 = 5,381,967 / 5,377,405 / 5,377,397`), which drifts
by a few kilobytes because the progress bar repaints a variable number of times; the
protocol-relevant `OSC 5113` wire bytes do not vary. The second transfer's raw stream is a
stable 4,133 bytes.

### Mechanism attribution (corroborated by the engine)

Driving the same 4 MB v1 → v2 pair directly through the rsync engine and inspecting the
delta confirms *why* transfer #2 is tiny: of the 4,000,000 bytes, only the **single
2000-byte block** containing the 68-byte edit travelled as literal `Data`
(`total_data_in_delta = 2000`), and applying that delta to v1 reconstructs v2 exactly
(`reconstruct_ok = True`). Every other block was referenced, not resent.

The block matching itself is performed in `Differ.next_op` (`kittens/transfer/rsync.pyi:42`),
which slides the rsync rolling **weak** checksum over the new file and, on a weak-hash hit,
confirms the match with the **XXH3-64 strong** hash before emitting a `Block`/`BlockRange`
reference; a miss emits a `Data` op. `Patcher.total_data_in_delta`
(`kittens/transfer/rsync.pyi:35`) is exactly the running total of those literal `Data`
bytes, and here it equals 2000 — which, after base64 expansion and `OSC 5113` framing,
becomes the 2,993 wire bytes of transfer #2. That is the source of the ≈ 1795× reduction in
actual TTY wire bytes.

---

## Coverage pass — every condition the questions imply

The governing rules treat every "e.g. / such as / including" example as a **required** item.
Each is exercised or observed below, with the evidence and citation.

### Directions — download *and* upload

The `--direction` option (`kittens/transfer/main.py:70`, choices `upload, download, send,
receive`) selects the direction; default is `download`. Both directions were driven through
the real kitten. Complete, unedited output:

```
--- default (download -> a 'send' session): kitten transfer <src> <dest> ---
first client->terminal command: id=181d52b1f;ac=send
--- --direction=receive (upload): kitten transfer --direction=receive <src> <dest> ---
first client->terminal command: id=181e43bb2;ac=receive;sz=1
full ordered commands:
  [0] id=181e43bb2;ac=receive;sz=1
  [1] id=181e43bb2;ac=file;fid=0;n=L3RtcC9raXR0eV90ZXN0X3RtcC9jb3ZfZDM0M3JoMjMvc3Jj
  [2] id=181e43bb2;ac=file;fid=1;n=L3RtcC9raXR0eV90ZXN0X3RtcC9jb3ZfZDM0M3JoMjMvc3Jj
  [3] id=181e43bb2;ac=finish
round-trip dest == src: True
```

The download (send) path opens with `ac=send`; the receive/upload path opens with
`ac=receive` (and advertises `sz=1`, the number of specs).

### Transmission modes — simple *and* rsync

`TransmissionType` enum (`kitty/file_transmission.py:198`) has members `simple` and `rsync`;
the field is `ttype` and serializes as the `tt` wire key (`kitty/file_transmission.py:257`).
The handler sets `needs_data_sent = self.ttype is not TransmissionType.simple`
(`kitty/file_transmission.py:462`) and `waiting_for_signature = ... ttype is
TransmissionType.rsync` (`kitty/file_transmission.py:653`). The **simple** mode is what
Q1/Q4 exercised (plain `data` chunks); the **rsync** mode is what Q5/Q6 exercised
(`--transmit-deltas`).

### Cancel path

`docs/file-transfer-protocol.rst:195` (*"Canceling a session"*): an `action=cancel`
produces `action=status … status=CANCELED`. The handler emits it via
`send_status_response(ErrorCode.CANCELED, ...)` (`kitty/file_transmission.py:936`; the
`ErrorCode` enum `OK STARTED CANCELED PROGRESS EINVAL EPERM EISDIR ENOENT` is at
`kitty/file_transmission.py:203`). Driven directly through the real handler
(`FileTransmission().handle_serialized_command(...)`); complete, unedited responses:

```
driver: FileTransmission().handle_serialized_command(receive); then handle_serialized_command(cancel)
   {'action': 'status', 'id': 'c1', 'status': 'OK'}
   {'action': 'status', 'id': 'c1', 'status': 'CANCELED'}
```

### Refusal / permission & file errors

Complete, unedited output (real handler):

```
driver: FileTransmission(allow=False).handle_serialized_command(receive, id=x, quiet=0)
EPERM responses (complete):
   {'action': 'status', 'id': 'x', 'status': 'EPERM:User refused the transfer'}
driver: handle_serialized_command(file, file_id=missing, name=XXX) on a non-existent path
ENOENT responses (complete):
   {'action': 'status', 'id': 'test', 'status': 'OK'}
   {'action': 'status', 'id': 'test', 'file_id': 'missing', 'status': 'ENOENT:Failed to read spec'}
   {'action': 'status', 'id': 'test', 'name': '/tmp/kitty_test_tmp/cov_home', 'status': 'OK'}
```

The `EPERM` message text originates at `kitty/file_transmission.py` (user-refusal path in
`handle_receive_confirmation`, see *Confirmation & bypass* below); the `ENOENT:Failed to
read spec` is the missing-file error (`TransmissionError` with `code='ENOENT'`).

### Quiet levels

`q=0` verbose, `q=1` errors only, `q=2` silent (keys table, `docs/file-transfer-protocol.rst`
rows at 565–574). Observed by repeating the EPERM refusal at each level; complete, unedited
output:

```
  quiet=0 -> responses: [{'action': 'status', 'id': 'x', 'status': 'EPERM:User refused the transfer'}]
  quiet=1 -> responses: [{'action': 'status', 'id': 'x', 'status': 'EPERM:User refused the transfer'}]
  quiet=2 -> responses: []
```

`q=0` and `q=1` still emit the error (an `EPERM` is an error, not an acknowledgement);
`q=2` suppresses **everything** — nothing is emitted.

### Compression — `zlib` (RFC 1950)

`docs/file-transfer-protocol.rst:487` specifies `compression=zlib` as RFC 1950 ZLIB deflate.
Compression is only applied to files **larger than 4096 bytes** (`kittens/transfer/send.go`
gates it on `stat_result.Size() > 4096`), so a 13,500-byte compressible file was used with
the real kitten `--compress=always`. Complete, unedited output:

```
file size: 13500 (> 4096 so compression_capable)
driver: kitten transfer --compress=always <src> <dest>
file command(s) on the wire (complete):
   id=18207c433;ac=file;zip=zlib;mod=1783487613511549842;fid=1;prm=420;n=L3RtcC9raXR0eV90ZXN0X3RtcC9jb3Z6X3NmeWZ3MnA0L2Rlc3Q
compressed data bytes on wire (decoded): 111 vs uncompressed file size: 13500
dest == src (handler decompressed via ZlibDecompressor): True
```

The file command carries `zip=zlib`, only **111 bytes** of compressed data travelled for the
13,500-byte (highly compressible) file, and the handler decompressed it via `ZlibDecompressor`
so the destination is byte-identical to the source.

### Confirmation & bypass

Transfers require **interactive user confirmation by default**. The actual confirmation
prompt is raised by `boss.confirm(...)` inside **`start_send`**
(`kitty/file_transmission.py:1161`, "The remote machine wants to read some files from this
computer. Do you want to allow the transfer?") for a send session, and inside
**`start_receive`** (`kitty/file_transmission.py:1191`, "The remote machine wants to send
some files to this computer…") for a receive session. The user's answer is processed by
**`handle_receive_confirmation`** (`kitty/file_transmission.py:1174`) and
**`handle_send_confirmation`** (`kitty/file_transmission.py:1204`) — accepting sends
`ErrorCode.OK`, refusing sends `ErrorCode.EPERM` (the `EPERM:User refused the transfer`
seen above).

The password-**bypass** option is looked up separately, at the top of the session objects:
`byp = get_options().file_transfer_confirmation_bypass` in `ActiveSend.__init__`
(`kitty/file_transmission.py:592`, inside the `__init__` at `:588`) and
`ActiveReceive.__init__` (`kitty/file_transmission.py:721`, inside the `__init__` at `:716`);
if it matches, `bypass_ok` is set and `start_send`/`start_receive` short-circuit the prompt.

The harness's `PtyFileTransmission(allow=True)` is the **canonical stand-in for the user
clicking "allow"** — it exercises the real permission-check code path
(`handle_receive_confirmation` / `handle_send_confirmation`); only the GPU confirmation
dialog is not rendered (disclosed in §0.2). The runs used the **default** (confirmation
enabled, answered "allow") and did **not** use the password-bypass scheme. For completeness,
the bypass scheme itself is `check_bypass()`, keyed by `KITTY_PUBLIC_KEY` or a
`sha256(session_id + ";" + password)` pre-shared hash
(`docs/file-transfer-protocol.rst:507`, *"Bypassing explicit user authorization"*).

### Resume / interrupt

Covered in full under [Q5](#q5--resumption-behavior) — the real `kitten transfer` child was
`SIGKILL`ed mid-transfer and restarted with `--transmit-deltas`; before / during / after
states were all observed.

### Two disclosed caveats

1. **No GPU window** was launched (headless container). No value in this document depends on
   the live window; the terminal-side logic was exercised through the real
   `kitty/file_transmission.py` handler via the pty harness (§0.2).
2. **The SSH transport hop was not exercised** — there is **no `sshd`** in this container, so
   a real `kitten ssh` → remote `kitten transfer` round-trip **could not be run** and any
   value depending on it is **non-canonical / not observed** (§0.2). The claim that the
   `OSC 5113` frames are **byte-identical over SSH** is **`INFERRED`** from SSH being a
   transparent byte pipe and from the `ssh` kitten merely *delivering* the `transfer` kitten
   to the remote (`kittens/ssh/main.py:167`); it was not observed here. Every other reported
   value comes from the real local client → C parser → handler → disk path, which **was**
   exercised byte-for-byte.

### Question ↔ location map

| Item | Answered in |
|------|-------------|
| Q1 end-to-end trace | [Q1](#q1--end-to-end-trace-of-file-data) |
| Q2 handshake & escape sequences | [Q2](#q2--handshake--the-exact-escape-sequences) |
| Q3 delta transfer & data structures | [Q3](#q3--rsync-style-delta-transfer--the-data-structures) |
| Q4 encoding & demultiplexing | [Q4](#q4--chunk-encoding--demultiplexing) |
| Q5 resumption behavior | [Q5](#q5--resumption-behavior) |
| Q6 empirical delta efficiency | [Q6](#q6--empirical-delta-efficiency-measured-2-runs) |
| download / upload directions | Coverage pass → *Directions* |
| simple / rsync modes | Coverage pass → *Transmission modes* |
| cancel / EPERM / ENOENT | Coverage pass → *Cancel* / *Refusal* |
| quiet levels | Coverage pass → *Quiet levels* |
| compression=zlib (RFC 1950) | Coverage pass → *Compression* |
| confirmation & bypass | Coverage pass → *Confirmation & bypass* |
| resume / interrupt | [Q5](#q5--resumption-behavior) |

---

## Appendix A — verified citation quick-reference

Every citation below was verified against HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.

**Control code / parser (C):**

- `kitty/control-codes.h:233` — `#define FILE_TRANSFER_CODE 5113`
- `kitty/data-types.c:596` — `PyModule_AddIntMacro(m, FILE_TRANSFER_CODE);`
- `gen/go_code.py:597` — generates `const FileTransferCode int = 5113`
- `kitty/vt-parser.c:547` — `case FILE_TRANSFER_CODE:` → `DISPATCH_OSC(file_transmission)`

**Send path (`kittens/transfer/send.go`):**

- `:363-364` — `SendManager.start_transfer()` returns
  `FileTransmissionCommand{Action: Action_send, Bypass: self.bypass}.Serialize()`
- `:384` — `prefix = fmt.Sprintf("\x1b]%d;id=%s;", kitty.FileTransferCode, self.request_id)`;
  `:385` — `suffix = "\x1b\\"`
- `:646-649` — `SendHandler.send_payload()` writes prefix + payload + suffix
- `:1198` — `func send_loop(...)`; `:1223-1232` — the **send session's** `OnEscapeCode`
  callback (`ftc_code := strconv.Itoa(kitty.FileTransferCode)` … `return
  handler.on_file_transfer_response(ftc)`) that routes terminal→client responses (Q1 step 7)
- `:38 / :40 / :41` — file states `WAITING_FOR_START` / `TRANSMITTING` / `FINISHED`
- `:280` — `SEND_WAITING_FOR_PERMISSION`; `:800` granted; `:802` denied

**FTC wire struct (`kittens/transfer/ftc.go`):**

- `:120` — `type FileTransmissionCommand struct` (field → short-key mapping in the
  [Q2 table](#q2--handshake--the-exact-escape-sequences))
- `:129` — `Bypass string json:"pw,omitempty" encoding:"base64"`; `:174-183` — serializer
  base64-encodes `reflect.String` fields tagged `encoding:"base64"`
  (**spec `docs/file-transfer-protocol.rst:567` calls `pw` a `safe_string`; the
  implementation base64-encodes it — documented divergence in the Q2 table**)

**Receive path (`kittens/transfer/receive.go`):**

- `:125` — `remote_file` struct; `:127` `expect_diff`; `:128` `patcher`;
  `:422-425` resume setup (`expect_diff=true`, `NewPatcher`, `CreateSignatureIterator`)
- `:200` — `remote_file.write_data()` (chunk reassembly)
- `:1120-1121` — the **receive direction's** `OnEscapeCode` callback (structurally identical
  to the send session's `send.go:1223-1232`)

**Terminal handler (`kitty/file_transmission.py`):**

- `:122` `EPERM` (`No permission to read spec`); `:198` `TransmissionType`; `:203` `ErrorCode`;
  `:257` `ttype` (`tt`); `:260` `bypass` field (`pw`, base64);
  `:450` `existing_stat`; `:462` `needs_data_sent`; `:470` `PatchFile`;
  `:653` `waiting_for_signature`; `:936` `ErrorCode.CANCELED`;
  `:1081` `transmit_rsync_signature` def (called `:1038` / `:1090` / `:1129`)
- **Confirmation prompts:** `:1161` `start_send` (raises the "read files" confirm);
  `:1174` `handle_receive_confirmation` (accepted→OK / refused→EPERM);
  `:1191` `start_receive` (raises the "send files" confirm);
  `:1204` `handle_send_confirmation`
- **Bypass-option lookup only:** `:588` `ActiveSend.__init__` (`:592`
  `file_transfer_confirmation_bypass`); `:716` `ActiveReceive.__init__` (`:721` same option)

**Rsync engine:**

- `kittens/transfer/rsync.pyi` — `RsyncError`@6, `Hasher`@9, `Patcher`@24,
  **`total_data_in_delta`@35**, `Differ`@38, **`next_op`@42**, `parse_ftc`@45
  *(observed lines — the AAP body's L30-32 / L38-40 were incorrect)*
- `kittens/transfer/algorithm.c:13` — `#include <xxhash.h>`
- `tools/rsync/api.go` — `Api`@47, `Differ`@56, `Patcher`@61,
  `CreateSignatureIterator`@195, `CreateDelta`@230

**SSH kitten / build / docs:**

- `kittens/ssh/main.py:164` — `remote_kitty` option; `:167` — help text: the transfer file
  kitten is delivered to the remote
- `setup.py:175` — `action: str = 'build'` (default action); `:1829` —
  `default=Options.action`; `:2115` — `if args.action == 'build':`;
  `go.mod:3` — `go 1.22`; `pyproject.toml:2` — `requires-python = ">=3.8"`
- `docs/file-transfer-protocol.rst` — Overall design @16, Canceling a session @195,
  Transmitting binary deltas @338, The format of signatures and deltas @409, Compression @487,
  Bypassing explicit user authorization @507, Encoding of transfer commands as escape codes @540
  (keys table rows @565-574; `bypass`/`pw` row @567)
- `docs/kittens/transfer.rst` — `--permissions-bypass` @68, Delta transfers @77,
  `--transmit-deltas` @81

---

## Appendix B — read-only guarantee (git status proof)

All observation scripts and sample files were created under `/tmp` (outside the repository
tree) and removed after use. The only path added to the repository is this document.
Verified at the repository root **after committing this document**:

```
$ git status --porcelain
    (no output — the working tree is clean)

$ git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1..HEAD
A	blitzy/documentation/kitty_815df1e210e0.md

$ find blitzy -type f
blitzy/documentation/kitty_815df1e210e0.md
```

The baseline `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` is the pinned HEAD this document
cites. The diff against it shows a **single added file** — this answer document — and the
working tree is clean (empty `git status --porcelain`). No existing source, test, doc,
build, or configuration file was modified, added, or deleted — the tracked tree is
byte-for-byte unchanged except for this answer document. (Build artifacts such as
`kitty/launcher/*` and `kitty/*.so` are `.gitignore`d and therefore do not appear.) This
confirms the user's read-only constraint was honored: *"Do not modify any source files.
Temporary test files are fine but clean them up afterwards."*
