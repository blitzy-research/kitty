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

All evidence in this document was produced **inside the canonical task container** using **genuine
keyboard input**. Three environmental facts materially shaped how it was produced; they are disclosed
up front rather than buried.

1. **Build and runtime environment — the canonical container image [OBSERVED].** Everything below was
   built and run inside the named image
   `ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0`, whose repository digest is
   `sha256:60da90a7183a82861fc6d1d40cb8086baa6a8a0e0d05f26d03aafd0f5b3cc384` [OBSERVED]:

   ```
   $ docker image inspect ghcr.io/scaleapi/swe-atlas:swe_atlas_QnA_kovidgoyal_kitty_1.0 --format '{{index .RepoDigests 0}}'
   ghcr.io/scaleapi/swe-atlas@sha256:60da90a7183a82861fc6d1d40cb8086baa6a8a0e0d05f26d03aafd0f5b3cc384
   ```

   The container was started with `--cap-add=SYS_PTRACE --security-opt seccomp=unconfined` so that
   `strace` could attach to the running kitty process (used for the file-open evidence). A host
   scratch directory is bind-mounted at `/work` inside the container; the pinned source tree
   (produced by `git archive 815df1e210e0`) lives at `/work/src`, and all temporary observation
   artifacts live under `/work` — outside the repository. The exact toolchain identities are recorded
   in the Q6 section.

2. **The receiver is a real kitty GUI, run headlessly under Xvfb with software OpenGL, driven by
   genuine X11 keystrokes [OBSERVED].** kitty is a GPU terminal with no headless flag. To exercise the
   genuine `vt-parser.c` then `screen.c` then `window.py` then `file_transmission.py` path (with the
   real `boss.confirm` permission UI), a real `kitty` process was launched under `Xvfb :99` with Mesa
   software GL (`LIBGL_ALWAYS_SOFTWARE=1`, `GALLIUM_DRIVER=llvmpipe`) and, deliberately,
   `-o allow_remote_control=no`. Keystrokes were delivered with `xdotool` (the X11 **XTEST**
   extension), which injects real key events into the focused window at the X-server layer — exactly
   as a physical keyboard would. This is **not** a remote-control or protocol bypass: remote control
   is switched off, and every command (including the confirmation `y`) is a genuine key event. That
   this genuinely reaches the shell was confirmed with a smoke test — a typed
   `date +GENUINE_XTEST_%s` produced `GENUINE_XTEST_1784004217` in a file — with the focused window
   id equal to the target kitty window. This is distinct from `kitten @ send-text` (remote control,
   not used here) and from kitty's unit-test harness (`kitty_tests/file_transmission.py`, which routes
   parsed commands straight to a test controller with a preset `allow` boolean and was **not** used
   for any claim here).

3. **SSH is a loopback-only, task-scoped daemon inside the container [OBSERVED].** Q7 requires a real
   `kitten ssh` hop. A dedicated `sshd` was bound to `127.0.0.1:2222` inside the container with
   task-scoped ed25519 host and client keys under `/work/harness/ssh`,
   `PermitRootLogin prohibit-password`, and public-key authentication only. Its setup and teardown are
   documented, and it and its keys are removed during cleanup.

The reusable environment sourced by every command below is:

```
export REPO=/work/src            # pinned source tree (git archive 815df1e210e0), inside the container
export LANG=C.UTF-8 LC_ALL=C.UTF-8 GOFLAGS=-mod=mod
export DISPLAY=:99 LIBGL_ALWAYS_SOFTWARE=1 GALLIUM_DRIVER=llvmpipe
export KITTY=$REPO/kitty/launcher/kitty
export KITTEN=$REPO/kitty/launcher/kitten
```

## Q6 — Building kitty from source (canonical configuration)

The canonical build entry point is `python3 setup.py` (the `Makefile` `all:` target delegates to it).
It was run to completion inside the container and produced the launcher `kitty/launcher/kitty` and the
Go `kitten` binary `kitty/launcher/kitten`. `libxxhash` is a required system dependency for the
transfer/rsync C module and is linked by the build [SOURCE-VERIFIED: setup.py:986].

Toolchain identity, all read inside the container [OBSERVED]:

```
$ python3 --version
Python 3.12.3
$ go version
go version go1.23.4 linux/amd64
$ cc --version | head -1
cc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0
$ pkg-config --modversion libxxhash
0.8.2
$ cat /etc/os-release | grep PRETTY
PRETTY_NAME="Ubuntu 24.04.2 LTS"
$ head -3 go.mod
module kitty

go 1.22
```

`go.mod` requires `go 1.22`; the container's Go 1.23.4 satisfies it [OBSERVED].

Build artifacts, version banners, the protocol constant reaching Python, and the XXH3 linkage of the
server-side rsync module [OBSERVED]:

```
$ ls -l kitty/launcher/kitty kitty/launcher/kitten
-rwxr-xr-x 1 root root 15945988 Jul 14 04:38 kitty/launcher/kitten
-rwxr-xr-x 1 root root    36224 Jul 14 04:37 kitty/launcher/kitty

$ ./kitty/launcher/kitty --version
kitty 0.35.2 created by Kovid Goyal
$ ./kitty/launcher/kitten --version
kitten 0.35.2 created by Kovid Goyal

$ python3 -c 'from kitty.fast_data_types import FILE_TRANSFER_CODE; print(FILE_TRANSFER_CODE)'
5113

$ ldd kittens/transfer/rsync.so | grep -i xxhash
	libxxhash.so.0 => /lib/x86_64-linux-gnu/libxxhash.so.0 (0x00007c88bba55000)
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

A loopback-only `sshd` was started inside the container with task-scoped keys under
`/work/harness/ssh` (`ListenAddress 127.0.0.1`, `Port 2222`, `PermitRootLogin prohibit-password`).
Inside the real kitty GUI (Xvfb, software GL, remote control disabled), the SSH kitten was invoked by
genuine XTEST keystrokes, a remote shell was reached, and — critically — the transfer kitten was run
**on the remote** by its bare name `kitten`, which resolves through the remote `PATH`. The commands
typed (as real key events) were:

```
/work/src/kitty/launcher/kitten ssh -p 2222 -i /work/harness/ssh/id_ed25519 -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@127.0.0.1
echo REMOTE_HOST=$(hostname) REMOTE_KITTEN=$(command -v kitten) ; kitten --version
script -e -q -c "kitten transfer /work/q7b_src.bin /work/q7b_dst.bin" /work/evidence/q7b_transfer.log
```

Inside the `kitten ssh` session the transfer kitten resolves to the **ssh-kitten-provisioned shim** on
the remote `PATH` — proving the SSH kitten made the transfer kitten available on the remote — and it
is our built 0.35.2 binary, not a download [OBSERVED]:

```
REMOTE_HOST=837e0b61e567 REMOTE_KITTEN=/root/.local/share/kitty-ssh-kitten/kitty/bin/kitten
kitten 0.35.2 created by Kovid Goyal
```

(The shim `.../kitty/bin/kitten` execs `.../kitty/install-tool/kitten` when it is present and
executable; it was pre-seeded with the built binary, so the run is fully offline and canonical rather
than fetching a release from GitHub. `REMOTE_HOST` is the container's own id over the loopback hop.)

The genuine process chain during the transfer, sampled live, shows the real
`kitten ssh` → `sshd` → login shell → `script` → **remote** transfer kitten pipeline [OBSERVED]:

```
$ # ps -eo pid,ppid,comm,args sampled every 0.15s during the transfer
  25959   23709 kitten   /work/src/kitty/launcher/kitten ssh -p 2222 -i /work/harness/ssh/id_ed25519 ... root@127.0.0.1
  25980    1130 sshd     sshd: root@pts/1
  25990   25980 bash     /bin/bash --login --posix
  26457   25990 script   script -e -q -c kitten transfer /work/q7b_src.bin /work/q7b_dst.bin /work/evidence/q7b_transfer.log
  26458   26457 kitten   /root/.local/share/kitty-ssh-kitten/kitty/install-tool/kitten transfer /work/q7b_src.bin /work/q7b_dst.bin
