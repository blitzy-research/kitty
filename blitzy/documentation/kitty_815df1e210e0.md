# kitty File Transfer Protocol (OSC 5113) over the SSH kitten — a runtime-evidenced trace

This document answers, from observed runtime behaviour, how file data travels through kitty's File
Transfer Protocol when driven through the SSH kitten. Every factual claim carries either a literal
runtime artifact or a specific `file:line` citation, and each claim is labelled:

- **[OBSERVED]** — a literal artifact captured at runtime (command output, wire bytes, a hash, an
  `strace` line). The producing command is shown alongside it.
- **[SOURCE-VERIFIED]** — a fact read directly from a named source line; the code was read, not
  executed, to establish it.
- **[INFERRED]** — a reasoned conclusion that follows from observed facts but was not itself directly
  observed.

All source citations are against HEAD commit `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`
(branch `kitty_815df1e210e0`).

## Methodology and honest disclosures (read this first)

Three environmental facts materially shaped how the evidence below was produced. They are disclosed
up front rather than buried.

1. **Build environment — native, not the named container image [OBSERVED].** The task named a Docker
   image (`ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`). That image was **not**
   pulled; the toolchain was instead reproduced natively on Ubuntu 25.10. The exact toolchain
   identities are recorded in the Q6 section so the reader can judge fidelity. This is a deviation
   from the named image and is called out explicitly.

2. **The receiver is a real kitty GUI, run headlessly under Xvfb with software OpenGL [OBSERVED].**
   kitty is a GPU terminal with no headless flag. To exercise the genuine
   `vt-parser.c` then `screen.c` then `window.py` then `file_transmission.py` path (with the real
   `boss.confirm` permission UI), a real `kitty` process was launched under `Xvfb :99` with Mesa
   software GL (`LIBGL_ALWAYS_SOFTWARE=1`, `GALLIUM_DRIVER=llvmpipe`). Keystrokes were injected with
   `kitten @ send-text`, which is the remote-control equivalent of a user typing at the keyboard: it
   feeds bytes into the terminal's child PTY exactly as a keyboard would. It is **not** a protocol
   bypass — the file-transfer OSC codes still travel the full parser and handler chain, and the
   confirmation overlay still runs. This is distinct from kitty's unit-test harness
   (`kitty_tests/file_transmission.py`), which routes parsed commands straight to a test controller
   with a preset `allow` boolean; the test harness was **not** used for any claim here.

3. **SSH is a loopback-only, task-scoped daemon [OBSERVED].** Q7 requires a real `kitten ssh` hop. A
   dedicated `sshd` was bound to `127.0.0.1:2222` with all host keys, the authorized key, and the
   `known_hosts` file placed under `/tmp/ft_obs/sshd`, and `PermitRootLogin prohibit-password`. Its
   setup and teardown are documented, and it and its keys are removed during cleanup.

The observation workspace is `/tmp/ft_obs` (outside the repository). A reusable environment file
sourced by every command below is:

```
export REPO=/tmp/blitzy/kitty/blitzy-dc55a543-7db2-41a4-8fbc-3537be540314_8d3e93
export LANG=C.UTF-8 LC_ALL=C.UTF-8 GOFLAGS=-mod=mod
export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe
export KITTY=$REPO/kitty/launcher/kitty
export KITTEN=$REPO/kitty/launcher/kitten
export SOCK=/tmp/ft_obs/kitty.sock
```

## Q6 — Building kitty from source (canonical configuration)

The canonical build entry point is `python3 setup.py` (the `Makefile` `all:` target delegates to it).
It was run to completion and produced the launcher `kitty/launcher/kitty` and the Go `kitten` binary
`kitty/launcher/kitten`. `libxxhash` is a required system dependency for the transfer/rsync C module
and is linked by the build [SOURCE-VERIFIED: setup.py:986].

Toolchain identity [OBSERVED]:

```
$ python3 --version
Python 3.13.7
$ go version
go version go1.24.4 linux/amd64
$ cc --version | head -1
cc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
$ pkg-config --modversion libxxhash
0.8.3
$ uname -a
Linux reverse-code-generator-a872215c-rw5bh 6.6.122+ #1 SMP Thu Apr  2 09:59:00 UTC 2026 x86_64 GNU/Linux
$ head -3 go.mod
module kitty

go 1.22
```

`go.mod` requires `go 1.22`; the installed Go 1.24.4 satisfies it [OBSERVED].

Build artifacts, version banners, the protocol constant reaching Python, and the XXH3 linkage of the
server-side rsync module [OBSERVED]:

```
$ ls -l kitty/launcher/kitty kitty/launcher/kitten
-rwxr-xr-x 1 root root 16429348 Jul 13 18:04 kitty/launcher/kitten
-rwxr-xr-x 1 root root    40384 Jul 13 18:04 kitty/launcher/kitty

$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
$ ./kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal

$ python3 -c 'from kitty.fast_data_types import FILE_TRANSFER_CODE; print(FILE_TRANSFER_CODE)'
5113

$ ldd kittens/transfer/rsync.so | grep -i xxhash
	libxxhash.so.0 => /lib/x86_64-linux-gnu/libxxhash.so.0 (0x00007e092f4dd000)
$ nm -D kittens/transfer/rsync.so | grep -iE 'XXH3|XXH128' | head
                 U XXH128_canonicalFromHash
                 U XXH3_128bits_digest
                 U XXH3_128bits_reset
                 U XXH3_128bits_update
                 U XXH3_64bits
                 U XXH3_64bits_digest
                 U XXH3_64bits_reset
                 U XXH3_64bits_update
                 U XXH3_createState
                 U XXH3_freeState
```

The `--transmit-deltas` help text (the flag used throughout for the rsync path) [OBSERVED]:

```
$ ./kitty/launcher/kitten transfer --help
  --transmit-deltas, -x
    If a file on the receiving side already exists, use the rsync algorithm to
    update it to match the file on the sending side, potentially saving lots of
    bandwidth and also automatically resuming partial transfers. Note that this
    will actually degrade performance on fast links or with small files, so use
    with care.
```

## Q7 — The canonical entry point: real `kitten ssh` then remote `kitten transfer`

A loopback-only `sshd` was started with task-scoped keys under `/tmp/ft_obs/sshd`
(`ListenAddress 127.0.0.1`, `Port 2222`, `PermitRootLogin prohibit-password`):

```
$ /usr/sbin/sshd -f /tmp/ft_obs/sshd/sshd_config
```

Inside the real kitty GUI (Xvfb, software GL), the SSH kitten was invoked, a remote shell was reached,
and — critically — the transfer kitten was run **on the remote** by its bare name, which resolves
through the remote `PATH` (the remote-supplied `kitten`, not a local absolute path). The commands
typed (via simulated keystrokes) were:

```
kitten ssh -p 2222 -i /tmp/ft_obs/sshd/id -o UserKnownHostsFile=/tmp/ft_obs/sshd/known_hosts -o StrictHostKeyChecking=yes -o IdentitiesOnly=yes root@127.0.0.1
command -v kitten
kitten transfer /tmp/ft_obs/q7_remote_src.bin /tmp/ft_obs/q7_local_dst.bin
```

On the remote, `command -v kitten` resolved to `/usr/local/bin/kitten` [OBSERVED]. The confirmation
overlay was accepted by typing `y`. The kitty screen, the receiver-written destination, matching
hashes, and the `strace` proof that the **kitty** process (PID 122499) — not the kitten client —
opened and wrote the destination [OBSERVED]:

```
Permission granted for this transfer
\u2714 /tmp/ft_obs/q7_remote_src.bin              3.1kB
-rw-r--r-- 1 root root 3072 Jul 13 18:36 /tmp/ft_obs/q7_local_dst.bin
4b9424f75a2a2bcb8775888a0f13e806bdecac93c5b369ff1c61a00fda74b1a9  /tmp/ft_obs/q7_remote_src.bin
4b9424f75a2a2bcb8775888a0f13e806bdecac93c5b369ff1c61a00fda74b1a9  /tmp/ft_obs/q7_local_dst.bin
122499 openat(AT_FDCWD, "/tmp/ft_obs/q7_local_dst.bin", O_RDWR|O_CREAT|O_TRUNC|O_CLOEXEC, 0644) = 11
```

The SSH kitten makes the transfer kitten available on the remote automatically
[SOURCE-VERIFIED: docs/kittens/ssh.rst:22; kittens/ssh/main.py:167]. The `strace` line is direct
evidence that the full server-side chain executed inside the terminal process and wrote the file
[OBSERVED]. (`\u2714` is the checkmark glyph U+2714 as recorded by the capture.)

## Q1 — Protocol handshake and OSC 5113 escape sequences

### The wire format

Commands are encoded as OSC escape codes of the form `<OSC> 5113 ; key=value ; ... <ST>`, where the
number 5113 is the numeralization of the word "file"
[SOURCE-VERIFIED: docs/file-transfer-protocol.rst:543-552]. OSC is the two bytes `ESC ]`
(`0x1b 0x5d`) and ST is `ESC \` (`0x1b 0x5c`), confirmed against the emitted bytes below.

### How the client initiates the handshake

The sending kitten builds each command with `FileTransmissionCommand.Serialize()`, then writes the
OSC prefix `\x1b]%d;id=%s;` (with `%d` = `kitty.FileTransferCode` = 5113) followed by the serialized
payload [SOURCE-VERIFIED: kittens/transfer/ftc.go:163-222; kittens/transfer/send.go:384,647-648]. The
first frame the client emits is an `action=send` command that opens the session.

### The handshake ordering (captured losslessly, both directions)

The transfer was run under `script`, which tees both directions of the PTY while the real kitty acts
as receiver; `--log-out` captures client-to-terminal bytes and `--log-in` captures terminal-to-client
bytes. Producing command [OBSERVED]:

```
$ script -q --log-out /tmp/ft_obs/q1_frames_out.bin --log-in /tmp/ft_obs/q1_frames_in.bin -c "$KITTEN transfer --compress=never /tmp/ft_obs/q1_src.bin /tmp/ft_obs/q1_dst.bin"
```

The complete frame inventory from that capture (every frame; each data field is shown by its exact
base64 length and decoded byte count, so nothing is elided) [OBSERVED]:

```
# CLIENT -> TERMINAL (sender kitten writes; captured via script --log-out)
  frame 0: id=1e64bf179 ac=send
  frame 1: id=1e64bf179 n=L3RtcC9mdF9vYnMvcTFfZHN0LmJpbg prm=420 ac=file fid=1 mod=1783967945418963894
  frame 2: id=1e64bf179 fid=1 d=<5462 base64 chars, decodes to 4096 bytes> ac=data
  frame 3: id=1e64bf179 fid=1 d=<2539 base64 chars, decodes to 1904 bytes> ac=end_data
  frame 4: id=1e64bf179 ac=finish
# TERMINAL -> CLIENT (real kitty responds; captured via script --log-in)
  frame 0: ac=status id=1e64bf179 st=T0s
  frame 1: ac=status id=1e64bf179 fid=1 n=L3RtcC9mdF9vYnMvcTFfZHN0LmJpbg st=U1RBUlRFRA
  frame 2: ac=status id=1e64bf179 fid=1 sz=4096 st=UFJPR1JFU1M
  frame 3: ac=status id=1e64bf179 fid=1 sz=6000 n=L3RtcC9mdF9vYnMvcTFfZHN0LmJpbg st=T0s
