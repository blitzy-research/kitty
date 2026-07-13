# kitty File Transfer Protocol (OSC 5113) over the SSH kitten — a runtime-evidenced trace

**Branch:** `kitty_815df1e210e0`  **HEAD commit:** `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
**Scope:** read-only code investigation. The only file written to the repository is this document.

---

## Summary

This document traces the complete journey of file data through kitty's **File Transfer Protocol** as it is
driven over an **SSH-kitten** connection, and answers seven objectives (Q1–Q7) **by name**, each grounded in a
specific `file:line` citation and, wherever possible, in **actual, unedited runtime output** captured from the
real code paths.

The end-to-end path is:

```
local file  ─►  kitten transfer (Go client)  ─►  OSC 5113 escape codes on the TTY  ─►
                terminal emulator VT parser (C, vt-parser.c)  ─►  screen.c  ─►  window.py  ─►
                file_transmission.py (Python server handler)  ─►  disk
```

The protocol is a **polyglot client/server system unified by one wire format (OSC 5113)**. The client
(`kitten transfer`) is **Go** (`kittens/transfer/*` + a pure-Go rsync engine at `tools/rsync/*`); the server
(the terminal emulator that receives OSC 5113 and writes files) is **C + Python** (`kitty/vt-parser.c`,
`kitty/screen.c`, `kitty/window.py`, `kitty/file_transmission.py`) with a C rsync engine at
`kittens/transfer/algorithm.c`. Two independent rsync implementations honor the **same** documented binary
contract.

Every finding below is tagged **[OBSERVED]** (captured at runtime) or **[INFERRED]** (derived from reading the
code after a genuine runtime attempt). Byte-sensitive results (the OSC bytes, the 12-byte rsync signature
header, delta operations, base64) are verified against the exact bytes the system emitted.

### A note on how the code was driven (headless accommodation — read this first)

The container has **no GUI / no `DISPLAY`** (the Wayland GLFW backend does not even build here; the build is
X11-only), so a literal GUI terminal window cannot be opened. kitty's **own test harness**
(`kitty_tests/file_transmission.py`) is built precisely for this situation, and I used the same canonical
mechanism it uses:

- The **client side is 100% canonical**: a real PTY is created and the **real `kitten transfer` binary**
  (`kitty/launcher/kitten`) is `fork`+`execvpe`'d into it — never a debug hook, remote-control command, or a
  synthetic `FileTransmissionCommand` stand-in.
- The bytes the client emits are captured **raw** from the PTY master (`PTY.received_bytes`) **and** fed to a
  **real kitty `Screen`** (the C object built from `kitty/screen.c` / `kitty/vt-parser.c`). On an OSC 5113
  sequence the real parser hits `case FILE_TRANSFER_CODE` and dispatches into the **genuine**
  `kitty/file_transmission.py` `FileTransmission.handle_serialized_command`.
- The **only** accommodation is that the `Screen` is driven programmatically instead of being painted in a GUI
  window. The parser, dispatch, handler, rsync engine, and disk writes are all the real ones. Where a step is
  a pure GUI-window concern (e.g. window painting) it is explicitly noted; nothing about the protocol logic is
  bypassed.

For **Q7** I additionally established a **genuine SSH connection through the real `kitten ssh` binary**
(loopback `ssh localhost`) and ran the real `kitten transfer` on the far side, proving the OSC 5113 stream
tunnels transparently over a real SSH TTY.

All temporary observation scripts and test files live **outside** the repository under `/tmp/ft_obs` and are
removed at the end; the repository is left byte-for-byte unchanged (proven in the final section).

---

## Environment & Build (Q6)

**[OBSERVED]** Toolchain (container-provided; unchanged by this task):

```
$ python3 --version   →  Python 3.13.7
$ go version          →  go version go1.24.4 linux/amd64   (satisfies go.mod `go 1.22`)
$ cc --version        →  cc (Ubuntu 15.2.0) 15.2.0
$ head -3 go.mod      →  module kitty / (blank) / go 1.22
```

**[OBSERVED]** Canonical build — the `Makefile` `all:` target delegates to `setup.py`; the canonical entry
point is `python3 setup.py`:

```
$ export LANG=C.UTF-8 LC_ALL=C.UTF-8 GOFLAGS=-mod=mod
$ python3 setup.py            # exit code 0
```

The build produces the launcher and the Go kitten binary:

```
$ ls -l kitty/launcher/kitty kitty/launcher/kitten
-rwxr-xr-x  kitty/launcher/kitty      (40384 bytes)
-rwxr-xr-x  kitty/launcher/kitten     (16429348 bytes, the Go binary)

$ ./kitty/launcher/kitty   --version   →  kitty 0.35.2 created by Kovid Goyal
$ ./kitty/launcher/kitten  --version   →  kitten 0.35.2 created by Kovid Goyal
```

**[OBSERVED]** The transfer/rsync **C** module requires **libxxhash**. `setup.py` line 986 links it:

```python
# setup.py:986
files('transfer', 'rsync', libraries=pkg_config('libxxhash', '--libs'),
      includes=pkg_config('libxxhash', '--cflags-only-I')),
```

Forcing a verbose rebuild of just that module (by touching `algorithm.c`'s mtime only — its md5
`df1463a61bb0f3a95eab8d4130564754` was identical before and after, so the repo is unmodified) captured the
exact compiler/link invocation:

```
COMPILE: gcc ... -std=c11 -pedantic-errors -Werror -O3 ... -c kittens/transfer/algorithm.c \
         -o build/rsync-kittens-transfer-algorithm.c.o
LINK:    gcc ... build/rsync-kittens-transfer-algorithm.c.o -lxxhash -ldl -lm ... \
         -o build/kittens/transfer/rsync.so
```

`kittens/transfer/algorithm.c` line 13 is `#include <xxhash.h>`, and the resulting `rsync.so` links
`libxxhash.so.0` (its dynamic symbol table references `XXH3_64bits`, `XXH3_128bits_digest/reset/update`,
`XXH128_canonicalFromHash`) — i.e. the **server-side C rsync uses XXH3-64/128**.

**[OBSERVED]** The protocol number reaches Python from the C definition:

```
$ python3 -c "from kitty.fast_data_types import FILE_TRANSFER_CODE; print(FILE_TRANSFER_CODE)"
5113
```

**[OBSERVED]** `kitten transfer --help` documents the delta/resume flag (evidence reused for Q4):

```
--transmit-deltas, -x
If a file on the receiving side already exists, use the rsync algorithm to update it to match the file on
the sending side, potentially saving lots of bandwidth and also automatically resuming partial transfers.
Note that this will actually degrade performance on fast links or with small files, so use with care.
```

---

## Canonical Entry Point (Q7) — the real `kitten ssh` → `kitten transfer` path

The documented canonical usage (`docs/kittens/transfer.rst:28-40`, `versionadded:: 0.30.0` at line 10) is:

```
<local>  $ kitten ssh my-remote-computer
<remote> $ kitten transfer some-file /path/on/local/computer
```

The SSH kitten automatically makes the transfer kitten available on the remote — `kittens/ssh/main.py:164`
declares the `remote_kitty` option and line 167 references the *transfer file kitten*; `docs/kittens/ssh.rst`
documents auto shell-integration + file transfer.

**[OBSERVED]** I set up `sshd` in the container (installed `openssh-server` as a *service*, generated host keys,
enabled passwordless root@localhost pubkey auth — none of this touches the repository) and ran the **real
`kitten ssh` binary**, which on the far side executed the **real `kitten transfer` binary**:

```
$ kitty/launcher/kitten ssh -o StrictHostKeyChecking=no -o BatchMode=yes -t -t localhost \
      '<abs-path>/kitty/launcher/kitten transfer <remote_src> <local_dest>'
```

Captured far-side output (unedited):

```
Found 1 files and directories, requesting transfer permission…
Permission granted for this transfer
 ✔ remote_src.bin 1.5kB
Connection to localhost closed.
TRANSFER_RC=0
```

Result: `kitten ssh` exit 0; `local_dest` created and **byte-identical** to the source (1472 bytes);
**4 OSC 5113 sequences** flowed over the SSH TTY, the first being
`\x1b]5113;id=11bcfc866;ac=send\x1b\\` (prefix hex `1b 5d 35 31 31 33`). This proves the OSC 5113 stream
**tunnels transparently over a real SSH TTY** — the protocol is transport-independent and rides the existing
terminal channel. I explicitly did **not** use `kitten __pytest__ ssh` (a debug hook kitty's own tests use)
because that would be non-canonical.

> Because the SSH transport is proven equivalent to the local TTY here, the byte-level captures in Q1–Q5 below
> are taken through the same real `kitten transfer` client over a local PTY (faster and identical on the wire),
> with the real C parser + real Python handler as the receiver.

---

## Q1 — Protocol handshake & OSC 5113 escape sequences

### The wire format

**[OBSERVED]** Commands are OSC escape codes of the form `<OSC> 5113 ; key=value ; ... <ST>`. The specification
is unambiguous at `docs/file-transfer-protocol.rst:543-552`:

> Transfer commands are encoded as `OSC` escape codes of the form `<OSC> 5113 ; key=value ; ... <ST>`. Here
> `OSC` is the bytes `0x1b 0x5d` and `ST` is the bytes `0x1b 0x5c` … The number `5113` … is the numeralization
> of the word `file`.

I verified these exact bytes against a real capture. The first command a `kitten transfer` emits is
`\x1b]5113;id=<id>;ac=send\x1b\\`; a hexdump of the prefix and terminator:

```
prefix bytes:  1b 5d 35 31 31 33   =  ESC ']' '5' '1' '1' '3'   =  "\x1b]5113"   (OSC + 5113)
ST terminator: 1b 5c               =  ESC '\'                     =  "\x1b\\"      (ST)
```

`5113` = "file" numeralized: it is defined **once** in C at `kitty/control-codes.h:233`
(`#define FILE_TRANSFER_CODE 5113`), exposed to Python at `kitty/data-types.c:596`
(`PyModule_AddIntMacro(m, FILE_TRANSFER_CODE)`), and mirrored to Go by code generation at
`gen/go_code.py:597` (`const FileTransferCode int = {FILE_TRANSFER_CODE}`, import at line 575). The Go client
uses that constant to build the OSC prefix.

### How the client initiates the handshake

**[OBSERVED]** The send-side prefix/suffix are built in `kittens/transfer/send.go`:

```go
// send.go:384-385
self.prefix = fmt.Sprintf("\x1b]%d;id=%s;", kitty.FileTransferCode, self.request_id)
self.suffix = "\x1b\\"
```

and written around `send.go:647-649` (`send_payload` writes `prefix` + payload + `suffix`). The payload itself
is built by `FileTransmissionCommand.Serialize()` at `kittens/transfer/ftc.go:163`.

### The handshake ordering

**[OBSERVED]** Driving a 20000-byte incompressible file (`--compress=never`) through the real client, I logged
**both** directions. The client → terminal action (`ac=`) sequence, in order, was:

```
send, file, data, data, data, data, end_data, finish
```

and the terminal → client status responses from the **genuine** server handler, in order, were:

```
status=OK          (accept)
status=STARTED     file_id=1
status=PROGRESS    size=4096
status=PROGRESS    size=8192
status=PROGRESS    size=12288
status=PROGRESS    size=16384
status=OK          size=20000     (after end_data)
```

This matches the documented flow `docs/file-transfer-protocol.rst:43-113` exactly:
`action=send → status=OK → action=file → STARTED → action=data → end_data → status=PROGRESS → status=OK →
finish`. The data is sent in chunks of **no larger than 4096 bytes** — the decoded chunk sizes were
`[4096, 4096, 4096, 4096, 3616]` (sum = 20000), confirming `docs/file-transfer-protocol.rst:78-81`.

The full set of 8 OSC 5113 commands captured (client → terminal), unedited (data payloads abbreviated as
`<base64>` where noted):

```
\x1b]5113;id=12024ce3b;ac=send\x1b\\
\x1b]5113;id=12024ce3b;ac=file;prm=420;n=L3RtcC9mdF9vYnMvdG1wcm9jY2hwcjYvcTFiX2RzdC5iaW4;fid=1;mod=1783961865632186366\x1b\\
\x1b]5113;id=12024ce3b;fid=1;ac=data;d=<base64>\x1b\\   (×4, last is ac=end_data)
\x1b]5113;id=12024ce3b;ac=finish\x1b\\
```

### Key abbreviations — every one addressed

**[OBSERVED]** Keys are abbreviated to reduce overhead (`docs/file-transfer-protocol.rst:561-574`), and the
`safe_string` charset `[0-9a-zA-Z_:./@-]` deliberately **excludes** the `;` separator
(`docs/file-transfer-protocol.rst:589-591`: “Note that the semi-colon is missing from this set.”). Each
abbreviation seen on my wire captures is correlated below:

| Full name | Abbrev | Value encoding | Observed on wire |
|-----------|--------|----------------|------------------|
| `action` | `ac` | enum: send, file, data, end_data, receive, cancel, status, finish | `ac=send/file/data/end_data/receive/finish` |
| `compression` | `zip` | enum: none, zlib | `zip=zlib` (compressible file; see Q3/Edge) |
| `file_type` | `ft` | enum: regular, directory, symlink, link | (regular files here; enum per spec) |
| `transmission_type` | `tt` | enum: simple, rsync | `tt=rsync` (with `-x`); absent ⇒ simple |
| `id` | `id` | safe_string (session id) | `id=12024ce3b` |
| `file_id` | `fid` | safe_string | `fid=1` |
| `bypass` | `pw` | base64_string | (not used in these runs) |
| `quiet` | `q` | integer | (default) |
| `mtime` | `mod` | integer (nanoseconds) | `mod=1783961865632186366` |
| `permissions` | `prm` | integer (octal value) | `prm=420` = `0o644` = `rw-r--r--` |
| `size` | `sz` | integer | (sent by server STARTED/PROGRESS) |
| `name` | `n` | base64_string | `n=…` decodes to `/tmp/ft_obs/tmprocchpr6/q1b_dst.bin` |
| `status` | `st` | base64_string | server → client (e.g. `EPERM:…`, see Edge 4) |
| `parent` | `pr` | safe_string | (directories only) |
| `data` | `d` | base64_bytes | `d=<base64>` (see Q3 for decode) |

`n=` was verified by decoding: base64 `L3RtcC9mdF9vYnMvdG1wcm9jY2hwcjYvcTFiX2RzdC5iaW4` →
`/tmp/ft_obs/tmprocchpr6/q1b_dst.bin`. `prm=420` is the decimal form of octal `0644`.


---

## Q2 — rsync-style delta transfer & the data structures that track signatures and differences

### Activation

**[OBSERVED]** The delta path is activated by `--transmit-deltas` / `-x`, which corresponds to
`transmission_type=rsync` (`tt=rsync`) on the wire. The client decides in `File.metadata_command`:

```go
// send.go:652-654
func (self *File) metadata_command(use_rsync bool) *FileTransmissionCommand {
    if use_rsync && self.rsync_capable {
        self.ttype = TransmissionType_rsync
    }
```

where `rsync_capable` is the SENDER-side gate (`send.go:131`):

```go
rsync_capable: file_type == FileType_regular && stat_result.Size() > 4096,
```

Running the same 200000-byte file twice (fresh dst, then with ~200 bytes changed), both with `-x`,
`tt=rsync` appeared on the wire in both, and the reported stats were:

```
Transfer 1 (fresh dst):     Delta size: 200 kB  Signature size: 0 B     Transmitted: 200 kB of 200 kB (100.0%)
Transfer 2 (200 B changed): Delta size: 1.0 kB  Signature size: 9.0 kB  Transmitted: 10 kB of 200 kB (5.0%)
```

### Which side computes what (both rsync engines)

The receiving side sends **block signatures**; the sending side computes a **delta**
(`docs/file-transfer-protocol.rst:338-488`). There are two independent rsync engines implementing the same
wire contract:

- **Go client** — `tools/rsync/algorithm.go` + `tools/rsync/api.go` (imports `github.com/zeebo/xxh3` at
  `algorithm.go:21`).
- **C server** — `kittens/transfer/algorithm.c` (`#include <xxhash.h>` at line 13), compiled into `rsync.so`.

On a SEND (`kitten transfer src dst`), the terminal/server holds the old `dst` and computes the signature; the
server’s decision is `kitty/file_transmission.py:1026-1028`:

```python
sz = df.existing_stat.st_size if df.existing_stat is not None else -1
ttype = TransmissionType.rsync \
    if sz > -1 and df.ttype is TransmissionType.rsync and df.ftype is FileType.regular else TransmissionType.simple
```

i.e. the server uses rsync (emits a signature) only when an existing `dst` is present; otherwise the signature
is empty (0 B) and the whole file becomes the delta (which is exactly the `Signature size: 0 B / 100%` seen in
Transfer 1 above). The signature is produced by `PatchFile` (`file_transmission.py:377`) via
`next_signature_block` (`:426`) / `signature_header` (`:431`).

### The Go client data structures (names, sizes, verified at their anchors)

**[OBSERVED]** `tools/rsync/algorithm.go`:

- `type OpType byte` (`:31`) with `OpBlock=0, OpData=1, OpHash=2, OpBlockRange=3` (`:34-37`).
- `type Operation struct { Type OpType; BlockIndex, BlockIndexEnd uint64; Data []byte }` (`:72`).
- `type BlockHash struct { Index uint64; WeakHash uint32; StrongHash uint64 }` (`:177`), with
  `const BlockHashSize = 20` (`:183`), serialized **little-endian** (`PutUint64/PutUint32/PutUint64` at offsets
  0/8/12).
- The **weak** hash is the rsync rolling checksum, rolled forward in **O(1)** by
  `rolling_checksum.add_one_byte` (`:355`; full recompute `full()` at `:341`).
- The **strong** block-identity hash is **XXH3-64** (`new_xxh3_64()`, `:59`); the **integrity** checksum is
  **XXH3-128** (`new_xxh3_128()`, `:65`).

**[OBSERVED]** `tools/rsync/api.go`: `type Api` (`:47`) holds `signature []BlockHash` (`:49`); the flow is
`Patcher.StartDelta` (`:159`), `Patcher.CreateSignatureIterator` (`:195`), `Differ.CreateDelta` (`:230`), and
`NewPatcher` (`:270`).

Corroboration (reference test, not modified):

```
$ go test ./tools/rsync/
ok      kitty/tools/rsync    0.008s
```

### The signature binary format — byte-verified against emitted bytes

**[OBSERVED]** With a 250000-byte file I captured the real signature stream (10012 bytes across the
signature FTCs) and parsed the **12-byte little-endian header** (`docs/file-transfer-protocol.rst:417-458`):

```
first 12 header bytes (hex):  00 00 00 00 00 00 00 00 f4 01 00 00

parsed little-endian:
  uint16 version          = 0
  uint16 checksum_type     = 0   → XXH3-128 (integrity)
  uint16 strong_hash_type  = 0   → XXH3-64  (block identity)
  uint16 weak_hash_type    = 0   → rsync rolling checksum
  uint32 block_size        = 500
```

`block_size = 500` equals **exactly √250000 = 500.0**, confirming the spec’s “block size is usually the square
root of the file size.” The first 20-byte per-block record (`BlockHashSize = 20`) parsed as:

```
index = 0   weak_hash = 0xb245bd82   strong_hash = 0x949c23b622bf5ddd
```

which matches the documented record layout `uint64 index / uint32 weak / uint64 strong`, terminated by
`action=end_data`.

### The delta operations — parsed from a real delta stream

**[OBSERVED]** With a 60000-byte file and a 150-byte change (`--compress=never`), I reassembled the
765-byte delta stream from the client’s `ac=data`/`ac=end_data` `d=` payloads and parsed the operations. The
op wire layout is confirmed by `tools/rsync/algorithm.go:110-125` (byte[0]=type; `OpBlock`=1+u64=9 B,
`OpBlockRange`=1+u64+u32=13 B, `OpHash`=1+u16+data, `OpData`=1+u32+data), matching
`docs/file-transfer-protocol.rst:460-488`:

```
op counts:  OpBlock=0  OpData=2  OpHash=1  OpBlockRange=2     (765/765 bytes parsed cleanly)
first ops:  OpBlockRange(index=0, +121) → OpData(490) → OpBlockRange(index=124, +119) → OpData(220) → OpHash(16)
```

Interpretation: the unchanged blocks are emitted as **`OpBlockRange` references** (2 ranges covering ~240
blocks); the changed region is emitted as **`OpData`** literals (490+220 = 710 bytes); the final `OpHash`
size=16 bytes is the **XXH3-128** (128-bit) integrity checksum. So a 60000-byte file is reconstructed from a
765-byte delta.

### The match mechanism

**[OBSERVED via stats + delta ops; INFERRED for the exact per-byte control flow]** The sender rolls the weak
rolling checksum byte-by-byte over its file (`rolling_checksum.add_one_byte`, `algorithm.go:355`); on a weak
hit it confirms with the **XXH3-64** strong hash before accepting a block match. Matched blocks are emitted as
`OpBlock`/`OpBlockRange` references and only changed regions become `OpData`. This is directly visible in the
op breakdown above (mostly block references, a little literal data) and quantified in Q5.


---

## Q3 — Chunk encoding, reassembly, & distinction from terminal output

### Encoding: base64 (RawStdEncoding)

**[OBSERVED]** Data chunks are base64-encoded. `FileTransmissionCommand.Serialize()`
(`kittens/transfer/ftc.go:163`) uses `base64.RawStdEncoding.EncodeToString` both for the base64-tagged string
fields (`Bypass`/`Name`/`Status`, tagged `encoding:"base64"` at `ftc.go:129-131`) at `ftc.go:180`, and for the
raw `data` byte slice (the `d=` field) at `ftc.go:189`. **RawStdEncoding means no `=` padding.**

I transferred a known 67-byte file (`--compress=never`) and pulled the real `d=` token off the wire; it has no
trailing `=` and decodes exactly to the original bytes:

```
d= token (from wire):
RklMRS1UUkFOU0ZFUi1QUk9UT0NPTCBiYXNlNjQgZGVtbzogVGhlIHF1aWNrIGJyb3duIGZveCAwMTIzNDU2Nzg5Lgo

$ printf '%s' '<token>' | base64 -d      # (RawStd → add padding for coreutils)
FILE-TRANSFER-PROTOCOL base64 demo: The quick brown fox 0123456789.
```

The decoded bytes `== ` the original file (verified `True`). (Chunk sizes are ≤ 4096 bytes — see Q1.)

### Reassembly & the server handler chain

**[OBSERVED — the whole chain executes at runtime]** Every transfer in this investigation produced a
destination file whose bytes equal the source, which is only possible if the full server chain ran. The hops,
each cited:

1. **`kitty/vt-parser.c:547-550`** — the OSC dispatch. The enclosing guard, verbatim:

   ```c
   case FILE_TRANSFER_CODE:
       START_DISPATCH
       DISPATCH_OSC(file_transmission);
       END_DISPATCH
   ```

2. **`kitty/screen.c:2311`** — `file_transmission(Screen *self, PyObject *data)` →
   `CALLBACK("file_transmission", "O", data)` calls back into Python.

3. **`kitty/window.py:1388-1389`** — `def file_transmission(self, data)` →
   `self.file_transmission_control.handle_serialized_command(data)` (the `file_transmission_control` property is
   at `window.py:619`).

4. **`kitty/file_transmission.py:858`** — `handle_serialized_command` base64-decodes and appends chunks and
   writes to disk via `DestFile` (`:441`) for a simple transfer or `PatchFile` (`:377`) for an rsync transfer;
   `ActiveReceive` (`:583`) buffers signature chunks in `signature_pending_chunks` (`:599`).

### Distinction: how transfer data is told apart from ordinary terminal output

**[OBSERVED]** The distinction is **structural, not heuristic**: only bytes wrapped as `\x1b]5113;…\x1b\\`
match `case FILE_TRANSFER_CODE:` in the VT parser; ordinary output never carries that OSC number, so it never
enters the branch. Side-by-side capture through the **same** real `Screen`:

```
normal output  (sh -c "printf 'hello world\n'; printf 'normal terminal output, no OSC 5113\n'")
  received_bytes = b'hello world\r\nnormal terminal output, no OSC 5113\r\n'
  contains b'\x1b]5113' ?  →  False

real transfer
  contains b'\x1b]5113' ?  →  True
```

The discriminator is exactly the `case FILE_TRANSFER_CODE:` guard quoted above (`vt-parser.c:547`). Normal
program output is routed to the normal screen/printing path; only the OSC-5113-wrapped bytes are handed to the
file-transmission handler.

### One definition of 5113 for three languages

**[OBSERVED]** `kitty/control-codes.h:233` `#define FILE_TRANSFER_CODE 5113` → exposed to Python at
`kitty/data-types.c:596` (`PyModule_AddIntMacro`) → mirrored to Go by code generation at `gen/go_code.py:597`.
The runtime check `python3 -c "from kitty.fast_data_types import FILE_TRANSFER_CODE; print(...)"` prints `5113`
(shown in Q6), and the Go client uses `kitty.FileTransferCode` in the OSC prefix (`send.go:384`).


---

## Q4 — Transfer resumption: what state allows resume, and where the metadata lives

### The key finding

**[OBSERVED]** The state that lets an interrupted transfer **resume rather than restart** is the **existing
(partial) destination file itself**; the “resumption metadata” is the **rsync signature computed on the fly**
from that partial file. **There is no separate sidecar resume file** (no `.part`/`.resume`/companion).

### Runtime demonstration (genuine interrupt → resume)

**[OBSERVED]** I transferred a 500000-byte file with the real client, **interrupted it mid-flight** by
`SIGKILL`-ing the spawned `kitten transfer` child once the destination had partial-but-incomplete content, then
re-ran the **same** transfer with `-x`. After the interruption:

```
partial dst exists = True   partial size = 151552   full size = 500000
dst dir listing (look for any .part/.resume sidecar):
    big.bin      500000        (source)
    big_dst.bin  151552        (partial target — the ONLY dst artifact)
partial == full? = False
```

The partial destination is 151552 bytes — above the 4096-byte threshold (below). Resuming with
`--transmit-deltas`:

```
after resume: dst size = 500000   matches full? = True
final dst dir listing (still only the target, no sidecar):
    big.bin      500000
    big_dst.bin  500000
--- resume rsync stats ---
    Rsync stats:
    Delta size: 349 kB Signature size: 7.8 kB
    Transmitted: 357 kB of a total of 500 kB (71.4%)
--- resume wire tokens ---
    tt tokens on wire: ['tt=rsync']
    ac ordering on wire: ac=send, ac=file, ac=data (×many), ac=end_data, ac=finish
```

The resume computed a **7.8 kB signature from the 151552-byte partial file** and transmitted only
**357 kB of 500 kB (71.4%)** — the ~151 kB already on disk was matched via the signature and **not re-sent**
(the 349 kB delta ≈ the missing tail of 348448 bytes). This was **stable across two runs** (byte-identical
output). The directory listing before and after shows **only** the target file — proving the **absence of any
sidecar**.

### The mechanism, cited

**[OBSERVED]** For a SEND (the case demonstrated above), the resumption signature of the existing `dst` is
produced by the terminal/server (`file_transmission.py:1026-1028`, quoted in Q2; `PatchFile` at `:377`), and
the client computes the delta. The wire shows `tt=rsync` precisely because an existing dst was present.

**[OBSERVED]** The **symmetric** gate on the party that *holds the existing file* is the on-the-fly signature in
`kittens/transfer/receive.go:404-425` (this runs on the client when it is the receiver, i.e.
`--direction=receive`). The enclosing guard and the **4096-byte threshold**, verbatim:

```go
// receive.go:404
read_signature := self.use_rsync && f.ftype == FileType_regular
if read_signature {
    if s, err := os.Lstat(f.expanded_local_path); err == nil {
        read_signature = s.Size() > 4096          // receive.go:406-407
    } else {
        read_signature = false
    }
}
// … Ttype = utils.IfElse(read_signature, TransmissionType_rsync, TransmissionType_simple)
if read_signature {
    fsf, _ := os.Open(f.expanded_local_path)      // the EXISTING file is the basis
    f.patcher = rsync.NewPatcher(f.expected_size) // receive.go:423
    s_it := f.patcher.CreateSignatureIterator(fsf, &output)  // receive.go:425 — signature computed on the fly
    …
}
```

Note there is **no file open other than the existing destination** — the signature is derived from it directly,
which is why no sidecar is needed. I exercised this `receive.go` threshold directly in the Edge-cases section
(existing local file 1000 B vs 20000 B).

**[OBSERVED]** The `--transmit-deltas`/`-x` help text (`kittens/transfer/main.py:115`+) states the rsync path
“…potentially saving lots of bandwidth and also **automatically resuming partial transfers**.”


---

## Q5 — Delta-efficiency experiment (measured byte counts, ≥2 runs)

**Goal:** transfer a file, modify a small portion, transfer again, and show the second transfer sends
substantially less data — identifying the mechanism that detected the unchanged portions.

**Scale & stability:** I used a **5,000,000-byte (5.0 MB)** incompressible file (`--compress=never`, so the
byte counts reflect real data movement, not compression). block_size ≈ √5e6 ≈ 2236, giving ~2236 blocks — large
enough for a representative, stable measurement. I ran the **entire baseline→modify→retransfer cycle twice** in
independent temp directories.

**Measurement source:** `print_rsync_stats` (`kittens/transfer/utils.go:109-114`), verbatim:

```go
func print_rsync_stats(total_bytes, delta_bytes, signature_bytes int64) {
    fmt.Println("Rsync stats:")
    fmt.Printf("  Delta size: %s Signature size: %s\n", humanize.Size(delta_bytes), humanize.Size(signature_bytes))
    frac := float64(delta_bytes+signature_bytes) / float64(utils.Max(1, total_bytes))
    fmt.Printf("  Transmitted: %s of a total of %s (%.1f%%)\n", humanize.Size(delta_bytes+signature_bytes), humanize.Size(total_bytes), frac*100)
}
```

(called at `send.go:1260` and `receive.go:1168`).

### Captured output (both runs, unedited)

**[OBSERVED]** RUN 1 and RUN 2 were **byte-for-byte identical**:

```
========== RUN 1 : file size = 5000000 bytes (5.0 MB) ==========
--- TRANSFER 1 (baseline, fresh dst) --- exit=0 dest_matches=True tt=['rsync']
    Rsync stats:
    Delta size: 5.0 MB Signature size: 0 B
    Transmitted: 5.0 MB of a total of 5.0 MB (100.0%)
--- modified 1024 bytes at offset 2500000 (0.0205% of file) ---
--- TRANSFER 2 (delta retransfer) --- exit=0 dest_matches=True tt=['rsync']
    Rsync stats:
    Delta size: 2.6 kB Signature size: 45 kB
    Transmitted: 47 kB of a total of 5.0 MB (0.9%)
   delta-op breakdown (reassembled 2595-byte delta stream): {'OpBlock': 0, 'OpData': 2, 'OpHash': 1, 'OpBlockRange': 2} ; literal OpData payload bytes=2540

========== RUN 2 : file size = 5000000 bytes (5.0 MB) ==========
--- TRANSFER 1 (baseline, fresh dst) --- exit=0 dest_matches=True tt=['rsync']
    Rsync stats:
    Delta size: 5.0 MB Signature size: 0 B
    Transmitted: 5.0 MB of a total of 5.0 MB (100.0%)
--- modified 1024 bytes at offset 2500000 (0.0205% of file) ---
--- TRANSFER 2 (delta retransfer) --- exit=0 dest_matches=True tt=['rsync']
    Rsync stats:
    Delta size: 2.6 kB Signature size: 45 kB
    Transmitted: 47 kB of a total of 5.0 MB (0.9%)
   delta-op breakdown (reassembled 2595-byte delta stream): {'OpBlock': 0, 'OpData': 2, 'OpHash': 1, 'OpBlockRange': 2} ; literal OpData payload bytes=2540

### STABILITY CHECK: RUN 1 vs RUN 2 retransfer stats (should match) ###
RUN 1 retransfer: ['Delta size: 2.6 kB Signature size: 45 kB', 'Transmitted: 47 kB of a total of 5.0 MB (0.9%)']
RUN 2 retransfer: ['Delta size: 2.6 kB Signature size: 45 kB', 'Transmitted: 47 kB of a total of 5.0 MB (0.9%)']
```

### Result & the detecting mechanism

**[OBSERVED]** The first (full) transfer sent **5.0 MB (100%)**. After changing just **1024 bytes (0.0205%)**,
the second (delta) transfer sent only **47 kB of 5.0 MB — 0.9%**, i.e. roughly a **106× reduction** — stable
across both runs.

The delta-op breakdown proves *why*: the entire 5 MB is represented by just **2 `OpBlockRange` references**
(all the unchanged blocks, sent as references — not data), **2 `OpData` ops** carrying **2540 literal bytes**
(the changed region, larger than the 1024 edited bytes because the change straddles block boundaries), and **1
`OpHash`** (the 16-byte XXH3-128 integrity checksum). The 45 kB signature matches
≈ 2236 blocks × 20 bytes (`BlockHashSize`) + 12-byte header ≈ 44732 B.

**Mechanism, by name:** the unchanged portions are detected by the rsync **rolling weak checksum** rolled
byte-by-byte over the sender’s file (`rolling_checksum.add_one_byte`, `tools/rsync/algorithm.go:355`; full
recompute `full()` at `:341`), and each weak-hash hit is **confirmed by the XXH3-64 strong hash**
(`new_xxh3_64`, `:59`). Confirmed matches are emitted as `OpBlock`/`OpBlockRange` references, while only changed
regions become `OpData`. The receiver’s block signatures come from `PatchFile.next_signature_block` /
`signature_header` (`kitty/file_transmission.py:426-431`); the client’s delta is produced by
`Differ.CreateDelta` (`tools/rsync/api.go:230`).


---

## Edge cases exercised (secondary paths, not just the happy path)

All results below are **[OBSERVED]** through the real `kitten transfer` binary. A crucial nuance is that there
are **two symmetric 4096-byte thresholds**, one per direction:

- **SEND-side** gate on the source being sent — `send.go:131`
  (`rsync_capable = file_type == FileType_regular && stat_result.Size() > 4096`).
- **RECEIVE-side** gate on the existing local file — `receive.go:406-407` (`read_signature = s.Size() > 4096`).

### Edge 1 — fresh transfer, no existing destination

```
[no -x]   exit=0 tt=[]        → simple path (tt absent ⇒ TransmissionType_simple)
[with -x] exit=0 tt=['rsync'] → client sets rsync (src>4096), but a fresh dst has no basis ⇒ empty signature (0 B) ⇒ effectively a full send
```

A brand-new destination has nothing to delta against, which is why Q2/Q5 “Transfer 1” reported
`Signature size: 0 B` and `100%`.

### Edge 2 — existing file below the 4096-byte threshold (both thresholds)

**SEND-side (`send.go:131`)** — the source being sent is ≤ 4096 bytes:

```
src=1000B  + -x → exit=0 tt=[]        → rsync_capable=false (size not >4096) ⇒ simple
src=5000B  + -x → exit=0 tt=['rsync'] → contrast: >4096 ⇒ rsync capable
```

**RECEIVE-side (`receive.go:406-407`)** — the existing local destination file (exercised with
`--direction=receive`), directly demonstrating the threshold boundary:

```
existing local dst = 1000B  + -x → exit=0 tt=[]        → s.Size()>4096 FALSE ⇒ no signature ⇒ simple
existing local dst = 20000B + -x → exit=0 tt=['rsync'] → s.Size()>4096 TRUE  ⇒ signature computed ⇒ rsync (dst matches source: True)
```

### Edge 3 — interrupted-then-resumed transfer

Covered in **Q4** above: a 151552-byte partial destination becomes the rsync basis on resume; only 71.4% of the
file is transmitted; no sidecar file is created.

### Edge 4 — transfer refused at the confirmation prompt (EPERM)

**[OBSERVED]** Declining confirmation (the server is driven with `allow=False`, the genuine refusal path):

```
exit code = 1            (child exits non-zero)
dst created? = False     (nothing is written)
server response (terminal → client): action=status  status='EPERM:User refused the transfer'  size=None
--- kitten screen output ---
    Scanning files…
    Found 1 files and directories, requesting transfer permission…
    Permission denied for this transfer
```

This is the safeguard documented at `docs/kittens/transfer.rst:38-40` (kitty asks for confirmation “so that the
file transfer protocol cannot be abused to read/write files”), and the `EPERM` status is part of the documented
flow `docs/file-transfer-protocol.rst:43-113`.

### Compression (`compression=zlib` / `zip=zlib`)

**[OBSERVED]** The only supported compression is RFC 1950 zlib deflate (`docs/file-transfer-protocol.rst:490-503`):

```
[compressible file, default --compress] zip on wire = ['zlib']
[--compress=never]                      zip on wire = []       (none)
```

The server decompresses with `zlib.decompressobj(wbits=0)` (`kitty/file_transmission.py:368`); the client uses
`compress/zlib` (Go stdlib). The compression capability is gated alongside rsync in `send.go` (a regular file
> 4096 bytes that “should be compressed”), and selected in `File.metadata_command` (`send.go:655-658`:
`self.compression = Compression_zlib`).


---

## Coverage pass — every named mechanism, function, struct, condition, file, flag & example

Each item is confirmed addressed **by name** above, with its `file:line` anchor.

**Objectives**
- [x] **Q1** — handshake & OSC 5113 escape sequences (§ Q1)
- [x] **Q2** — rsync delta transfer & signature/difference data structures (§ Q2)
- [x] **Q3** — chunk encoding, reassembly, distinction from terminal output (§ Q3)
- [x] **Q4** — resumption state & metadata location (§ Q4)
- [x] **Q5** — delta-efficiency experiment, ≥2 runs, stable (§ Q5)
- [x] **Q6** — build from source (§ Environment & Build)
- [x] **Q7** — canonical `kitten ssh` → `kitten transfer` entry point (§ Canonical Entry Point)

**Wire format & 5113**
- [x] OSC 5113 form `<OSC> 5113 ; key=value … <ST>`; OSC=`0x1b 0x5d`, ST=`0x1b 0x5c`; byte hexdump verified — `docs/file-transfer-protocol.rst:543-552`
- [x] `5113` = numeralization of “file” — `docs/file-transfer-protocol.rst:551-552`
- [x] `FILE_TRANSFER_CODE` single source of truth — `kitty/control-codes.h:233` → `kitty/data-types.c:596` → `gen/go_code.py:597` (import `:575`)
- [x] `case FILE_TRANSFER_CODE:` / `DISPATCH_OSC(file_transmission)` — `kitty/vt-parser.c:547-550`
- [x] handler chain `vt-parser.c` → `screen.c:2311` → `window.py:1388-1389` (property `:619`) → `file_transmission.py`

**Key abbreviations (all)** — `ac, zip, ft, tt, id, fid, pw, q, mod, prm, sz, n, st, pr, d` — table in § Q1; `safe_string` excludes `;` (`docs/file-transfer-protocol.rst:589-591`)

**Client serialization & OSC prefix**
- [x] `FileTransmissionCommand.Serialize()` — `kittens/transfer/ftc.go:163`
- [x] `base64.RawStdEncoding.EncodeToString` — `ftc.go:180` (string fields `:129-131`) & `:189` (data)
- [x] OSC prefix `\x1b]%d;id=%s;` / suffix `\x1b\\` — `send.go:384-385`; `send_payload` `:647-649`
- [x] `tt=rsync` set in `File.metadata_command` — `send.go:652-654`; `rsync_capable` gate `send.go:131`

**Server handler**
- [x] `handle_serialized_command` — `file_transmission.py:858`
- [x] `DestFile` `:441`, `PatchFile` `:377`, `ActiveReceive` `:583`, `signature_pending_chunks` `:599`
- [x] server rsync decision `sz > -1 …` — `file_transmission.py:1026-1028`; `next_signature_block` `:426` / `signature_header` `:431`

**Go rsync data structures**
- [x] `OpType` (`OpBlock=0, OpData=1, OpHash=2, OpBlockRange=3`) — `tools/rsync/algorithm.go:31,34-37`
- [x] `Operation{Type,BlockIndex,BlockIndexEnd,Data}` — `algorithm.go:72`; op serialize `:110-125`
- [x] `BlockHash{Index,WeakHash,StrongHash}` + `BlockHashSize=20` — `algorithm.go:177,183`
- [x] `rolling_checksum` weak hash + `add_one_byte` O(1) — `algorithm.go:355` (full `:341`)
- [x] XXH3-64 `new_xxh3_64` `:59`; XXH3-128 `new_xxh3_128` `:65`; import `github.com/zeebo/xxh3` `:21`
- [x] `Api` `:47` / `signature []BlockHash` `:49` / `StartDelta` `:159` / `CreateSignatureIterator` `:195` / `CreateDelta` `:230` / `NewPatcher` `:270` — `tools/rsync/api.go`

**Binary formats (byte-verified)**
- [x] 12-byte signature header (version/checksum_type/strong_hash_type/weak_hash_type/block_size≈√size) + per-block records — `docs/file-transfer-protocol.rst:417-458`
- [x] delta op layout `Block(0)/Data(1)/Hash(2)/BlockRange(3)` — `docs/file-transfer-protocol.rst:460-488`

**Resumption / delta flags**
- [x] `--transmit-deltas`/`-x`, “automatically resuming partial transfers” — `kittens/transfer/main.py:115`+
- [x] 4096-byte threshold, both sides — `send.go:131` and `receive.go:404-407`; `NewPatcher` `:423`, `CreateSignatureIterator` `:425`
- [x] no sidecar file (directory-listing proof) — § Q4

**Stats / efficiency**
- [x] `print_rsync_stats` printing “Delta size” / “Signature size” / “Transmitted …” — `kittens/transfer/utils.go:109-114` (called `send.go:1260`, `receive.go:1168`)

**Server-side C rsync**
- [x] `kittens/transfer/algorithm.c:13` `#include <xxhash.h>`; linked via `setup.py:986`; `rsync.so` → `libxxhash.so.0`

**Compression**
- [x] `compression=zlib`/`zip=zlib` observed; server `zlib.decompressobj(wbits=0)` — `kitty/file_transmission.py:368`; only RFC 1950 zlib — `docs/file-transfer-protocol.rst:490-503`

**SSH kitten & build**
- [x] `kitten ssh` → `kitten transfer`; `remote_kitty` — `kittens/ssh/main.py:164,167`; `docs/kittens/ssh.rst`
- [x] canonical usage `versionadded:: 0.30.0` — `docs/kittens/transfer.rst:10,28-40`
- [x] build `python3 setup.py` → `kitty/launcher/kitty` + `kitty/launcher/kitten`; `libxxhash` — `setup.py:986`

**Edge cases (all four + compression)**
- [x] fresh/no-dst (simple) · existing file < 4096 (both sides) · interrupted-then-resumed · refused (EPERM) · zlib compression — § Edge cases

**Out of scope (correctly excluded):** the graphics protocol (`kitty/graphics.c`, `graphics.h`,
`parse-graphics-command.h`) and its unrelated `transmission_type`, the clipboard protocol, notifications, and
`kittens/tui/*` — matched keyword sweeps but are not the file transfer protocol.

---

## Observed vs inferred — summary

Everything in Q1–Q7 and the edge cases is **[OBSERVED]** at runtime through the real `kitten transfer` (and, for
Q7, the real `kitten ssh`) binary, with the sole accommodation that the receiving terminal is a **headless real
kitty `Screen`** (no GUI window) rather than a painted GUI window. The only **[INFERRED]** element is the exact
per-byte internal control flow of the rolling-checksum match loop (§ Q2 “match mechanism”), which is derived
from the code after observing its externally-visible effects (the delta-op breakdown and the Q5 byte savings);
its inputs and outputs are observed, only the intermediate loop is read rather than single-stepped.


---

## Repository left unchanged (read-only proof)

This was a strictly read-only investigation. All temporary observation scripts and test transfer files lived
outside the repository under `/tmp/ft_obs` and were deleted. No existing source file was modified, added to, or
removed. The transfer/rsync C module `kittens/transfer/algorithm.c` — whose mtime was touched once to capture
the verbose `libxxhash` link line in Q6 — has an unchanged content hash:

```
$ md5sum kittens/transfer/algorithm.c
df1463a61bb0f3a95eab8d4130564754  kittens/transfer/algorithm.c
```

The definitive proof that the only change to the repository is this answer document:

```
$ git diff --stat
(no output — zero source modifications)

$ git status --porcelain --untracked-files=all
?? blitzy/documentation/kitty_815df1e210e0.md

$ git rev-parse HEAD
815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
```

The single untracked path is this document; every other tracked file is byte-for-byte identical to
`HEAD` (`815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`).