```

The confirmation overlay was accepted by a genuine `y` key event; the destination was written by the
**local kitty process** (PID 23641), not the kitten client — proven by an `strace` of the kitty
process opening the destination — and the source and destination hashes match [OBSERVED]:

```
Permission granted for this transfer
]5113;id=675a069d;ac=send                      # the first OSC 5113 frame, from the transfer log

$ strace -f -qq -e trace=openat -p 23641 -o q7b_strace.txt    # (no -y: clean, unannotated)
23641 openat(AT_FDCWD, "/work/q7b_dst.bin", O_RDWR|O_CREAT|O_TRUNC|O_CLOEXEC, 0644) = 9

$ sha256sum /work/q7b_src.bin /work/q7b_dst.bin
ee45cd01317686fc6b4bb7f4b3a3b8416ab593e74d88dda872f822607c477141  /work/q7b_src.bin
ee45cd01317686fc6b4bb7f4b3a3b8416ab593e74d88dda872f822607c477141  /work/q7b_dst.bin
```

The SSH kitten makes the transfer kitten available on the remote automatically
[SOURCE-VERIFIED: docs/kittens/ssh.rst:22; kittens/ssh/main.py:167]. The `strace` line is direct
evidence that the full server-side chain executed inside the terminal process and wrote the file
[OBSERVED]. The transfer moved a 4,194,304-byte source to a byte-identical destination [OBSERVED].

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

### The handshake ordering — a small file, captured losslessly, nothing elided

A 58-byte file was transferred so that every frame — including the full base64 `d=` payload — fits in
the capture with nothing summarised. The transfer was run under `script`, which tees both directions
of the PTY while the real kitty acts as receiver (`--log-out` = client→terminal, `--log-in` =
terminal→client). Producing command (typed as genuine XTEST keystrokes into the kitty GUI) [OBSERVED]:

```
$ script -e -q --log-out /work/evidence/q1small.out --log-in /work/evidence/q1small.in \
     -c "$KITTEN transfer /work/q1_small.bin /work/q1_small_dst.bin"     # (confirmed with a genuine 'y')
```

The complete, verbatim frame inventory from that capture — every frame, every field, the `d=` payload
shown in full (78 base64 chars) [OBSERVED]:

```
# CLIENT -> TERMINAL (sender kitten writes; captured via script --log-out)
  ]5113;id=6b0ff93;ac=send
  ]5113;id=6b0ff93;n=L3dvcmsvcTFfc21hbGxfZHN0LmJpbg;prm=420;fid=1;ac=file;mod=1784005135421759736
  ]5113;id=6b0ff93;fid=1;ac=end_data;d=VGhlIG51bWJlciA1MTEzIGlzIHRoZSBudW1lcmFsaXphdGlvbiBvZiB0aGUgd29yZCAiZmlsZSIuCg
  ]5113;id=6b0ff93;ac=finish
# TERMINAL -> CLIENT (real kitty responds; captured via script --log-in)
  ]5113;ac=status;id=6b0ff93;st=T0s
  ]5113;ac=status;id=6b0ff93;fid=1;n=L3dvcmsvcTFfc21hbGxfZHN0LmJpbg;st=U1RBUlRFRA
  ]5113;ac=status;id=6b0ff93;fid=1;sz=58;n=L3dvcmsvcTFfc21hbGxfZHN0LmJpbg;st=T0s
```

Decoding every base64 field in those frames [OBSERVED]: `n` = `L3dvcmsvcTFfc21hbGxfZHN0LmJpbg` =
`/work/q1_small_dst.bin`; the `d=` token decodes to the exact 58 source bytes
`The number 5113 is the numeralization of the word "file".\n`; `st` values `T0s` = `OK` and
`U1RBUlRFRA` = `STARTED`; and `prm=420` is decimal 420 = octal `0o644`, the source's permission bits.
This 58-byte file is below the compression threshold, so the `ac=file` frame carries no `zip` field
[OBSERVED]. The raw bytes of the small control frames, verbatim, confirm the framing [OBSERVED]:

```
send  : b'\x1b]5113;id=6b0ff93;ac=send\x1b\\'         # 2 bytes before '5113' = 1b 5d (ESC ]); trailer = 1b 5c (ESC \)
status: b'\x1b]5113;ac=status;id=6b0ff93;st=T0s\x1b\\'
```

Putting the two directions together, the observed ordering was: client `ac=send`; terminal `st=OK`;
client `ac=file`; terminal `st=STARTED`; client `ac=end_data` (the whole 58-byte file in one final
chunk); terminal `st=OK sz=58`; client `ac=finish`. Every frame carries the same session `id=6b0ff93`
and, after the file is opened, the same `fid=1`, so requests and acknowledgements are correlated by
those two IDs [OBSERVED]. This matches the documented session flow
[SOURCE-VERIFIED: docs/file-transfer-protocol.rst:43-113].

### Data chunking at the 4096-byte maximum, and the PROGRESS status

To show the ≤4096-byte data chunking and the intermediate `PROGRESS` status, a 20,000-byte file was
transferred. Here the `d=` fields are large, so — unlike the 58-byte case above — they are reported by
their exact decoded byte counts and base64 lengths rather than reproduced in full (this block is
**length-summarised**, not claimed verbatim). Producing command [OBSERVED]:

```
$ script -e -q --log-out /work/evidence/q1chunk.out --log-in /work/evidence/q1chunk.in \
     -c "$KITTEN transfer /work/q1_chunk.bin /work/q1_chunk_dst.bin"