```

The raw bytes of the small control frames, verbatim [OBSERVED]:

```
send  : b'\x1b]5113;id=1e64bf179;ac=send\x1b\\'
file  : b'\x1b]5113;id=1e64bf179;n=L3RtcC9mdF9vYnMvcTFfZHN0LmJpbg;prm=420;ac=file;fid=1;mod=1783967945418963894\x1b\\'
status: b'\x1b]5113;ac=status;id=1e64bf179;st=T0s\x1b\\'
```

The `st=` status field is base64; decoded, `T0s` = `OK`, `U1RBUlRFRA` = `STARTED`, and
`UFJPR1JFU1M` = `PROGRESS` [OBSERVED]. Putting the two directions together, the observed ordering for
this 6000-byte file was: client `ac=send`; terminal `st=OK`; client `ac=file`; terminal `st=STARTED`;
client `ac=data` (a 4096-byte chunk); terminal `st=PROGRESS sz=4096`; client `ac=end_data` (the final
1904-byte chunk); terminal `st=OK sz=6000`; client `ac=finish`. Every frame carries the same session
`id=1e64bf179` and, after the file is opened, the same `fid=1`, so requests and acknowledgements are
correlated by those two IDs [OBSERVED]. This matches the documented session flow
[SOURCE-VERIFIED: docs/file-transfer-protocol.rst:43-113]. The `prm=420` metadata is decimal 420 =
octal `0o644`, the source file's permission bits [OBSERVED].

### Key abbreviations — every one, and a real specification/implementation drift

Keys are abbreviated on the wire to reduce overhead
[SOURCE-VERIFIED: docs/file-transfer-protocol.rst:558-576]:

| Full key | Abbrev | Type on the wire |
|---|---|---|
| action | `ac` | keyword (e.g. `send`, `file`, `data`, `end_data`, `status`, `finish`) |
| compression | `zip` | keyword (`none` / `zlib`) |
| file_type | `ft` | keyword |
| transmission_type | `tt` | keyword (`simple` / `rsync`) |
| id | `id` | safe_string (session id) |
| file_id | `fid` | safe_string |
| bypass | `pw` | see drift note below |
| quiet | `q` | integer (0 = verbose / 1 = only errors / 2 = totally silent) |
| mtime | `mod` | integer nanoseconds |
| permissions | `prm` | integer (octal value in decimal) |
| size | `sz` | integer |
| name | `n` | base64_string |
| status | `st` | base64_string |
| parent | `pr` | safe_string |
| data | `d` | base64 bytes |

**Drift note (bypass / `pw`) [SOURCE-VERIFIED].** The protocol specification lists `bypass` (`pw`) as
a `safe_string`, described as a "hash of the bypass password and the session id"
[docs/file-transfer-protocol.rst:567]. The implementations, however, encode `pw` as **base64**: the
Go client tags the field `encoding:"base64"` [kittens/transfer/ftc.go:129] and the Python server
declares `metadata={'base64': True, 'sname': 'pw'}` [kitty/file_transmission.py:260]. This is a
genuine spec-versus-implementation drift; the base64 behaviour must not be attributed to the cited
specification text, which says `safe_string`.

The `safe_string` character set is `[0-9a-zA-Z_:./@-]`, which deliberately excludes the `;`
separator so unencoded fields can never break framing
[SOURCE-VERIFIED: docs/file-transfer-protocol.rst:589-601].

## Q2 — rsync-style delta transfer and the data structures

### Activation and which side computes what

The delta path is activated by `transmission_type=rsync` (`tt=rsync`), requested with
`--transmit-deltas`/`-x` [SOURCE-VERIFIED: docs/file-transfer-protocol.rst:338-488;
kittens/transfer/main.py:116-122]. The two sides run **two independent rsync engines** that implement
the identical wire format — a deliberate cross-language contract [SOURCE-VERIFIED]:

- The **sender** (`kitten transfer`, Go) computes the **delta** with the Go engine
  `tools/rsync/algorithm.go` and `tools/rsync/api.go`.
- The **receiver** (kitty) computes the **signature** of the existing file with the C engine
  `kittens/transfer/algorithm.c`. The Python module `kittens.transfer.rsync` (`rsync.so`) is compiled
  from that C source [SOURCE-VERIFIED: setup.py:986], and `PatchFile` imports its `Patcher`
  [kitty/file_transmission.py:380-381].

### The Go client data structures (names, sizes, at their source anchors)

[SOURCE-VERIFIED: tools/rsync/algorithm.go]:

- `OpType` enumerates `OpBlock=0`, `OpData=1`, `OpHash=2`, `OpBlockRange=3` [algorithm.go:31-37].
- `Operation{Type, BlockIndex, BlockIndexEnd, Data}` is one delta instruction [algorithm.go:71-77],
  with serialized sizes `OpBlock`=9, `OpBlockRange`=13, `OpHash`=`3+len`, `OpData`=`5+len`
  [algorithm.go:95-102]. `OpBlockRange` serializes `BlockIndexEnd - BlockIndex` as its `uint32`, so a
  range covers blocks `[BlockIndex, BlockIndexEnd]` inclusive.
- `BlockHash{Index uint64, WeakHash uint32, StrongHash uint64}` is the per-block signature record,
  20 bytes serialized little-endian: `index@0`, `weak@8`, `strong@12` [algorithm.go:177-204].
- `rolling_checksum` is the weak rsync hash; `full()` computes it in O(n) and `add_one_byte()` rolls
  the window one byte forward in O(1) [algorithm.go:336-361]. The strong block hash is XXH3-64 and the
  whole-file integrity checksum is XXH3-128 [algorithm.go:40-66,69-102].
- `Api`/`Patcher` hold the block list and drive signature/delta/patch
  [SOURCE-VERIFIED: tools/rsync/api.go:47-230,270-282].

### The signature binary format — byte-verified against emitted bytes

For a 250,000-byte file transferred with `-x`, the signature the receiver sent (captured on the
terminal-to-client channel) parsed to [OBSERVED]:

```
=== SIGNATURE (terminal->client, q2_in.bin) ===
signature payload bytes: 10012
12-byte header hex: 00 00 00 00 00 00 00 00 f4 01 00 00
  version=0 checksum_type=0 strong_hash_type=0 weak_hash_type=0 block_size=500
  sqrt(250000)=500  block_size matches sqrt? True
per-block records: 500 (expected 250000/500 = 500)
  first BlockHash: index=0 weak_hash=0xf88bf464 strong_hash=0x0418a1ad0d668657
