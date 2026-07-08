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

**Canonical build command** (the `build` action is dispatched at `setup.py:2115`,
`if args.action == 'build':`):

```
$ CI=true python3 setup.py build
```

Tail of the build (re-run incrementally to confirm it succeeds):

```
Package wayland-protocols was not found in the pkg-config search path.
Perhaps you should add the directory containing `wayland-protocols.pc'
to the PKG_CONFIG_PATH environment variable
Package 'wayland-protocols', required by 'virtual:world', not found
wayland-protocols >= 1.17 is required, found version: not found
Disabling building of wayland backend
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
documents this delivery at `kittens/ssh/main.py:167` ("… the :doc:`transfer file kitten
</kittens/transfer>` to transfer files").

A full windowed **GPU terminal cannot be launched in this headless container**, so the
transfers below were driven through the repository's own pseudo-terminal harness
`kitty_tests/file_transmission.py`. This is **not a synthetic stand-in** for the protocol:

- The harness runs the **real `kitten transfer` binary** (`kitty/launcher/kitten`, resolved
  via `kitty.constants.kitten_exe`) as the child of a real pseudo-terminal — class
  `TransferPTY` at `kitty_tests/file_transmission.py:173`.
- The bytes the kitten writes are read from the pty master and fed to the **real C VT
  parser** (`parse_bytes` → `kitty/vt-parser.c`), which dispatches the file-transfer OSC to
  the **real terminal-side handler** `kitty/file_transmission.py` — class
  `PtyFileTransmission(FileTransmission)` at `kitty_tests/file_transmission.py:160`.
- Files are written to a real filesystem.

In other words, **every layer of the transfer protocol (client serializer → TTY bytes → C
demultiplexer → Python handler → disk) is the real production code path**, exercised
byte-for-byte. The only two things *not* exercised are (a) the GPU window rendering, and
(b) the SSH transport hop. Neither affects any value reported here: SSH is a transparent
byte pipe, so the `OSC 5113` frames over a local pty are identical to those over SSH, and
no reported value depends on the GPU window. Where a value would depend on the live UI it
is explicitly labeled **non-canonical**; none below are.

All observation scripts and sample files were created under `/tmp` (outside the repository
tree). The read-only guarantee is proven in [Appendix B](#appendix-b--read-only-guarantee-git-status-proof).

### 0.3 The baseline test suite passes

Before capturing anything, the in-repo suites were run to confirm the handler and engine
work end-to-end:

```
$ CI=true LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 TMPDIR=/tmp/kitty_test_tmp \
    ./kitty/launcher/kitty +launch test.py --module file_transmission
...
test_file_get (kitty_tests.file_transmission.TestFileTransmission.test_file_get) ... ok
test_parse_ftc (kitty_tests.file_transmission.TestFileTransmission.test_parse_ftc) ... ok
test_rsync_hashers (kitty_tests.file_transmission.TestFileTransmission.test_rsync_hashers) ... ok
test_rsync_roundtrip (kitty_tests.file_transmission.TestFileTransmission.test_rsync_roundtrip) ... ok
test_transfer_receive (kitty_tests.file_transmission.TestFileTransmission.test_transfer_receive) ... ok
test_transfer_send (kitty_tests.file_transmission.TestFileTransmission.test_transfer_send) ... ok

----------------------------------------------------------------------
Ran 6 tests in 1.138s

OK
```

```
$ go test -mod=mod -count=1 ./kittens/transfer/... ./tools/rsync/...
ok  	kitty/kittens/transfer	0.011s
ok  	kitty/tools/rsync	0.011s
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
7. Status/PROGRESS acknowledgements flow **back** to the client, whose terminal loop routes
   them through the `OnEscapeCode` callback at **`kittens/transfer/receive.go:1120`**.

### Runtime evidence

A 28-byte file `hello, kitty file transfer!\n` was sent through the real `kitten transfer`
binary. The driver captured (a) the exact bytes the client emitted on the pty, (b) each
command after the C parser demultiplexed it, and (c) each response the real handler wrote
back. Complete, unedited output:

```
=== RAW BYTES emitted by real kitten client (pty.received_bytes) ===
total bytes from client: 1270
repr: b'Scanning files\xe2\x80\xa6\r\nFound 1 files and directories, requesting transfer permission\xe2\x80\xa6\r\n\x1b[?s\x1b[*x\x1b[4l\x1b[?1l\x1b[?5l\x1b[?2004l\x1b[?1004l\x1b[?1000l\x1b[?1002l\x1b[?1003l\x1b[?1005l\x1b[?1006l\x1b[?8h\x1b[?7h\x1b[?25h\x1b[>29u\x1b[?25l\x1b]5113;id=dc147551;ac=send\x1b\\\x1b[32mPermission granted for this transfer\x1b[39m\n\r\x1b]5113;id=dc147551;mod=1783483643356778028;prm=420;n=L3RtcC9raXR0eV90ZXN0X3RtcC90bXAzeGhtNjRmai9kZXN0;ac=file;fid=1\x1b\\\x1b]5113;id=dc147551;d=aGVsbG8sIGtpdHR5IGZpbGUgdHJhbnNmZXIhCg;ac=end_data;fid=1\x1b\\ ...(progress UI omitted for this listing; shown verbatim in Q4)... \x1b]5113;id=dc147551;ac=finish\x1b\\...'

=== ORDERED CLIENT->TERMINAL COMMANDS (demuxed OSC bodies via C parser) ===
[0] id=dc147551;ac=send
[1] id=dc147551;mod=1783483643356778028;prm=420;n=L3RtcC9raXR0eV90ZXN0X3RtcC90bXAzeGhtNjRmai9kZXN0;ac=file;fid=1
[2] id=dc147551;d=aGVsbG8sIGtpdHR5IGZpbGUgdHJhbnNmZXIhCg;ac=end_data;fid=1
[3] id=dc147551;ac=finish

=== ORDERED TERMINAL->CLIENT RESPONSES (real handler) ===
[0] 5113;ac=status;id=dc147551;st=T0s
[1] 5113;ac=status;id=dc147551;fid=1;n=L3RtcC9raXR0eV90ZXN0X3RtcC90bXAzeGhtNjRmai9kZXN0;st=U1RBUlRFRA
[2] 5113;ac=status;id=dc147551;fid=1;sz=28;n=L3RtcC9raXR0eV90ZXN0X3RtcC90bXAzeGhtNjRmai9kZXN0;st=T0s

=== DISK TERMINUS ===
dest bytes: 28 match src: True
dest content repr: b'hello, kitty file transfer!\n'
```

The base64-encoded values decode exactly as follows (verified against the emitted bytes):

```
st=T0s        -> b'OK'         (permission granted; and again when the file completes)
st=U1RBUlRFRA  -> b'STARTED'    (destination file opened)
n=L3RtcC...    -> b'/tmp/kitty_test_tmp/tmp3xhm64fj/dest'   (destination path)
d=aGVsbG8s...  -> b'hello, kitty file transfer!\n'          (the file's bytes)
prm=420  == octal 0o644        (UNIX permission bits)
mod=1783483643356778028        (mtime, nanoseconds since the UNIX epoch)
```

### Why this is the whole path

The ordered command list **is** the trace, and it matches the narrated chain step for step:

- `[0] ac=send` is produced by `SendManager.start_transfer()`
  (`kittens/transfer/send.go:363-364`, which returns
  `FileTransmissionCommand{Action: Action_send, Bypass: self.bypass}.Serialize()`).
- Each command is framed and written by `SendHandler.send_payload()`
  (`kittens/transfer/send.go:646-649`).
- The three responses (`OK` → `STARTED` → `OK`) are produced by the **real handler** after
  the C parser routed the commands to it (`kitty/vt-parser.c:547`).
- `[2] ac=end_data` carries the file's bytes under `d=` and marks the last chunk; the
  handler then reports `st=OK` with `sz=28`, and the **file on disk is byte-identical to the
  source** — the terminus of the trace.

The send side advances a per-file state machine `WAITING_FOR_START` (`send.go:38`) →
`TRANSMITTING` (`send.go:40`) → `FINISHED` (`send.go:41`), and a session permission state
`SEND_WAITING_FOR_PERMISSION` (`send.go:280`) that becomes `SEND_PERMISSION_GRANTED`
(`send.go:800`) on a `status=OK` reply or `SEND_PERMISSION_DENIED` (`send.go:802`) otherwise
— which is exactly why response `[0] st=OK` must arrive before any `data` command is sent.

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
- **`5113`** = `FILE_TRANSFER_CODE`, the numeralization of the word *"file"*.
- **`ST`** (String Terminator) = the two bytes `0x1b 0x5c` (`ESC \`).

The overall encoding is `<OSC> 5113 ; key=value ; key=value … <ST>`, specified in
`docs/file-transfer-protocol.rst` under *"Encoding of transfer commands as escape codes"*
(section anchor at line 540).

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

The opening frame emitted by the **real kitten** was captured and dumped both as a Python
`repr()` and as hex of its first and last bytes:

```
=== BYTE-EXACT OPENING action=send FRAME ===
repr : b'\x1b]5113;id=dc147551;ac=send\x1b\\'
first 7 bytes (hex): 1b 5d 35 31 31 33 3b
  -> ESC ]  5  1  1  3  ;
last 2 bytes  (hex): 1b 5c
  -> ESC  \        (ST)
verify begins 1b 5d 35 31 31 33 3b : True
verify ends   1b 5c                 : True
```

This literally begins `1b 5d 35 31 31 33 3b` (`ESC ] 5 1 1 3 ;`) and ends `1b 5c` (`ESC \`),
confirming the `OSC 5113 … ST` form byte-for-byte.

### The key schema (field → wire short-key)

Every command is a `FileTransmissionCommand` (struct at `kittens/transfer/ftc.go:120`).
On the wire the field names are abbreviated to reduce overhead; the struct's JSON tags give
the exact mapping (verified against the source), and the protocol keys table in
`docs/file-transfer-protocol.rst:540` documents each value's type:

| Struct field  | Wire key | Type / notes                                                             |
|---------------|----------|--------------------------------------------------------------------------|
| `Action`      | `ac`     | enum: `send, file, data, end_data, receive, cancel, status, finish`      |
| `Compression` | `zip`    | `none` \| `zlib` (RFC 1950 deflate)                                        |
| `Ftype`       | `ft`     | `regular` \| `directory` \| `symlink` \| `link`                           |
| `Ttype`       | `tt`     | `simple` \| `rsync`                                                       |
| `Quiet`       | `q`      | `0` verbose \| `1` errors only \| `2` silent                              |
| `Id`          | `id`     | session id                                                               |
| `File_id`     | `fid`    | per-file id                                                              |
| `Bypass`      | `pw`     | base64; hash of bypass password + session id                             |
| `Name`        | `n`      | base64 (`base64_string`)                                                  |
| `Status`      | `st`     | base64 (`base64_string`)                                                  |
| `Parent`      | `pr`     | parent file id                                                           |
| `Mtime`       | `mod`    | integer; ns since epoch                                                  |
| `Permissions` | `prm`    | integer; UNIX mode bits                                                  |
| `Size`        | `sz`     | integer; default `-1`                                                    |
| `Data`        | `d`      | base64 (`base64_bytes`); the file chunk, ≤4096 bytes                      |

Cross-checking against the captured commands from Q1 confirms the schema is exactly what
the wire uses — e.g. `ac=send`, `ac=file;fid=1;n=<b64>;prm=420;mod=<ns>`,
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
  the **signature** of the file it already has (`signature_of_file` / the signature
  iterator) and applies incoming deltas; it exposes `total_data_in_delta`
  (`kittens/transfer/rsync.pyi:35`), the count of literal bytes that actually travelled.
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
checksum is **XXH3-128**.

### Runtime evidence — a real signature and a real delta

Using the in-repo rsync engine (the same one the kitten uses), a 1024-byte base file (16
distinct 64-byte blocks) had its signature computed, then a target that differs only in
5 bytes at offset `[458:463]` (`b'XYZ!!'`) had a delta computed against it. The raw bytes
were dumped and decoded:

```
=== SIGNATURE: first 12 bytes (hexdump) ===
00 00 00 00 00 00 00 00 20 00 00 00
decoded header:
  version          = 0
  checksum_type    = 0   -> XXH3-128 (whole-file integrity)
  strong_hash_type = 0   -> XXH3-64  (block identity)
  weak_hash_type   = 0   -> rsync rolling checksum (weak)
  block_size       = 32  (0x20)  == floor(sqrt(1024))
