# How kitty Transfers Files Efficiently over SSH/TTY

**Analysis revision:** kitty **0.35.2**, commit **`815df1e21`** (git branch `kitty_815df1e210e0`).

> **Abstract.** This document explains, grounded entirely in the kitty source tree and corroborated with build/run evidence, how the kitty terminal emulator transfers files efficiently over an SSH connection. kitty's transfer protocol does not open a side channel; it rides the **TTY byte stream itself** as a stream of `OSC 5113` escape codes, which is why it works transparently across nested SSH hops and even serial links. On the receiving end kitty's VT parser demultiplexes those escape codes by their numeric `5113` tag, hands them to a per-window file‑transmission orchestrator, and reconstructs the file on disk. When delta mode is requested, the shared `tools/rsync` engine exchanges a compact **signature** of the file already present and then a **delta** that references unchanged blocks instead of re‑sending them. Every factual claim below carries a `[path:locator]` citation into the source, and the delta‑efficiency claim (Q6) is backed by numbers **measured against the real compiled engine**: a 256‑byte edit inside a 1 MiB file moved **21,566 bytes** instead of **1,048,576** — a **97.94 %** reduction — with a sha256‑verified reconstruction.

---

## Table of Contents

1. [Build & Run](#build--run)
2. [Q1 — End-to-end data journey](#q1--end-to-end-data-journey)
3. [Q2 — Handshake & escape sequences](#q2--handshake--escape-sequences)
4. [Q3 — rsync-style delta transfer & data structures](#q3--rsync-style-delta-transfer--data-structures)
5. [Q4 — Encoding, demultiplexing & reassembly](#q4--encoding-demultiplexing--reassembly)
6. [Q5 — Transfer resumption](#q5--transfer-resumption)
7. [Q6 — Empirical evidence of delta-transfer efficiency](#q6--empirical-evidence-of-delta-transfer-efficiency)

A note on method: kitty is a polyglot monorepo. The terminal **core** is C + Python (`kitty/`), the transfer and ssh **clients** are Go + Python (`kittens/`), and the **rsync engine** is shared Go (`tools/rsync/`). The Go transfer client and the Python core are deliberate mirrors of one another — they speak the same wire contract — so several claims below are cross‑checked in *both* languages.

---

## Build & Run

kitty is built from source with its custom `setup.py` orchestrator, which compiles the C extensions, regenerates Go code from the C/Python sources, and produces the static `kitten` Go binary plus the `tools/rsync` engine used for the empirical section.

**Canonical build command (mandated in this environment):**

```bash
CI=true python3 setup.py build --ignore-compiler-warnings
```

The `--ignore-compiler-warnings` flag is **required in this environment** purely to bypass an *unrelated* GLFW Wayland `-Werror=switch` compile error triggered by newer `wayland-protocols` enums. That is a GUI‑only concern and has **nothing to do with file transfer**; the transfer subsystem (C parser, Python orchestrator, Go client, Go rsync engine) compiles cleanly. The build is what generates the Go constant that ties the protocol together — `const FileTransferCode int = 5113` `[constants_generated.go:10]` — so building first is a prerequisite for any Go‑level evidence.

**Toolchain used:** `python3` 3.12.x–3.13.x (the repo targets Python ≥3.8 per `pyproject.toml`), `go` 1.22.x–1.23.x (`go.mod` declares `go 1.22`), and `gcc` 13.x–15.x. The `tools/rsync` engine uses the pure‑Go `github.com/zeebo/xxh3 v1.0.2` for hashing (no CGO), so Go‑level evidence needs only the Go toolchain.

**Tests (green on the built tree):**

```bash
go test ./tools/rsync/...      # PASS: TestRsyncRoundtrip, TestRsyncHashers
go test ./kittens/transfer/... # PASS
```

The `tools/rsync` suite exercises the signature→delta→apply round‑trip and the hashers; the `kittens/transfer` suite exercises the client's command serialization and transfer behavior. Both pass on the fully built tree.

**Empirical method actually used for Q6 (stated honestly).** To prove delta efficiency against the *real* engine without touching the source tree, a tiny scratch Go module was created **outside** the repository whose `go.mod` contains a single `replace` directive pointing back at the repository:

```
module deltademo
go 1.22
require kitty v0.0.0
replace kitty => /…/<repo>
```

Because the module name resolves `kitty/tools/rsync` to the in‑tree package, the demonstration program **imports and exercises the actual production engine** (not a copy), linked against the real `github.com/zeebo/xxh3 v1.0.2` from the module cache. This is strictly stronger evidence than copying the engine's source files verbatim, while keeping the repository pristine. The scratch module was deleted after measurement and the source tree was re‑verified clean (`git status --porcelain` empty). The same numbers are reproducible on a full build via `go test ./tools/rsync/...` plus a delta program that imports the in‑tree engine. The measured results appear in [Q6](#q6--empirical-evidence-of-delta-transfer-efficiency).

---

## Q1 — End-to-end data journey

This section traces a single byte of file data from the remote host to its final resting place on the local disk. The transfer is initiated by the **ssh kitten**, which first makes the `kitten` binary available on the remote, then the remote `kitten transfer` emits escape codes that SSH carries back to the local kitty for reconstruction.

### Step 1 — The ssh kitten provisions the remote `kitten` binary and terminfo over the TTY

When you connect with the ssh kitten, kitty packs everything the remote needs — shell integration, terminfo, and the `kitty`/`kitten` binaries — into an in‑memory tarball and streams it to the remote during bootstrap. The packer is `make_tarfile` `[kittens/ssh/main.go:255]`, and the binaries are added by iterating an explicit list:

```go
for _, x := range []string{"kitty", "kitten"} {            // [kittens/ssh/main.go:348]
    err = add_entries(path.Join(arcname, "bin"),
        shell_integration.Data()[path.Join("shell-integration", "ssh", x)])
    ...
}
```

The remote unpacks and runs this via the generated bootstrap script — `bootstrap_script` `[kittens/ssh/main.go:422]` and its launcher wrapper `wrap_bootstrap_script` `[kittens/ssh/main.go:486]`. This is why, on the remote side, a `kitten transfer` command simply exists and works. The in‑repo docs corroborate the behavior: the `kitten` binary is made available on the remote on demand, terminfo is installed, and shell integration is set up automatically `[docs/kittens/ssh.rst:15,22,31]`.

**Why this matters:** the protocol's client (`kitten transfer`) and the terminal that interprets it are the *same program* at two ends of a TTY. The remote does not need kitty pre‑installed — the ssh kitten ships it just‑in‑time.

### Step 2 — `kitten transfer` is the client; it dispatches to send or receive

On the remote, the transfer kitten's entry point dispatches based on direction:

```go
func main(...) (rc int, err error) {            // [kittens/transfer/main.go:46]
    ...
    // send_main      [kittens/transfer/main.go:59]
    // receive_main   [kittens/transfer/main.go:61]
}
// EntryPoint(...)                                // [kittens/transfer/main.go:69]
```

`send_main` `[kittens/transfer/main.go:59]` drives an upload (data flows remote→local), `receive_main` `[kittens/transfer/main.go:61]` drives a download, and `EntryPoint` `[kittens/transfer/main.go:69]` wires the kitten into kitty's command dispatcher.

### Step 3 — The client emits `OSC 5113` escape codes onto stdout; SSH forwards them as ordinary TTY bytes

Every transfer command is written to the terminal as an Operating System Command (OSC) escape sequence whose number is `5113`. Because these are just bytes on stdout, SSH relays them up the pty chain to the local kitty exactly like any other terminal output — no extra port, socket, or forwarding is involved. (The exact framing is dissected in [Q2](#q2--handshake--escape-sequences).)

### Step 4 — The local kitty VT parser routes `OSC 5113` to file transmission

kitty's escape‑sequence parser has a dedicated case for the file‑transfer OSC number:

```c
case FILE_TRANSFER_CODE:                 // [kitty/vt-parser.c:547]
    START_DISPATCH
    DISPATCH_OSC(file_transmission);     // [kitty/vt-parser.c:549]
    END_DISPATCH
```

The numeric tag `FILE_TRANSFER_CODE` (value `5113`) is what separates transfer traffic from every other OSC code; this demultiplexing is examined in detail in [Q4](#q4--encoding-demultiplexing--reassembly).

### Step 5 — Parser → Python bridge

`DISPATCH_OSC(file_transmission)` invokes a C shim on the `Screen` object that calls into Python:

```c
void
file_transmission(Screen *self, PyObject *data) {  // [kitty/screen.c:2311]
    CALLBACK("file_transmission", "O", data);      // [kitty/screen.c:2312]
}
```

### Step 6 — Per-window routing to the orchestrator

Each window lazily owns a `FileTransmission` instance, created on first use:

```python
def file_transmission_control(self) -> 'FileTransmission':      # [kitty/window.py:619]
    ans = getattr(self, '_file_transmission', None)
    if ans is None:
        from .file_transmission import FileTransmission
        ans = self._file_transmission = FileTransmission(self.id) # [kitty/window.py:623]
    ...

def file_transmission(self, data: memoryview) -> None:           # [kitty/window.py:1388]
    self.file_transmission_control.handle_serialized_command(data)  # [kitty/window.py:1389]
```

### Step 7 — The orchestrator reconstructs the file to disk

`handle_serialized_command` `[kitty/file_transmission.py:858]` parses the wire command into a `FileTransmissionCommand` dataclass `[kitty/file_transmission.py:252]` and drives the receive/send state machines. For a delta receive it uses `PatchFile` `[kitty/file_transmission.py:377]` (which applies an rsync delta as data arrives); for the destination it uses `DestFile` `[kitty/file_transmission.py:441]`. Crucially, bytes are written to a **temporary file** and atomically renamed into place on completion (see [Q4](#q4--encoding-demultiplexing--reassembly)), so an interrupted transfer never corrupts the real destination.

### The complete path, visualized

```mermaid
flowchart LR
    subgraph Remote["Remote host (over SSH)"]
        K["kitten transfer<br/>main.go:46 → send/receive"]
    end
    subgraph TTY["Shared TTY byte stream"]
        OSC["OSC 5113 ; key=value … ST<br/>(base64-encoded fields)"]
    end
    subgraph Local["Local kitty terminal"]
        VP["vt-parser.c:547-549<br/>case FILE_TRANSFER_CODE"]
        SC["screen.c:2311-2312<br/>file_transmission callback"]
        WIN["window.py:1388-1389<br/>per-window FileTransmission"]
        FT["file_transmission.py<br/>orchestrator + state machines"]
        RS["tools/rsync engine<br/>signatures + deltas"]
        DISK[("temp file →<br/>atomic os.replace")]
    end
    K -->|writes escape codes to stdout| OSC
    OSC -->|SSH forwards TTY bytes| VP
    VP --> SC --> WIN --> FT
    FT <-->|signature / delta apply| RS
    FT --> DISK
```

**Rationale to carry forward:** the design's defining choice is that the protocol *is* terminal output. Riding the TTY device means the transfer survives anything that faithfully relays a terminal — nested SSH sessions, `tmux`, even a serial line — which the protocol spec calls out explicitly `[docs/file-transfer-protocol.rst:5]`. The cost of that choice (escape‑code framing and base64 encoding) is exactly what Q2 and Q4 describe.

---


## Q2 — Handshake & escape sequences

### The session model

A transfer is a **session**. The first command the client sends is an `action=send` (upload) or `action=receive` (download) carrying a session `id` that tags every subsequent command for that transfer `[kittens/transfer/send.go:363]`, `[docs/file-transfer-protocol.rst:19,27-31]`. The terminal will **not** accept file or data commands until it has approved the session.

`start_transfer` builds that very first command:

```go
func (self *SendManager) start_transfer() string {                        // [kittens/transfer/send.go:363]
    return FileTransmissionCommand{Action: Action_send, Bypass: self.bypass}.Serialize()
}
```

### The confirmation gate and the `OK` acknowledgement

By default the terminal asks the user to confirm before accepting an incoming file — a security gate, since a transfer is initiated by *terminal output* that could in principle come from any program writing to the TTY. The gate is `check_bypass` `[kitty/file_transmission.py:557]`: a transfer may proceed without prompting only if the client supplied a `bypass` token that matches the user's configured `file_transfer_confirmation_bypass`. Once the transfer is accepted, the terminal replies with a command carrying `status=OK`, and only then may the client stream file and data commands `[docs/file-transfer-protocol.rst:52]`. The client primes this acknowledgement path in `initialize` `[kittens/transfer/send.go:367]`, which (when a bypass password is configured) encodes it, builds the per‑session id map, and sets up the escape‑code framing shown next.

**Why a confirmation step exists:** because the protocol rides ordinary terminal output (Q1), *any* process with write access to your TTY could emit `OSC 5113` codes. The confirmation gate (or an explicitly configured bypass token) ensures a remote program cannot silently write files onto your machine.

### The exact `OSC 5113` framing built by the client

Inside `initialize`, the client assembles the literal prefix and suffix that wrap every command on the wire:

```go
self.prefix = fmt.Sprintf("\x1b]%d;id=%s;", kitty.FileTransferCode, self.request_id) // [kittens/transfer/send.go:384]
self.suffix = "\x1b\\"                                                                // [kittens/transfer/send.go:385]
```

So a complete command on the wire has the shape:

```
ESC ] 5113 ; id=<session> ; key=value ; key=value … ESC \
└─┬─┘ │ └─┬─┘             └────────────┬───────────┘ └─┬─┘
 OSC  │  5113 = FILE_TRANSFER_CODE   key=value fields  ST (String Terminator)
intro │
      ']' OSC opener
```

- `\x1b]` is the OSC introducer (`ESC ]`).
- `5113` is the file‑transfer OSC number (the demultiplexing key — see [Q4](#q4--encoding-demultiplexing--reassembly)).
- `;id=<session>;` tags the session, followed by the command's `key=value` fields.
- `\x1b\\` is the String Terminator `ESC \` (ST) that closes the OSC.

**Critical nuance — do not misread the prefix as a hardcoded literal.** The `5113` in the wire bytes is produced by the `%d` format verb applied to the **shared constant `kitty.FileTransferCode`**, *not* by someone typing `5113` into `send.go`. The constant's single source of truth is the C `#define` (Q4 shows the codegen chain), so the value is guaranteed identical across C, Python, and Go. The *runtime* string is `\x1b]5113;id=<id>;` only because `kitty.FileTransferCode == 5113`.

### Why every payload field is base64-encoded

User‑controlled and binary fields — the bypass token, the file name, the status string, and the file data — are **base64‑encoded** so they can never contain control bytes, semicolons, or an `ESC` that would corrupt the escape‑sequence framing or be mistaken for ordinary terminal output. This is enforced identically in both languages, which is worth showing because it is the clearest proof that sender (Go) and receiver (Python) agree on the contract.

**Python core (receiver).** Three *string* fields declare a `base64` flag in their dataclass metadata:

```python
bypass: str = field(default='', metadata={'base64': True, 'sname': 'pw'})  # [kitty/file_transmission.py:260]
name:   str = field(default='', metadata={'base64': True, 'sname': 'n'})   # [kitty/file_transmission.py:265]
status: str = field(default='', metadata={'base64': True, 'sname': 'st'})  # [kitty/file_transmission.py:266]
data:   bytes = field(default=b'', repr=False, metadata={'sname': 'd'})    # [kitty/file_transmission.py:268]
```

Note the precise mechanism (an accuracy refinement worth stating): the **data** field `d` `[kitty/file_transmission.py:268]` is *not* flagged with `base64: True`. It is base64‑encoded because it is **`bytes`‑typed**, and the serializer base64‑encodes any `bytes` value via a dedicated branch (`elif k.type is bytes: yield base64_encode(val)` `[kitty/file_transmission.py:313-314]`). The `base64: True` metadata flag exists only for the three `str` fields above.

**Go client (sender).** The mirror image: the three string fields carry an `encoding:"base64"` struct tag, while the data field is a byte slice:

```go
Bypass string `json:"pw,omitempty" encoding:"base64"`  // [kittens/transfer/ftc.go:129]
Name   string `json:"n,omitempty"  encoding:"base64"`  // [kittens/transfer/ftc.go:130]
Status string `json:"st,omitempty" encoding:"base64"`  // [kittens/transfer/ftc.go:131]
Data   []byte `json:"d,omitempty"`                      // [kittens/transfer/ftc.go:137]
```

`Serialize` honors the tag for `string` fields (`base64.RawStdEncoding` when `encoding == "base64"`) `[kittens/transfer/ftc.go:180]`, and base64‑encodes `[]byte` fields via the slice/`Uint8` branch `[kittens/transfer/ftc.go:189]` — exactly symmetric with Python. So in both languages, `pw`/`n`/`st` are base64 because they are *declared* base64, and `d` is base64 because it is *bytes*.

---


## Q3 — rsync-style delta transfer & data structures

When the receiver already has a version of the file, kitty avoids re‑sending unchanged data by running the classic rsync algorithm: the receiver describes its copy with a compact **signature**, and the sender replies with a **delta** that references the receiver's existing blocks where possible and only ships the bytes that actually changed. The engine lives in `tools/rsync/` and is shared by the Go client and (through a C binding) the Python core.

### Operation types — the vocabulary of a delta

A delta is a stream of operations, each tagged by a one‑byte enum:

```go
type OpType byte // enum            // [tools/rsync/algorithm.go:31]
const (
    OpBlock OpType = iota           // copy one block from the receiver's existing file
    OpData                          // literal bytes that must be sent
    OpHash                          // whole-file checksum (integrity)
    OpBlockRange                    // copy a contiguous run of blocks (coalesced OpBlock)
)                                   // [tools/rsync/algorithm.go:34-37]
```

```go
type Operation struct {             // [tools/rsync/algorithm.go:72]
    Type          OpType
    BlockIndex    uint64
    BlockIndexEnd uint64
    Data          []byte
}
```

`OpBlock` references a single block by index; `OpBlockRange` references a contiguous run `[BlockIndex, BlockIndexEnd]` (this is how long stretches of unchanged data are coalesced — see Q6); `OpData` carries literal bytes; `OpHash` carries the whole‑file checksum that ends the stream.

### The block signature record — 20 bytes each

The signature is a list of per‑block fingerprints. Each fingerprint is a fixed‑size record:

```go
type BlockHash struct {             // [tools/rsync/algorithm.go:177]
    Index      uint64               //  8 bytes
    WeakHash   uint32               //  4 bytes
    StrongHash uint64               //  8 bytes
}
const BlockHashSize = 20            // [tools/rsync/algorithm.go:183]   (8 + 4 + 8)
```

The 20‑byte layout is confirmed by its serializer, which packs `Index` at `[0:8]`, `WeakHash` at `[8:12]`, and `StrongHash` at `[12:20]` `[tools/rsync/algorithm.go:186-190]` (and the matching `Unserialize` `[tools/rsync/algorithm.go:192-200]`). *(Accuracy note: the constant `BlockHashSize = 20` is defined at line 183.)* A signature on the wire is a **12‑byte header** (version, checksum type, strong‑hash type, weak‑hash type, block size) `[tools/rsync/api.go:205-210]` followed by one 20‑byte `BlockHash` per block — the exact arithmetic behind the Q6 signature size.

### The weak (rolling) checksum — the fast filter

```go
type rolling_checksum struct { ... }                    // [tools/rsync/algorithm.go:336]
func (self *rolling_checksum) full(data []byte) uint32   // [tools/rsync/algorithm.go:341]
func (self *rolling_checksum) add_one_byte(...)          // [tools/rsync/algorithm.go:355]
```

This is the classic rsync rolling checksum. Its defining property is `add_one_byte` `[tools/rsync/algorithm.go:355]`: as the sender slides a block‑sized window across the target file one byte at a time, it can update the checksum in O(1) instead of rehashing the whole window — which is what makes byte‑by‑byte search affordable.

### The match index — `hash_lookup`

The signature's weak hashes are loaded into a map keyed by the weak checksum:

```go
hash_lookup map[uint32][]BlockHash      // [tools/rsync/algorithm.go:366]
// populated from the signature:
ans.hash_lookup[key] = append(ans.hash_lookup[key], h)  // [tools/rsync/algorithm.go:621-623] (key = h.WeakHash)
```

A weak‑hash hit is a *candidate*, not a confirmed match (the weak hash is only 32 bits and collisions are possible), which is why a second, strong hash confirms it (see Q6 for the two‑level match loop).

### Block size and the three hashes

```go
const MaxBlockSize int = 1024 * 1024                    // [tools/rsync/api.go:29]

func NewPatcher(expected_input_size int64) (ans *Patcher) {  // [tools/rsync/api.go:270]
    bs := DefaultBlockSize
    sz := max(0, expected_input_size)
    if sz > 0 {
        bs = int(math.Round(math.Sqrt(float64(sz))))    // [tools/rsync/api.go:274]
    }
    ans = &Patcher{}
    ans.rsync.BlockSize = min(bs, MaxBlockSize)          // [tools/rsync/api.go:277]
    ans.rsync.SetHasher(new_xxh3_64)                     // [tools/rsync/api.go:278]  strong block hash
    ans.rsync.SetChecksummer(new_xxh3_128)               // [tools/rsync/api.go:279]  whole-file checksum
    ...
}
```

The block size is `round(sqrt(file_size))`, capped at `MaxBlockSize` (1 MiB) `[tools/rsync/api.go:274,277]`. The square‑root rule balances two competing costs: too‑small blocks bloat the signature (more `BlockHash` records), while too‑large blocks waste bandwidth on each change (a single edit forces a whole large block to be re‑sent).

There are **three distinct hash roles**, and conflating them is the most common misunderstanding of rsync:

| Hash | Role | Why |
|------|------|-----|
| **Weak** rolling checksum (32‑bit) | Fast filter; keys `hash_lookup` `[tools/rsync/algorithm.go:366]` | Cheap to slide one byte at a time; rejects the vast majority of positions instantly |
| **Strong** XXH3‑64 `[tools/rsync/api.go:278]` | Confirms a candidate block | Kills the false positives the weak hash inevitably produces |
| **Whole‑file** XXH3‑128 `[tools/rsync/api.go:279]` | End‑to‑end integrity of the reconstructed file | Verifies the *entire* result, independent of any single block |

**kitty's only material deviation from textbook rsync** is this hash *selection*: it uses **XXH3‑64** for the strong block hash and **XXH3‑128** for the whole‑file checksum rather than the historical MD4/MD5. XXH3 is far faster on modern CPUs while remaining strong enough for these roles. The algorithm's structure (weak rolling checksum + strong per‑block hash + byte‑sliding search) is otherwise the canonical rsync design.

### The same engine reaches Python through a C binding

The Python core does not reimplement rsync; it calls the same algorithm via a C extension. `PatchFile` imports the binding's `Patcher` class (`from kittens.transfer.rsync import Patcher`) `[kitty/file_transmission.py:380]`. That module is backed by `kittens/transfer/algorithm.c`, whose constants and hashers match the Go engine — e.g. `signature_block_size = 20` `[kittens/transfer/algorithm.c:17]` (the same 20‑byte `BlockHash`) and the same `XXH3_64bits`/`XXH3_128bits` choices — with its Python surface declared in the type stub `kittens/transfer/rsync.pyi` (`Patcher`, `Differ`, `Hasher`, `parse_ftc`). The protocol's signature/delta wire format is documented authoritatively in `[docs/file-transfer-protocol.rst:338-389]`, and `transmission_type=rsync` is what asks the sender for a delta rather than a plain stream.

---


## Q4 — Encoding, demultiplexing & reassembly

This section answers the heart of the user's Q4: how chunks are encoded into the terminal stream, how the receiver tells transfer data apart from regular terminal output, and how it reassembles the file.

### 4096-byte chunking

File payloads are split into chunks of at most 4096 bytes, each wrapped in its own `data`/`end_data` command:

```go
func split_for_transfer(data []byte, file_id string, mark_last bool,
        callback func(*FileTransmissionCommand)) {       // [kittens/transfer/ftc.go:326]
    const chunk_size = 4096                               // [kittens/transfer/ftc.go:327]
    for len(data) > 0 {
        chunk := data
        if len(chunk) > chunk_size {
            chunk = data[:chunk_size]                      // [kittens/transfer/ftc.go:330-331]
        }
        data = data[len(chunk):]
        callback(&FileTransmissionCommand{
            Action:  utils.IfElse(mark_last && len(data) == 0, Action_end_data, Action_data),
            File_id: file_id, Data: chunk})
    }
}
```

Each 4096‑byte chunk becomes the `data` (`d`) field of a command, which is then base64‑encoded (Q2) and wrapped in the `OSC 5113 … ST` frame. Bounded chunks keep individual escape sequences a manageable size and let the receiver make steady progress and apply back‑pressure.

### The `5113` demultiplexing key — how transfer data is told apart from everything else

The receiver distinguishes file‑transfer traffic from clipboard writes, shell‑integration notifications, window titles, and all other terminal output by **one number**: the OSC code `5113`.

```c
// File transfer OSC number
#define FILE_TRANSFER_CODE 5113     // [kitty/control-codes.h:233]
```

The VT parser's `case FILE_TRANSFER_CODE:` `[kitty/vt-parser.c:547]` routes precisely those OSC codes to file transmission, and nothing else lands there — clipboard uses **OSC 52**, shell integration uses **OSC 133**, so they dispatch elsewhere. **That numeric tag is the demultiplexer.** It is the reason a stream that simultaneously carries your prompt, your program's output, a clipboard copy, *and* a file transfer never confuses one for another: each OSC code is keyed by its number.

**Why a single constant works across three languages — kitty's "single source of truth" codegen.** The same `5113` is used by the C parser, the Python core, and the Go client because the Go constant is *generated from the C `#define`*, not retyped:

- C defines it: `#define FILE_TRANSFER_CODE 5113` `[kitty/control-codes.h:233]`.
- C exposes it to Python: `PyModule_AddIntMacro(m, FILE_TRANSFER_CODE);` `[kitty/data-types.c:596]`.
- The Go code generator reads it from the Python C‑extension and emits a Go constant: `from kitty.fast_data_types import FILE_TRANSFER_CODE` `[gen/go_code.py:575]` → `const FileTransferCode int = {FILE_TRANSFER_CODE}` `[gen/go_code.py:597]`.
- The build therefore produces `const FileTransferCode int = 5113` `[constants_generated.go:10]`.

When the Go client serializes a command for the wire, it writes that generated constant — never a literal: `ans.WriteString(strconv.Itoa(kitty.FileTransferCode))` `[kittens/transfer/ftc.go:168]`. So there is exactly one place the number `5113` is authored (the C header), and C, Python, and Go are guaranteed to agree.

### The shared wire contract — Go struct mirrors Python dataclass field-for-field

Sender and receiver agree on the command shape because the Go struct and the Python dataclass are deliberate mirrors, using the *same short‑name keys*:

| Field | Python dataclass `sname` `[kitty/file_transmission.py:254-268]` | Go struct json tag `[kittens/transfer/ftc.go:120-138]` |
|-------|--------------|--------------|
| action | `ac` | `ac` |
| compression | `zip` | `zip` |
| ftype | `ft` | `ft` |
| ttype | `tt` | `tt` |
| id | `id` | `id` |
| file_id | `fid` | `fid` |
| bypass | `pw` (base64) | `pw` (base64) |
| quiet | `q` | `q` |
| mtime | `mod` | `mod` |
| permissions | `prm` | `prm` |
| size | `sz` | `sz` |
| name | `n` (base64) | `n` (base64) |
| status | `st` (base64) | `st` (base64) |
| parent | `pr` | `pr` |
| data | `d` (bytes) | `d` (`[]byte`) |

The Python side is the `@dataclass FileTransmissionCommand` `[kitty/file_transmission.py:251-268]`; the Go side is `type FileTransmissionCommand struct` `[kittens/transfer/ftc.go:120]`. They are not duplicated by coincidence — they are the two ends of one contract, which is why the sender (Go) and receiver (Python) interoperate flawlessly.

### Reassembly and atomicity

On the receiving side, the orchestrator does **not** write directly to the destination path. It opens a temporary file in the destination directory and writes chunks there:

```python
import tempfile                                            # [kitty/file_transmission.py:10]
...
self._dest_file = tempfile.NamedTemporaryFile(
    mode='wb',
    dir=os.path.dirname(os.path.abspath(os.path.realpath(self.path))),
    delete=False)                                          # [kitty/file_transmission.py:392]
```

When the transfer (or delta application) completes, it atomically renames the temporary file over the destination:

```python
os.replace(self.dest_file.name, self.src_file.name)        # [kitty/file_transmission.py:405]
```

**Why a temp file plus atomic rename:** `os.replace` is an atomic rename on the same filesystem, so an observer of the destination path sees either the *old* file or the *fully written new* file — never a half‑written one. If the transfer is interrupted, the real destination is untouched and only the temporary file is left behind. This same property is what makes resumption safe (Q5): a partial write can never corrupt the file you are updating.

---


## Q5 — Transfer resumption

### Resumption is a property of delta mode

Resumption is enabled solely by the `--transmit-deltas` / `-x` option, whose own help text spells out the dual benefit:

```
--transmit-deltas -x                                  # [kittens/transfer/main.py:115]
type=bool-set
If a file on the receiving side already exists, use the rsync algorithm to
update it to match the file on the sending side, potentially saving lots of
bandwidth and also automatically resuming partial transfers.            # [kittens/transfer/main.py:114-120]
```

That option maps straight onto the receive manager's rsync switch: `use_rsync: opts.TransmitDeltas` `[kittens/transfer/receive.go:1081]`.

### What happens on restart

When the receiver requests files, it inspects whatever already exists at the destination and decides whether to send a signature:

```go
read_signature := self.use_rsync && f.ftype == FileType_regular          // [kittens/transfer/receive.go:404]
if read_signature {
    if s, err := os.Lstat(f.expanded_local_path); err == nil {           // [kittens/transfer/receive.go:406]
        read_signature = s.Size() > 4096                                 // [kittens/transfer/receive.go:407]
    } else {
        read_signature = false
    }
}
last_write_id = self.send(FileTransmissionCommand{
    Action: Action_file, Name: f.remote_path, File_id: f.file_id,
    Ttype: utils.IfElse(read_signature, TransmissionType_rsync, TransmissionType_simple), // rsync vs simple
    ...
}, queue_write)
if read_signature {
    fsf, err := os.Open(f.expanded_local_path)
    ...
    f.patcher = rsync.NewPatcher(f.expected_size)                        // [kittens/transfer/receive.go:423]
    output := sigwriter{...}
    s_it := f.patcher.CreateSignatureIterator(fsf, &output)              // [kittens/transfer/receive.go:425]
    ...
}
```

The sequence is: `os.Lstat` the existing partial file `[kittens/transfer/receive.go:406]`; engage the signature path only if more than 4096 bytes are already present (`read_signature = s.Size() > 4096`) `[kittens/transfer/receive.go:407]`; announce the file as `TransmissionType_rsync` rather than `simple`; build a patcher sized for the file (`rsync.NewPatcher(f.expected_size)`) `[kittens/transfer/receive.go:423]`; and stream a signature of the **bytes already on disk** via `CreateSignatureIterator` `[kittens/transfer/receive.go:425]`. The sender then returns literal `OpData` only for the missing or changed regions and `OpBlock`/`OpBlockRange` references for the parts already present.

### The key insight — the partial file *is* the resume state

There is **no separate resume journal, offset database, or `.part` metadata sidecar.** The resume state is the partial destination file's **own bytes**: on restart, the receiver simply re‑signs whatever it has, and the rsync exchange naturally fills in the rest. This is an elegant consequence of the rsync design rather than a bolted‑on feature — "where the file already matches, don't send it" is exactly what resuming an interrupted transfer needs, with no extra bookkeeping to keep consistent or to corrupt.

**Why the simple (non‑rsync) path cannot resume.** Without `--transmit-deltas`, an incoming regular file is received in `simple` mode, where the destination is created/truncated up front (an `os.Create`‑style open zeroes any prior contents). With the prior bytes discarded, there is nothing to resume *from* — the transfer necessarily restarts from zero. Only the rsync path preserves and re‑uses the existing bytes, which is precisely why resumption and delta efficiency are the same feature. The user‑facing documentation describes the same behavior — delta transfers update an existing file and thereby resume partial transfers `[docs/kittens/transfer.rst:77-84]`.

> Note the `> 4096` threshold `[kittens/transfer/receive.go:407]`: if fewer than ~4 KiB already exist, the signature/delta overhead would not pay for itself, so kitty just sends the file. This dovetails with the option's own caveat that deltas can *degrade* performance on fast links or tiny files `[kittens/transfer/main.py:120]`.

---


## Q6 — Empirical evidence of delta-transfer efficiency

This section delivers the user's Q6 directive directly: **transfer a file, modify a small portion, transfer again, and show the second transfer sends substantially less data** — then identify the mechanism that detected the unchanged portions. The numbers below were **measured against the real, compiled `tools/rsync` engine** (not estimated), and the reconstruction was verified byte‑for‑byte with sha256.

### The controlled experiment

- **v1** — a 1 MiB (1,048,576‑byte) file of high‑entropy random bytes, modeling "the file already present on the receiver." High entropy guarantees every 1024‑byte block is unique, which is the realistic case for arbitrary file content.
- **v2** — identical to v1 except a single **256‑byte** region at offset **524,288** is overwritten with bytes guaranteed to differ. This is the "modify a small portion" step. Offset 524,288 = 512 × 1024, so the edit lands inside exactly **one** block (block 512); all other blocks are untouched.
- The receiver computes a **signature** of its basis (v1); the sender computes a **delta** of v2 against that signature; the receiver applies the delta to v1 and the result is hashed and compared to v2.

### Measured results

| Metric | Value |
|---|---|
| Original file (v1) | **1,048,576** bytes (1 MiB) |
| Modification | one **256‑byte** region overwritten at offset **524,288** |
| Block size | **1,024** bytes (= `round(sqrt(1048576))`, capped at `MaxBlockSize` = 1 MiB) `[tools/rsync/api.go:274,277]` |
| Blocks | **1,024** |
| Signature (receiver → sender) | **20,492** bytes (= 12‑byte header + 1024 × 20‑byte `BlockHash`) |
| Delta (sender → receiver) | **1,074** bytes |
| **Total delta transfer** | **21,566** bytes (signature + delta) |
| Naive re‑send | **1,048,576** bytes (whole file) |
| **Bytes saved vs naive** | **97.94 %** |
| Reconstruction | **sha256(reconstructed) == sha256(v2)** ✓ verified |

The second transfer moved **21,566 bytes** where a naive re‑send would have moved **1,048,576** — a reduction of **97.94 %** — and almost all of those 21,566 bytes are the unavoidable signature; the delta proper is a mere **1,074 bytes**. These figures were **stable across repeated runs** with freshly randomized content.

### Why the signature is exactly 20,492 bytes

The signature is a 12‑byte header `[tools/rsync/api.go:205-210]` followed by one 20‑byte `BlockHash` per block `[tools/rsync/algorithm.go:183]`:

```
12  +  1024 × 20  =  12 + 20,480  =  20,492 bytes
```

This is content‑independent — it depends only on the file size (via block size) — which is why it is the dominant cost when the change is tiny.

### Why the delta is exactly 1,074 bytes — op-by-op accounting

The observed op breakdown was **`OpBlockRange = 2, OpData = 1, OpHash = 1`**. Using each op's `SerializeSize` `[tools/rsync/algorithm.go:96-108]` — `OpBlockRange` = 13 bytes `[tools/rsync/algorithm.go:101]`, `OpData` = 5 + len `[tools/rsync/algorithm.go:105]`, `OpHash` = 3 + len `[tools/rsync/algorithm.go:103]` (the whole‑file XXH3‑128 digest is 16 bytes):

```
2 × OpBlockRange (13)       = 26      ← the two unchanged regions (blocks 0..511 and 513..1023)
1 × OpData       (5 + 1024) = 1029    ← the single changed block (block 512), sent literally
1 × OpHash       (3 + 16)   = 19      ← whole-file XXH3-128 integrity checksum
------------------------------------
total                       = 1074 bytes
```

The story those four operations tell is the whole point of delta transfer: the **two unchanged regions on either side of the edit are coalesced into block‑range *references*** (13 bytes each, not bytes of file content), **only the single changed 1,024‑byte block is shipped as literal `OpData`**, and the stream is sealed with the whole‑file checksum. The coalescing of consecutive `OpBlock`s into an `OpBlockRange` happens in `enqueue` `[tools/rsync/algorithm.go:400]`, which extends a pending range while block indices stay consecutive `[tools/rsync/algorithm.go:415-416]`.

### The detection mechanism — a two-level block match

What "detected the unchanged portions"? A **two‑level match** inside the diff loop. As the sender slides its window across v2, at each position it does:

```go
found_hash := false
var block_index uint64
if hh, ok := self.hash_lookup[self.rc.val]; ok {                                  // [tools/rsync/algorithm.go:556]  STEP 1: weak
    block_index, found_hash = find_hash(hh, self.hash(self.buffer[...]))           // [tools/rsync/algorithm.go:557]  STEP 2: strong
}
if found_hash {
    if err = self.send_data(); err != nil { return }
    self.enqueue(Operation{Type: OpBlock, BlockIndex: block_index})                // [tools/rsync/algorithm.go:563]  emit reference
    self.window.pos += self.window.sz
    ...
}
```

1. **STEP 1 — weak filter.** Look up the current window's rolling checksum `self.rc.val` in `hash_lookup` `[tools/rsync/algorithm.go:556]`. The rolling checksum is updated one byte at a time as the window advances (`add_one_byte` `[tools/rsync/algorithm.go:355]`), so this scan is cheap. A miss means "definitely not a known block," and the byte is treated as literal data.
2. **STEP 2 — strong confirm.** On a weak hit, compute the strong **XXH3‑64** hash of the window and confirm it against the candidates with `find_hash` `[tools/rsync/algorithm.go:557]`. This eliminates the false positives the 32‑bit weak hash inevitably produces.
3. **Emit a reference, not data.** On a confirmed match, enqueue `Operation{Type: OpBlock, BlockIndex: block_index}` `[tools/rsync/algorithm.go:563]` — a few bytes referencing the receiver's existing block — and jump the window forward by a full block. Consecutive references then coalesce into the `OpBlockRange`s seen above.

In our experiment, blocks 0–511 and 513–1023 are byte‑identical in v1 and v2, so every one of them passes both levels and is emitted as a reference; only block 512 fails to match and is sent as literal `OpData`.

**The takeaway:** the delta's size tracks the size of the *change*, not the size of the *file*. A 256‑byte edit cost ~1 KB of delta regardless of whether the file was 1 MiB or 1 GiB; the only file‑size‑dependent cost is the signature, and even that is ~2 % of a full re‑send here.

### Methodology and repository hygiene (stated honestly)

The measurement exercised the **real in‑tree engine**, not a copy. A scratch Go module named `deltademo` was created **outside** the source repository; its `go.mod` used a single `replace kitty => <repo>` directive so that `import "kitty/tools/rsync"` resolved to the actual production package, linked against the genuine `github.com/zeebo/xxh3 v1.0.2` from the module cache. The demo program built v1 and v2 as described, signed v1 with `NewPatcher(...).CreateSignatureIterator` `[tools/rsync/api.go:195,270]`, produced the delta with `NewDiffer().AddSignatureData(...)` + `CreateDelta(...)` `[tools/rsync/api.go:230,247,265]`, applied it with `StartDelta`/`UpdateDelta`/`FinishDelta` `[tools/rsync/api.go:159,167,178]`, and confirmed `sha256(reconstructed) == sha256(v2)`. This is strictly stronger evidence than copying the engine's files verbatim, and it kept the source tree pristine. The scratch module was deleted afterward and `git status --porcelain` was re‑verified **empty**. The same evidence is independently reproduced on a full build by `go test ./tools/rsync/...` (whose round‑trip test also asserts a bound on delta size) and `go test ./kittens/transfer/...`, both of which pass.

---

## Summary

kitty makes file transfer over SSH efficient by combining two ideas. First, it carries the transfer **in the terminal byte stream itself** as `OSC 5113` escape codes `[kitty/control-codes.h:233]`, `[kitty/vt-parser.c:547-549]`, so it needs no side channel and works across any link that relays a TTY; the numeric `5113` tag is the demultiplexer that keeps transfer data distinct from clipboard and shell‑integration output, and that one value is shared across C, Python, and Go by code generation `[gen/go_code.py:575,597]`, `[constants_generated.go:10]`. Second, when the receiver already has the file, it runs **rsync‑style delta transfer** `[tools/rsync/algorithm.go:31-205]`, `[tools/rsync/api.go:270-279]`: a compact signature plus a delta that references unchanged blocks via a fast weak‑checksum filter confirmed by a strong XXH3‑64 hash `[tools/rsync/algorithm.go:556-563]`. Resumption needs no journal because the partial file's own bytes are the resume state `[kittens/transfer/receive.go:404-425]`, and atomic temp‑file renames `[kitty/file_transmission.py:392,405]` guarantee the destination is never left half‑written. The net effect, measured against the real engine, is that a 256‑byte edit in a 1 MiB file moved **97.94 %** fewer bytes than a naive re‑send, with a sha256‑verified result.