```

The 12-byte little-endian header is `uint16 version`, `uint16 checksum_type`, `uint16
strong_hash_type`, `uint16 weak_hash_type`, `uint32 block_size`
[SOURCE-VERIFIED: docs/file-transfer-protocol.rst:417-458; emitted by
kittens/transfer/algorithm.c:226-238]. The block size is `round(sqrt(file_size))`
[SOURCE-VERIFIED: kittens/transfer/algorithm.c:204; tools/rsync/api.go:274]; `round(sqrt(250000))` =
500, matching the emitted field [OBSERVED]. Each record is 20 bytes (`le64 index`, `le32 weak`,
`le64 strong`) [SOURCE-VERIFIED: kittens/transfer/algorithm.c:246-260].

### The delta operations — parsed from a real delta stream

The delta the sender emitted for the same file (a small-edit case) parsed to [OBSERVED]:

```
=== DELTA (client->terminal, q2_out.bin) ===
delta payload bytes: 550
op counts: {'OpBlock': 0, 'OpData': 1, 'OpHash': 1, 'OpBlockRange': 2} (550/550 bytes parsed)
first ops: OpBlockRange(start=0,+239) -> OpData(size=500) -> OpBlockRange(start=241,+258) -> OpHash(size=16)
```

Matched runs of blocks became two `OpBlockRange` references, the changed block became one
`OpData(500)`, and a final `OpHash(16)` carried the XXH3-128 integrity checksum [OBSERVED]. The op
type/width layout matches the specification: `Block(0)`=`uint64 index`; `Data(1)`=`uint32 size` +
payload; `Hash(2)`=`uint16 size` + checksum; `BlockRange(3)`=`uint64 start` + `uint32 (end-start)`
[SOURCE-VERIFIED: docs/file-transfer-protocol.rst:460-488; tools/rsync/algorithm.go:104-140].

### The match mechanism

The sender's delta is produced by `Differ.CreateDelta(src, output)`
[SOURCE-VERIFIED: tools/rsync/api.go:230] — invoked by the transfer kitten as
`self.delta_loader = self.differ.CreateDelta(self.actual_file, self.deltabuf)`
[SOURCE-VERIFIED: kittens/transfer/send.go:768] — which loads the previously received signature and
delegates to the streaming `(*rsync).CreateDiff` [SOURCE-VERIFIED: tools/rsync/api.go:239]. The
low-level convenience wrapper `(*rsync).CreateDelta` funnels through the same `CreateDiff` and
collects the ops into a slice [SOURCE-VERIFIED: tools/rsync/algorithm.go:599-601]. That routine
detects matches by rolling the weak checksum byte-by-byte and, on a weak-hash hit,
confirming with the strong hash [SOURCE-VERIFIED: tools/rsync/algorithm.go]: `hash_lookup
map[uint32][]BlockHash` is keyed by weak hash and built from the received signature
[algorithm.go:366,617-623]; the window advances via `self.rc.add_one_byte(...)` [algorithm.go:543];
`if hh, ok := self.hash_lookup[self.rc.val]; ok` is the weak-hash lookup [algorithm.go:556] and
`find_hash(...)` confirms with `block.StrongHash == hv` (XXH3-64) before emitting a reference
[algorithm.go:557,641-643].

### Moved/shifted-block detection (position-independent matching)

Because the weak checksum is rolled at every byte offset, blocks are matched by content regardless of
where they moved to in the file. This was demonstrated by prepending 777 novel bytes to a
200,000-byte basis and re-transferring with `-x` [OBSERVED]:

```
delta payload bytes: 1010 (vs full src 200777 bytes)
op counts: {'OpBlock': 0, 'OpData': 2, 'OpHash': 1, 'OpBlockRange': 1}
first ops: OpData(size=777) -> OpBlockRange(start=0,+446) -> OpData(size=191) -> OpHash(size=16)
literal OpData bytes total: 968 (~= the 777 prepended novel bytes)
blocks referenced (OpBlock + sum OpBlockRange spans): 447 of 448 basis blocks (200000/447)
```

Here the basis file's block size is `round(sqrt(200000))` = **447** bytes
[SOURCE-VERIFIED: kittens/transfer/algorithm.c:204] (distinct from the 500-byte block size of the
250,000-byte example above, which is `round(sqrt(250000))` = 500), so the 200,000-byte basis divides
into 447 full 447-byte blocks (447 × 447 = 199,809 bytes) plus a 191-byte partial trailing block —
**448 basis blocks** in total, matching the 448 signature records the receiver emitted. Only the 777
prepended bytes plus that 191-byte partial tail were sent as literal `OpData` (447 × 447 + 191 + 777 =
200,777 = the full new-file size, confirmed by the reconstruction cross-check [OBSERVED]); the partial
tail cannot be matched as a whole block because it is shorter than the 447-byte rolling window. The
447 full original blocks were referenced by their **original index** through a single
`OpBlockRange[0..446]`, even though every one had shifted 777 bytes later in the file [OBSERVED]. The
position-independent lookup that makes this possible is `self.hash_lookup[self.rc.val]`
[SOURCE-VERIFIED: tools/rsync/algorithm.go:532-567]; the C server's analogue is
[SOURCE-VERIFIED: kittens/transfer/algorithm.c:719-747].

## Q3 — Chunk encoding, reassembly, and distinction from terminal output

### Encoding: base64 (RawStdEncoding)

The `d=` data field is base64, using Go's RawStdEncoding (no `=` padding)
[SOURCE-VERIFIED: kittens/transfer/ftc.go:178-189]. An exact-known 89-byte file was transferred and
every stage hashed [OBSERVED]:

```
d= token (raw, from wire):
a2l0dHkgT1NDIDUxMTMgUTMgZXhhY3QtYnl0ZSBkZW1vOiAwMTIzNDU2Nzg5IEFCQ0RFRkdISUpLTE1OT1BRUlNUVVZXWFlaIGFiY2RlZmdoaWprbG1ubwo
token has trailing '=' padding?  False
source   bytes: 89 sha256: 3d52fde3c7d726c3b05fcf1612c0bb4f2ac279c677c0789502b576fad85eed3d
decoded  bytes: 89 sha256: 3d52fde3c7d726c3b05fcf1612c0bb4f2ac279c677c0789502b576fad85eed3d
dest     bytes: 89 sha256: 3d52fde3c7d726c3b05fcf1612c0bb4f2ac279c677c0789502b576fad85eed3d
decoded == source ? True
dest    == source ? True
```

The token carries no `=` padding (RawStdEncoding), decodes to exactly 89 bytes, and the source, the
base64-decoded token, and the destination kitty wrote all share SHA-256
`3d52fde3c7d726c3b05fcf1612c0bb4f2ac279c677c0789502b576fad85eed3d` [OBSERVED].

### Reassembly and the server handler chain (precise citations)

Incoming OSC 5113 bytes travel `kitty/vt-parser.c`, then `kitty/screen.c`, then `kitty/window.py`,
then `kitty/file_transmission.py`. The exact hop citations are [SOURCE-VERIFIED]:

- `kitty/vt-parser.c:547-550` — `case FILE_TRANSFER_CODE:` then `DISPATCH_OSC(file_transmission)`.
- `kitty/screen.c:2311-2312` — `file_transmission(Screen*, PyObject*)` calls back into Python.
- `kitty/window.py:1388-1389` — `file_transmission()` forwards to
  `self.file_transmission_control.handle_serialized_command(data)`.

Within `file_transmission.py` the work is **distributed across several methods** — the frequently
cited `handle_serialized_command` only deserializes and dispatches; it does not itself decode, buffer,
decompress, or write [SOURCE-VERIFIED]:

- `handle_serialized_command(:858)` — calls `FileTransmissionCommand.deserialize(data)` and
  dispatches; that is all it does.
- `deserialize(:329-351)` — base64-decodes `bytes` fields via `base64_decode(val)` (this is where the
  `d=` payload is decoded) and decodes base64 vs `safe_string` for string fields per each field's
  metadata.
- `ActiveReceive.add_data(:623-631)` — routes a decoded chunk to `DestFile.write_data`.
- `DestFile.write_data(:510)` — for a regular file: decompresses at `:542`
  (`self.decompressor(...)`), lazily opens the destination with
  `os.O_RDWR | os.O_CREAT | os.O_TRUNC | os.O_CLOEXEC` at `:546-547`, and writes the decompressed
  bytes with `af.write(decompressed)` at `:550`.
- `ZlibDecompressor(:367-371)` — `zlib.decompressobj(wbits=0)` then `self.d.decompress(data)`.
- `PatchFile.write / write_to_dest(:420-423)` — the rsync path applies delta operations instead of a
  plain write.
- The send side serializes outgoing commands around `:1086-1128`.

### Distinction: how transfer data is told apart from ordinary output

Only byte sequences wrapped as `\x1b]5113;...\x1b\\` reach the file-transmission branch; ordinary
terminal output is never wrapped that way, so it never enters `case FILE_TRANSFER_CODE:`. This was
confirmed by capturing ordinary command output and a real transfer through the same terminal
[OBSERVED]:

```
=== NORMAL command output capture (q3_normal.bin) ===
bytes repr: b'Script started on 2026-07-13 18:42:20+00:00 [COMMAND="printf \'hello world\\n\'; printf \'ordinary output, no OSC 5113\\n\'" T'
contains b'\x1b]5113' ? -> False