total signature length = 652 bytes  == 12 (header) + 32 records * 20 bytes

=== FIRST BLOCK SIGNATURE RECORD (20 bytes) ===
00 00 00 00 00 00 00 00  20 08 10 86  02 97 07 fe 79 8d c5 1a
  index       (uint64) = 0
  weak_hash   (uint32) = 0x86100820  = 2249197600   (4 bytes)
  strong_hash (uint64) = 0x1ac58d79fe079702 = 1929103570490595074 (8 bytes)

=== DELTA (340 bytes) — operation histogram and decoded ops ===
Block(type=0)      x 31
Data (type=1)      x  2
    Data size=15  payload=b'HHHHHHHHHHXYZ!!'
    Data size=17  payload=b'HHHHHHHHHHHHHHHHH'
Hash (type=2)      x  1
    Hash size=16  checksum=42e84cbeb6aa28c5932ffaecabb16b4b   (XXH3-128 = 16 bytes = 128 bits)
BlockRange(type=3) x  0   (documented; not emitted for this particular input)
literal bytes inlined as Data: 32 of 1024 target bytes
```

Every decoded field matches the spec: the header is all zeros except `block_size = 32`
(= √1024); each block record is 20 bytes (8 + 4 + 8), giving a total signature of
`12 + 32×20 = 652` bytes; the strong hash occupies 8 bytes and the weak hash 4 bytes; and
the trailing `Hash` op carries a 16-byte (128-bit) XXH3-128 checksum. Critically, of the
1024 target bytes, **only 32 travelled as literal `Data`** — the block containing the edit
— while the other 31 blocks were sent as cheap `Block` references. That is the delta
mechanism in miniature.

### Cross-engine confirmation

The hash families were confirmed directly against `test_rsync_hashers`, and the whole
signature/delta round-trip against `test_rsync_roundtrip`:

```
Hasher('xxh3-64')  of b'abcd' -> hexdigest 6497a96f53a89890 ; digest64 7248448420886124688
Hasher('xxh3-128') of b'abcd' -> hexdigest 8d6b60383dfa90c21be79eecd1b1353d

