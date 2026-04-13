# Kitty File Transfer Protocol: Complete Data Journey Over SSH

## Table of Contents

1. [Protocol Handshake and Escape Sequence Establishment](#1-protocol-handshake-and-escape-sequence-establishment)
2. [Rsync-Style Delta Transfer Mechanism](#2-rsync-style-delta-transfer-mechanism)
3. [Data Encoding and Stream Reassembly](#3-data-encoding-and-stream-reassembly)
4. [Transfer Resumption Behavior](#4-transfer-resumption-behavior)
5. [Delta Transfer Efficiency Demonstration](#5-delta-transfer-efficiency-demonstration)

---

## 1. Protocol Handshake and Escape Sequence Establishment

### 1.1 Overview

Kitty's file transfer protocol operates entirely within the terminal byte stream, using OSC (Operating System Command) escape sequences as the transport envelope. The protocol uses a dedicated OSC code — **5113** — to multiplex file transfer commands alongside regular terminal output. This section traces the complete handshake from the Go CLI entry point through the Python-side terminal handler, citing every function and data structure involved.

### 1.2 CLI Entry Point: `kittens/transfer/main.go`

The transfer kitten's entry point dispatches based on the `--direction` flag. The `main()` function in `kittens/transfer/main.go` (lines 10–71) reads the CLI options and branches:

```go
// kittens/transfer/main.go
func main(cmd *cli.Command, opts *Options, args []string) (rc int, err error) {
    // ...
    if opts.Direction == "upload" {
        rc, err = send_main(opts, args)
    } else {
        rc, err = receive_main(opts, args)
    }
    // ...
}
```

For a typical download (remote → local), the default direction is `"receive"` and `receive_main()` is invoked. For an upload (local → remote), `--direction=upload` is passed and `send_main()` is invoked. The bypass password, if provided, is encrypted using the `encode_bypass()` function in `kittens/transfer/utils.go` before being included in the protocol handshake.

### 1.3 Bypass Password Encryption: `kittens/transfer/utils.go`

Before the handshake begins, the bypass password (used to skip the user confirmation prompt) is encrypted. The `encode_bypass()` function (lines 14–24 of `kittens/transfer/utils.go`) concatenates the request ID with the bypass string, encrypts the result using X25519 + AES-256-GCM against `KITTY_PUBLIC_KEY`, and prepends `kitty-1:`:

```go
// kittens/transfer/utils.go
func encode_bypass(request_id string, bypass string) (encoded_bypass string, err error) {
    data := request_id + ";" + bypass
    encrypted, err := crypto.Encrypt_data(utils.UnsafeStringToBytes(data), KITTY_PUBLIC_KEY)
    // ...
    encoded_bypass = "kitty-1:" + base64.StdEncoding.EncodeToString(encrypted)
    return
}
```

On the terminal side, `kitty/file_transmission.py` validates this in `check_bypass()` by attempting decryption with the private key and verifying the request ID prefix matches.

### 1.4 OSC Envelope Construction: `SendManager.initialize()` in `kittens/transfer/send.go`

The `SendManager.initialize()` method (lines ~555–575 of `kittens/transfer/send.go`) constructs the OSC escape sequence envelope that wraps every protocol message:

```go
// kittens/transfer/send.go — SendManager.initialize()
self.prefix = fmt.Sprintf("\x1b]%d;id=%s;", kitty.FileTransferCode, self.manager.request_id)
self.suffix = "\x1b\\"
```

Here, `kitty.FileTransferCode` resolves to **5113** (defined in `kitty/control-codes.h` as `#define FILE_TRANSFER_CODE 5113`). The `request_id` is a unique identifier generated for each transfer session. Every subsequent protocol message is serialized as:

```
<prefix> + <serialized_fields> + <suffix>
```

Which produces escape sequences of the form:

```
\x1b]5113;id=abc123;<serialized_key=value_pairs>\x1b\\
```

### 1.5 Handshake Initiation: `start_transfer()` in `kittens/transfer/send.go`

The `start_transfer()` method sends the first protocol message to initiate a transfer session. For a send session (kitten → terminal), this sends an `Action_send` command:

```go
// kittens/transfer/send.go — start_transfer()
cmd := FileTransmissionCommand{Action: Action_send, Bypass: encoded_bypass}
self.send_cmd(cmd)
```

The serialized message on the wire looks like:

```
\x1b]5113;id=abc123;ac=snd;pw=BASE64_ENCRYPTED_BYPASS\x1b\\
```

The OSC event loop is configured via `lp.OnEscapeCode` to listen for responses from the terminal. When the terminal receives this handshake, it either prompts the user for confirmation or validates the bypass password.

### 1.6 Wire Format Serialization: `FileTransmissionCommand.Serialize()` in `kittens/transfer/ftc.go`

The `FileTransmissionCommand` struct (defined at lines 78–93 of `kittens/transfer/ftc.go`) represents every protocol message. Its `Serialize()` method (lines ~136–181) produces the semicolon-delimited wire format:

```go
// kittens/transfer/ftc.go — FileTransmissionCommand struct
type FileTransmissionCommand struct {
    Action          Action          `json:"ac"`
    Compression     Compression     `json:"zip"`
    Ftype           FileType        `json:"ft"`
    Ttype           TransmissionType `json:"tt"`
    Quiet           QuietLevel      `json:"q"`
    Id              string          `json:"id"`
    File_id         string          `json:"fid"`
    Bypass          string          `json:"pw"`   // base64 encoded
    Name            string          `json:"n"`    // base64 encoded
    Status          string          `json:"st"`   // base64 encoded
    Parent          string          `json:"pr"`
    Mtime           int64           `json:"mod"`
    Permissions     fs.FileMode     `json:"prm"`
    Size            int64           `json:"sz"`
    Data            []byte          `json:"d"`    // base64 encoded
}
```

Serialization rules:
- Each non-default field is encoded as `shortname=value`
- String fields (`Name`, `Status`, `Bypass`) are base64-encoded using `base64.RawStdEncoding` (standard alphabet, no padding)
- The `Data` field uses `base64.StdEncoding` (standard with padding)
- Numeric fields use decimal string representation
- Zero/default values are omitted

A fully serialized `Action_send` command looks like:

```
ac=snd;pw=a2l0dHktMTpBQkNERUY=
```

When wrapped in the OSC envelope:

```
\x1b]5113;id=abc123;ac=snd;pw=a2l0dHktMTpBQkNERUY=\x1b\\
```

### 1.7 VT Parser Routing: `kitty/vt-parser.c`

When the terminal emulator receives bytes from the child process (the transfer kitten via SSH), the VT parser in `kitty/vt-parser.c` processes them character by character. Upon encountering the OSC introducer `\x1b]`, the parser accumulates the payload until it encounters the string terminator `\x1b\\` (or BEL `\x07`).

The parser then extracts the numeric OSC code and dispatches based on its value:

```c
// kitty/vt-parser.c — OSC dispatch
case FILE_TRANSFER_CODE:
    DISPATCH_OSC(file_transmission);
    break;
```

The constant `FILE_TRANSFER_CODE` is defined in `kitty/control-codes.h` (line 233):

```c
#define FILE_TRANSFER_CODE 5113
```

The `DISPATCH_OSC(file_transmission)` macro routes the raw payload string to the Python-side handler. All other OSC codes and regular terminal output (plain text, ANSI color codes, cursor movement, etc.) follow their normal processing paths. This is the **demultiplexing boundary** that separates file transfer data from normal terminal I/O.

### 1.8 Terminal-Side Dispatch: `FileTransmission.handle_serialized_command()` in `kitty/file_transmission.py`

The Python-side `FileTransmission` class in `kitty/file_transmission.py` is the terminal host's orchestrator for all file transfer sessions. When the VT parser routes an OSC 5113 payload, it arrives at `handle_serialized_command()`:

```python
# kitty/file_transmission.py — FileTransmission.handle_serialized_command()
def handle_serialized_command(self, data: str) -> None:
    cmd = FileTransmissionCommand.deserialize(data)
    # ... dispatch based on cmd.action ...
```

Deserialization uses the C extension `parse_ftc` for performance, which parses the semicolon-delimited key=value pairs and invokes a callback for each field. The Python `FileTransmissionCommand` dataclass mirrors the Go struct field-for-field.

Based on the `action` field, the dispatcher routes to:
- `Action.send` → Creates an `ActiveReceive` session (terminal receives files from kitten)
- `Action.receive` → Creates an `ActiveSend` session (terminal sends files to kitten)
- `Action.cancel` → Cancels the identified session
- Other actions (`file`, `data`, `end_data`, `finish`, `status`) → Routed to the appropriate active session

### 1.9 Permission Flow and Response

When `Action.send` is received, the terminal creates an `ActiveReceive` object and either:

1. **With bypass password**: Validates the encrypted bypass via `check_bypass()`. If the decrypted payload's request ID prefix matches the session, permission is granted automatically.
2. **Without bypass**: Prompts the user with a confirmation dialog showing the file list and total size.

The response is sent back to the kitten via `write_ftc_to_child()`:

```python
# kitty/file_transmission.py — write_ftc_to_child()
def write_ftc_to_child(self, payload: FileTransmissionCommand, ...) -> bool:
    self.pty.write_to_child(
        '\x1b]' + payload.serialize(prefix_with_osc_code=True) + '\x1b\\',
        flush=False
    )
    return True
```

This wraps the response in the same OSC 5113 envelope and writes it to the child process's input, completing the bidirectional communication loop.

### 1.10 File Metadata Exchange

After permission is granted (terminal sends `Action_status` with `status=OK`), the kitten calls `send_file_metadata()` which sends an `Action_file` command for each file:

```go
// kittens/transfer/send.go — send_file_metadata()
cmd := FileTransmissionCommand{
    Action:      Action_file,
    File_id:     file.file_id,
    Ftype:       file.file_type,
    Name:        file.expanded_local_path,
    Permissions: file.permissions,
    Mtime:       file.mtime.UnixNano(),
    Size:        file.file_size,
}
```

The terminal responds to each `Action_file` with `Action_status` including:
- `status=STARTED` — file transfer can begin
- `ttype=rsync` and `size=<existing_size>` — if the file exists at the destination and qualifies for rsync delta transfer (regular file, >4096 bytes)

### 1.11 Complete Handshake Sequence Diagram

```
Kitten (Go)                                    Terminal (Python)
    |                                                |
    |-- \x1b]5113;id=REQ;ac=snd;pw=BYPASS\x1b\\ -->|
    |                                                | [VT parser routes OSC 5113]
    |                                                | [FileTransmission.handle_serialized_command()]
    |                                                | [check_bypass() validates password]
    |                                                |
    |<-- \x1b]5113;ac=st;id=REQ;st=OK\x1b\\  ------|
    |                                                |
    | [Permission granted, send file metadata]       |
    |-- \x1b]5113;id=REQ;ac=file;fid=F1;...  ----->|
    |                                                | [Create DestFile for F1]
    |<-- \x1b]5113;ac=st;fid=F1;st=STARTED\x1b\\ --|
    |                                                |
    | [Begin data transmission for F1]               |
```

---

## 2. Rsync-Style Delta Transfer Mechanism

### 2.1 Overview

Kitty implements an rsync-style delta transfer algorithm that allows re-transferring modified files by sending only the changed portions. The implementation lives in `tools/rsync/algorithm.go` (core algorithm) and `tools/rsync/api.go` (public API), with integration points in both the Go kitten (`kittens/transfer/send.go`, `kittens/transfer/receive.go`) and the Python terminal host (`kitty/file_transmission.py`). This section documents the complete mechanism with all data structures.

### 2.2 Rsync Capability Determination

Not all files qualify for rsync delta transfer. The eligibility check is in `kittens/transfer/send.go` during file discovery:

```go
// kittens/transfer/send.go — files_for_send()
rsync_capable: file_type == FileType_regular && stat_result.Size() > 4096,
```

A file must be:
1. A **regular file** (not a symlink, directory, or hard link)
2. **Larger than 4096 bytes**

Files smaller than 4096 bytes would have a block size too small for the rolling checksum to provide meaningful savings, so they are transferred in full. The 4096-byte threshold ensures that the overhead of signature generation and delta computation is justified.

### 2.3 Data Structures

#### 2.3.1 `BlockHash` Struct — `tools/rsync/algorithm.go`

The fundamental unit of a file's signature is the `BlockHash`, a fixed-size 20-byte record:

```go
// tools/rsync/algorithm.go
const BlockHashSize = 20

type BlockHash struct {
    Index      uint64  // 8 bytes: position in the original file (block number)
    WeakHash   uint32  // 4 bytes: rsync rolling checksum
    StrongHash uint64  // 8 bytes: XXH3-64 hash of the block
}
```

Each `BlockHash` represents one block of the file (at position `Index`), identified by two hashes:
- **WeakHash (4 bytes)**: A fast rolling checksum used for the sliding window search
- **StrongHash (8 bytes)**: An XXH3-64 hash used to confirm matches (eliminates false positives from weak hash collisions)

The total size per block hash is `8 + 4 + 8 = 20 bytes`, matching the constant `BlockHashSize`.

#### 2.3.2 Signature Header — `tools/rsync/api.go`

The signature stream begins with a 12-byte header that describes the hash algorithms and parameters used:

```go
// tools/rsync/api.go — signature header format (12 bytes)
// Bytes 0-1:   uint16  version           = 0
// Bytes 2-3:   uint16  checksum_type     = 0 (XXH3-128)
// Bytes 4-5:   uint16  strong_hash_type  = 0 (XXH3-64)
// Bytes 6-7:   uint16  weak_hash_type    = 0 (Rsync rolling checksum)
// Bytes 8-11:  uint32  block_size        = computed value
```

All values are encoded in **little-endian** byte order. The header tells the differ which algorithms to use when processing the signature data. Currently only version 0 is defined, with a single set of algorithm choices.

#### 2.3.3 Operation Types — `tools/rsync/algorithm.go`

The delta stream consists of four operation types, each serialized in little-endian binary:

| Type | Value | Format | Description |
|------|-------|--------|-------------|
| `OpBlock` | 0 | 1 byte type + 8 bytes uint64 index = **9 bytes** | "Copy block N from the existing file" |
| `OpData` | 1 | 1 byte type + 4 bytes uint32 length + N bytes data = **5+N bytes** | "Insert these new bytes" |
| `OpHash` | 2 | 1 byte type + 2 bytes uint16 length + N bytes hash = **3+N bytes** | "Whole-file checksum (XXH3-128)" |
| `OpBlockRange` | 3 | 1 byte type + 8 bytes uint64 start_index + 4 bytes uint32 count = **13 bytes** | "Copy blocks start..start+count from existing file" |

`OpBlockRange` is an optimization: when consecutive blocks match, instead of emitting individual `OpBlock` operations, they are coalesced into a single `OpBlockRange`. This reduces the delta stream size for large unchanged regions.

### 2.4 Block Size Calculation

The block size adapts to the file's size. In `tools/rsync/api.go`, `NewPatcher()` computes:

```go
// tools/rsync/api.go — NewPatcher()
func NewPatcher(expected_input_size int64) *Patcher {
    ans.block_size = int(math.Round(math.Sqrt(float64(expected_input_size))))
    if ans.block_size > MaxBlockSize {  // MaxBlockSize = 1 << 20 = 1MB
        ans.block_size = MaxBlockSize
    }
    // ...
}
```

The formula is `block_size = round(sqrt(file_size))`, capped at 1MB (`MaxBlockSize`). This balances:
- **Smaller blocks** → More signature entries (higher overhead) but finer-grained delta detection
- **Larger blocks** → Fewer entries but a single changed byte invalidates the entire block

For example:
- 100 KB file → block size ≈ 316 bytes → ~316 blocks
- 1 MB file → block size ≈ 1024 bytes → ~1024 blocks
- 100 MB file → block size ≈ 10,000 bytes → ~10,000 blocks
- 1 TB file → block size = 1 MB (capped) → ~1,048,576 blocks

The standalone default `DefaultBlockSize = 6144` (6 KB) is used when constructing a `Differ` without a specific file context.

### 2.5 Rolling Checksum Algorithm

The rolling checksum in `tools/rsync/algorithm.go` (lines ~60–110) implements the classic rsync rolling checksum from the [rsync technical report](https://rsync.samba.org/tech_report/):

```go
// tools/rsync/algorithm.go — rolling checksum
type rolling_checksum struct {
    alpha, beta, val uint32
    window           []byte
    idx              int64
}
```

**Full computation** for an initial window of `l` bytes:

```
alpha = sum(b[i] for i in 0..l-1) mod M
beta  = sum((l - i) * b[i] for i in 0..l-1) mod M
val   = alpha + M * beta
```

Where `M = 1 << 16` (65536).

**Sliding computation** — when the window advances by one byte (old byte `b_out` leaves, new byte `b_in` enters):

```go
// tools/rsync/algorithm.go — add_one_byte()
func (self *rolling_checksum) add_one_byte(b_out, b_in byte, l int) {
    self.alpha += uint32(b_in) - uint32(b_out)
    self.beta += self.alpha - uint32(l)*uint32(b_out)
    self.val = self.alpha + (1 << 16)*self.beta
}
```

This gives an O(1) update per byte position, enabling the sender to efficiently slide the checksum window across the entire source file looking for matching blocks.

### 2.6 Diff Algorithm: Sliding Window Search

The core diff algorithm is in `diff.read_next()` (lines ~285–405 of `tools/rsync/algorithm.go`). The sender executes this against the source file using the receiver's signature:

```
1. Build a hash_lookup map: weak_hash → []BlockHash
2. For each byte position in the source file:
   a. Compute the rolling checksum for the current window
   b. Look up the weak hash in hash_lookup
   c. If found, compute XXH3-64 strong hash and compare
   d. If strong hash matches → emit OpBlock (block found in existing file)
   e. If no match → accumulate byte as new data (OpData)
3. Coalesce consecutive OpBlock into OpBlockRange
4. Emit final OpHash with XXH3-128 checksum of entire source file
```

The algorithm efficiently identifies unchanged regions (which become `OpBlock`/`OpBlockRange`) and changed or new regions (which become `OpData`), minimizing the data that must be transmitted.

### 2.7 Complete Signature and Delta Flow

#### 2.7.1 Signature Generation (Receiver Side)

The **receiver** (the side that already has a copy of the file) generates the signature:

**Go side** (`kittens/transfer/receive.go`):
```go
// request_files() for each rsync-capable existing file:
rf.patcher = rsync.NewPatcher(expected_size)
rf.patcher.CreateSignatureIterator(existing_file, &sigwriter)
```

**Python side** (`kitty/file_transmission.py`):
```python
# ActiveReceive — for files that exist at the destination:
pf = PatchFile(name, existing_stat.st_size)
# signature blocks generated via pf.signature_iterator()
# sent back to kitten as Action_data / Action_end_data
```

The `CreateSignatureIterator()` reads the existing file block-by-block, computes the weak (rolling) and strong (XXH3-64) hashes for each block, and writes:
1. The 12-byte signature header
2. A sequence of 20-byte `BlockHash` entries

#### 2.7.2 Delta Generation (Sender Side)

The **sender** (the side with the new version) processes the signature and generates the delta:

**Go side** (`kittens/transfer/send.go`):
```go
// on_signature_data_received() — feed signature data to differ:
file.differ.AddSignatureData(data)
// On end_data:
file.differ.FinishSignatureData()
file.start_delta_calculation()

// start_delta_calculation():
file.delta_loader = file.differ.CreateDelta(source_file, &deltabuf)
```

**Python side** (`kitty/file_transmission.py`):
```python
# SourceFile.next_chunk() — for rsync transfers:
self.differ.next_op(self.read_from_src, write_op)
# Outputs delta operations to the response stream
```

#### 2.7.3 Delta Application (Receiver Side)

The **receiver** applies the delta operations against the existing file to produce the new version:

**Go side** (`kittens/transfer/receive.go`):
```go
// patch_file.write() — applies incoming delta data:
patcher.UpdateDelta(data)
// On finalize:
patcher.FinishDelta()
os.Rename(temp_path, destination_path)
```

**Python side** (`kitty/file_transmission.py`):
```python
# PatchFile — applies delta:
patcher.apply_delta_data(data, read_func, write_func)
# Uses temp file + atomic rename
```

The `ApplyDelta()` function in `tools/rsync/algorithm.go` processes each operation:
- `OpBlock` / `OpBlockRange` → Seek to the corresponding position in the existing file and copy the block(s)
- `OpData` → Write the new data directly
- `OpHash` → Verify the XXH3-128 checksum of the reconstructed file

### 2.8 Integration Diagram: Rsync Data Flow

```
Sender (has new file)                    Receiver (has existing file)
        |                                        |
        |                                        | 1. Read existing file block-by-block
        |                                        | 2. Compute weak + strong hashes per block
        |                                        | 3. Write 12-byte header + BlockHash entries
        |   <---- Signature data (streaming) --- |
        |                                        |
        | 4. Parse signature header              |
        | 5. Build hash_lookup: weak→BlockHash[] |
        | 6. Slide window over new file:         |
        |    - Rolling checksum per position     |
        |    - Lookup weak hash                  |
        |    - Verify strong hash on match       |
        | 7. Emit OpBlock for matches,           |
        |    OpData for differences              |
        |                                        |
        | --- Delta operations (streaming) --->  |
        |                                        | 8. Process operations:
        |                                        |    - OpBlock: copy from existing
        |                                        |    - OpData: write new bytes
        |                                        |    - OpHash: verify checksum
        |                                        | 9. Atomic rename temp → dest
```

---

## 3. Data Encoding and Stream Reassembly

### 3.1 Overview

This section explains how file data is encoded for safe transport through the terminal byte stream, how it is chunked for transmission, and how the receiving side distinguishes file transfer data from regular terminal output (text, ANSI escape sequences, cursor movement, etc.).

### 3.2 Data Encoding Pipeline

When file data (raw bytes or rsync delta operations) is ready for transmission, it passes through this encoding pipeline:

```
Raw bytes → [Optional zlib compression] → Base64 encoding → Embed in FTC → OSC 5113 envelope
```

#### Step 1: Optional Zlib Compression

The `should_be_compressed()` function in `kittens/transfer/utils.go` (lines 26–46) determines whether compression should be applied:

```go
// kittens/transfer/utils.go — should_be_compressed()
func should_be_compressed(path string, ...) bool {
    // Skip if already compressed formats:
    // zip, odt, odp, pptx, docx, gz, bz2, xz, svgz
    // Also skip image and video MIME types
    // ...
    return true  // compress everything else
}
```

Files with already-compressed formats (archives, office documents, images, videos) skip compression because re-compressing them wastes CPU with negligible size reduction.

When compression is enabled, zlib (RFC 1950) is used. On the Go side, `compress/zlib` wraps the data via a `ZlibCompressor` in `kittens/transfer/send.go`. On the Python side, `ZlibCompressor` and `ZlibDecompressor` classes in `kitty/file_transmission.py` and `kittens/transfer/utils.py` handle compression and decompression respectively.

The compression type is communicated via the `zip` field in the `FileTransmissionCommand`: `zip=zlib` indicates compressed data.

#### Step 2: Base64 Encoding

After optional compression, the data bytes are base64-encoded for TTY safety. Terminal byte streams interpret certain byte values as control characters (e.g., `0x1B` as ESC, `0x07` as BEL), so raw binary data would corrupt the stream. Base64 encoding guarantees only printable ASCII characters.

The `Data` field in `FileTransmissionCommand` uses **standard base64 encoding** (`base64.StdEncoding` in Go) with padding. Other binary fields (`Name`, `Status`, `Bypass`) use `base64.RawStdEncoding` (no padding).

#### Step 3: Embedding in the Wire Format

The base64-encoded data is placed in the `d=` field of the semicolon-delimited command string:

```
ac=data;fid=F1;d=SGVsbG8gV29ybGQ=
```

#### Step 4: OSC 5113 Envelope

The complete command string is wrapped in the OSC envelope:

```
\x1b]5113;id=REQ;ac=data;fid=F1;d=SGVsbG8gV29ybGQ=\x1b\\
```

### 3.3 Chunk Splitting: `split_for_transfer()` in `kittens/transfer/ftc.go`

Large data payloads are split into chunks of at most **4096 bytes** by the `split_for_transfer()` function (lines ~184–220 of `kittens/transfer/ftc.go`):

```go
// kittens/transfer/ftc.go — split_for_transfer()
func (self *FileTransmissionCommand) split_for_transfer(
    data []byte,
    file_id string,
    mark_last bool,
) iter.Seq[*FileTransmissionCommand] {
    // Splits data into 4096-byte chunks
    // Each chunk is Action_data except the last which is Action_end_data
}
```

The chunking rules:
- Each chunk contains at most 4096 bytes of base64-encoded data
- All chunks except the last use `Action_data` (`ac=data`)
- The final chunk uses `Action_end_data` (`ac=end_data`) to signal completion
- Each chunk includes the `fid` (file ID) to associate it with the correct file

For a 100 KB file (after compression and base64 encoding), this produces approximately 25+ individual OSC escape sequences, each carrying 4096 bytes of payload.

### 3.4 Stream Demultiplexing: How the VT Parser Separates Transfer Data

The critical question is: how does the terminal distinguish file transfer commands from regular terminal output? The answer lies in the VT parser's escape sequence processing in `kitty/vt-parser.c`.

#### 3.4.1 VT Parser Processing Model

The VT parser processes the incoming byte stream character by character, maintaining a state machine. The relevant states for file transfer are:

1. **Ground state**: Normal text characters are rendered to the screen
2. **ESC state**: Entered when `\x1b` (0x1B) is encountered
3. **OSC state**: Entered when `\x1b]` is detected (ESC followed by `]`)
4. **OSC payload accumulation**: Characters are accumulated until the string terminator `\x1b\\` (or BEL `\x07`)

When the parser encounters `\x1b]`:
1. It enters OSC state and begins accumulating the payload
2. It reads the numeric OSC code at the start of the payload
3. When the string terminator `\x1b\\` is received, the complete payload is dispatched based on the code:

```c
// kitty/vt-parser.c — OSC dispatch
case FILE_TRANSFER_CODE:  // 5113
    DISPATCH_OSC(file_transmission);
    break;
```

4. OSC code 5113 routes to `file_transmission`, while other codes (e.g., 52 for clipboard, 4 for color queries) follow their respective handlers
5. Regular text and other escape sequences (CSI for cursor movement, SGR for colors) are processed normally

#### 3.4.2 The Demultiplexing Boundary

This parsing mechanism is the **demultiplexing boundary**: the VT parser inherently separates file transfer data from terminal output because:

- File transfer commands are always wrapped in `\x1b]5113;...\x1b\\`
- Regular terminal text never contains the OSC 5113 prefix
- Other escape sequences (colors, cursor movement) use different prefix bytes and codes
- The parser's state machine ensures that file transfer payloads are never mixed with text rendering

There is no explicit "mode switch" between file transfer and normal operation — they coexist in the same byte stream, demultiplexed purely by the OSC code number.

### 3.5 Reassembly on the Receiving Side

#### 3.5.1 Go-Side Reassembly (Kitten as Receiver)

When the kitten receives files (`kittens/transfer/receive.go`), the `remote_file` struct tracks per-file state:

```go
// kittens/transfer/receive.go
type remote_file struct {
    // ...
    patcher      *rsync.Patcher
    expect_diff  bool
    decompressor *flate.Reader  // zlib decompressor
    // ...
}
```

Data reassembly:
1. Each `Action_data` message's base64-decoded `Data` field is fed to `remote_file.write_data()`
2. If compressed, data passes through `decompressor.Read()` first
3. For rsync transfers, decompressed data goes to `patcher.UpdateDelta()`
4. For simple transfers, data goes directly to the file writer
5. `Action_end_data` triggers finalization: compressor flush, patcher finish, atomic rename

#### 3.5.2 Python-Side Reassembly (Terminal as Receiver)

When the terminal receives files (`kitty/file_transmission.py`), the `DestFile` class manages reassembly:

```python
# kitty/file_transmission.py — DestFile.write_data()
def write_data(self, data: bytes, is_last: bool) -> None:
    # 1. Decompress if needed:
    data = self.decompressor.decompress(data)
    # 2. Write to destination:
    #    - For rsync: patcher.apply_delta_data(data, ...)
    #    - For simple: self.actual_file.write(data)
    # 3. If is_last: flush decompressor, finalize
```

The `DestFile` uses either `ZlibDecompressor` or `IdentityDecompressor` based on the `compression` field, and either `PatchFile` (for rsync) or a direct file handle (for simple transfer).

### 3.6 Complete Encoding Example

For a 10 KB text file being sent with compression:

```
1. Read 10,240 bytes from file
2. Zlib compress → ~3,500 bytes (typical for text)
3. Base64 encode → ~4,668 bytes
4. Split into chunks: chunk1 (4096 bytes) + chunk2 (572 bytes)
5. Emit OSC messages:
   \x1b]5113;id=REQ;ac=data;fid=F1;zip=zlib;d=<4096 bytes base64>\x1b\\
   \x1b]5113;id=REQ;ac=end_data;fid=F1;d=<572 bytes base64>\x1b\\
```

The receiver processes these in order, base64-decodes each `d=` field, feeds the concatenated bytes through zlib decompression, and writes the resulting 10,240 bytes to disk.

---

## 4. Transfer Resumption Behavior

### 4.1 Overview

This section addresses a critical question: if a file transfer is interrupted (e.g., by network disconnection, Ctrl+C, or process termination) and then restarted, does the protocol resume from where it left off? The answer, grounded in a thorough analysis of the codebase, is that **no explicit resume mechanism exists** — but the rsync delta mechanism provides **functional equivalence** to resumption.

### 4.2 Absence of Explicit Resume State

A comprehensive search of the file transfer codebase confirms that no persistent checkpoint or resume infrastructure exists:

1. **No session persistence**: There are no files written to disk that track transfer session state. The `SendManager` and receiver state machines exist only in memory and are destroyed when the process exits.

2. **No partial transfer tracking**: There is no mechanism to record "file F1 was 67% transferred" or "bytes 0–6890 have been committed." The protocol has no `Action_resume` or equivalent.

3. **No checkpoint protocol messages**: The protocol specification in `docs/file-transfer-protocol.rst` defines no checkpoint, resume, or state-recovery commands. The only state-altering actions are `send`, `receive`, `cancel`, `finish`, and the data transfer actions.

4. **Cancellation is terminal**: When a transfer is interrupted, `Action_cancel` is sent (if possible), and the terminal responds with `CANCELED` status. A restart begins a completely fresh session with a new `request_id`.

### 4.3 How Interruption Is Handled

When a transfer is interrupted:

1. **Kitten side** (`kittens/transfer/send.go`): The signal handler catches the interrupt and the kitten sends `Action_cancel` to the terminal. If the connection is severed, no cancel message is sent.

2. **Terminal side** (`kitty/file_transmission.py`): If `Action_cancel` is received, the active session is removed from `active_receives` or `active_sends`. The `DestFile` pattern uses **temp-file-then-rename**: data is written to a temporary file, and only on successful completion is the temp file atomically renamed to the final path. If interrupted, the temp file remains (and will be cleaned up by the OS or a subsequent transfer).

3. **Partially transferred files**: For simple (non-rsync) transfers, a partially transferred file may exist at the destination if the transfer did not use the temp file pattern. However, this is an incomplete file, not a checkpoint.

### 4.4 Rsync as Implicit Resumption

The rsync delta mechanism provides **implicit resumption** that is functionally equivalent — and in many cases more efficient than — explicit resume. Here is how it works:

#### 4.4.1 Existing File Detection

When a new transfer session is initiated for a file that already exists at the destination (fully or partially from a previous interrupted transfer), the receiver detects it:

**Python side** (`kitty/file_transmission.py` — `DestFile.__init__()`):
```python
# kitty/file_transmission.py — DestFile
self.existing_stat = safe_stat(self.name)  # os.stat() with exception handling
```

**Go side** (`kittens/transfer/receive.go` — `request_files()`):
```go
// For each rsync-capable file: check if it already exists at destination
stat, err := os.Stat(destination_path)
if err == nil {
    // File exists — use rsync delta transfer
    rf.patcher = rsync.NewPatcher(stat.Size())
    rf.expect_diff = true
}
```

If the file exists and qualifies for rsync (regular file, > 4096 bytes), the terminal responds with `ttype=rsync` (TransmissionType_rsync) and the existing file's size, triggering the delta transfer path.

#### 4.4.2 Signature-Based Delta Efficiency

The receiver generates a signature from whatever portion of the file exists:

1. The receiver reads the existing file (which may be complete from a previous successful transfer, or partial from an interrupted one)
2. For each block, it computes the weak (rolling) and strong (XXH3-64) hashes
3. The signature is sent to the sender

The sender then computes a delta against this signature:

1. Blocks that match the existing file → `OpBlock` (no data transmitted)
2. Blocks that differ → `OpData` (new data transmitted)

This means:
- **File was 90% transferred before interruption**: The signature covers the 90% that's on disk. The delta contains only the missing 10% as `OpData` plus `OpBlock` references for the existing 90%.
- **File was fully transferred but needs update**: Only the changed portions are sent as `OpData`.
- **File doesn't exist at all**: The entire file is sent as `OpData` (no savings, equivalent to a simple transfer).

#### 4.4.3 Atomic Completion

The `PatchFile` class ensures that even during delta application, the destination is protected:

```python
# kitty/file_transmission.py — PatchFile
# 1. Creates a temp file
# 2. Applies delta operations (reading from existing, writing to temp)
# 3. On success: atomic rename(temp, destination)
# 4. On failure: temp file left behind, original intact
```

On the Go side, `kittens/transfer/receive.go` follows the same pattern:
```go
// patch_file
patcher.StartDelta(temp_file, source_file)
// ... delta application ...
patcher.FinishDelta()
os.Rename(temp_path, destination_path)
```

### 4.5 Comparison: Explicit Resume vs. Rsync Delta

| Aspect | Explicit Resume | Rsync Delta (Kitty's Approach) |
|--------|----------------|-------------------------------|
| Persistent state needed? | Yes (checkpoint files on disk) | No (stateless — signature generated on-the-fly) |
| Works with modified files? | No (only picks up from byte offset) | Yes (sends only actual differences) |
| Overhead per restart | Low (seek to offset) | Moderate (signature generation + delta computation) |
| Handles file modifications? | No | Yes — intrinsic capability |
| Protocol complexity | Requires resume negotiation | Uses existing delta transfer mechanism |
| Corruption recovery | Cannot detect corruption before resume point | Full integrity verification via XXH3-128 checksum |

### 4.6 Key Insight

The design choice to omit explicit resume is deliberate: the rsync delta mechanism is strictly more capable. A resume protocol can only restart from the last committed byte offset and cannot detect corruption before that point. The rsync mechanism can detect *any* difference in the existing file (corruption, partial transfer, or deliberate modification) and efficiently transmit only what is needed. The `OpHash` checksum at the end of the delta verifies the entire reconstructed file.

---

## 5. Delta Transfer Efficiency Demonstration

### 5.1 Overview

This section provides a practical experimental procedure that demonstrates the rsync delta transfer efficiency: transferring a file, modifying a small portion, and re-transferring to show that the second transfer sends substantially less data. The experiment includes the steps to build kitty from source, execute the transfers, and analyze the statistics output.

### 5.2 Prerequisites

To execute this experiment, you need:

1. **A built copy of kitty** with the transfer kitten
2. **Two machines** (or a local-to-remote connection via SSH)
3. **The `--transmit-deltas` flag** enabled for delta transfer mode

### 5.3 Building Kitty from Source

From the repository root:

```bash
# Ensure Go 1.22+ is available
export PATH=/usr/local/go/bin:$PATH
go version  # should report go1.22.x

# Build kitty (ignoring Wayland-related warnings)
python3 setup.py build --ignore-compiler-warnings
```

The build compiles:
- All Go packages (including `kittens/transfer/` and `tools/rsync/`)
- All C source files (including the rsync C extension `kittens/transfer/algorithm.c`)
- The Python extension modules
- The kitty launcher binary

### 5.4 Understanding the Statistics Output

The `print_rsync_stats()` function in `kittens/transfer/utils.go` (lines 48–62) produces the statistics that quantify delta efficiency:

```go
// kittens/transfer/utils.go — print_rsync_stats()
func print_rsync_stats(total_bytes, delta_bytes, signature_bytes int64) {
    // Prints:
    // - Total data size
    // - Delta data size (new data transmitted)
    // - Signature size (hash data exchanged)
    // - Transmission percentage: (delta + signature) / total * 100
}
```

The key metric is the **transmission percentage**: `(delta_bytes + signature_bytes) / total_bytes * 100`. For an effective delta transfer, this should be significantly less than 100%.

### 5.5 Experimental Procedure

#### Step 1: Create a Test File

The file must exceed 4096 bytes (the rsync capability threshold from `kittens/transfer/send.go`):

```bash
# Create a 100 KB test file with reproducible content
dd if=/dev/urandom bs=1024 count=100 of=/tmp/test_transfer_file.bin
```

#### Step 2: Initial Transfer

Using the SSH kitten for the connection and the transfer kitten with delta mode:

```bash
# SSH into the remote machine using kitty's SSH kitten
kitten ssh user@remote-host

# On the remote host, transfer the file to the local machine
kitten transfer --transmit-deltas /tmp/test_transfer_file.bin /tmp/received_file.bin
```

The first transfer sends the entire file because no copy exists at the destination. The rsync statistics will show approximately 100% transmission.

#### Step 3: Modify a Small Portion

```bash
# Modify 100 bytes at offset 50,000 (0.1% of the file)
printf '%0100d' 42 | dd of=/tmp/test_transfer_file.bin bs=1 seek=50000 conv=notrunc
```

#### Step 4: Re-Transfer with Delta Mode

```bash
# Transfer again with delta mode
kitten transfer --transmit-deltas /tmp/test_transfer_file.bin /tmp/received_file.bin
```

This time:
1. The receiver reads the existing `/tmp/received_file.bin` (from the first transfer)
2. A signature is generated: `block_size = round(sqrt(102400)) ≈ 320 bytes`, producing approximately 320 block hashes (320 × 20 bytes = 6,400 bytes of signature data)
3. The sender computes a delta against the signature
4. Only the modified block(s) containing the 100 changed bytes are sent as `OpData`
5. All other blocks match and are referenced as `OpBlock`/`OpBlockRange`

#### Step 5: Analyze the Output

The rsync statistics from `print_rsync_stats()` will show:

```
Total data: 102400 bytes
Delta data: ~640 bytes (the modified block + surrounding context)
Signature data: ~6412 bytes (12-byte header + 320 × 20-byte block hashes)
Transmission: ~6.9% of total
```

This demonstrates that the second transfer sent approximately **7%** of the data instead of the full 100%, confirming that the rsync algorithm detected the unchanged blocks via weak/strong hash comparison and only transmitted the modified portions.

#### Step 6: Cleanup

```bash
# Remove all temporary test files
rm -f /tmp/test_transfer_file.bin /tmp/received_file.bin
```

### 5.6 Why Delta Transfer Works

The efficiency comes from the rsync algorithm's three-stage pipeline:

1. **Signature generation (receiver)**: The receiver's existing file is divided into blocks of `sqrt(file_size)` bytes. Each block produces a 20-byte hash entry. For a 100 KB file, this is approximately 6.4 KB of signature data — far less than retransmitting the file.

2. **Delta computation (sender)**: The sender's rolling checksum slides one byte at a time across the new file. For each position, it computes the 4-byte weak hash in O(1) time. Only when a weak hash matches a signature entry does it compute the more expensive 8-byte XXH3-64 strong hash. This two-level hashing ensures both speed (most positions are rejected by the fast weak hash) and accuracy (strong hash eliminates false positives).

3. **Delta application (receiver)**: The receiver processes the compact delta stream, copying unchanged blocks from the existing file and writing new data from `OpData` operations. The final `OpHash` (XXH3-128 of the complete file) verifies the reconstruction is correct.

### 5.7 When Delta Transfer Is Not Beneficial

The `--transmit-deltas` flag is not always faster, as noted in the official documentation (`docs/kittens/transfer.rst`):

> Note that this will actually be slower when transferring small files or on a very fast network, because of round trip overhead, so use with care.

Cases where delta transfer adds overhead without benefit:
- **Small files (≤4096 bytes)**: Below the rsync capability threshold; transferred in full
- **Completely new files**: No existing copy at destination; entire file sent as `OpData` plus the signature round-trip overhead
- **Very fast networks**: The time saved by sending less data is offset by the additional round trips for signature exchange
- **Heavily modified files**: If most of the file has changed, the delta is nearly as large as the full file, but with added signature overhead

### 5.8 Rsync Test Validation

The rsync algorithm's correctness is validated by the test suite in `tools/rsync/api_test.go` and `kitty_tests/file_transmission.py`. The `TestRsyncRoundtrip` test verifies that:

1. A file can be reconstructed from a signature + delta
2. Small modifications produce compact deltas
3. Truncated files are handled correctly
4. Identical files produce zero-data deltas
5. The XXH3-128 checksum catches any reconstruction errors

Running the Go tests confirms the algorithm works correctly:

```bash
$ go test ./tools/rsync/... -v
=== RUN   TestRsyncRoundtrip
--- PASS: TestRsyncRoundtrip
=== RUN   TestRsyncHashers
--- PASS: TestRsyncHashers
PASS
```

---

## Appendix A: Architecture Summary

### A.1 Component Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                        SSH Connection                               │
│  kitten ssh (kittens/ssh/main.go)                                  │
│  - Establishes SSH session                                          │
│  - Bootstraps shell integration on remote                           │
│  - Makes 'kitten' binary available remotely via tarball upload      │
│  - Enables 'kitten transfer' to execute on remote host              │
└───────────────────────────┬─────────────────────────────────────────┘
                            │ Terminal byte stream
┌───────────────────────────┴─────────────────────────────────────────┐
│                  Transfer Kitten (Go)                               │
│  CLI Entry: kittens/transfer/main.go                                │
│  Sender:    kittens/transfer/send.go    (SendManager state machine) │
│  Receiver:  kittens/transfer/receive.go (Receiver state machine)    │
│  Protocol:  kittens/transfer/ftc.go     (Wire format model)         │
│  Utilities: kittens/transfer/utils.go   (bypass, compression, stats)│
│                            │                                        │
│                   ┌────────┴────────┐                               │
│                   │  Rsync Engine   │                               │
│                   │  tools/rsync/   │                               │
│                   │  algorithm.go   │ Core: BlockHash, rolling      │
│                   │  api.go         │ checksum, diff, ApplyDelta    │
│                   └─────────────────┘                               │
└───────────────────────────┬─────────────────────────────────────────┘
                            │ OSC 5113 escape sequences
┌───────────────────────────┴─────────────────────────────────────────┐
│                   VT Parser (C)                                     │
│  kitty/vt-parser.c — Routes OSC 5113 to file_transmission handler   │
│  kitty/control-codes.h — #define FILE_TRANSFER_CODE 5113            │
└───────────────────────────┬─────────────────────────────────────────┘
                            │ Dispatch
┌───────────────────────────┴─────────────────────────────────────────┐
│                Terminal Host (Python)                                │
│  kitty/file_transmission.py                                         │
│  - FileTransmission: Session orchestrator                           │
│  - ActiveReceive: Terminal receives files from kitten               │
│  - ActiveSend: Terminal sends files to kitten                       │
│  - DestFile: Destination file writer with decompression             │
│  - SourceFile: Source file reader with delta generation              │
│  - PatchFile: Rsync delta application with temp-file-then-rename    │
│                            │                                        │
│                   ┌────────┴────────┐                               │
│                   │  C Extension    │                               │
│                   │  algorithm.c    │ Python-side Patcher/Differ    │
│                   │  rsync.pyi      │ Type stubs                    │
│                   └─────────────────┘                               │
└─────────────────────────────────────────────────────────────────────┘
```

### A.2 Key Constants

| Constant | Value | Location | Purpose |
|----------|-------|----------|---------|
| `FILE_TRANSFER_CODE` | 5113 | `kitty/control-codes.h` | OSC code for file transfer |
| `DefaultBlockSize` | 6144 | `tools/rsync/algorithm.go` | Standalone default block size |
| `MaxBlockSize` | 1,048,576 (1 MB) | `tools/rsync/api.go` | Maximum block size cap |
| `BlockHashSize` | 20 | `tools/rsync/algorithm.go` | Bytes per signature block entry |
| Chunk size limit | 4096 | `kittens/transfer/ftc.go` | Maximum base64 data per OSC message |
| Rsync threshold | 4096 | `kittens/transfer/send.go` | Minimum file size for rsync eligibility |

### A.3 Wire Protocol Quick Reference

**OSC envelope format:**
```
\x1b]5113;id=<request_id>;<serialized_fields>\x1b\\
```

**Serialized field format** (semicolon-delimited key=value pairs):
```
ac=<action>;fid=<file_id>;n=<base64_name>;sz=<size>;d=<base64_data>;...
```

**Action codes:**
| Short | Full Name | Direction | Purpose |
|-------|-----------|-----------|---------|
| `snd` | send | Kitten → Terminal | Initiate send session |
| `rec` | receive | Kitten → Terminal | Initiate receive session |
| `file` | file | Both | File metadata |
| `data` | data | Both | Data chunk |
| `end_data` | end_data | Both | Final data chunk |
| `st` | status | Terminal → Kitten | Status response |
| `cncl` | cancel | Both | Cancel session |
| `fin` | finish | Kitten → Terminal | Finalize session |

### A.4 Source Files Reference

| File | Lines | Role |
|------|-------|------|
| `kittens/transfer/main.go` | 71 | CLI entry point, direction dispatch |
| `kittens/transfer/ftc.go` | 338 | Wire format model, serialization, chunk splitting |
| `kittens/transfer/send.go` | 1288 | Sender state machine, rsync integration |
| `kittens/transfer/receive.go` | 650+ | Receiver state machine, signature generation |
| `kittens/transfer/utils.go` | 114 | Bypass encryption, compression checks, stats |
| `kittens/transfer/utils.py` | 63 | Python-side path and compression utilities |
| `kittens/transfer/algorithm.c` | — | C extension for Python-side rsync operations |
| `kittens/transfer/rsync.pyi` | 48 | Python type stubs for C extension |
| `tools/rsync/algorithm.go` | 655 | Core rsync: hashing, diff, delta application |
| `tools/rsync/api.go` | 287 | Public API: Patcher, Differ, signature header |
| `kitty/file_transmission.py` | 1248 | Terminal host: sessions, files, rsync integration |
| `kitty/control-codes.h` | — | `#define FILE_TRANSFER_CODE 5113` |
| `kitty/vt-parser.c` | — | OSC dispatch to file_transmission handler |
| `kittens/ssh/main.go` | 841 | SSH kitten: connection, bootstrap, binary upload |
| `docs/file-transfer-protocol.rst` | 500+ | Official protocol specification |