=== REAL TRANSFER capture (q3_out.bin) ===
contains b'\x1b]5113' ? -> True
```

The discriminator is structural — the OSC introducer plus the code 5113
[SOURCE-VERIFIED: kitty/vt-parser.c:547-550]. Ordinary output carries no such wrapper, so it is never
routed to the handler [INFERRED, from the observed presence/absence above and the parser branch].

### One definition of 5113 for three languages

The protocol number is defined once and propagated [SOURCE-VERIFIED]:

```
kitty/control-codes.h:233   #define FILE_TRANSFER_CODE 5113
kitty/data-types.c:596      PyModule_AddIntMacro(m, FILE_TRANSFER_CODE);
gen/go_code.py:575          from kitty.fast_data_types import FILE_TRANSFER_CODE
gen/go_code.py:597          const FileTransferCode int = {FILE_TRANSFER_CODE}
```

The `gen/go_code.py` f-string template `{FILE_TRANSFER_CODE}` is expanded at code-generation time to
the literal `5113` in the emitted Go source, so the same integer is shared by the C core, the Python
module, and the Go kitten.

## Q4 — Transfer resumption: the state that allows resume, and where the metadata lives

The state that lets an interrupted transfer resume rather than restart is the **existing (partial)
destination file** itself. The "resumption metadata" is the rsync **signature computed on the fly
from that partial file** — there is **no persistent resume-metadata sidecar**. The `-x` help text
states the flag uses rsync "potentially saving lots of bandwidth and also automatically resuming
partial transfers" [SOURCE-VERIFIED: kittens/transfer/main.py:116-122]. The receiver builds the
signature from the existing regular file only when it exceeds 4096 bytes, then sets up
`NewPatcher(...)` and `CreateSignatureIterator(...)`
[SOURCE-VERIFIED: kittens/transfer/receive.go:404-425].

### Runtime demonstration (genuine interrupt then resume, two runs)

A 314,572,800-byte file was transferred; the sender kitten was caught mid-flight and terminated with
a targeted signal (its exact PID captured and verified `comm=kitten` before signalling), then the
transfer was resumed with `-x`. Two runs interrupted at different fractions
(`SRC sha256=c629fd32604032ee8f1b16639701e368904a8daa4905eebfd2d854e3f14c5795`).

Run 1 — interrupted at ~7% (21,970,702 of 314,572,800), then resumed [OBSERVED]:

```
=== BEFORE (t0): dst listing ===
ls: cannot access '/tmp/ft_obs/q4_dst.bin': No such file or directory
=== DURING (t1): caught partial dst=20054036 sender PID=131525 comm=kitten ===
CAUGHT_SIZE=20054036 KILLED_PID=131525
=== AFTER INTERRUPT (t2): partial dst listing ===
-rw-r--r-- 1 root root 21970702 Jul 13 18:50 /tmp/ft_obs/q4_dst.bin
PARTIAL_SIZE=21970702  (full=314572800; partial? YES)
partial dst sha256      : e787e7c29e05aae93836cbc7c1df3c9a03c9a69bc8570769da31ce26cc1f33c1
source[:21970702] sha256 : e787e7c29e05aae93836cbc7c1df3c9a03c9a69bc8570769da31ce26cc1f33c1
PREFIX MATCH: YES
```

The resume command and its result [OBSERVED]:

```
$ script -q --log-out /tmp/ft_obs/q4_out.bin --log-in /tmp/ft_obs/q4_in.bin -c "$KITTEN transfer -x /tmp/ft_obs/q4_src.bin /tmp/ft_obs/q4_dst.bin"
resume completed to full size? yes  final_size=314572800
final dst sha256: c629fd32604032ee8f1b16639701e368904a8daa4905eebfd2d854e3f14c5795
source  sha256: c629fd32604032ee8f1b16639701e368904a8daa4905eebfd2d854e3f14c5795
HASH MATCH (resume produced identical file): YES
```

The resume used the rsync path, and the receiver's signature was computed **over the partial**, not
the full source — the decisive evidence being the signature's block size [OBSERVED]:

```
ac=file frame fields: {'id': '20862f1db', 'mod': '1783968518192690389', 'prm': '420', 'ac': 'file', 'zip': 'zlib', 'fid': '1', 'n': 'L3RtcC9mdF9vYnMvcTRfZHN0LmJpbg', 'tt': 'rsync'}
Rsync stats:
  Delta size: 293 MB Signature size: 94 kB
  Transmitted: 293 MB of a total of 315 MB (93.1%)
signature payload bytes decoded from q4_in.bin: 93772
12-byte header hex: 00 00 00 00 00 00 00 00 4f 12 00 00
  block_size=4687
  round(sqrt(partial=21970702)) = 4687  => block_size matches sqrt(PARTIAL)? True
  round(sqrt(full=314572800)) = 17736  => NOT used (proves signature is over the EXISTING PARTIAL)
  first BlockHash: index=0 weak_hash=0xca422098 strong_hash=0xd8a9006b419def05
```

`block_size` = `round(sqrt(21970702))` = 4687, the square root of the **partial** size, not of the
full source (whose root is 17736) [OBSERVED; formula SOURCE-VERIFIED: kittens/transfer/algorithm.c:204].
Because the already-present ~22 MB was matched by rsync and not re-sent, only a 293 MB delta was
transmitted for a 315 MB file [OBSERVED].

Run 2 — interrupted at ~32% (101,662,737 bytes), reproducing the behaviour [OBSERVED]:

```
=== DURING (t1): caught partial dst=100151453 sender PID=134361 comm=kitten ===
=== AFTER INTERRUPT (t2): partial=101662737  (partial? YES) ===
prefix match: YES  partial_sha256=dec498cc52f0b3a601bfc83920992c4b4a8b5d69455d360d40edb15b9431790d
tt=rsync
Rsync stats:
  Delta size: 213 MB Signature size: 202 kB
  Transmitted: 213 MB of a total of 315 MB (67.8%)