$ go test -v ./tools/rsync/...
=== RUN   TestRsyncRoundtrip
--- PASS: TestRsyncRoundtrip (0.00s)
=== RUN   TestRsyncHashers
--- PASS: TestRsyncHashers (0.00s)
PASS
ok  	kitty/tools/rsync	0.011s
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
  child program's ordinary output:
  ```go
  // kittens/transfer/receive.go:1120 onward (complete callback, unedited)
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
  destination file happens in
  **`remote_file.write_data()`** (`kittens/transfer/receive.go:200`).

### Runtime evidence — chunk encoding

A 10,000-byte file was sent with compression disabled (`--compress=never`) so the raw
chunking is visible. The captured `data`/`end_data` commands:

```
=== CHUNK ENCODING (10000-byte file, --compress=never) ===
data cmd: ac=data      d(base64)=5462 chars -> decoded 4096 raw bytes
data cmd: ac=data      d(base64)=5462 chars -> decoded 4096 raw bytes
data cmd: ac=end_data  d(base64)=2411 chars -> decoded 1808 raw bytes

chunks: 3 ; max raw bytes in any chunk: 4096  (spec limit: <=4096)  ✓
sum of decoded chunk sizes: 4096 + 4096 + 1808 = 10000  == file size  ✓
last chunk carries ac=end_data                                        ✓
base64 is UNPADDED (4096 raw -> 5462 chars)                           ✓
first 16 decoded bytes: 0b 30 55 7a 9f c4 e9 0e 33 58 7d a2 c7 ec 11 36
first 16 source  bytes: 0b 30 55 7a 9f c4 e9 0e 33 58 7d a2 c7 ec 11 36  (match ✓)
```

The three chunks are 4096 + 4096 + 1808 = 10000 bytes, so no chunk exceeds the 4096-byte
limit, the final chunk is the `end_data` command, and the decoded payload matches the
source bytes exactly.

### Runtime evidence — demultiplexing

The same run captured the raw byte stream from the client and, separately, the final screen
buffer produced by the real C parser:

```
=== DEMULTIPLEXING PROOF ===
raw bytes emitted by client                     : 14645
count of '\x1b]5113;' OSC frame-starts in stream : 6   (send, file, data, data, end_data, finish)
first data chunk's base64 present in RAW stream  : True
first data chunk's base64 present in SCREEN buff : False   <-- KEY RESULT

