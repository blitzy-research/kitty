# How file data travels end-to-end through kitty's file-transfer protocol (over an SSH-kitten connection)

> An empirically-grounded deep-dive. Every value quoted below was produced by **building and running** the code in the canonical container and **capturing the real output** — not by reading alone. Every claim is tied to an exact `file:line` citation, re-confirmed against the source tree at HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1` (branch `kitty_815df1e210e0`).

## Overview

kitty transfers files over the terminal's own byte stream. There is no side-channel socket: the sender (`kitten transfer`) emits `OSC 5113` escape codes that ride the same TTY/PTY that carries ordinary terminal output; the receiving kitty terminal recognizes those escape codes in its VT parser and routes them to an in-process Python orchestrator (`kitty/file_transmission.py`) that reassembles the file. When the destination already contains a version of the file, kitty switches to an **rsync-style delta** transfer: the receiver signs its existing copy, the sender computes a delta of *copy-block* and *literal-data* operations against that signature, and only the differences travel across the link.

Four cooperating layers implement this, and this document explains them together:

| Layer | Files | Responsibility |
|-------|-------|----------------|
| **Protocol channel** | `kitty/control-codes.h`, `kitty/vt-parser.c`, `kitty/data-types.c`, `gen/go_code.py` | Defines `OSC 5113` and the VT-parser routing that separates transfer frames from ordinary output |
| **Orchestrator** | `kitty/file_transmission.py` | In-process receiver/state-machine, `FileTransmissionCommand` (de)serialization, `DestFile`/`PatchFile` assembly, `OSC 5113` emission |
| **Client + algorithm** | `kittens/transfer/*`, `tools/rsync/*` | The Go CLI sender/receiver and the shared rsync delta engine (C core + Go library) |
| **Transport bootstrap** | `kittens/ssh/*` | Makes the protocol reachable across an SSH connection (`remote_kitty`, `shell_integration`) |

The document answers ten discrete objectives (O1–O10) in their own sections, each embedding **(a)** the exact command run, **(b)** verbatim captured output, **(c)** `file:line` citations, and **(d)** the rationale. A closing coverage pass confirms nothing was missed and that the source tree is unchanged.

## Conventions used in this document

- `<ESC>` denotes the byte `0x1b`. An `OSC` introducer is the two bytes `0x1b 0x5d` (`ESC ]`); an `ST` (string terminator) is the two bytes `0x1b 0x5c` (`ESC \`). This is stated normatively in `docs/file-transfer-protocol.rst:L547-L548`.
- Fenced code blocks labelled **command** show exactly what was executed; blocks labelled **captured output** show verbatim bytes/log lines. Where a capture was infeasible in the headless container, this is stated explicitly and the next-best observable is used.
- `file:line` citations point at the source as it exists on this branch; every literal below was re-opened and confirmed before quoting.

## Environment & build summary

The empirical work was performed in the user-provided container (`andrewparkscaleai/coding-agent:kovidgoyal__kitty__815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`). The observed toolchain:

**command**
```bash
go version
/opt/kitty-venv/bin/python --version
cc --version | head -1
```
**captured output**
```
go version go1.22.12 linux/amd64
Python 3.12.13
cc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
```

These reconcile with the project's declared minimums: `go 1.22` [`go.mod:L3`] and `requires-python = ">=3.8"` [`pyproject.toml:L2`]. The container's Python 3.12.13 satisfies the `>=3.8` floor; Go 1.22.12 is the 1.22 line required by `go.mod`.

The built kitty/kitten report version 0.35.2:

**command**
```bash
./kitty/launcher/kitten --version
```
**captured output**
```
kitten 0.35.2 created by Kovid Goyal
```

## End-to-end data flow (over SSH)

```mermaid
flowchart LR
    A["kitten transfer<br/>(sender, on remote host)"] -->|"emits OSC 5113 frames<br/>ac=send / ac=file / ac=data"| B["Remote PTY"]
    B -->|"SSH connection<br/>(transport-agnostic byte pipe)"| C["Local kitty terminal"]
    C --> D["VT parser<br/>kitty/vt-parser.c:L547"]
    D -->|"case FILE_TRANSFER_CODE:<br/>DISPATCH_OSC(file_transmission)"| E["Orchestrator<br/>kitty/file_transmission.py"]
    E -->|"deserialize<br/>FileTransmissionCommand"| F["ActiveReceive / DestFile<br/>file_transmission.py:L441"]
    F -->|"rsync? yes"| G["PatchFile<br/>sign existing file, apply delta<br/>to temp file  L377-L439"]
    F -->|"rsync? no"| H["DestFile<br/>write full data"]
    G --> I["atomic os.replace<br/>temp -> destination  L405"]
    H --> I
    E -.->|"status=OK / STARTED / PROGRESS acks<br/>flow back via OSC 5113"| C
```

The remainder of the document walks this diagram, grounding each hop in captured output and source citations.

---

## O1 — Build & connect

**What was asked:** build kitty from source, establish an SSH connection using the SSH kitten, and initiate a file transfer with the transfer kitten.

### Build

The canonical build entry point is `setup.py` [`setup.py`]; the `Makefile`'s `all:` target wraps `python3 setup.py $(VVAL)`. In the container the build interpreter is the prepared venv, so the exact command run was:

**command**
```bash
PATH=/usr/local/go/bin:$PATH CI=true /opt/kitty-venv/bin/python setup.py build --verbose
```

The build completed with exit status 0 in ~23 s, emitting 103 lines. It detected the compiler and compiled the C core under strict flags:

**captured output (build-log head)**
```
CC: ['gcc'] (15, 0)
gcc (Ubuntu 15.2.0-4ubuntu4) 15.2.0
...
Detected: CompilerType.gcc
```

Representative lines proving each of the three build products was produced (C core, the rsync C extension, and the Go kitten):

**captured output (the rsync C core is compiled — `kittens/transfer/algorithm.c`)**
```
gcc -MMD -DNDEBUG ... -std=c11 -pedantic-errors -Werror -O3 ... -flto ... -march=native ... \
  -c kittens/transfer/algorithm.c -o build/rsync-kittens-transfer-algorithm.c.o
```

**captured output (the rsync extension is linked — note `-lxxhash`, empirically confirming the XXH3 hash suite)**
```
gcc ... -shared -flto build/rsync-kittens-transfer-algorithm.c.o -lxxhash ... \
  -o build/kittens/transfer/rsync.so
```

**captured output (the C core `fast_data_types.so` is linked — it includes `vt-parser.c.o` and `data-types.c.o`, the two files that define/route `OSC 5113`)**
```
gcc ... -shared -flto ... build/fast_data_types-kitty-data-types.c.o ... \
  build/fast_data_types-kitty-vt-parser.c.o ... -o build/kitty/fast_data_types.so
```

**captured output (the Go `kitten` binary is built; the VCS revision matches this branch's HEAD)**
```
/usr/local/go/bin/go build -v -ldflags '-X kitty.VCSRevision=815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1 -s -w' \
  -o kitty/launcher/kitten /tmp/blitzy/kitty/.../tools/cmd
```

The C flags `-std=c11 -pedantic-errors -Werror` (visible in the compile line) confirm the core builds cleanly under warnings-as-errors — no `--ignore-compiler-warnings` was needed. The `data-types.c` compile line additionally carries `-DKITTY_VCS_REV="815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1"` and `-DWRAPPED_KITTENS="... ssh ... transfer ..."`, showing the `ssh` and `transfer` kittens are wrapped into the launcher. The four resulting artifacts are `kitty/launcher/kitty`, `kitty/launcher/kitten` (Go), `kitty/fast_data_types.so` (C core), and `kittens/transfer/rsync.so` (the rsync C extension). All of these paths are `.gitignore`d, so building leaves the tracked tree unchanged.

> A capture-friendly debug build is also available: `python3 setup.py build --debug --extra-logging=event-loop` (the `Makefile`'s `debug-event-loop:` target). It compiles without `-DNDEBUG` and enables event-loop logging useful for observing the escape-code channel.

### Connect (SSH kitten) and initiate a transfer (transfer kitten)

The transfer kitten's own help text states the intended topology — the protocol runs over a TTY, typically an SSH one:

**command**
```bash
./kitty/launcher/kitten transfer --help
```
**captured output (first lines)**
```
Usage: kitten transfer [options] source_files_or_directories destination_path

Transfer files over the TTY device. Can be used to send files between any two
computers provided there is a TTY connection between them, such as over SSH.
```

**The SSH bootstrap.** The SSH kitten makes `kitten` available on the *remote* host so a remote `kitten transfer` can speak the protocol back through the SSH PTY. This is controlled by the `remote_kitty` option, whose verbatim documentation explains the mechanism:

- `kittens/ssh/main.py:L164` — `opt('remote_kitty', 'if-needed', choices=('if-needed', 'no', 'yes'), long_text='''` — the long text states (verbatim from source): *"kitten is not actually copied to the remote host, instead a small bootstrap script is copied which will download and run kitten when kitten is first executed on the remote host."*
- `kittens/ssh/main.py:L122` — `opt('shell_integration', 'inherited', long_text='''` — controls remote shell integration, which is what triggers the terminfo/`kitten` bootstrap on the remote side.

Because the constant that identifies transfer frames is language-independent (see O3) and the protocol rides the terminal byte stream, the *transport* (SSH vs. a local PTY) is irrelevant to the protocol itself. That is why the protocol can be exercised and captured over a local PTY and the observations apply unchanged to the SSH path.

**Honest limitation (stated explicitly).** kitty is a GUI terminal; the SSH kitten refuses to run outside a kitty window, and the container is headless:

**command**
```bash
./kitty/launcher/kitten ssh localhost
```
**captured output**
```
Error: The SSH kitten is meant to run inside a kitty window
```

Therefore a full GUI SSH round-trip could **not** be captured headlessly. The fallback (used throughout this document) is to drive the *same* protocol code paths directly: the real compiled `kitten` binary under a PTY harness (O3), and the real compiled Python orchestrator (`FileTransmissionCommand`, `PatchFile`, `DestFile`) and rsync extension (`rsync.so`) via direct calls (O5–O10). `sshd`, `ssh`, and host keys are present in the container (`/etc/ssh`), but the GUI requirement of the SSH kitten cannot be satisfied without a display. This is a container constraint, not a protocol limitation.

---

## O2 — Handshake initiation

**What was asked:** how does the transfer kitten initiate the protocol handshake — the initial `action=send` (or `action=receive`) and the `status=OK` acknowledgement that opens the session?

### The observed handshake, from the real kitten binary

Running the real compiled `kitten transfer` under a `pty.fork()` harness, the kitten printed its scan progress and then emitted the **session-opening frame** verbatim:

**command (essence of the harness — full script kept under `/root`, removed afterward)**
```python
# src.txt is 18000 bytes (> 4096, so rsync-capable); TERM=xterm-kitty
pid, fd = pty.fork()
if pid == 0:  # child: pty.fork() gives it a controlling TTY
    os.execvpe(KITTEN, [KITTEN, "transfer", "--transmit-deltas", SRC, DST], env)
# parent: read raw bytes the kitten writes to fd, sniff for the OSC 5113 frame
```
**captured output (raw bytes the kitten wrote to the PTY)**
```
Scanning files…
Found 1 files and directories, requesting transfer permission…
<ESC>]5113;id=d6476d88;ac=send<ESC>\
```

Hex of that frame (proving the exact framing bytes):
```
1b 5d 35 31 31 33 3b 69 64 3d 64 36 34 37 36 64 38 38 3b 61 63 3d 73 65 6e 64 1b 5c
└┬─┘ └──────┬──────┘ └────────────┬───────────────┘ └┬─┘
 OSC   "5113" (ASCII)   ; id=d6476d88 ; ac=send        ST
0x1b 0x5d               payload                        0x1b 0x5c
```

So the transfer kitten opens the session with `ac=send` (the short-name for `action=send`; see O6). `id` is the session id (`d6476d88` here).

### The full command sequence and the acknowledgements

The normative flow is documented under **"Overall design"** (`docs/file-transfer-protocol.rst:L16`). The verbatim example lines quoted by the spec are:

- `docs/file-transfer-protocol.rst:L362` — `→ action=file id=someid file_id=f1 name=/path/to/destination transmission_type=rsync`
- `docs/file-transfer-protocol.rst:L367` — `← action=status id=someid file_id=f1 status=STARTED transmission_type=rsync`

Putting the observed and documented pieces together, a session proceeds as:

1. **Open:** sender → `ac=send` (or `ac=receive`); receiver → `ac=status ... st=OK` (or refuses with `EPERM`). The `send`/`receive`/`status`/`file`/`data`/`end_data`/`finish` actions are the `Action` enum at `kitty/file_transmission.py:L166`.
2. **Per file:** sender → `ac=file ... fid=f1 ...`; receiver → `ac=status ... fid=f1 st=STARTED`.
3. **Payload:** sender → repeated `ac=data ... d=<base64>` then `ac=end_data`.
4. **Close:** sender → `ac=finish`.

Using the **real** compiled `FileTransmissionCommand.serialize()` (from the built `kitty/fast_data_types` import surface), the exact acknowledgement frames were produced and captured:

**captured output (the `STARTED` and `OK` acks, real serializer)**
```
STARTED : <ESC>]5113;ac=status;tt=rsync;id=deadbeef;fid=f1;st=U1RBUlRFRA<ESC>\
OK      : <ESC>]5113;ac=status;id=deadbeef;st=T0s<ESC>\
```

The status text is carried base64-encoded in the `st=` field (see O6). Decoding confirms the values verbatim:
```
st=U1RBUlRFRA  -> b'STARTED'
st=T0s         -> b'OK'
```

### The status vocabulary

The acknowledgement values come from the `ErrorCode` enum, quoted verbatim:

- `kitty/file_transmission.py:L203` — `ErrorCode = Enum('ErrorCode', 'OK STARTED CANCELED PROGRESS EINVAL EPERM EISDIR ENOENT')`

So `OK` opens/permits, `STARTED` acknowledges a file, `PROGRESS` reports mid-transfer, `CANCELED` aborts, and `EINVAL`/`EPERM`/`EISDIR`/`ENOENT` are the error refusals (e.g., `EPERM` when the user denies permission). Transfers require user approval unless a bypass token (`pw=`, base64) is supplied — the frame the kitten emitted (`ac=send` with no `pw=`) is exactly the "request transfer permission" step its console message announced.

**Rationale.** The handshake is a request/acknowledge exchange carried entirely as `OSC 5113` frames: the sender declares intent (`ac=send`), the receiver authorizes (`st=OK`) or refuses (`st=EPERM`), and each file is separately acknowledged (`st=STARTED`) before its bytes flow. This keeps the protocol strictly ordered and lets the receiving terminal gate every transfer behind user consent.

---

## O3 — Session-establishing escape sequences

**What was asked:** identify the exact escape sequences that establish the transfer session over the terminal connection.

### The `OSC 5113` control code

Transfer commands are `OSC` escape codes of the form (quoted verbatim from the spec):

- `docs/file-transfer-protocol.rst:L543` — `Transfer commands are encoded as ``OSC`` escape codes of the form::`
- `docs/file-transfer-protocol.rst:L545` — `    <OSC> 5113 ; key=value ; key=value ... <ST>`
- `docs/file-transfer-protocol.rst:L547` — `Here ``OSC`` is the bytes ``0x1b 0x5d`` and ``ST`` is the bytes ...` (`0x1b 0x5c`, L548).

The numeric `5113` is a single named constant, defined once in C:

- `kitty/control-codes.h:L232-L233`
  ```c
  // File transfer OSC number
  #define FILE_TRANSFER_CODE 5113
  ```

The live frame captured in O2 is exactly this form: `0x1b 0x5d` (`<OSC>`), the ASCII bytes `5113`, `;`-separated `key=value` pairs, then `0x1b 0x5c` (`<ST>`).

### Cross-language consistency of `5113`

The single C definition is surfaced to all three languages, and all three agree on `5113`:

- **C → Python.** `kitty/data-types.c:L596` — `PyModule_AddIntMacro(m, FILE_TRANSFER_CODE);` exports the macro as a Python-visible integer. The orchestrator imports it: `kitty/file_transmission.py:L23` — `from kitty.fast_data_types import ESC_OSC, FILE_TRANSFER_CODE, ...`.
- **C → Go.** `gen/go_code.py:L597` — `const FileTransferCode int = {FILE_TRANSFER_CODE}` is a code-generation template. The `{FILE_TRANSFER_CODE}` placeholder is filled from the C macro's value at generation time, so the Go constant is *derived from* the same C header rather than hard-coded independently. This is why the three layers cannot drift.

Empirically, the string form of the constant that prefixes every frame was captured as `5113`:

**captured output (from the real Python serializer)**
```
ftc_prefix (FILE_TRANSFER_CODE as str): 5113
```
This corresponds to `ftc_prefix = str(FILE_TRANSFER_CODE)` in `kitty/file_transmission.py` (module-level, near L30).

### Emission path

The orchestrator emits a frame by serializing the command's fields (with the `OSC` code prefixed) and writing them to the child PTY as an OSC escape code:

- `kitty/file_transmission.py:L1149` — `data = tuple(payload.get_serialized_fields(prefix_with_osc_code=True))`
- `kitty/file_transmission.py:L1150` — `queued = window.screen.send_escape_code_to_child(ESC_OSC, data)`

The Go client has the parallel encoder: `kittens/transfer/ftc.go:L163` — `func (self FileTransmissionCommand) Serialize(prefix_with_osc_code ...bool) string {`. Both sides therefore produce byte-identical `OSC 5113 ; key=value ; ... ST` frames; the normative contract they both target is **"Encoding of transfer commands as escape codes"** (`docs/file-transfer-protocol.rst:L540`).

### Captured session-establishing frames (real serializer)

**captured output (verbatim; `<ESC>` = `0x1b`)**
```
send      : <ESC>]5113;ac=send;id=deadbeef<ESC>\
file rsync: <ESC>]5113;ac=file;tt=rsync;id=deadbeef;fid=f1;n=L3BhdGgvdG8vZGVzdGluYXRpb24<ESC>\
STARTED   : <ESC>]5113;ac=status;tt=rsync;id=deadbeef;fid=f1;st=U1RBUlRFRA<ESC>\
```
The `file` frame's `n=` field base64-decodes to the destination path, matching the spec's `name=/path/to/destination` example at L362:
```
n=L3BhdGgvdG8vZGVzdGluYXRpb24 -> b'/path/to/destination'
```

**Rationale.** A dedicated, rarely-collided OSC number (`5113`) is what lets file-transfer traffic share the terminal byte stream with ordinary output while remaining unambiguously identifiable (see O7). Defining it once in C and deriving the Python and Go constants from that single definition guarantees the sender, the receiver, and the algorithm library all frame and recognize the exact same escape sequence.

---

## O4 — rsync-style delta transfer

**What was asked:** how does kitty implement rsync-style delta transfer (weak rolling checksum plus strong-hash block matching) via its C core and Go binding?

### The canonical rsync algorithm (validated externally)

kitty's implementation maps faithfully onto the classic rsync algorithm (Tridgell & Mackerras, `rsync.samba.org/tech_report`). The canonical three-step structure is: the receiver splits its file into fixed-size blocks and computes a cheap **weak rolling checksum** and an expensive **strong hash** per block; the sender slides a one-byte window over its file, computing the rolling checksum at each offset in **O(1)** per step; on a weak-checksum hit it verifies with the strong hash, and only then emits a *block reference*; unmatched regions are emitted as *literal data*. The rolling checksum's defining property is that "successive values can be computed very efficiently using the recurrence relations" (rsync tech report, node3) — i.e. constant work per byte. kitty's C code cites this exact reference in-line (see below).

kitty modernizes only the **hash suite**, not the algorithm: classic rsync uses an Adler-32-style weak checksum and MD4 (strong); kitty uses an rsync rolling checksum (weak), **XXH3-64** as the strong per-block id, and **XXH3-128** as a whole-file integrity checksum (see O5). This is a hash-suite modernization, not an algorithmic departure — the block-matching structure is identical.

### The rolling checksum in the Go binding

- `tools/rsync/algorithm.go:L335` — `// see https://rsync.samba.org/tech_report/node3.html` (the in-code citation)
- `tools/rsync/algorithm.go:L336` — `type rolling_checksum struct {`
- `tools/rsync/algorithm.go:L341` — `func (self *rolling_checksum) full(data []byte) uint32 {` — computes the checksum over a whole block
- `tools/rsync/algorithm.go:L355` — `func (self *rolling_checksum) add_one_byte(first_byte, last_byte byte) {` — the O(1) incremental update as the window advances by one byte (given the byte entering and the byte leaving)

`add_one_byte` is exactly the "constant work per byte" property the canonical algorithm relies on: sliding the window forward requires only the byte leaving (`first_byte`) and the byte entering (`last_byte`), never a re-scan of the block.

### The C core

The same algorithm is implemented in C in `kittens/transfer/algorithm.c` and compiled into the `kittens.transfer.rsync` extension (`rsync.so`). O1's build log shows `kittens/transfer/algorithm.c` compiled and linked with `-lxxhash`. The C core and the Go library are kept API-compatible so either can produce/consume the other's signatures (see O5's parity note).

### Observed: the round-trip actually works

Both implementations were exercised and pass:

**command (Go library round-trip + hashers)**
```bash
go test -count=1 -v ./tools/rsync/
```
**captured output**
```
=== RUN   TestRsyncRoundtrip
--- PASS: TestRsyncRoundtrip (0.00s)
=== RUN   TestRsyncHashers
--- PASS: TestRsyncHashers (0.00s)
PASS
ok      kitty/tools/rsync       0.009s
```

The Python side (the compiled `rsync.so` `Patcher`/`Differ`) was driven directly in the O10 experiment below and reconstructs the modified file correctly (`reconstruction correct (recon==changed): True`). The normative sections backing this are **"Transmitting binary deltas"** (`docs/file-transfer-protocol.rst:L338`, which references the rsync tech report) and **"The format of signatures and deltas"** (`docs/file-transfer-protocol.rst:L409`).

**Rationale.** By computing the weak checksum incrementally, the sender can test *every* byte offset (not just block boundaries) in a single pass; the weak checksum screens out almost all non-matches cheaply, and the strong hash eliminates the rare weak-hash collisions. This is what makes the delta both correct and fast, and it is why only the changed bytes (plus small per-block metadata) need to cross the link (quantified in O10).

---

## O5 — Signature/difference data structures

**What was asked:** identify the data structures that track file signatures and differences.

### `BlockHash` — the per-block signature record

- `tools/rsync/algorithm.go:L177-L183`
  ```go
  type BlockHash struct {
      Index      uint64
      WeakHash   uint32
      StrongHash uint64
  }

  const BlockHashSize = 20
  ```

Each block of the receiver's existing file is summarized by one `BlockHash`: its `Index`, its `WeakHash` (the rsync rolling checksum) and its `StrongHash` (XXH3-64). The serialized size is fixed at 20 bytes (`8 + 4 + 8`), captured as `BlockHashSize = 20`.

The record is constructed while iterating the signature:
- `tools/rsync/algorithm.go:L261` — `ans = BlockHash{Index: self.index, WeakHash: self.rc.full(b), StrongHash: self.hasher.Sum64()}` — note `WeakHash` comes from the rolling checksum's `full()` and `StrongHash` from the block hasher's `Sum64()`.

### The weak-hash lookup map — how differences are found

The sender indexes the receiver's `BlockHash` records by weak hash so it can test each sliding-window offset in (amortized) O(1):

- `tools/rsync/algorithm.go:L362` — `type diff struct {`
- `tools/rsync/algorithm.go:L365-L366`
  ```go
  // A single β hash may correlate with many unique hashes.
  hash_lookup map[uint32][]BlockHash
  ```

The map key is the `uint32` weak hash; the value is a slice of `BlockHash` because (as the comment notes) one weak hash may collide across several blocks — the strong hash then disambiguates. This is precisely the canonical rsync "hash table → sorted-list scan → strong-checksum verify" three-level check, expressed as a Go map of slices.

### The public API surface

- `tools/rsync/api.go:L47-L61` — the `Api`, `Differ`, and `Patcher` structs:
  - `Api struct` at L47 (fields L47-L54),
  - `Differ struct` at L56 (fields L56-L59),
  - `Patcher struct` at L61.
- `tools/rsync/api.go:L270` — `func NewPatcher(expected_input_size int64) (ans *Patcher) {` — constructs a patcher sized for the expected input.

### The hash-type enums (the modernized suite)

- `tools/rsync/api.go:L31-L43`
  ```go
  type StrongHashType uint16
  type WeakHashType uint16
  type ChecksumType uint16

  const (
      XXH3 StrongHashType = iota
  )
  const (
      XXH3128Sum ChecksumType = iota
  )
  const (
      Rsync WeakHashType = iota
  )
  ```
  So the strong per-block hash is `XXH3` (XXH3-64), the whole-file integrity checksum is `XXH3128Sum` (XXH3-128), and the weak hash is `Rsync` (the rolling checksum). The build's `-lxxhash` link (O1) is the runtime confirmation that XXH3 is the strong-hash provider.

### The Python extension surface

The same structures are exposed to the Python orchestrator through the compiled `rsync.so` type stub:
- `kittens/transfer/rsync.pyi:L24` — `class Patcher:`
- `kittens/transfer/rsync.pyi:L38` — `class Differ:`
- `kittens/transfer/rsync.pyi:L45` — `def parse_ftc(...)` (parses a `FileTransmissionCommand` frame in C)

### Observed: the on-wire signature format matches these structures exactly

Signing an 18000-byte file produced a signature of **2712 bytes** over **135 blocks** with a chosen block size of **134** (≈ √18000):

**captured output**
```
block size chosen by Patcher         : 134
number of signature blocks           : 135
signature size (sent receiver->sender): 2712 bytes
```

The spec (`docs/file-transfer-protocol.rst`) defines the signature as a **12-byte header** (`uint16 version + uint16 checksum_type + uint16 strong_hash_type + uint16 weak_hash_type + uint32 block_size`, L417-L424) followed by per-block records of **20 bytes** each (`uint64 index + uint32 weak_hash + uint64 strong_hash`, L447-L451). The arithmetic checks out exactly:
```
12  (header)  +  135 blocks × 20 bytes/block  =  12 + 2700  =  2712 bytes  ✓
```
This is a direct empirical confirmation that the wire record equals `BlockHashSize = 20` [`tools/rsync/algorithm.go:L183`], and that `block_size` is "usually the square root of the file size" as the spec states (L426-L428): √18000 ≈ 134.16, and the Patcher chose 134.

**Rationale.** `BlockHash` is the atomic unit of a signature; the `hash_lookup` map is the acceleration structure that turns "does any receiver block match here?" into a cheap map probe keyed on the weak hash, with the strong hash resolving collisions. Fixing the record at 20 bytes and integers at little-endian (`docs/file-transfer-protocol.rst:L412`) is what lets the C core and the Go library interoperate on the exact same bytes.

---

## O6 — Chunk encoding in the terminal stream

**What was asked:** how are data chunks encoded — base64 `d=` payloads no larger than 4096 bytes, optionally zlib-compressed, wrapped in `OSC 5113`?

### The `FileTransmissionCommand` fields (compact short-names)

The wire format uses two/three-letter field names, defined on the dataclass:

- `kitty/file_transmission.py:L251` — `@dataclass` ; `L252` — `class FileTransmissionCommand:` ; fields `L254-L267`, each carrying a short-name in its metadata:

| short-name | field | notes |
|-----------|-------|-------|
| `ac` | action | `send`/`file`/`data`/`end_data`/`receive`/`cancel`/`status`/`finish` |
| `zip` | compression | e.g. `zlib` |
| `ft` | ftype | file type |
| `tt` | ttype | transmission type (e.g. `rsync`) |
| `fid` | file_id | |
| `pw` | bypass | base64 (permission bypass token) |
| `q` | quiet | |
| `mod` | mtime | |
| `prm` | permissions | |
| `sz` | size | |
| `n` | name | base64 |
| `st` | status | base64 |
| `pr` | parent | |
| `d` | data | the payload |

(The `id` field has no short-name — it is written literally as `id=`.)

### The 4096-byte chunk threshold

- `kittens/transfer/send.go:L64` — `ans.b.Grow(4096)` — the send buffer is grown in 4096-byte units.
- `kittens/transfer/send.go:L131` — `rsync_capable: file_type == FileType_regular && stat_result.Size() > 4096,` — a file only qualifies for rsync-delta treatment when it is a regular file **larger than 4096 bytes** (below that, a full send is cheaper than a signature exchange).

Chunk payloads (`d=`) are base64-encoded and, per the spec, are kept no larger than 4096 bytes each; larger files are split across multiple `ac=data` frames.

### The compressors

- `kittens/transfer/utils.py:L41` — `class IdentityCompressor:` (pass-through)
- `kittens/transfer/utils.py:L50` — `class ZlibCompressor:` (zlib)
- Imported by the orchestrator: `kitty/file_transmission.py:L22` — `from kittens.transfer.utils import IdentityCompressor, ZlibCompressor`.

When compression is active the frame carries `zip=zlib`; the spec shows `compression=zlib` examples at L497 and L503.

### Observed data frames (real serializer)

**captured output (an uncompressed data chunk — `d=` is base64)**
```
<ESC>]5113;ac=data;id=deadbeef;fid=f1;d=VGhlIHF1aWNrIGJyb3duIGZveAABAg<ESC>\
```

**captured output (a zlib-compressed data chunk — note the `zip=zlib` field)**
```
<ESC>]5113;ac=data;zip=zlib;id=deadbeef;fid=f1;d=eHh4eHh4eHh4eHh4eHh4eHh4eHh4eHh4eHh4eHh4eHg<ESC>\
```

**captured output (the frames that bracket the payload)**
```
end_data: <ESC>]5113;ac=end_data;id=deadbeef;fid=f1<ESC>\
finish  : <ESC>]5113;ac=finish;id=deadbeef<ESC>\
```

A base64 round-trip confirms `d=` really carries the file bytes (here the literal `ABC`):
```
d=QUJD -> b'ABC'
```

**Rationale.** Short field names keep per-frame overhead minimal; base64 makes arbitrary binary data safe to embed in a text escape sequence; the 4096-byte cap keeps individual frames small enough to interleave cleanly with terminal output and to fit terminal I/O buffers; and optional zlib compression (`zip=zlib`) shrinks compressible payloads further. All of it is wrapped in the `OSC 5113 ; ... ST` envelope so the receiver's VT parser can pick it out (O7).

---

## O7 — Receiver reassembly & discrimination from terminal output

**What was asked:** how does the receiving side reassemble chunks and distinguish transfer data from regular terminal output?

### The discriminator: the VT parser routes `OSC 5113` specifically

Ordinary text and most escape sequences flow through kitty's VT parser and end up on screen. `OSC 5113` is special-cased: when the parser finishes reading an OSC whose numeric code is `FILE_TRANSFER_CODE`, it dispatches to the file-transmission handler instead of treating the payload as text:

- `kitty/vt-parser.c:L547-L549`
  ```c
  case FILE_TRANSFER_CODE:
      START_DISPATCH
      DISPATCH_OSC(file_transmission);
  ```

This `case` is exactly the line that separates transfer bytes from ordinary output: any OSC that is *not* `5113` (and the other handled codes) is processed as a normal terminal control; `5113` is handed to `file_transmission`. Because the discriminator is the OSC number itself, transfer frames can be interleaved arbitrarily with normal output on the same stream without ambiguity.

### Reassembly in the orchestrator

`DISPATCH_OSC(file_transmission)` lands in the Python orchestrator, which deserializes the frame into a `FileTransmissionCommand` and drives file assembly:

- `kitty/file_transmission.py:L377` — `class PatchFile:` (rsync-delta assembly)
- `kitty/file_transmission.py:L441` — `class DestFile:` (destination management / full-write assembly)

(The `DestFile`/`PatchFile` assembly machinery spans roughly `L377-L468`.)

### Observed: the same bytes deserialize back into a command

The frame the real kitten emitted was fed back through the real deserializer, and through the compiled C `parse_ftc`:

**captured output (Python `FileTransmissionCommand.deserialize`)**
```
captured wire (from kitten): <ESC>]5113;id=d6476d88;ac=send<ESC>\
deserialized -> action='send' id='d6476d88'
```

**captured output (the compiled rsync C extension `parse_ftc` on a data frame)**
```
{'ac': 'data', 'id': 'zz', 'fid': 'f1', 'd': 'QUJD'}
```

So the exact bytes emitted on the wire (O2/O3) parse cleanly back into the structured command the receiver acts on — the round-trip is closed. As an end-to-end check, the file-transmission test module exercises this send/receive path in full:

**command**
```bash
TMPDIR=/root/kitty_test_tmp ./kitty/launcher/kitty +launch test.py --module file_transmission
```
**captured output (tail)**
```
Ran 6 tests ... OK
```
(6/6 in ~0.99 s: `test_file_get`, `test_parse_ftc`, `test_rsync_hashers`, `test_rsync_roundtrip`, `test_transfer_receive`, `test_transfer_send`. A non-setgid `TMPDIR` is used because the container's `/tmp` is setgid, which otherwise trips two directory-mode assertions — an environment artifact, not a code defect.)

**Rationale.** Using a single reserved OSC number as the discriminator means the receiver needs no out-of-band signalling to tell file bytes from screen bytes: the VT parser already tokenizes OSC codes, so routing `5113` to `file_transmission` is a one-line branch. The orchestrator then owns reassembly, so the wire protocol stays simple (ordered `data` frames) while all buffering/patching logic lives in one place.

---

## O8 — Resumption behavior

**What was asked:** what state allows an interrupted transfer to resume rather than restart?

The answer is: the **rsync signature computed over the existing on-disk destination file**. There is no separate progress cursor — the partial file *is* the state.

### The signature-over-existing-file path

When the destination already exists and rsync is in use, the receiver opens the existing file and signs it block-by-block; the sender then only needs to send blocks that differ:

- `kitty/file_transmission.py:L426` — `def next_signature_block(...)` begins the signing.
- `kitty/file_transmission.py:L430` — `self.src_file = open(self.path, 'rb')` — opens the **existing destination** for reading.
- `kitty/file_transmission.py:L431` — emits `patcher.signature_header(buf)`.
- `kitty/file_transmission.py:L434` — emits `patcher.sign_block(...)` per block.

`DestFile` decides this by stat-ing any existing destination and yielding a `PatchFile` over it:
- `kitty/file_transmission.py:L450` — `os.stat(self.name, follow_symlinks=False)` (existing destination stat)
- `kitty/file_transmission.py:L470` — `PatchFile(self.name, existing_stat.st_size, ...)` from the `signature_iterator` (def at L469)

### Observed: the signature is really computed over the on-disk file

**captured output**
```
dest dir contents BEFORE: ['file.bin']
existing destination file size: 18000 bytes
signature computed over existing file: 2712 bytes  (src_file opened = True)
delta produced by sender: 260 bytes
```

`src_file opened = True` is the observable proof that `open(self.path, 'rb')` at L430 ran against the pre-existing `file.bin`, and the 2712-byte signature (12 + 135×20; see O5) is computed from that file's *current* contents.

**Rationale.** Because the signature is derived from whatever is already on disk, resuming an interrupted transfer is not a special case: the receiver simply re-signs the partial file and the sender computes a fresh delta against it. Whatever bytes already arrived are represented as *copy* operations and are never re-sent. The transfer kitten's own `--transmit-deltas` help text calls this out verbatim — `kittens/transfer/main.py:L117` describes using the rsync algorithm to update an existing file, *"automatically resuming partial transfers"*.

---

## O9 — Resumption metadata location

**What was asked:** where is resumption metadata stored — is there a sidecar/journal, or is the partial destination file itself the basis?

**There is NO sidecar and NO journal.** The resumption basis is the on-disk destination file itself plus its *recomputed* rsync signature. During patching, output is written to a **temporary file in the destination directory**, and the transfer finishes with an **atomic replace**:

- `kitty/file_transmission.py:L392` — the destination temp file is created (inside the `dest_file` property, L390-L393):
  ```python
  self._dest_file = tempfile.NamedTemporaryFile(mode='wb', dir=os.path.dirname(os.path.abspath(os.path.realpath(self.path))), delete=False)
  ```
- `kitty/file_transmission.py:L405` — the finalize step:
  ```python
  os.replace(self.dest_file.name, self.src_file.name)
  ```
  (Note: this is line **405** on this branch — verified against the current file.)

The Go receiver mirrors this exactly:
- `kittens/transfer/receive.go:L91` — `err = os.Rename(pf.temp.Name(), pf.src.Name())`

### Observed: temp file appears during patch, then vanishes via atomic replace; nothing else is left behind

**captured output**
```
dest dir contents DURING patch (note temp file alongside): ['file.bin', 'tmp4qaq8h48']
temp file created in destination directory: 'tmp4qaq8h48' (dir=/root/kitty_scratch/o89)
dest dir contents AFTER close (atomic replace): ['file.bin']
final dest == sender's changed content: True
NO sidecar/journal file present — resumption basis is the on-disk file + its recomputed signature: True
```

The directory listing proves the sequence: **BEFORE** = `['file.bin']`; **DURING** the patch a sibling temp file `tmp4qaq8h48` exists alongside `file.bin` in the *same* destination directory (from `tempfile.NamedTemporaryFile(dir=...)` at L392); **AFTER** `close()` only `file.bin` remains — the temp file was atomically renamed over the destination via `os.replace` (L405). At no point does any `.journal`/`.sig`/sidecar file appear.

**Rationale.** Writing to a temp file in the *same* directory guarantees the final `os.replace`/`os.Rename` is atomic (same filesystem, no cross-device copy), so a crash can never leave a half-written destination — the old file stays intact until the new one is complete. Storing no separate metadata means there is nothing to keep in sync or to garbage-collect: the file on disk, re-signed on demand, is the single source of truth for what still needs to be sent.

---

## O10 — Delta-efficiency evidence (the modify-and-retransfer experiment)

**What was asked:** transfer a file, modify a small portion, transfer again with deltas enabled, and show the second transfer sends substantially fewer bytes — identifying the mechanism that detected the unchanged portions.

### The experiment

A source file of **18000 bytes** was used (deliberately `> 4096` so it is `rsync_capable` per `kittens/transfer/send.go:L131`). It was transferred once to populate the destination; then **35 bytes** in the middle were modified; then it was re-transferred using the rsync delta path (as `--transmit-deltas` selects — `kittens/transfer/main.py:L115`, `docs/kittens/transfer.rst:L80`). Byte accounting was measured by driving the real compiled `rsync.so` `Differ`/`Patcher`.

**captured output**
```
=================== O10: DELTA-EFFICIENCY EXPERIMENT (real rsync.so) ===================
original (destination) size : 18000 bytes
changed  (sender)      size : 18000 bytes
bytes actually modified     : 35

TRANSFER #1 (no pre-existing destination -> full send):
  literal data bytes sent (whole file) : 18000

TRANSFER #2 (destination exists -> rsync delta, --transmit-deltas):
  block size chosen by Patcher         : 134
  number of signature blocks           : 135
  signature size (sent receiver->sender): 2712 bytes
  delta size (sent sender->receiver)   : 259 bytes
  literal bytes in delta (total_data_in_delta): 178
  reconstruction correct (recon==changed): True

=================== SUMMARY ===================
  full send literal bytes : 18000
  delta send total bytes  : 259  (of which literal: 178)
  reduction (delta vs full): 98.6% fewer bytes
```

### The measured result

| Metric | Transfer #1 (full) | Transfer #2 (delta) |
|--------|--------------------|--------------------|
| Bytes sender→receiver | **18000** (all literal) | **259** (delta ops) |
| — of which literal data | 18000 | **178** |
| Signature receiver→sender | — | 2712 |
| Reconstruction correct | — | **True** |

Transfer #2 sent **259 bytes** of delta versus **18000 bytes** for the full send — a **98.6% reduction** — and correctly reconstructed the modified file. Even counting the receiver→sender signature (2712 bytes) as overhead, the delta path moves ~2971 bytes total versus 18000, still a large saving, and that overhead is what buys the ability to send only 178 literal bytes for a 35-byte edit (the 178 covers the changed region plus the boundary block(s) straddling it).

### The mechanism that detected the unchanged portions

The 134 unchanged blocks were represented as **copy operations** (`Block`, type 0 — "copy block index from the existing file unmodified") rather than **literal operations** (`Data`, type 1 — literal bytes), per the delta format in `docs/file-transfer-protocol.rst` (§"The format of signatures and deltas", L409+). Those copies were chosen because the sender's sliding-window **rolling checksum** (weak) matched a receiver block's `WeakHash`, and the **XXH3-64** strong hash then confirmed the match — looked up via `hash_lookup map[uint32][]BlockHash` [`tools/rsync/algorithm.go:L362-L366`] against the `BlockHash` records built at `tools/rsync/algorithm.go:L261`. Only the offsets whose rolling checksum found no confirmed match were emitted as literal `Data`.

This is the same result the canonical rsync literature predicts — e.g., a benchmark updating a 24 MB kernel tarball across versions transferred only tens of bytes rather than the whole file — confirming kitty's implementation behaves as a faithful rsync.

**Rationale.** The savings come entirely from replacing literal bytes with block-index references for the regions that did not change. Because the rolling checksum is testable at every byte offset in O(1) (O4), a small edit that shifts nothing else still lets the sender re-align on the very next unchanged block and resume copying — which is why a 35-byte change costs only ~178 literal bytes, not a re-send of everything after the edit.

---

## Cross-file consistency (why the four layers agree)

- **Constant consistency — `FILE_TRANSFER_CODE` = `5113`.** Defined once in C at `kitty/control-codes.h:L233`, surfaced to Python at `kitty/data-types.c:L596` (`PyModule_AddIntMacro`), and templated into Go at `gen/go_code.py:L597` (`const FileTransferCode int = {FILE_TRANSFER_CODE}`). The Python side observed the string form as `5113`; the C→Go template fills the same value, so the three layers cannot drift.
- **Wire-format parity.** The Python serializer (`kitty/file_transmission.py:L251-L267`, emitted at L1149-L1150) and the Go encoder (`kittens/transfer/ftc.go:L163`) produce the same `key=value;…` frames, both targeting the normative "Encoding of transfer commands as escape codes" (`docs/file-transfer-protocol.rst:L540`). Empirically, the frame the Go `kitten` emitted (`<ESC>]5113;id=d6476d88;ac=send<ESC>\`) deserialized cleanly through the Python `FileTransmissionCommand` (O7).
- **rsync parity.** The C core (`kittens/transfer/algorithm.c`) and the Go library (`tools/rsync/*.go`) implement the same `BlockHash`/signature/delta format (`tools/rsync/algorithm.go:L177-L183`, `BlockHashSize = 20`). The observed signature size (2712 = 12 + 135×20) matches the spec's byte layout, so either side can produce/consume the other's signatures.

---

## Coverage pass

Every objective was answered with **(a)** a real command, **(b)** verbatim captured output, and **(c)** `file:line` citations:

| # | Objective | Command / capture | Key citations |
|---|-----------|-------------------|---------------|
| O1 | Build & connect | `setup.py build --verbose` (exit 0, 103 lines; `-lxxhash` link; Go kitten with VCS rev); `kitten transfer --help`; SSH-kitten limitation captured | `setup.py`, `go.mod:L3`, `pyproject.toml:L2`, `kittens/ssh/main.py:L122,L164` |
| O2 | Handshake initiation | live `<ESC>]5113;...;ac=send<ESC>\`; real `STARTED`/`OK` acks; status enum | `docs/…:L16,L362,L367`, `kitty/file_transmission.py:L166,L203` |
| O3 | Session-establishing escape sequences | live + serialized `OSC 5113` frames; `ftc_prefix=5113`; emission path | `kitty/control-codes.h:L233`, `kitty/data-types.c:L596`, `gen/go_code.py:L597`, `kitty/file_transmission.py:L23,L1149-L1150`, `docs/…:L540-L548` |
| O4 | rsync-style delta transfer | `go test ./tools/rsync/` PASS; rolling-checksum functions | `tools/rsync/algorithm.go:L335-L360`, `docs/…:L338,L409` |
| O5 | Signature/difference data structures | signature = 2712 = 12+135×20; block size 134 | `tools/rsync/algorithm.go:L177-L183,L261,L362-L366`, `tools/rsync/api.go:L31-L43,L47-L61,L270`, `kittens/transfer/rsync.pyi:L24,L38,L45` |
| O6 | Chunk encoding | captured `ac=data … d=<base64>` and `zip=zlib` frames; `d=QUJD`→`ABC` | `kitty/file_transmission.py:L251-L267,L22`, `kittens/transfer/send.go:L64,L131`, `kittens/transfer/utils.py:L41,L50`, `docs/…:L497,L503` |
| O7 | Receiver reassembly & discrimination | VT-parser `case FILE_TRANSFER_CODE`; deserialize + `parse_ftc`; 6/6 tests OK | `kitty/vt-parser.c:L547-L549`, `kitty/file_transmission.py:L377,L441` |
| O8 | Resumption behavior | signature computed over existing file (`src_file opened=True`, 2712 B) | `kitty/file_transmission.py:L426-L439,L450,L469-L470`, `kittens/transfer/main.py:L117` |
| O9 | Resumption metadata location | temp file `tmp4qaq8h48` during patch → atomic replace → only `file.bin`; no sidecar | `kitty/file_transmission.py:L392,L405`, `kittens/transfer/receive.go:L91` |
| O10 | Delta-efficiency evidence | **18000 → 259 bytes = 98.6% fewer**; reconstruction True; Block-vs-Data | `tools/rsync/algorithm.go:L261,L362-L366`, `kittens/transfer/main.py:L115`, `docs/kittens/transfer.rst:L80`, `docs/…:L338` |

**Explicitly flagged infeasible capture and its fallback:** a full **GUI SSH round-trip** could not be captured because the SSH kitten refuses to run outside a kitty window and the container is headless (`Error: The SSH kitten is meant to run inside a kitty window`, O1). Fallback: the identical protocol code paths were driven directly — the real compiled `kitten` binary under a `pty.fork()` harness (which produced the live `OSC 5113` frame in O2/O3), and the real compiled Python orchestrator and `rsync.so` extension via direct calls (O5–O10). Because the protocol is transport-agnostic (it rides the terminal byte stream regardless of whether that stream is a local PTY or an SSH channel), these observations apply unchanged to the SSH path. All other objectives were captured directly.

**Source tree left unchanged.** All scratch scripts and test files were kept outside the repository (under `/root/` and `/tmp/`) and removed after capture. The only addition to the repository is this document:

**command**
```bash
git status --porcelain
git status --porcelain -uall   # expand the untracked directory
git diff --stat                # any tracked-file changes?
```
**captured output**
```
?? blitzy/
?? blitzy/documentation/kitty_815df1e210e0.md
```
(`git status --porcelain` collapses the new directory to `?? blitzy/`; `-uall` expands it to the single file `blitzy/documentation/kitty_815df1e210e0.md`. `git diff --stat` produced **no output** — no tracked source file under `kitty/`, `kittens/`, `tools/`, `docs/`, `gen/`, or any manifest was modified.)

---

*Compiled from four cooperating source layers — protocol channel + orchestrator (`kitty/`), client + rsync algorithm (`kittens/`, `tools/rsync/`), transport bootstrap (`kittens/ssh/`), and normative documentation (`docs/`) — plus verbatim runtime output captured during the build/run/measure investigation at HEAD `815df1e210e0a9ab4622f5c7f2d6891d7dbeddf1`.*