signature decoded bytes=201672 block_size=10083  round(sqrt(partial=101662737))=10083
final sha256 : c629fd32604032ee8f1b16639701e368904a8daa4905eebfd2d854e3f14c5795
source sha256: c629fd32604032ee8f1b16639701e368904a8daa4905eebfd2d854e3f14c5795
HASH MATCH: YES
```

Both runs completed to a byte-identical copy of the source; the mechanism is stable [OBSERVED].

### No persistent sidecar, but a transient patch file (precise wording)

There is no persistent resume-metadata sidecar — the signature is computed **on the fly**, block by
block, by `PatchFile.next_signature_block` [SOURCE-VERIFIED: kitty/file_transmission.py:426] and
streamed straight back to the sender by `FileTransmission.transmit_rsync_signature`
[SOURCE-VERIFIED: kitty/file_transmission.py:1081]; the only in-memory retention is the transient
backpressure buffer `ActiveReceive.signature_pending_chunks` (a `Deque[FileTransmissionCommand]`)
[SOURCE-VERIFIED: kitty/file_transmission.py:599], into which a signature chunk is appended only when
the child pipe is momentarily full [SOURCE-VERIFIED: kitty/file_transmission.py:1119,1121] and which
is drained first on the next scheduled pass before more blocks are read
[SOURCE-VERIFIED: kitty/file_transmission.py:1086-1088]. During an rsync **patch**, however,
`PatchFile` creates a **transient** output file:
`tempfile.NamedTemporaryFile(mode='wb', dir=os.path.dirname(realpath(path)), delete=False)`
[SOURCE-VERIFIED: kitty/file_transmission.py:392], and on close it does
`os.replace(self.dest_file.name, self.src_file.name)`
[SOURCE-VERIFIED: kitty/file_transmission.py:405]. That transient file was observed live via `strace`
of the kitty receiver during Run 2, then seen renamed onto the destination [OBSERVED]:

```
122499 openat(AT_FDCWD</tmp/blitzy/kitty/blitzy-dc55a543-7db2-41a4-8fbc-3537be540314_8d3e93>, "/tmp/ft_obs/tmp1pwl3tqe", O_RDWR|O_CREAT|O_EXCL|O_NOFOLLOW|O_CLOEXEC, 0600) = 17</tmp/ft_obs/tmp1pwl3tqe>
122499 rename("/tmp/ft_obs/tmp1pwl3tqe", "/tmp/ft_obs/q4_dst.bin") = 0
dir-watch caught the transient temp file live: /tmp/ft_obs/tmp1pwl3tqe
after completion, temp file is gone (renamed onto dest): no tmp* remains
```

The precise statement is therefore: **no persistent resume sidecar exists; the resume basis is the
partial destination file and its on-the-fly signature; a transient patch-output temp file exists only
during rsync reconstruction and is atomically renamed into place** [OBSERVED + SOURCE-VERIFIED].

## Q5 — Delta-efficiency experiment (measured, corrected, two stable runs)

A deterministic, incompressible 5,000,000-byte file was generated, transferred in full, then a
1024-byte region at offset 2,500,000 was overwritten and the file re-transferred with `-x`. The
generation and edit are fixed-seed scripts, so the byte counts are exactly reproducible [OBSERVED]:

```
$ python3 /tmp/ft_obs/q5_gen.py /tmp/ft_obs/q5_src.bin 5000000
original src size=5000000 sha256=dd0c0eaa8d732ef292f0d8d9a44a8cbd176f2e5bec84ed947e97c37230aab4ec
$ gzip -c /tmp/ft_obs/q5_src.bin | wc -c
5000791
$ python3 /tmp/ft_obs/q5_edit.py /tmp/ft_obs/q5_src.bin 2500000 1024
modified src size=5000000 sha256=a66d5a3b35d3dd37310b24d1cc03e3565e81a5ebd56c62ee7fd2b8bd97ef53c3
```

The gzip size (5,000,791) exceeding the raw size confirms the content is incompressible, so blocks
are unique and rsync cannot match by accident [OBSERVED].

### The two transfers, with raw `print_rsync_stats`

Transfer #1 (baseline, `-x`, no basis) sends the whole file as delta; transfer #2 (`-x`, prior copy
as basis) sends only the change [OBSERVED]:

```
$ script -q --log-out /tmp/ft_obs/q5_out1.bin --log-in /tmp/ft_obs/q5_in1.bin -c "$KITTEN transfer -x /tmp/ft_obs/q5_src.bin /tmp/ft_obs/q5_dst.bin"
Rsync stats:
  Delta size: 5.0 MB Signature size: 0 B
$ script -q --log-out /tmp/ft_obs/q5_out2.bin --log-in /tmp/ft_obs/q5_in2.bin -c "$KITTEN transfer -x /tmp/ft_obs/q5_src.bin /tmp/ft_obs/q5_dst.bin"
Rsync stats:
  Delta size: 2.6 kB Signature size: 45 kB
  Transmitted: 47 kB of a total of 5.0 MB (0.9%)
```

Both transfers produced destinations whose SHA-256 matched their source (full:
`dd0c0eaa8d732ef292f0d8d9a44a8cbd176f2e5bec84ed947e97c37230aab4ec`; modified:
`a66d5a3b35d3dd37310b24d1cc03e3565e81a5ebd56c62ee7fd2b8bd97ef53c3`) [OBSERVED].

### Exact byte counts (parsed from the wire), and the corrected arithmetic

`print_rsync_stats` rounds for display, so the signature and delta were parsed from the actual
emitted bytes [OBSERVED]:

```
file_size=5000000 block_size=2236 edit_offset=2500000 -> edit_block=1118
signature raw payload bytes: 44752  (12-byte header 00 00 00 00 00 00 00 00 bc 08 00 00 => block_size=2236; records=2237; 12+2237*20=44752)
delta op sequence (uncompressed serialized bytes):
  OpBlockRange   blocks [0..1117] inclusive (1118 blocks matched)   13
  OpData         2236 literal bytes  (CHANGED block 1118)           2241
  OpBlockRange   blocks [1119..2235] inclusive (1117 blocks matched) 13
  OpData         304 literal bytes   (partial tail block 2236)      309
  OpHash         16-byte XXH3-128 integrity checksum                19