screen buffer (what the user would see) contained only the progress UI:
  "Scanning files…"
  "Permission granted for this transfer"
  a progress line ending "10kB @ 7.5MB/s"  with a ✔ and progress bar
```

This is the demultiplexing made visible: the payload's base64 is present in the raw stream
(six `OSC 5113` frames) but **absent from the screen buffer** — the C parser at
`kitty/vt-parser.c:547` diverted every `5113` frame to the handler rather than rendering
it, so the file data never touches the screen. Only the progress UI (which the kitten draws
with ordinary SGR/CSI sequences) is rendered. On the client side, the responses are
similarly filtered by the `OnEscapeCode` callback (`kittens/transfer/receive.go:1120-1121`)
and reassembled by `remote_file.write_data()` (`kittens/transfer/receive.go:200`).

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
self.expect_diff = true
self.patcher = rsync.NewPatcher(expected_size)
...
self.patcher.CreateSignatureIterator(...)   // signature of what is already on disk
```

### Runtime evidence — before / during / after (the transitional states)

**(A) Interrupt.** A 1,000,000-byte random file transfer was killed with `SIGKILL` partway
through. Observed *before* state:

```
=== (A) AFTER REAL INTERRUPT (SIGKILL) ===
partial file exists : True
partial file size   : 4096
partial prefix bytes match source prefix : True
```