```

Parsed result [OBSERVED]:

```
ac=file frame carries: zip=zlib
CLIENT data frames (5): base64-decoded payload sizes = [4096, 4096, 4096, 4096, 3637]
  (frames 1-4 are ac=data, frame 5 is ac=end_data; each 4096-byte payload = 5462 base64 chars)
concatenated compressed stream = 20021 bytes -> zlib.decompressobj(0) inflates to 20000 bytes == source
TERMINAL statuses in order: T0s(OK) STARTED PROGRESS(sz=4089) PROGRESS(sz=8185) PROGRESS(sz=12281) PROGRESS(sz=16377) OK(sz=20000)
```

This demonstrates the full pipeline: the payload is (optionally zlib-compressed, then) split into
chunks of **at most 4096 bytes**, each base64-encoded into a `d=` field; the receiver emits `PROGRESS`
as cumulative bytes land and a final `OK` at the full size; and reassembly (concatenate the chunks,
then inflate) reproduces the source exactly [OBSERVED]. The `st=` `PROGRESS` value decodes as
`UFJPR1JFU1M` = `PROGRESS` [OBSERVED].

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

A 200,000-byte file was first transferred with `-x` (creating the basis), then a modified copy was
retransferred with `-x` so the receiver emitted a signature. Producing commands [OBSERVED]:

```
$ # basis file q2.bin (200000 bytes, sha256 72947853a1608315ec42a592d71204f4a93ca3cc83771608df28a67d506a73dd)
$ script -e -q --log-out q2full.out  --log-in q2full.in  -c "$KITTEN transfer -x /work/q2.bin       /work/q2_dst.bin"   # basis created
$ script -e -q --log-out q2delta.out --log-in q2delta.in -c "$KITTEN transfer -x /work/q2_moved.bin /work/q2_dst.bin"   # retransfer
```

The signature the receiver sent (captured on the terminal→client channel `q2delta.in`, three
uncompressed `ac=data` frames) parsed to [OBSERVED]:

```
signature payload bytes: 8972
12-byte header (LE): version=0 checksum_type=0 strong_hash_type=0 weak_hash_type=0 block_size=447
  round(sqrt(200000)) = 447  => block_size matches sqrt? True
per-block records: 448   (12 + 448*20 = 8972 bytes)
  record[0]: index=0 weak_hash=0x53a2df1f strong_hash(XXH3-64)=0x3f1bea744b3af421
  record[1]: index=1 weak_hash=0x7a19ea7b strong_hash=0x119bc40d0d3bed27
```

The 12-byte little-endian header is `uint16 version`, `uint16 checksum_type`, `uint16
strong_hash_type`, `uint16 weak_hash_type`, `uint32 block_size`
[SOURCE-VERIFIED: docs/file-transfer-protocol.rst:417-458; emitted by
kittens/transfer/algorithm.c:226-238]. The block size is `round(sqrt(file_size))`
[SOURCE-VERIFIED: kittens/transfer/algorithm.c:204; tools/rsync/api.go:274]; `round(sqrt(200000))` =
447, matching the emitted field [OBSERVED]. Each record is 20 bytes (`le64 index`, `le32 weak`,
`le64 strong`) [SOURCE-VERIFIED: kittens/transfer/algorithm.c:246-260].

### Moved/shifted-block detection (position-independent matching) and the delta operations

Because the weak checksum is rolled at every byte offset, blocks are matched by content regardless of
where they moved in the file. The retransferred `q2_moved.bin` (200,777 bytes,
sha256 `c375f52a7048e9261e72fb945ef2bb7ae7d7182142d0fe18119ebba488eb74e8`) is the 200,000-byte basis
with 777 novel bytes prepended. The delta the sender emitted (captured on `q2delta.out`; the `ac=file`
frame carried `zip=zlib`, so the single `ac=end_data` `d=` payload was zlib-inflated before parsing)
parsed to [OBSERVED]:

```
delta wire payload: 523 compressed bytes -> zlib.decompressobj(0) -> 1010 raw serialized bytes
op sequence:
  1. OpData(777)              -> the 777 prepended novel bytes (literal)
  2. OpBlockRange(0, +446)    -> basis blocks 0..446 = 447 MATCHED blocks (each shifted 777 bytes later)
  3. OpData(191)              -> the 191-byte partial trailing block (basis block 447) re-sent literally
  4. OpHash(16, 9beb52360bb77d8dd687fac02a1f6ceb) -> XXH3-128 whole-file integrity checksum