EXACT delta bytes (sum) = 2595
signature bytes = 44752  delta bytes = 2595  total = 47347
vs full file 5000000: total is 0.94694%  => 105.603x reduction
WIRE (zlib-compressed + base64 + OSC-framed) capture sizes: q5_out2.bin=3743 bytes  q5_in2.bin=60688 bytes
```

The exact figures are therefore: block size `round(sqrt(5,000,000)) = 2236`; `2237` signature
records; signature `12 + 2237*20 = 44,752` bytes; delta `2,595` bytes; total logical payload
`47,347` bytes, which is `0.94694%` of the file, a `105.603x` reduction [OBSERVED]. The edit at
offset 2,500,000 lands entirely inside a **single** block — block `2,500,000 // 2236 = 1118` (spanning
bytes 2,499,848 to 2,502,083); the edit occupies 2,500,000 to 2,501,023, so it does **not** straddle a
block boundary, and it appears as exactly one `OpData(2236)` in the op sequence [OBSERVED].

### Stability across two runs

With deterministic inputs, a second full run reproduced the first byte-for-byte [OBSERVED]:

```
RUN2 raw rsync stats:
Rsync stats:
  Delta size: 2.6 kB Signature size: 45 kB
  Transmitted: 47 kB of a total of 5.0 MB (0.9%)
RUN2 parsed: signature bytes = 44752  delta bytes = 2595  total = 47347
STABILITY vs RUN1: signature 44752==44752 True  delta 2595==2595 True  total 47347==47347 True
```

### The detecting mechanism and a metric qualification

Unchanged portions are detected by the sender's rsync match: it rolls the weak checksum byte-by-byte
and, on a weak-hash hit, confirms with the XXH3-64 strong hash before emitting a block reference;
matched blocks become `OpBlockRange`/`OpBlock` and only changed regions become `OpData`
[SOURCE-VERIFIED: tools/rsync/algorithm.go:543,556-557,641-643; see the Q2 match-mechanism section].
`print_rsync_stats` reports the **uncompressed logical rsync payload** (`p.total_transferred` and
`p.signature_bytes`), not the on-wire byte count
[SOURCE-VERIFIED: kittens/transfer/send.go:1252-1261; kittens/transfer/utils.go:109-114]. On the wire
the delta is additionally zlib-compressed and then base64-encoded and OSC-framed; for transfer #2 the
actual captured wire sizes were 3,743 bytes (client to terminal) and 60,688 bytes (terminal to
client), so the reported "47 kB" is the logical payload, not the TTY byte total [OBSERVED].

## Edge cases exercised (secondary paths, not just the happy path)

All edge cases were run through the real kitty receiver; each is shown with its command shape, the
observed `tt`, and a source/destination hash match [OBSERVED]:

```
EDGE A  fresh, no -x            : ac=file tt=(absent => simple)   rsync-stats ABSENT   full send   dst sha256=bf8e4e71585fd2e56be18def3077da6f6e0dae9581fc74e0018fe13b5cb3e9ff  MATCH
EDGE B  fresh, -x, no basis     : ac=file tt=rsync                stats present 100.0% dst sha256=83dd8abc2653f4a0edfeafae8028a01cc55f01aa9c6d6c6a5320115d8167779b  MATCH
EDGE C1 src=4096B (NOT > 4096)  : first -x => tt=simple; retransfer -x with basis => STILL tt=simple (gate blocks rsync)   MATCH
EDGE C2 src=8192B (> 4096)      : first -x => tt=rsync 100%; retransfer -x with basis => tt=rsync 24.1% transmitted        MATCH
EDGE D1 default compression     : ac=file zip=zlib   ; 18000B compressible text => 125 bytes on-wire data payload         MATCH
EDGE D2 --compress=never        : ac=file zip=(none) ; 18000B sent uncompressed                                           MATCH
```

- **Edge A/B (simple vs rsync path).** Without `-x` the `ac=file` frame has no `tt` field (simple
  path) and no rsync stats are printed; with `-x` but no basis, `tt=rsync` is offered but the
  signature is empty (0 B), so 100% is transmitted [OBSERVED].
- **Edge C1/C2 (the 4096-byte threshold, both gates).** The sender offers rsync only for a regular
  source larger than 4096 bytes [SOURCE-VERIFIED: kittens/transfer/send.go:131]; the receive
  direction has the symmetric gate [SOURCE-VERIFIED: kittens/transfer/receive.go:404-410]; and the
  server engages rsync only when an existing file is present and the sender requested it
  [SOURCE-VERIFIED: kitty/file_transmission.py:1026-1028]. At exactly 4096 bytes the file stays on the
  simple path even with `-x` and a basis; at 8192 bytes rsync engages and yields a real 24.1% delta —
  the small-file inefficiency the `-x` help text warns about [OBSERVED].
- **Edge 3 (interrupt then resume)** is covered in full in Q4 (two runs, PID capture, targeted kill,
  before/during/after state, hashes, transient patch file).
- **Edge 4 (refusal / EPERM).** The confirmation is a real `boss.confirm` overlay
  [SOURCE-VERIFIED: kitty/file_transmission.py:1191,1199-1202]; refusal drives EPERM
  [SOURCE-VERIFIED: kitty/file_transmission.py:203]. Accept and refuse were both exercised through the
  live UI [OBSERVED]:

```
ACCEPT: overlay 'The remote machine wants to send some files to this computer. Do you want to allow the transfer?'; typed 'y' => 'Permission granted for this transfer'
        src==dst sha256 b7566217c253a2d319a135ee29cd8e286eb115be0bd99d0fb066691be14583a7 (MATCH)
        113113 openat(AT_FDCWD, "/tmp/ft_obs/dst2.bin", O_RDWR|O_CREAT|O_TRUNC|O_CLOEXEC, 0644) = 11
REFUSE: typed 'n' => 'Permission denied for this transfer'; exit code 1; dst_refuse.bin NOT created (EPERM)
```

### Compression (`compression=zlib` / `zip=zlib`)

The only supported compression is RFC 1950 zlib deflate, selected via `compression=zlib`; the server
decompresses with `zlib.decompressobj(wbits=0)`
[SOURCE-VERIFIED: kitty/file_transmission.py:367-368] and the client compresses with `compress/zlib`
[SOURCE-VERIFIED: kittens/transfer/send.go:7,65]. As shown in Edge D1/D2, an 18,000-byte compressible
text file reduced to a 125-byte on-wire data payload by default and was 18,000 bytes uncompressed with
`--compress=never` [OBSERVED]. Compression eligibility is additionally gated by size and a MIME check
[SOURCE-VERIFIED: kittens/transfer/send.go:132].

## Coverage pass — every named mechanism, function, struct, condition, file, flag