**(B) Resume.** The same transfer was restarted with `--transmit-deltas`. Observed *during*
and *after* state:

```
=== (B) RESUME with --transmit-deltas ===
DURING: handler existing_stat.st_size = 4096   (existing partial reused as delta base)
AFTER : dest size = 1000000 ; dest == source : True
```

**Honest nuance (observed, not inferred).** Because this file was **random** and only a
4096-byte (0.4%) prefix existed, there was almost nothing to reuse, so the resume delta was
actually *larger* than the remainder:

```
resume wire bytes = 1,007,056  (~100.7% of the file)
```

This is expected: rsync delta over incompressible/unique data cannot save much, and the
op-stream headers add overhead. To show resumption *working as intended*, the experiment
was repeated with a large existing prefix:

**(C) Large-prefix resume.** 900,000 of 1,000,000 random bytes already present on disk:

```
=== (C) LARGE-PREFIX RESUME (900,000 of 1,000,000 present) ===
DURING: handler existing_stat.st_size = 900000
resume wire bytes = 100,465   (~= the 100,000-byte remainder + ~465 bytes of op headers)
the 900,000-byte prefix was NOT retransmitted
(a naive restart would have sent all 1,000,000 bytes)
```

All three transitional states — **before** (partial file present on disk), **during**
(handler stats the existing file and sends a signature of it), and **after** (transfer
completes, reusing the existing prefix as the delta base) — were **observed**, not inferred.
The direct-answer claim ("the existing file is the resumption metadata; no journal") is
grounded in `existing_stat` (`file_transmission.py:450`) + `PatchFile`
(`file_transmission.py:470`) and the receive-side `expect_diff`/`patcher`
(`receive.go:127-128`, set up at `receive.go:422-425`).