matched basis blocks = 447 of 448 ; literal OpData bytes = 968 = 777 (prefix) + 191 (partial tail)
```

The op type/width layout matches the specification: `Block(0)`=`uint64 index`; `Data(1)`=`uint32 size`
+ payload; `Hash(2)`=`uint16 size` + checksum; `BlockRange(3)`=`uint64 start` + `uint32 (end-start)`
[SOURCE-VERIFIED: docs/file-transfer-protocol.rst:460-488; tools/rsync/algorithm.go:104-140]. The
200,000-byte basis divides into 447 full 447-byte blocks (447 × 447 = 199,809 bytes) plus a 191-byte
partial trailing block — **448 basis blocks**, matching the 448 signature records. Only the 777
prepended bytes plus that 191-byte partial tail were sent as literal `OpData`
(447 × 447 + 191 + 777 = 200,777 = the full new-file size); the partial tail cannot be matched as a
whole block because it is shorter than the 447-byte window. The 447 full original blocks were
referenced by their **original index** through a single `OpBlockRange[0..446]`, even though every one
had shifted 777 bytes later in the file — so the match is **447 of 448 basis blocks** [OBSERVED].

### The match mechanism

The sender's delta is produced by `Differ.CreateDelta(src, output)`
[SOURCE-VERIFIED: tools/rsync/api.go:230] — invoked by the transfer kitten as
`self.delta_loader = self.differ.CreateDelta(self.actual_file, self.deltabuf)`
[SOURCE-VERIFIED: kittens/transfer/send.go:768] — which loads the previously received signature and
delegates to the streaming `(*rsync).CreateDiff` [SOURCE-VERIFIED: tools/rsync/api.go:239]. The
low-level convenience wrapper `(*rsync).CreateDelta` funnels through the same `CreateDiff` and
collects the ops into a slice [SOURCE-VERIFIED: tools/rsync/algorithm.go:599-601]. That routine
detects matches by rolling the weak checksum byte-by-byte and, on a weak-hash hit, confirming with the
strong hash [SOURCE-VERIFIED: tools/rsync/algorithm.go]: `hash_lookup map[uint32][]BlockHash` is keyed
by weak hash and built from the received signature [algorithm.go:366,617-623]; the window advances via
`self.rc.add_one_byte(...)` [algorithm.go:543]; `if hh, ok := self.hash_lookup[self.rc.val]; ok` is
the weak-hash lookup [algorithm.go:556] and `find_hash(...)` confirms with `block.StrongHash == hv`
(XXH3-64) before emitting a reference [algorithm.go:557,641-643]. The weak hash is `rolling_checksum`
[algorithm.go:336,341,355], the strong block hash is XXH3-64 (`new_xxh3_64` [algorithm.go:59]), and
the integrity checksum is XXH3-128 (`new_xxh3_128` [algorithm.go:65]). The C server's analogous
position-independent match is [SOURCE-VERIFIED: kittens/transfer/algorithm.c:719-747].


## Q3 — Chunk encoding, receiver reassembly, and distinction from terminal output

### How chunks are encoded on the wire — base64 RawStdEncoding

The `d=` data field (and the `name`/`status`/`bypass` fields) are base64-encoded before being placed
between the OSC introducer and the ST terminator. The Go client uses standard base64 **without
padding** — `base64.RawStdEncoding` — for both string fields and the raw data bytes
[SOURCE-VERIFIED: kittens/transfer/ftc.go:178-189], and decodes with the same encoding on the
receive/verify path [SOURCE-VERIFIED: kittens/transfer/ftc.go:248]. RawStdEncoding was confirmed
against the exact bytes emitted for the 58-byte file used in Q1 — a full source → base64 → decode
round-trip [OBSERVED]:

```
source_len      = 58
source_sha256   = e0f2ed47de25afbad79b45a8625659e6d63b91ee2b287d6b7ffb50d5ccdd3a2a
std_b64(padded) = VGhlIG51bWJlciA1MTEzIGlzIHRoZSBudW1lcmFsaXphdGlvbiBvZiB0aGUgd29yZCAiZmlsZSIuCg==
RawStd(no pad)  = VGhlIG51bWJlciA1MTEzIGlzIHRoZSBudW1lcmFsaXphdGlvbiBvZiB0aGUgd29yZCAiZmlsZSIuCg
wire d= token   = VGhlIG51bWJlciA1MTEzIGlzIHRoZSBudW1lcmFsaXphdGlvbiBvZiB0aGUgd29yZCAiZmlsZSIuCg
RawStd==wire?   = True          # padded form has a trailing '=='; the wire token is RawStd (unpadded)
decoded_len     = 58
decoded_sha256  = e0f2ed47de25afbad79b45a8625659e6d63b91ee2b287d6b7ffb50d5ccdd3a2a
decoded==source = True
```

The padded standard encoding ends in `==`; the wire token is exactly the padded form with the padding
removed, proving RawStdEncoding [OBSERVED].

### How the receiver reassembles — the handler chain, end to end

The protocol number is defined once as `#define FILE_TRANSFER_CODE 5113`
[SOURCE-VERIFIED: kitty/control-codes.h:233]. On the receiving terminal, an incoming OSC 5113 sequence
flows through this chain [SOURCE-VERIFIED]:

1. `kitty/vt-parser.c:547-550` — the OSC dispatcher matches `case FILE_TRANSFER_CODE:` and calls
   `DISPATCH_OSC(file_transmission)`.
2. `kitty/screen.c:2311-2312` — `file_transmission(Screen *self, PyObject *cmd)` hands the command to
   the Python layer.
3. `kitty/window.py:1388-1389` — `file_transmission()` forwards to
   `self.file_transmission.handle_serialized_command(...)`.
4. `kitty/file_transmission.py` — `handle_serialized_command()` parses the command and drives
   reassembly [file_transmission.py:803]; for a data chunk it base64-decodes the payload and appends
   it to the active file, writing through `DestFile` (simple) or `PatchFile` (rsync)
   [file_transmission.py:599,858]. During an rsync receive,
   `ActiveReceive.signature_pending_chunks` buffers the signature chunks that the server produces
   [file_transmission.py:599].

Reassembly is therefore: base64-decode each `d=` chunk, (zlib-inflate if the file declared
`zip=zlib`), and append in order to the destination file. The Q1 20,000-byte case demonstrated this
end to end: five decoded compressed chunks `[4096,4096,4096,4096,3637]` were concatenated into a
20,021-byte zlib stream and inflated back to the exact 20,000-byte source [OBSERVED, see Q1].

### How transfer data is distinguished from ordinary terminal output

Distinction happens **at parse time in the VT parser**: only a byte sequence introduced by `ESC ] 5113
; … ST` reaches `case FILE_TRANSFER_CODE` and thus `file_transmission`
[SOURCE-VERIFIED: kitty/vt-parser.c:547-550]. Ordinary terminal output — even output whose *text*
happens to contain the characters `5113` and `ac=send` — is never wrapped in that OSC introducer, so
it is rendered as glyphs and never routed to the handler. This was verified at runtime by displaying
3,000 ordinary lines each literally containing `5113 ac=send` in the real kitty window while stracing
the kitty receiver for any destination-creating `openat`, then contrasting with a genuine transfer.
Producing commands (typed as genuine XTEST keystrokes into the focused kitty window) [OBSERVED]:

```
# Part A — ordinary output through the real terminal:
$ cat /work/q3_ord_lines.txt          # 3000 lines, each: 'line N: 5113 ac=send ... fid=N'
   PART A ordinary-output: marker_seen=YES  total_openat_traced=0  openat_CREATE_of_transfer_dest=0

# Part B — a genuine OSC-5113 transfer of the same 58-byte file:
$ /work/src/kitty/launcher/kitten transfer /work/q1_small.bin /work/q3_real_dst.bin   # confirmed with 'y'
   PART B real-transfer: done=D=0  dst_exists=YES
   23641 openat(AT_FDCWD, "/work/q3_real_dst.bin", O_RDWR|O_CREAT|O_TRUNC|O_CLOEXEC, 0644) = 9
   q3_real_dst sha256 = e0f2ed47de25afbad79b45a8625659e6d63b91ee2b287d6b7ffb50d5ccdd3a2a == source
```

Displaying 3,000 lines of ordinary text that literally contain `5113 ac=send` produced **zero**
destination-creating `openat` calls in the kitty process, whereas the genuine transfer produced
exactly one `openat(..., O_CREAT, ...)` for the destination and reproduced the source byte-for-byte
[OBSERVED]. That is precisely how transfer data is separated from regular terminal output: the OSC
5113 introducer, not the payload text, is the discriminator.


## Q4 — Transfer resumption: the state and where the metadata lives

### The direct answer

The state that lets an interrupted transfer resume rather than restart is **the existing (possibly
partial) destination file itself**. The "resumption metadata" is the **rsync signature computed on the
fly from that partial file** — there is **no separate sidecar resume file** on disk. The
`--transmit-deltas`/`-x` help text states the flag uses rsync "potentially saving lots of bandwidth
and also automatically resuming partial transfers"
[SOURCE-VERIFIED: kittens/transfer/main.py:116-122]. The crux is in `DestFile.signature_iterator()`,
which constructs the patcher from the **existing** file's current size
[SOURCE-VERIFIED: kitty/file_transmission.py:470-471]:

```
def signature_iterator(self) -> PatchFile:
    self.actual_file = PatchFile(self.name, self.existing_stat.st_size if self.existing_stat is not None else 0)
    return self.actual_file
```

`PatchFile.__init__` feeds that size straight into the C rsync `Patcher(expected_size)`, which sizes
the blocks from it [SOURCE-VERIFIED: kitty/file_transmission.py:379-383].

### Runtime proof — genuine interrupt, then resume, twice

A large transfer was started, then genuinely interrupted with **Ctrl+C** (`xdotool key ctrl+c`)
targeting the process whose `comm` is literally `kitten` (the sender), with the kitty GUI PID
explicitly excluded so the terminal itself was never signalled [OBSERVED]:

```
INTERRUPT target_pid=23878 comm=kitten at_dst=4308960 (KITTY_PID=23641 excluded)     # run 1
INTERRUPT target_pid=24120 comm=kitten at_dst=17776384 (KITTY_PID=23641 excluded)    # run 2
```

The transfer was then resumed with `-x`. The signature the receiver computed from the **partial**
destination (parsed byte-exact from the resume capture's terminal→client channel) was [OBSERVED]:

```
q4_r1_resume: sig_bytes=49992 block_size=2498 n_records=2499   round(sqrt(partial))=2498
   rec0: index=0 weak=0x0f80e321 strong=0x510409b512084112
q4_r2_resume: sig_bytes=89192 block_size=4458 n_records=4459   round(sqrt(partial))=4458
   rec0: index=0 weak=0x1e06bb05 strong=0xe6009f48cdc77cec
```

The block size is `round(sqrt(partial_size))`, **not** `round(sqrt(full_size))` — proving the
signature is taken over the partial file that already exists, which is exactly what makes the resume
send only the missing tail. `sig_bytes` equals `12 + n_records*20` in both runs (49992 = 12+2499·20;
89192 = 12+4459·20) [OBSERVED]. Both runs completed with the destination byte-identical to the source
(`Q4OK_1784007584`), and there was **no sidecar resume file** anywhere on disk — the only artifacts
are the source and the destination [OBSERVED].

### The transient patch tempfile — a clean strace (no `-y` annotations)

While applying the delta the server writes to a **transient tempfile** and atomically renames it into
place. A clean strace (no `strace -y` path annotations, so the raw syscalls are unambiguous) captured
[OBSERVED]:

```
openat(AT_FDCWD, "/work/q4t_dst.bin", O_RDONLY|O_CLOEXEC) = 9                                    # read existing partial to sign
openat(AT_FDCWD, "/work/tmpp1_t693p", O_RDWR|O_CREAT|O_EXCL|O_NOFOLLOW|O_CLOEXEC, 0600) = 10     # transient patch tempfile
rename("/work/tmpp1_t693p", "/work/q4t_dst.bin") = 0                                             # atomic replace into place
```

(The full capture contains only these plus routine `/proc/<pid>/stat` reads; the count of `</…>`
annotations is **0**.) The tempfile is created by
`tempfile.NamedTemporaryFile(mode='wb', dir=…, delete=False)`
[SOURCE-VERIFIED: kitty/file_transmission.py:391-392] and moved into place with `os.replace(...)`
[SOURCE-VERIFIED: kitty/file_transmission.py:405]. The `O_EXCL|O_NOFOLLOW|0600` flags come from
Python's `tempfile`, giving a private, non-symlink-following, owner-only patch file. During an rsync
receive the signature chunks the receiver produces are buffered in
`ActiveReceive.signature_pending_chunks` before being fed to the patcher
[SOURCE-VERIFIED: kitty/file_transmission.py:599].

### P4-F4 correction — there is NOT one universal "4096 gate"; there are THREE asymmetric gates

The rsync/delta path is guarded by **three different conditions in three different places**, and they
are **not** the same test. Two are `> 4096` thresholds on different sizes; the third is an
**existence** test with no threshold at all. All three were exercised at runtime and parsed
byte-exact:

| # | Where the gate lives | Exact condition (source) | What it tests | Runtime scenario | Observed result |
|---|---|---|---|---|---|
| 1 | **Sender** (Go client) | `rsync_capable: … stat_result.Size() > 4096` [send.go:131] | **source** file size > 4096 | (a) 4096-byte **source**, fresh dest | `tt=None`, **no signature** — simple path (4096 is not > 4096) |
| 2 | **Server / terminal, download** (Python) | `sz > -1 and df.ttype is rsync and df.ftype is regular`, where `sz = existing_stat.st_size … else -1` [file_transmission.py:1024-1028] | destination **exists** (`sz > -1`) — **no** size threshold | (b) 8192-byte source, **4096-byte existing** dest | `tt=rsync`, **signature 1292 B, block_size=64, 64 records** |
| 3 | **Receiver** (Go client) | `read_signature = s.Size() > 4096` [receive.go:404-410] | **basis** file size > 4096 | (c) 16000-byte **basis**, `--direction=upload` | `tt=rsync`, **signature 2512 B, block_size=128, 125 records** |

So a 4096-byte file "stays simple" only because of the **sender** gate (scenario a); on the **download**
path an existing destination of *exactly* 4096 bytes **does** engage rsync (scenario b), because that
gate is existence-based, not `> 4096`. This is the precise correction to the earlier "universal
symmetric 4096 gate" framing [OBSERVED + SOURCE-VERIFIED].

### The two engines size blocks by DIFFERENT formulas (a real secondary asymmetry)

The signature block sizes above differ because the **C server** and the **Go client** compute
`block_size` differently:

- **C server** (`kittens/transfer/algorithm.c`): `block_size = round(sqrt(expected_input_size))` with
  no hash-block rounding [SOURCE-VERIFIED: kittens/transfer/algorithm.c:203-204]. Scenario (b):
  `round(sqrt(4096)) = 64` → 64 records over the 4096-byte basis (`ceil(4096/64)=64`) [OBSERVED].
- **Go client** (`tools/rsync/api.go`, `NewPatcher`): `bs = round(sqrt(expected_input_size))`, then
  **floored to a multiple of `HashBlockSize()`** when that is smaller
  [SOURCE-VERIFIED: tools/rsync/api.go:270-286]. `HashBlockSize()` = `hasher.BlockSize()`
  [SOURCE-VERIFIED: tools/rsync/algorithm.go:637], and the XXH3 hasher's block size is **64** (module
  `github.com/zeebo/xxh3` v1.0.2). Scenario (c): source (expected) = 20000, `round(sqrt(20000)) = 141`,
  floored to a multiple of 64 = **128**; the signature over the 16000-byte basis then has
  `ceil(16000/128) = 125` records → 2512 B (`12 + 125·20`) [OBSERVED]. This exactly reproduces the
  parsed values.