| # | Named item | Where addressed | Basis |
|---|---|---|---|
| 1 | OSC=`1b 5d`, ST=`1b 5c`, code 5113 | Q1 wire format | OBSERVED + SOURCE (spec:543-552) |
| 2 | 5113 = numeralization of "file" | Q1 wire format | SOURCE (spec:543-552) |
| 3 | C then Python then Go constant propagation | Q3 one-definition | SOURCE (control-codes.h:233; data-types.c:596; go_code.py:575,597) |
| 4 | `FileTransmissionCommand.Serialize()` + OSC prefix | Q1 initiation | SOURCE (ftc.go:163-222; send.go:384,647-648) |
| 5 | Session id / file id correlation, action/status interleaving | Q1 ordering | OBSERVED |
| 6 | All 15 key abbreviations (ac/zip/ft/tt/id/fid/pw/q/mod/prm/sz/n/st/pr/d) | Q1 table | SOURCE (spec:558-576) |
| 7 | `pw` spec (safe_string) vs impl (base64) drift | Q1 drift note | SOURCE (spec:567; ftc.go:129; file_transmission.py:260) |
| 8 | `safe_string` excludes `;` | Q1 abbreviations | SOURCE (spec:589-601) |
| 9 | base64 RawStdEncoding, no padding | Q3 encoding | OBSERVED + SOURCE (ftc.go:178-189) |
| 10 | zlib RFC1950; distinct from delta | Compression; Q5 metric | OBSERVED + SOURCE (file_transmission.py:367-368; send.go:7,65) |
| 11 | `vt-parser.c` `FILE_TRANSFER_CODE` dispatch | Q3 chain / distinction | SOURCE (vt-parser.c:547-550) |
| 12 | `screen.c` callback bridge | Q3 chain | SOURCE (screen.c:2311-2312) |
| 13 | `window.py` forwarding | Q3 chain | SOURCE (window.py:1388-1389) |
| 14 | `handle_serialized_command` deserialize+dispatch only | Q3 chain | SOURCE (file_transmission.py:858) |
| 15 | `deserialize` base64-decodes `d=` and string fields | Q3 chain | SOURCE (file_transmission.py:329-351) |
| 16 | `DestFile.write_data`: decompress, open O_RDWR/O_CREAT/O_TRUNC, `af.write` | Q3 chain | SOURCE (file_transmission.py:510,542,546-547,550) |
| 17 | `PatchFile.write` (rsync patch path) | Q3 chain; Q4 | SOURCE (file_transmission.py:420-423) |
| 18 | `--transmit-deltas`/`-x`, `tt=rsync` | Q2 activation; Q4 | OBSERVED + SOURCE (main.py:116-122) |
| 19 | 12-byte LE signature header; block_size=round(sqrt(size)) | Q2 signature | OBSERVED + SOURCE (algorithm.c:204,226-238; api.go:274) |
| 20 | 20-byte `BlockHash` (index/weak/strong) | Q2 structures | OBSERVED + SOURCE (algorithm.go:177-204; algorithm.c:246-260) |
| 21 | Four delta ops (Block/Data/Hash/BlockRange), widths | Q2 delta ops | OBSERVED + SOURCE (spec:460-488; algorithm.go:104-140) |
| 22 | Rolling weak checksum (O(1) roll) | Q2 mechanism | SOURCE (algorithm.go:336-361) |
| 23 | XXH3-64 strong block confirmation | Q2 mechanism | SOURCE (algorithm.go:69-102,641-643) |
| 24 | XXH3-128 integrity | Q2 delta ops (OpHash) | OBSERVED + SOURCE (algorithm.go:40-66) |
| 25 | Moved/shifted-block support | Q2 moved-block | OBSERVED + SOURCE (algorithm.go:532-567; algorithm.c:719-747) |
| 26 | Send-side regular source > 4096 gate | Edge 2 | OBSERVED + SOURCE (send.go:131) |
| 27 | Receive-side existing basis > 4096 gate | Edge 2; Q4 | SOURCE (receive.go:404-410) |
| 28 | Server engages rsync only if existing file + sender requested | Edge 2 | SOURCE (file_transmission.py:1026-1028) |
| 29 | Partial destination as on-demand signature basis | Q4 | OBSERVED + SOURCE (algorithm.c:204) |
| 30 | No persistent resume sidecar vs transient patch file | Q4 | OBSERVED + SOURCE (file_transmission.py:392,405) |
| 31 | `print_rsync_stats` = uncompressed logical payload | Q5 metric | OBSERVED + SOURCE (send.go:1252-1261; utils.go:109-114) |
| 32 | Default launcher build/invocation, versions | Q6 | OBSERVED |
| 33 | SSH kitten makes transfer kitten available remotely | Q7 | OBSERVED + SOURCE (ssh.rst:22; ssh/main.py:167) |
| 34 | Confirmation accept and refusal (EPERM) via real UI | Edge 4 | OBSERVED + SOURCE (file_transmission.py:1191,1199-1202,203) |
| 35 | Two stable Q5 runs; source/dest hashes; ordinary-output control | Q5; Q3 distinction | OBSERVED |

## Observed vs inferred — summary

- **[OBSERVED]** items are backed by the literal artifacts above: the build banners and `nm`/`ldd`
  output; the Q7 `strace openat` and matching hashes; the Q1 bidirectional frame inventory and raw
  control-frame bytes; the Q2 signature/delta byte parses and the moved-block delta; the Q3 89-byte
  round-trip hashes and the normal-vs-transfer discriminator; the Q4 two-run interrupt/resume with
  hashes, `tt=rsync`, on-partial signature block sizes, and the transient temp-file `strace`; the Q5
  parsed 44,752 / 2,595 / 47,347 figures across two identical runs; and the edge-case
  `tt`/hash/stat captures.
- **[SOURCE-VERIFIED]** items are facts read from specific lines (the dispatch chain, the struct and
  op layouts, the block-size formula, the 4096 gates, the `pw` drift, the confirmation and EPERM
  paths).
- **[INFERRED]** is used sparingly and only where a conclusion follows from observed facts — for
  example, that ordinary output never enters the transfer branch because it lacks the OSC 5113
  wrapper.

## Repository read-only proof and cleanup

This investigation added exactly one file — this document — and modified no tracked source. The only
tracked difference from the baseline is this deliverable; the working tree has no modified tracked
files; and the rebuilt `rsync.so` is git-ignored, not tracked [OBSERVED]:

```
$ git diff --name-status 815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1
A	blitzy/documentation/kitty_815df1e210e0.md
$ git status --porcelain --untracked-files=no
$ git check-ignore kittens/transfer/rsync.so
kittens/transfer/rsync.so
```

All runtime evidence was produced under `/tmp/ft_obs` (outside the repository); the loopback `sshd`,
the `Xvfb`/`kitty` processes, and all task-created SSH keys and temporary files are torn down and
removed during cleanup, and no dependency or tracked file was altered to perform the investigation.