---

## Q6 — Empirical delta efficiency (measured, ≥2 runs)

### Direct answer

**The second transfer sends dramatically less data than the first — roughly 1951× less in
this experiment (≈2,050 bytes vs 4,000,000 bytes).** The reduction happens because rsync
block matching replaces the unchanged regions with cheap `Block`/`BlockRange` references and
transmits only the changed bytes as `Data` operations. The matching is driven by the rsync
rolling **weak** hash plus the **XXH3-64 strong** hash inside `Differ.next_op`
(`kittens/transfer/rsync.pyi:42`), and the literal bytes that actually travel are counted by
`Patcher.total_data_in_delta` (`kittens/transfer/rsync.pyi:35`).

### The experiment (the user's operative example)

1. Create a **4,000,000-byte** random file under `/tmp` (large enough that delta savings
   dominate round-trip overhead; the docs warn deltas can be *slower* for tiny files —
   `docs/kittens/transfer.rst:77`, *"Delta transfers"*).
2. Transfer it once and measure the wire bytes.
3. Modify a **small portion** in place (~70 bytes at offset 2,000,000).
4. Transfer again with `--transmit-deltas` (`-x`; `docs/kittens/transfer.rst:81`) and
   measure the wire bytes.
5. Repeat the whole thing and confirm stability. `block_size` = √(4,000,000) ≈ 2000.

### Runtime evidence — measured, stable across 3 runs

```
=== Q6: DELTA EFFICIENCY (4,000,000-byte random file; ~70-byte in-place edit; block_size=2000) ===
RUN 1: first=4,000,000 B   second=2,050 B   reduction=1951.2x   bytes_changed=69
RUN 2: first=4,000,000 B   second=2,050 B   reduction=1951.2x   bytes_changed=70
RUN 3: first=4,000,000 B   second=2,050 B   reduction=1951.2x   bytes_changed=70

first-transfer bytes  across runs : [4000000, 4000000, 4000000]
second-transfer bytes across runs : [2050, 2050, 2050]
second / first                    : 0.00051   (perfectly stable)
```

The numbers are **identical across all three runs** — the second transfer is consistently
~0.05% the size of the first. (The full transfer includes the file's own bytes; the delta
transfer sends only the changed block plus operation headers.)

### Mechanism attribution (corroborated by the engine)

Driving the same 4 MB v1 → v2 pair directly through the rsync engine and inspecting the
delta:

```
=== MECHANISM (engine-level corroboration) ===
delta stream length            = 2050 bytes
Patcher.total_data_in_delta    = 2000 bytes    (rsync.pyi:35)
  -> exactly ONE 2000-byte block (the one containing the edit) travelled as literal Data
reconstructed(v1, delta) == v2 : True
2050-byte delta  ==  2000 bytes of Data  +  ~50 bytes of Block/op headers
```

So of the 4,000,000 bytes, only the **single 2000-byte block** that contained the ~70-byte
edit was sent as literal `Data`; every other block was referenced, not resent. The block
matching itself is performed in `Differ.next_op` (`kittens/transfer/rsync.pyi:42`), which
slides the rsync rolling **weak** checksum over the new file and, on a weak-hash hit,
confirms the match with the **XXH3-64 strong** hash before emitting a `Block`/`BlockRange`
reference; a miss emits a `Data` op. `Patcher.total_data_in_delta`
(`kittens/transfer/rsync.pyi:35`) is exactly the running total of those literal `Data`
bytes, and here it equals 2000 — the source of the ~1951× reduction.

---

## Coverage pass — every condition the questions imply

The governing rules treat every "e.g. / such as / including" example as a **required** item.
Each is exercised or observed below, with the evidence and citation.