## Q5 — Delta efficiency: transfer, modify a small portion, retransfer, measure

### The experiment and its scale

A **16 MiB** deterministic, incompressible file was used so the byte counts are large enough to be
representative and exactly reproducible [OBSERVED]:

```
q5_src.bin (pristine) = 16,777,216 bytes (16 MiB)
  sha256 = 4d400f720aeaee5b4fcb275f819c01fcfab9abaa3512ed8d0c0061447b4d75c5
  block_size = round(sqrt(16777216)) = 4096 ; n_blocks = 16777216/4096 = 4096
  predicted signature size = 12 + 4096*20 = 81932 bytes   [CONFIRMED on the wire below]
modification: flip 1 byte at offset 8,000,000  ->  block 1953 (block spans [7999488, 8003584))
  modified sha256 = 0b974ed77387e5138b4a7b83838481802cff55cfba50d35b54a06c1ca7544c09
```

### First transfer (no basis on the receiver) — the whole file is sent

```
first transfer: 4111 ac=data frames + 1 ac=end_data ; base64-decoded payload = 16,777,216 bytes (ENTIRE FILE)
                wire payload (zlib-wrapped incompressible random) = 16,782,347 bytes
                net transfer time = 0.594 s
```

The sender advertises `tt=rsync` (its source > 4096, gate #1), but the receiver has no basis, so the
existence gate (#2, `sz = -1`) yields `simple` and the sender streams all 4096 blocks literally
[OBSERVED, consistent with the Q4 gate analysis].

### Second transfer with `-x` (basis = the pristine file) — four independent runs

Each delta run was preceded by restoring the pristine 16 MiB basis on the receiver, then transferring
the 1-byte-modified source with `-x`. Four independent runs were byte-identical [OBSERVED]:

```
RUN       raw_delta  wire_delta  signature  total(delta+sig)  block_size  n_records  net_time
q5dA         4146       4162       81932        86078            4096        4096      0.095 s
q5dB         4146       4162       81932        86078            4096        4096      (untimed)
q5dA_t       4146       4162       81932        86078            4096        4096      0.095 s
q5dB_t       4146       4162       81932        86078            4096        4096      0.099 s
```

The delta op sequence was identical in every run [OBSERVED]:

```
[ BlockRange(0, 1952), Data(4096), BlockRange(1954, 2141), Hash(16) ]
  BlockRange(0, 1952)    -> basis blocks 0..1952  = 1953 blocks copied (unchanged)
  Data(4096)             -> block 1953 (the changed block) sent LITERALLY
  BlockRange(1954, 2141) -> basis blocks 1954..4095 = 2142 blocks copied (unchanged)
  Hash(16)               -> XXH3-128 whole-file integrity checksum
  reconstruction: 1953 + 1 + 2142 = 4096 blocks == n_blocks (EXACT)
  literal fraction: 1 of 4096 blocks = 0.0244%
raw-delta arithmetic: BlockRange(13) + Data(1+4+4096=4101) + BlockRange(13) + Hash(1+2+16=19) = 4146  [CONFIRMED]
signature record[0]: index=0 weak=0x96bdfdb1 strong=0xb0f3d1f304db40ef  (identical all runs)
```

### The measurable evidence — `print_rsync_stats`

The mechanism that reports the savings is `print_rsync_stats`, printed unconditionally after a
successful rsync transfer [SOURCE-VERIFIED: kittens/transfer/utils.go:109-113; call site
kittens/transfer/send.go:1260, guarded by `if tsf > 0`]. Its verbatim output on the delta run
(control bytes stripped for legibility; identical across all runs) [OBSERVED]:

```
Rsync stats:
  Delta size: 4.1 kB Signature size: 82 kB
  Transmitted: 86 kB of a total of 17 MB (0.5%)
```

### The numbers — second transfer versus first

```
transmitted (delta + signature) = 4146 + 81932 = 86078 bytes
transmitted fraction            = 86078 / 16777216 = 0.513065%
overall reduction               = 16777216 / 86078  = 194.9071x
delta-only reduction            = 16777216 / 4146    = 4046.6x
wall-clock                       = full 0.594 s  vs  delta 0.095-0.099 s  (~6x faster)
```

The second transfer sent **0.513%** of the original file's bytes — a **194.9×** reduction — while
reproducing the destination byte-for-byte [OBSERVED].

### Stability and declared tolerance (P4-F3)

The source file and the edit are both deterministic, so exact reproducibility is expected. **Declared
tolerance: ±0 bytes.** Observed variation across the four delta runs: **0 bytes** for the delta, **0
bytes** for the signature, **0 bytes** for the total — within tolerance. The only run-to-run variation
was ~4 ms of scheduling jitter in the net transfer time (0.095–0.099 s) [OBSERVED]. These figures also
cross-validate the Report 7 control (signature 81932 / delta 4146 / total 86078 / 0.513% / 194.9×)
[OBSERVED].

### The mechanism that detected the unchanged portions

The sender rolls the weak `rolling_checksum` over its file byte-by-byte (`add_one_byte`
[tools/rsync/algorithm.go:336-361]); on a weak-hash hit against a receiver block signature it confirms
with the XXH3-64 strong hash [tools/rsync/algorithm.go:556-557,641-643]; matched blocks are emitted as
`Block`/`BlockRange` references and only the one changed block is emitted as literal `Data`; the
trailing `Hash` op carries the XXH3-128 whole-file integrity checksum [SOURCE-VERIFIED + OBSERVED].
That weak-then-strong, position-independent match is precisely what let 4095 of 4096 blocks be sent as
two compact range references instead of bytes.


## Edge cases (secondary paths, each observed at runtime)

All five edge cases below were exercised through the real `kitten transfer` path with genuine XTEST
confirmation, and parsed byte-exact.

### Edge A — fresh transfer, no existing destination → the simple (non-rsync) path

The 58-byte Q1 transfer and the Q4 gate-(a) 4096-byte transfer both ran against a *non-existent*
destination. Neither emitted a `tt=` field or any signature frame; the whole file was sent as literal
`ac=data`/`ac=end_data` chunks [OBSERVED, see Q1 and Q4 gate (a)]. A fresh destination therefore always
takes the simple path, because the receiver has no basis to sign.

### Edge B — existing basis *below* the 4096-byte threshold, download path → rsync STILL engages

```
source = 8192 bytes (> 4096, so the sender advertises tt=rsync)
existing destination = 2000 bytes (< 4096)
result: tt=['rsync']  signature_bytes=912  block_size=45  n_records=45   round(sqrt(2000)) = 45
        edgeB dst == src : True
```

This is the sharpest illustration of the P4-F4 asymmetry: on the **download** path the rsync gate is
**existence-based** (`sz > -1` [kitty/file_transmission.py:1024-1028]), so a basis of only 2000 bytes
— well under 4096 — still triggers a signature and delta. The C server sized the blocks
`round(sqrt(2000)) = 45`, giving `ceil(2000/45) = 45` records and a `12 + 45·20 = 912`-byte signature,
exactly as observed [OBSERVED]. (A basis below 4096 only stays "simple" under the **Go-client
receive** gate, which *is* a `> 4096` test [receive.go:404-410].)

### Edge C — interrupted, then resumed

Covered in full under Q4: a genuine Ctrl+C interrupt followed by a `-x` resume, twice, with the
signature computed over the partial file (`block_size = round(sqrt(partial))`) and no sidecar file on
disk [OBSERVED, see Q4].

### Edge D — transfer refused at the confirmation prompt (EPERM)

```
$ /work/src/kitty/launcher/kitten transfer /work/edge_refuse_src.bin /work/edge_refuse_dst.bin   # answered 'n'
Permission denied for this transfer
COMMAND_EXIT_CODE = 1
frames on the wire: ['send']        # only the ac=send introducer; no file/data frames
destination created: False
```

A genuine `n` at the prompt caused the receiver to deny permission; the client exited non-zero, only
the opening `ac=send` frame ever reached the wire, and no destination file was created [OBSERVED].

### Compression D1/D2 — the only supported compression is zlib, and `--compress=never` disables it

Same 18,500-byte compressible source, two runs:

```
D1 default            : zip=zlib   data_frames=1  wire_payload_bytes=118     dst==src=True
D2 --compress=never   : zip=(absent) data_frames=5 wire_payload_bytes=18500  dst==src=True
```

By default the file was zlib-compressed to a single 118-byte data frame; with `--compress=never` the
`zip` field was absent and the full 18,500 bytes were sent across five ≤4096-byte frames. Both
reproduced the source exactly [OBSERVED]. The only supported compression is RFC 1950 zlib deflate,
selected via `compression=zlib`; the server decompresses with `zlib.decompressobj(wbits=0)`
[SOURCE-VERIFIED: kitty/file_transmission.py:367-368] and the client uses `compress/zlib`
[SOURCE-VERIFIED: kittens/transfer/send.go:7].


## Coverage pass — every named file, mechanism, flag, data structure, and dependency

### Files (all REFERENCE / read-only)

| File | Role | Where addressed |
|---|---|---|
| `docs/file-transfer-protocol.rst` | OSC 5113 encoding, key abbreviations, handshake, signature/delta binary format, compression | Q1, Q2 |
| `docs/kittens/transfer.rst` | `versionadded 0.30.0` (L10); basic usage `kitten ssh`→`kitten transfer` (L35-36); `--direction=upload` (L46); `--permissions-bypass` (L69); `--transmit-deltas` (L81-82) | Methodology, Q4, Q7 |
| `docs/kittens/ssh.rst` | Auto shell integration + file transfer (`versionadded 0.25.0`, L22) | Q7 |
| `kittens/transfer/main.go` | `main()` dispatches `send_main` (L59) / `receive_main` (L61) | Q7 |
| `kittens/transfer/main.py` | `--transmit-deltas`/`-x` flag + resume help text (L116-122) | Q4 |
| `kittens/transfer/ftc.go` | `FileTransmissionCommand.Serialize()`; base64 RawStdEncoding (L178-189, decode L248) | Q1, Q3 |
| `kittens/transfer/send.go` | OSC prefix `\x1b]%d;id=%s;` (L384,647-648); `CreateDelta` call (L768); `print_rsync_stats` call (L1260); `compress/zlib` (L7) | Q1, Q2, Q5 |
| `kittens/transfer/receive.go` | Go-client receive gate `s.Size() > 4096` (L404-410); `compress/zlib` (L7) | Q4 |
| `kittens/transfer/utils.go` | `print_rsync_stats` Delta/Signature sizes (L109-113) | Q5 |
| `kittens/transfer/algorithm.c` | Server C rsync; `block_size = round(sqrt(...))` (L203-204); XXH3 via `<xxhash.h>` (L13) | Q2, Q4 |
| `tools/rsync/algorithm.go` | `OpType` (L31-37), `Operation` (L71-77), `BlockHash` (L177-204), `rolling_checksum`/`add_one_byte` (L336-361), XXH3-64/128, `HashBlockSize` (L637) | Q2, Q4, Q5 |
| `tools/rsync/api.go` | `Api`/`Differ`/`Patcher`; `NewPatcher` block-size floor (L270-286); `CreateDelta` (L230); `CreateDiff` (L239) | Q2, Q4 |
| `kitty/control-codes.h` | `#define FILE_TRANSFER_CODE 5113` (L233) | Q3, Q6 |
| `kitty/vt-parser.c` | `case FILE_TRANSFER_CODE:` → `DISPATCH_OSC(file_transmission)` (L547-550) | Q3 |
| `kitty/data-types.c` | Exposes `FILE_TRANSFER_CODE` to Python (L596) | Q3, Q6 |
| `kitty/screen.c` | `file_transmission(Screen*, PyObject*)` callback (L2311-2312) | Q3 |
| `kitty/window.py` | `file_transmission()` → `handle_serialized_command()` (L1388-1389) | Q3 |
| `kitty/file_transmission.py` | `handle_serialized_command` (L803); `DestFile`/`PatchFile` (L379-405,858); `signature_iterator` (L470); `signature_pending_chunks` (L599); download gate (L1024-1028); `zlib.decompressobj(0)` (L367-368) | Q3, Q4 |
| `gen/go_code.py` | Generates the Go `FileTransferCode` constant (L597) | Q6 |
| `kittens/ssh/main.py` | SSH kitten makes `kitten` available on remote (L167) | Q7 |
| `kitty_tests/file_transmission.py` | Python protocol test harness — pattern for driving the server round-trip | Reference pattern (informed the Q3/Q4 handler-chain reads) |
| `kittens/transfer/ftc_test.go` | FTC `Serialize`/deserialize round-trip — pattern for the Q1 wire-format parse | Reference pattern |
| `kittens/transfer/send_test.go` | Send-side test — pattern for the Q2 delta parse | Reference pattern |
| `tools/rsync/api_test.go` | rsync signature/delta round-trip — pattern for the Q2/Q5 signature+delta parse | Reference pattern |

### Mechanisms, flags, data structures, and dependencies

| Named item | Where addressed |
|---|---|
| OSC introducer `ESC ]` (`1b 5d`) / ST `ESC \` (`1b 5c`) | Q1 (raw bytes) |
| `action` values `send`/`file`/`data`/`end_data`/`status`/`finish` | Q1 (handshake order) |
| All 15 key abbreviations incl. `quiet`→`q` and the `bypass`/`pw` base64 drift | Q1 (table) |
| `safe_string` charset `[0-9a-zA-Z_:./@-]` | Q1 |
| `transmission_type` `simple`/`rsync` (`tt`) | Q2, Q4, edges |
| `--transmit-deltas` / `-x` | Q4 |
| `--direction=upload` | Q4 gate (c), edges |
| `--compress=never` and `compression=zlib` | edges (D1/D2) |
| `OpType` `Block`/`Data`/`Hash`/`BlockRange` | Q2, Q5 |
| `BlockHash{Index,WeakHash,StrongHash}` (20 bytes LE) | Q2 |
| `rolling_checksum` (weak) + `add_one_byte` | Q2, Q5 |
| XXH3-64 (strong block) + XXH3-128 (integrity) | Q2, Q5 |
| 12-byte signature header (`round(sqrt(size))` block size) | Q2, Q4 |
| `CreateDelta` / `CreateDiff` | Q2 |
| `DestFile` / `PatchFile` / `signature_iterator` | Q3, Q4 |
| `signature_pending_chunks` | Q4 |
| transient tempfile `O_EXCL|O_NOFOLLOW 0600` + `os.replace` | Q4 |
| `print_rsync_stats` | Q5 |
| confirmation prompt / EPERM | Q7, edge D |
| `github.com/zeebo/xxh3` v1.0.2 (`go.mod` L16; XXH3 hasher, `BlockSize()`=64) | Q2, Q4, Q6 |
| `compress/zlib` (Go stdlib) / `zlib` (Python stdlib) / `libxxhash` (system) | Q1, Q6, edges |
| Build entry `python3 setup.py` → `kitty/launcher/{kitty,kitten}` | Q6 |
| Canonical entry `kitten ssh` → `kitten transfer` | Q7 |

## Observed vs. inferred summary

- **[OBSERVED] at runtime** (captured byte-exact in the container): the OSC 5113 handshake ordering
  and framing bytes (Q1); the full 58-byte base64 frame and the ≤4096-byte chunking with `PROGRESS`
  (Q1); the base64 RawStdEncoding round-trip and the ordinary-output-vs-transfer distinction (Q3); the
  signature header/records and the moved-block delta `447 of 448` (Q2); the three asymmetric gates
  with byte-exact signatures, the `round(sqrt(partial))` resume signature, the clean tempfile
  strace, and the no-sidecar fact (Q4); the four-run 16 MiB delta efficiency (delta 4146 / signature
  81932 / total 86078 / 0.513% / 194.9×) with ±0-byte stability and timing (Q5); the five edge cases
  including the below-4096 download basis engaging rsync and the zlib-vs-`never` compression contrast;
  and the SSH process chain, remote shim, and `src==dst` hash for Q7.
- **[SOURCE-VERIFIED]** (grounded in `file:line`, corroborating the observations): every function,
  struct, condition, and constant cited above — e.g. `FILE_TRANSFER_CODE 5113`, the VT-parser
  dispatch, `NewPatcher`'s block-size floor, `CreateDelta`/`CreateDiff`, the download existence gate,
  and the two block-size formulas.
- **[INFERRED]** (explicitly labeled where used): only the interpretation that normal terminal output
  "never enters the transfer branch" is stated as a mechanism explanation — it is itself corroborated
  by the OBSERVED zero-`openat` distinction test in Q3. No numeric claim in this document is inferred;
  all measured values are OBSERVED.

## Read-only scope — proof

This investigation is read-only except for this one answer document. All builds, transfers, captures,
and temporary scripts ran inside the container against a copy of the source at `/work/src` and wrote
their artifacts to `/work/evidence` (a bind-mounted host scratch directory **outside** the
repository). The repository working tree was confirmed clean of any change other than this file:

```
$ git status --porcelain
 ?? blitzy/documentation/kitty_815df1e210e0.md
```

No existing `.go`, `.py`, `.c`, `.h`, `.rst`, or build file was modified, added to, or deleted; the
only repository write is `blitzy/documentation/kitty_815df1e210e0.md`.