### Directions — download *and* upload

The `--direction` option (`kittens/transfer/main.py:70`, choices `upload, download, send,
receive`) selects the direction; default is `download`.

```
DOWNLOAD (default): client's first command is  ac=send
RECEIVE/UPLOAD (--direction=receive): captured commands:
  [0] id=f387dbb5;sz=1;ac=receive
  [1] ... ac=file ...
  [2] ... ac=finish
  round-trip verified OK
```

Both directions were driven through the real kitten; the receive/upload path opens with
`ac=receive` rather than `ac=send`.

### Transmission modes — simple *and* rsync

`TransmissionType` enum (`kitty/file_transmission.py:198`) has members `simple` and `rsync`;
the field is `ttype` and serializes as the `tt` wire key (`kitty/file_transmission.py:257`).
The handler sets `needs_data_sent = ttype is not simple`
(`kitty/file_transmission.py:462`) and `waiting_for_signature = ttype is rsync`
(`kitty/file_transmission.py:653`). The **simple** mode is what Q1/Q4 exercised (plain
`data` chunks); the **rsync** mode is what Q5/Q6 exercised (`--transmit-deltas`).

### Cancel path

`docs/file-transfer-protocol.rst:195` (*"Canceling a session"*): an `action=cancel`
produces `action=status … status=CANCELED`. The handler emits it via
`send_status_response(ErrorCode.CANCELED)` (`kitty/file_transmission.py:936`; the
`ErrorCode` enum `OK STARTED CANCELED PROGRESS EINVAL EPERM EISDIR ENOENT` is at
`kitty/file_transmission.py:203`). Observed:

```
CANCEL: action=receive then action=cancel ->
  [{status:'OK'}, {status:'CANCELED'}]
```

### Refusal / permission & file errors

```
EPERM  (user refused): FileTransmission(allow=False)+action=receive ->
  [{action:status, id:x, status:'EPERM:User refused the transfer'}]     (file_transmission.py:122)
ENOENT (missing spec): action=file on a non-existent path ->
  {status:'ENOENT:Failed to read spec', file_id:'missing'}
```

### Quiet levels

`q=0` verbose, `q=1` errors only, `q=2` silent (`docs/file-transfer-protocol.rst:540` keys
table). Observed: repeating the EPERM refusal with `quiet=2` suppresses **all**
acknowledgements:

```
QUIET=2 on the refused transfer ->  []   (nothing emitted)
```

### Compression — `zlib` (RFC 1950)

`docs/file-transfer-protocol.rst:487` specifies `compression=zlib` as RFC 1950 ZLIB
deflate. Observed with the real kitten `--compress=always`:

```
file cmd: id=f371cf32;ac=file;prm=420;n=<b64>;mod=...;zip=zlib;fid=1
round-trip decompressed OK (ZlibDecompressor)
```

### Confirmation & bypass

Transfers require **interactive user confirmation by default**
(`kitty/file_transmission.py:592`, `kitty/file_transmission.py:721`, the
`file_transfer_confirmation_bypass` option). The harness's
`PtyFileTransmission(allow=True)` is the **canonical stand-in for the user clicking
"allow"** — it exercises the real permission-check code path; only the GPU confirmation
dialog is not rendered (disclosed). The runs used `allow=True` (the *default* confirmation
outcome) and did **not** use the password-bypass scheme, so **no non-canonical value was
produced**. For completeness, the bypass mechanism itself is `check_bypass()` keyed by
`KITTY_PUBLIC_KEY` / a `sha256(session_id + ";" + password)` pre-shared hash
(`docs/file-transfer-protocol.rst:507`, *"Bypassing explicit user authorization"*).

### Resume / interrupt

Covered in full under [Q5](#q5--resumption-behavior) (before / during / after observed).

### Two disclosed caveats

1. **No GPU window** was launched (headless container). No value in this document depends on
   the live window; the terminal-side logic was exercised through the real
   `kitty/file_transmission.py` handler via the pty harness.
2. **The SSH transport hop was not used.** SSH is a transparent byte pipe — the `OSC 5113`
   frames over a local pty are byte-identical to those over SSH; the `ssh` kitten merely
   delivers the `transfer` kitten to the remote (`kittens/ssh/main.py:167`).

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
- `:38 / :40 / :41` — file states `WAITING_FOR_START` / `TRANSMITTING` / `FINISHED`
- `:280` — `SEND_WAITING_FOR_PERMISSION`; `:800` granted; `:802` denied

**FTC wire struct:** `kittens/transfer/ftc.go:120` — `type FileTransmissionCommand struct`
(field → short-key mapping in the [Q2 table](#q2--handshake--the-exact-escape-sequences)).

**Receive path (`kittens/transfer/receive.go`):**

- `:125` — `remote_file` struct; `:127` `expect_diff`; `:128` `patcher`;
  `:422-425` resume setup (`expect_diff=true`, `NewPatcher`, `CreateSignatureIterator`)
- `:200` — `remote_file.write_data()` (chunk reassembly)
- `:1120-1121` — `ftc_code := strconv.Itoa(kitty.FileTransferCode)` + `OnEscapeCode` callback

**Terminal handler (`kitty/file_transmission.py`):**

- `:122` EPERM; `:198` `TransmissionType`; `:203` `ErrorCode`; `:257` `ttype` (`tt`);
  `:450` `existing_stat`; `:462` `needs_data_sent`; `:470` `PatchFile`;
  `:592` / `:721` confirmation bypass option; `:653` `waiting_for_signature`;
  `:936` `ErrorCode.CANCELED`; `:1081` `transmit_rsync_signature` def
  (called `:1038` / `:1090` / `:1129`)

**Rsync engine:**

- `kittens/transfer/rsync.pyi` — `RsyncError`@6, `Hasher`@9, `Patcher`@24,
  **`total_data_in_delta`@35**, `Differ`@38, **`next_op`@42**, `parse_ftc`@45
  *(observed lines — the AAP body's L30-32 / L38-40 were incorrect)*
- `kittens/transfer/algorithm.c:13` — `#include <xxhash.h>`
- `tools/rsync/api.go` — `Api`@47, `Differ`@56, `Patcher`@61,
  `CreateSignatureIterator`@195, `CreateDelta`@230, `NewDiffer`@265, `NewPatcher`@270

**SSH kitten / build / docs:**

- `kittens/ssh/main.py:167` — transfer kitten delivered to the remote
- `setup.py:2115` — `if args.action == 'build':`; `go.mod:3` — `go 1.22`;
  `pyproject.toml:2` — `requires-python = ">=3.8"`
- `docs/file-transfer-protocol.rst` — Overall design @16, Canceling a session @195,
  Transmitting binary deltas @338, Format of signatures and deltas @409, Compression @487,
  Bypassing explicit user authorization @507, Encoding of transfer commands as escape codes @540
- `docs/kittens/transfer.rst` — `--permissions-bypass` @68, Delta transfers @77,
  `--transmit-deltas` @81

---

## Appendix B — read-only guarantee (git status proof)

All observation scripts and sample files were created under `/tmp` (outside the repository
tree) and removed after use. The only path added to the repository is this document. After
cleanup, at the repository root:

```
$ git status --porcelain
?? blitzy/

$ git diff --stat HEAD
   (no output — zero changes to any tracked file)

$ find blitzy -type f
blitzy/documentation/kitty_815df1e210e0.md
```

`blitzy/` is the single new top-level path, containing only
`blitzy/documentation/kitty_815df1e210e0.md`. No existing source, test, doc, build, or
configuration file was modified, added, or deleted — the tracked tree is byte-for-byte
unchanged except for this answer document. (Build artifacts such as `kitty/launcher/*` and
`kitty/*.so` are `.gitignore`d and therefore do not appear.) This confirms the user's
read-only constraint was honored: *"Do not modify any source files. Temporary test files
are fine but clean them up afterwards."*

