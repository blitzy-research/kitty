# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification

### 0.1.1 Core Documentation Objective

Based on the provided requirements, the Blitzy platform understands that the documentation objective is to **create new documentation** that provides a deep, code-grounded technical walkthrough of Kitty's diff kitten runtime behavior — specifically how it compares directories, detects renames, manages caches, parallelizes syntax highlighting, and handles heterogeneous file types (text, binary, and images).

- **Documentation Category:** Create new documentation
- **Documentation Type:** Technical deep-dive / Architectural walkthrough
- **Target Audience:** A developer onboarding into the Kitty repository who wants an intuitive, runtime-oriented understanding of the diff kitten's internal behavior

The user's questions decompose into the following discrete documentation requirements:

- **Directory comparison lifecycle:** Trace the complete runtime flow from the moment two directory paths are accepted through to a fully classified `Collection` of changes, renames, additions, and removals
- **Rename detection mechanism:** Explain how the diff kitten recognizes that a file was renamed rather than deleted-and-recreated, including the MD5-hash-based matching algorithm in `kittens/diff/collect.go`
- **Caching architecture:** Document all seven LRU caches (`mimetypes_cache`, `data_cache`, `hash_cache`, `size_cache`, `lines_cache`, `highlighted_lines_cache`, `is_text_cache`) — their purpose, capacity, thread-safety guarantees, and how they avoid redundant I/O
- **Parallel syntax highlighting:** Explain how `highlight_all()` in `kittens/diff/highlight.go` uses `images.Context.Parallel()` to run Chroma-based highlighting across multiple goroutines without data races
- **Binary and image handling:** Document the branching logic in `kittens/diff/render.go` that distinguishes text diffs, binary file metadata display, and inline image rendering via the Kitty graphics protocol
- **Diff algorithm internals:** Explain the built-in anchored diff algorithm in `kittens/diff/diff.go` (a patience-diff variant running in O(n log n) time) and how it finds matching regions through unique-line anchoring
- **Async orchestration pipeline:** Document the four-stage asynchronous pipeline (`COLLECTION → DIFF → HIGHLIGHT → IMAGE_LOAD`) coordinated through Go channels in `kittens/diff/ui.go`

### 0.1.2 Special Instructions and Constraints

- **Read-only constraint (CRITICAL):** The user explicitly states "the repository itself should remain unchanged and anything temporary should be cleaned up afterward." No existing files in the source repository may be modified.
- **Implementation rule:** A new markdown document named `kitty_815df1e210e0.md` must be created and placed in the `blitzy/documentation` directory (the source branch name is `kitty_815df1e210e0`).
- **Rationale requirement:** The documentation must provide thinking and rationale behind the answers, not just descriptions.
- **Code-as-truth principle:** All answers must be grounded in the actual codebase, not assumptions or generalizations.
- **No design system specified:** No component library or design system is relevant to this documentation task.
- **No Figma attachments:** No UI/design assets were provided.

### 0.1.3 Technical Interpretation

These documentation requirements translate to the following technical documentation strategy:

- To document the **directory comparison lifecycle**, we will create a new markdown file (`blitzy/documentation/kitty_815df1e210e0.md`) with a dedicated section that traces the call path from `main()` in `kittens/diff/main.go` → `create_collection()` → `collect_files()` in `kittens/diff/collect.go`, detailing the `walk()` function's directory traversal, set-intersection logic for common names, and data-comparison for detecting changes.
- To document the **rename detection mechanism**, we will explain the hash-and-verify approach in `collect_files()` (lines 332–368 of `kittens/diff/collect.go`): computing MD5 hashes for all added and removed files, matching by hash equality, then verifying via full data comparison before classifying as a rename.
- To document the **caching architecture**, we will catalog all seven LRU caches initialized in `init_caches()` (lines 26–37 of `kittens/diff/collect.go`), referencing the thread-safe `LRUCache` generic type in `tools/utils/cache.go` that uses `sync.RWMutex` for concurrent access.
- To document **parallel syntax highlighting**, we will trace `highlight_all()` in `kittens/diff/highlight.go` through `images.Context.Parallel()` in `tools/utils/images/utils.go`, explaining how work items are distributed via a buffered channel across `runtime.NumCPU()` goroutines with `sync.WaitGroup` coordination.
- To document **binary and image handling**, we will explain the `render()` function's dispatch logic in `kittens/diff/render.go` (lines 696–770), which branches on `is_path_text()` and `is_image()` to select between `lines_for_diff()`, `binary_lines()`, and `image_lines()`.
- To document the **diff algorithm**, we will explain the `Diff()` and `tgs()` functions in `kittens/diff/diff.go`, derived from the Go standard library, which implement an anchored unified-diff using Szymanski's longest-common-subsequence algorithm.
- To document the **async pipeline**, we will trace the `Handler.handle_async_result()` state machine in `kittens/diff/ui.go` (lines 245–276), explaining the goroutine-and-channel architecture for COLLECTION, DIFF, HIGHLIGHT, IMAGE_LOAD, and IMAGE_RESIZE result types.

### 0.1.4 Inferred Documentation Needs

Based on code analysis, the following implicit documentation needs have been identified:

- **File classification pipeline:** The chain `mimetype_for_path()` → `is_image()` / `is_path_text()` in `kittens/diff/collect.go` is central to the diff kitten's behavior but is not directly asked about; it must be documented as supporting context for the binary/image handling question.
- **External diff backend selection:** The `set_diff_command()` function in `kittens/diff/patch.go` selects between `builtin`, `git`, `diff`, and custom backends — this is relevant to understanding how the diff algorithm operates in practice and should be included.
- **Ignore-pattern filtering:** The `walk()` function respects `conf.Ignore_name` glob patterns via the `allowed()` function, which is important context for directory comparison behavior.
- **Center-of-change detection:** The `changed_center()` function in `kittens/diff/patch.go` computes the specific character-level change region within matching line pairs, which is the "matching regions" aspect of the user's question about the diff algorithm.
- **SSH remote directory support:** The `get_remote_file()` / `get_ssh_file()` path in `kittens/diff/main.go` enables diffing remote directories via SSH+tar, which is relevant infrastructure context.


## 0.2 Documentation Discovery and Analysis

### 0.2.1 Existing Documentation Infrastructure Assessment

Repository analysis reveals a **Sphinx-based documentation framework** with reStructuredText source files, covering user-facing guides and protocol specifications. The documentation infrastructure has strong coverage for end-user features but minimal coverage of internal runtime behavior.

- **Documentation framework:** Sphinx (version not pinned in `docs/requirements.txt`; uses `furo` theme)
- **Documentation generator configuration:** `docs/conf.py` (imports `kitty.constants.str_version` for version management)
- **Documentation dependencies** (from `docs/requirements.txt`):
  - `sphinx` — Core documentation generator
  - `furo` — Theme
  - `sphinx-copybutton` — Copy button for code blocks
  - `sphinxext-opengraph` — OpenGraph metadata
  - `sphinx-inline-tabs` — Tab rendering
  - `sphinx-autobuild` — Live rebuild during development
- **Source format:** reStructuredText (`.rst` files in `docs/`)
- **Diagram tools detected:** Mermaid is used in generated technical spec content; the existing docs reference screenshots but do not use inline diagram tools natively
- **API documentation tools:** None detected for Go code; Python type annotations are enforced via strict mypy but no automated Python API doc generation (no Sphinx `autodoc` extension observed)

**Existing diff kitten documentation found:**

| Documentation File | Content | Coverage Status |
|---|---|---|
| `docs/kittens/diff.rst` | End-user guide: features list, installation, usage, keyboard shortcuts, git integration, configuration reference | Complete for user-facing concerns; no internal architecture or runtime behavior documentation |
| `docs/kittens_intro.rst` | Kittens overview page listing all kittens with brief descriptions | Mentions diff kitten with one-line summary |
| `docs/screenshots/diff.png` | Screenshot of the diff kitten UI | Visual reference only |

**Key gap identified:** The existing documentation at `docs/kittens/diff.rst` is exclusively user-facing (how to invoke, keyboard shortcuts, configuration options). There is **zero documentation** of the runtime behavior, caching architecture, rename detection algorithm, parallel execution model, or diff algorithm internals. The user's questions are entirely unaddressed by existing documentation.

### 0.2.2 Repository Code Analysis for Documentation

Search patterns employed to identify all source code relevant to the diff kitten's runtime behavior:

- **Diff kitten core:** `kittens/diff/*.go` — 10 Go source files comprising the complete implementation
- **Diff kitten metadata:** `kittens/diff/*.py` — 2 Python files for CLI/config definitions
- **Shared utilities:** `tools/utils/cache.go` — LRU cache implementation used by all seven diff caches
- **Parallelism infrastructure:** `tools/utils/images/utils.go` — `Context.Parallel()` goroutine pool used by diff, highlight, and search
- **Unit tests:** `kittens/diff/collect_test.go` — Walk traversal coverage

Key directories examined:

| Directory | Relevance | Files Retrieved |
|---|---|---|
| `kittens/diff/` | Primary — all diff kitten source code | `main.go`, `collect.go`, `diff.go`, `patch.go`, `highlight.go`, `ui.go`, `render.go`, `search.go`, `mouse.go`, `main.py`, `__init__.py`, `collect_test.go` |
| `tools/utils/` | Supporting — LRU cache generic type | `cache.go` |
| `tools/utils/images/` | Supporting — parallel execution infrastructure | `utils.go` |
| `docs/kittens/` | Existing documentation | `diff.rst` |
| `docs/` | Documentation infrastructure | `conf.py`, `requirements.txt`, `kittens_intro.rst` |

Related documentation found that provides context:

- Technical specification section 4.8 (Kittens Framework Execution Flow) — describes kitten resolution and the built-in kitten catalog
- Technical specification section 3.3 (Open Source Dependencies) — documents the Chroma v2.14.0 dependency used for syntax highlighting

### 0.2.3 Web Search Research Conducted

No web search was required for this documentation task. All answers are derived directly from the source code, consistent with the user's "code as truth" directive. The key external references that would be relevant for the reader are:

- The Go standard library `internal/diff/diff.go` (the origin of the anchored diff algorithm adapted in `kittens/diff/diff.go`)
- Szymanski's 1975 paper "A Special Case of the Maximal Common Subsequence Problem" (Princeton TR #170), cited directly in the source code comments at `kittens/diff/diff.go:191`
- The Chroma v2 syntax highlighting library at `github.com/alecthomas/chroma/v2` (v2.14.0 as pinned in `go.mod`)


## 0.3 Documentation Scope Analysis

### 0.3.1 Code-to-Documentation Mapping

The following modules require documentation to answer the user's questions. Each module is mapped to the specific public APIs and internal mechanisms that must be explained.

**Module: `kittens/diff/collect.go` — File Collection, Caching, and Rename Detection**

- Public/exported types: `Collection` struct (fields: `changes`, `renames`, `type_map`, `adds`, `removes`, `all_paths`, `paths_to_highlight`, `added_count`, `removed_count`)
- Key internal functions: `create_collection()`, `collect_files()`, `walk()`, `hash_for_path()`, `data_for_path()`, `lines_for_path()`, `highlighted_lines_for_path()`, `mimetype_for_path()`, `is_path_text()`, `is_image()`, `init_caches()`, `allowed()`, `sanitize()`
- Current documentation: **Missing** — no internal documentation exists
- Documentation needed: Runtime walkthrough of directory traversal, set-intersection logic, rename detection by MD5 hash matching, caching layer architecture with all seven LRU caches

**Module: `kittens/diff/diff.go` — Built-in Anchored Diff Algorithm**

- Key functions: `Diff()`, `tgs()`, `lines()`
- Key types: `pair` struct
- Current documentation: **Missing** — only inline code comments exist (adapted from Go stdlib)
- Documentation needed: Algorithm explanation (unique-line anchoring, O(n log n) complexity), how `tgs()` implements Szymanski's longest-common-subsequence algorithm, how matching regions are expanded

**Module: `kittens/diff/patch.go` — Diff Execution and Parsing Engine**

- Key functions: `set_diff_command()`, `run_diff()`, `do_diff()`, `diff()` (parallel batch), `parse_patch()`, `parse_hunk_header()`, `changed_center()`
- Key types: `Patch`, `Hunk`, `Chunk`, `Center`, `diff_job`
- Current documentation: **Missing**
- Documentation needed: External backend selection logic, parallel batch diff execution, unified diff parsing, character-level center-of-change computation

**Module: `kittens/diff/highlight.go` — Parallel Syntax Highlighting**

- Key functions: `highlight_file()`, `highlight_all()`, `ansi_formatter()`, `clear_background()`
- Current documentation: **Missing**
- Documentation needed: Lexer selection (filename → extension → content analysis), Chroma style resolution, parallel highlighting via `images.Context.Parallel()`, ANSI escape code formatting, per-line token formatting

**Module: `kittens/diff/ui.go` — Async Pipeline and UI Controller**

- Key types: `Handler`, `AsyncResult`, `ResultType`, `ScrollPos`
- Key functions: `initialize()`, `generate_diff()`, `highlight_all()`, `load_all_images()`, `handle_async_result()`, `render_diff()`, `draw_screen()`
- Current documentation: **Missing**
- Documentation needed: Four-stage async pipeline (COLLECTION → DIFF → HIGHLIGHT → IMAGE_LOAD), goroutine-channel coordination, wakeup-driven event processing

**Module: `kittens/diff/render.go` — Display Rendering**

- Key functions: `render()`, `lines_for_diff()`, `all_lines()`, `binary_lines()`, `image_lines()`, `rename_lines()`
- Key types: `LogicalLine`, `LogicalLines`, `ScreenLine`, `HalfScreenLine`, `DiffData`
- Current documentation: **Missing**
- Documentation needed: Text/binary/image dispatch logic, side-by-side line rendering, width-aware wrapping

**Module: `kittens/diff/main.go` — Entry Point**

- Key functions: `main()`, `load_config()`, `get_remote_file()`, `get_ssh_file()`
- Current documentation: **Missing** for internal behavior
- Documentation needed: Initialization sequence, SSH remote file retrieval, cache and formatter setup

**Supporting Module: `tools/utils/cache.go` — LRU Cache**

- Key type: `LRUCache[K, V]` (generic)
- Key methods: `Get()`, `Set()`, `GetOrCreate()`, `MustGetOrCreate()`
- Current documentation: **Missing**
- Documentation needed: Thread-safety via `sync.RWMutex`, eviction policy (linked-list LRU), capacity management

**Supporting Module: `tools/utils/images/utils.go` — Parallel Execution**

- Key type: `Context` struct
- Key method: `Parallel(start, stop, fn)`
- Current documentation: **Missing**
- Documentation needed: Goroutine pool via `runtime.NumCPU()`, buffered channel distribution, `sync.WaitGroup` synchronization

### 0.3.2 Documentation Gap Analysis

Given the requirements and repository analysis, documentation gaps include:

- **Undocumented runtime behavior:** All internal functions in the diff kitten's 10 Go source files have zero external documentation. Only `docs/kittens/diff.rst` exists, covering exclusively user-facing concerns (invocation, keyboard shortcuts, git integration).
- **Missing architectural documentation:** No architecture diagrams or data-flow documentation exists for the diff kitten's async pipeline, caching layer, or parallel execution model.
- **Missing algorithm documentation:** The anchored diff algorithm in `diff.go` has inline code comments referencing Szymanski's paper, but no prose explanation of how the algorithm works or why it was chosen over standard O(n²) alternatives.
- **Missing caching documentation:** The seven LRU caches are silently initialized in `init_caches()` with no documentation of their purpose, capacity, or interaction patterns.
- **Missing parallel execution documentation:** The `images.Context.Parallel()` utility is used in three places within the diff kitten (diff batch execution, syntax highlighting, search) but has no documentation explaining its concurrency model.
- **Missing binary/image handling documentation:** The dispatch logic that routes files through text-diff, binary-metadata, or image-rendering paths is undocumented.


## 0.4 Documentation Implementation Design

### 0.4.1 Documentation Structure Planning

The deliverable is a single comprehensive markdown document placed at `blitzy/documentation/kitty_815df1e210e0.md`. The document will be organized to mirror the user's natural line of inquiry, progressing from high-level runtime flow down to algorithm internals.

```
blitzy/documentation/
└── kitty_815df1e210e0.md
    ├── Introduction and Scope
    ├── Runtime Lifecycle: From Two Directories to a Classified Collection
    │   ├── Entry Point and Initialization
    │   ├── Directory Traversal (walk)
    │   ├── Set-Intersection Classification
    │   └── Rename Detection via MD5 Hashing
    ├── The Asynchronous Pipeline
    │   ├── Stage 1: Collection (goroutine)
    │   ├── Stage 2: Diff Generation (goroutine, parallel batch)
    │   ├── Stage 3: Syntax Highlighting (goroutine, parallel)
    │   ├── Stage 4: Image Loading (goroutine)
    │   └── Wakeup-Driven Result Processing
    ├── Caching Architecture
    │   ├── The Seven LRU Caches
    │   ├── Thread-Safety and Eviction
    │   └── Cache Interaction Patterns
    ├── The Diff Algorithm: Anchored Unified Diff
    │   ├── Why Anchored (Patience) Diff
    │   ├── Unique-Line Matching via tgs()
    │   ├── Match Expansion and Chunk Emission
    │   └── External Backend Fallback (git, diff)
    ├── Parallel Syntax Highlighting
    │   ├── Lexer Selection Pipeline
    │   ├── Chroma Tokenization and ANSI Formatting
    │   └── Goroutine Pool Execution
    ├── Handling Binary Files and Images
    │   ├── File Classification (text vs binary vs image)
    │   ├── Binary Metadata Display
    │   └── Inline Image Rendering via Graphics Protocol
    ├── Character-Level Change Detection
    │   └── Center-of-Change Computation
    └── Summary
```

### 0.4.2 Content Generation Strategy

**Information Extraction Approach:**

- Extract the directory comparison lifecycle by tracing the call chain: `main()` → `create_collection()` → `collect_files()` → `walk()` in `kittens/diff/collect.go`
- Extract rename detection logic from lines 332–368 of `kittens/diff/collect.go`, documenting the MD5-hash-then-verify algorithm
- Catalog all caches from `init_caches()` (lines 26–37 of `kittens/diff/collect.go`) and cross-reference with `tools/utils/cache.go`
- Extract the async pipeline from `Handler.initialize()` (lines 114–140 of `kittens/diff/ui.go`) through `handle_async_result()` (lines 245–276)
- Extract the diff algorithm explanation from `Diff()` and `tgs()` in `kittens/diff/diff.go`, preserving the source code's own comment about Szymanski's paper
- Extract parallel execution from `highlight_all()` (lines 217–228 of `kittens/diff/highlight.go`) and `diff()` (lines 352–377 of `kittens/diff/patch.go`)
- Extract binary/image dispatch from `render()` (lines 696–770 of `kittens/diff/render.go`)

**Documentation Standards:**

- Markdown formatting with proper headers (`#`, `##`, `###`)
- Mermaid diagram integration for the async pipeline and directory comparison flow
- Code snippets using fenced code blocks with Go syntax highlighting, kept brief (2–3 lines)
- Source citations as inline references: `Source: kittens/diff/collect.go:296`
- Tables for cache catalog and function-to-purpose mappings
- Consistent terminology aligned with the codebase (e.g., "Collection" not "file list", "Patch" not "diff result")

### 0.4.3 Diagram and Visual Strategy

Mermaid diagrams to create within the documentation:

- **Directory comparison lifecycle flowchart:** Traces from `main()` through `create_collection()` → `collect_files()` → `walk()` → set operations → rename detection → `finalize()`
- **Async pipeline sequence diagram:** Shows the four-stage goroutine pipeline with channel communication and `WakeupMainThread()` signaling
- **Caching layer diagram:** Shows how the seven caches interrelate — `data_cache` feeds into `hash_cache`, `lines_cache`, and `is_text_cache`; `highlighted_lines_cache` depends on `lines_cache`
- **File classification decision tree:** Flowchart showing how `is_path_text()` and `is_image()` route files to `lines_for_diff()`, `binary_lines()`, or `image_lines()`
- **Rename detection algorithm flowchart:** Step-by-step flow of the MD5-hash-and-verify approach


## 0.5 Documentation File Transformation Mapping

### 0.5.1 File-by-File Documentation Plan

The following table maps every documentation file to be created. Per the implementation rules, no existing files will be modified and no files will be deleted.

| Target Documentation File | Transformation | Source Code/Docs | Content/Changes |
|---|---|---|---|
| `blitzy/documentation/kitty_815df1e210e0.md` | CREATE | `kittens/diff/main.go`, `kittens/diff/collect.go`, `kittens/diff/diff.go`, `kittens/diff/patch.go`, `kittens/diff/highlight.go`, `kittens/diff/ui.go`, `kittens/diff/render.go`, `kittens/diff/search.go`, `kittens/diff/mouse.go`, `kittens/diff/main.py`, `kittens/diff/__init__.py`, `kittens/diff/collect_test.go`, `tools/utils/cache.go`, `tools/utils/images/utils.go` | Comprehensive technical deep-dive document answering all user questions about the diff kitten's runtime behavior: directory comparison lifecycle, rename detection, caching architecture, parallel highlighting, binary/image handling, diff algorithm internals, and async pipeline orchestration. Includes Mermaid diagrams, code-grounded explanations with source citations, and reasoning/rationale behind each answer. |

**Reference files used for style and context (not modified):**

| Reference File | Mode | Purpose |
|---|---|---|
| `docs/kittens/diff.rst` | REFERENCE | Understand existing end-user documentation coverage and terminology |
| `docs/kittens_intro.rst` | REFERENCE | Understand kittens ecosystem context |
| `docs/conf.py` | REFERENCE | Understand documentation infrastructure and version management |
| `docs/requirements.txt` | REFERENCE | Understand documentation tooling dependencies |

### 0.5.2 New Documentation File Detail

```
File: blitzy/documentation/kitty_815df1e210e0.md
Type: Technical deep-dive / Architectural walkthrough
Source Code:
    - kittens/diff/main.go (entry point, SSH handling, initialization)
    - kittens/diff/collect.go (collection, caches, rename detection, walk)
    - kittens/diff/diff.go (anchored diff algorithm, tgs())
    - kittens/diff/patch.go (diff execution, parsing, parallel batch)
    - kittens/diff/highlight.go (Chroma highlighting, parallel execution)
    - kittens/diff/ui.go (async pipeline, Handler state machine)
    - kittens/diff/render.go (display rendering, text/binary/image dispatch)
    - kittens/diff/search.go (regex search, parallel matching)
    - kittens/diff/mouse.go (mouse handling, clipboard)
    - kittens/diff/main.py (configuration options, CLI definition)
    - kittens/diff/__init__.py (syntax_aliases helper)
    - kittens/diff/collect_test.go (walk unit tests)
    - tools/utils/cache.go (LRU cache generic implementation)
    - tools/utils/images/utils.go (Parallel goroutine pool)
Sections:
    - Introduction and Scope (purpose, audience, document conventions)
    - Runtime Lifecycle (entry point → directory traversal → set classification → rename detection)
    - Asynchronous Pipeline (COLLECTION → DIFF → HIGHLIGHT → IMAGE_LOAD stages)
    - Caching Architecture (seven LRU caches, thread-safety, eviction, interaction)
    - Diff Algorithm Internals (anchored/patience diff, tgs(), O(n log n) guarantee)
    - Parallel Syntax Highlighting (Chroma, lexer selection, goroutine pool)
    - Binary and Image Handling (MIME classification, metadata display, graphics protocol)
    - Character-Level Change Detection (center-of-change computation)
    - Summary (key takeaways)
Diagrams:
    - Directory comparison lifecycle flowchart (Mermaid)
    - Async pipeline sequence diagram (Mermaid)
    - Caching layer relationship diagram (Mermaid)
    - File classification decision tree (Mermaid)
    - Rename detection algorithm flowchart (Mermaid)
Key Citations:
    - kittens/diff/collect.go (lines 26–37 for caches, 260–294 for walk, 296–369 for collect_files)
    - kittens/diff/diff.go (lines 49–167 for Diff(), 192–264 for tgs())
    - kittens/diff/patch.go (lines 282–328 for run_diff, 352–377 for parallel diff)
    - kittens/diff/highlight.go (lines 161–215 for highlight_file, 217–228 for highlight_all)
    - kittens/diff/ui.go (lines 114–140 for initialize, 142–159 for generate_diff, 245–276 for handle_async_result)
    - kittens/diff/render.go (lines 696–770 for render dispatch)
    - tools/utils/cache.go (lines 13–72 for LRUCache)
    - tools/utils/images/utils.go (lines 27–56 for Parallel)
```

### 0.5.3 Documentation Configuration Updates

No documentation configuration files need to be created or updated. The output file is a standalone markdown document in the `blitzy/documentation` directory, independent of the Sphinx documentation infrastructure in `docs/`.

### 0.5.4 Cross-Documentation Dependencies

- The new document is **self-contained** and does not create navigation dependencies with the existing Sphinx documentation
- Internal cross-references within the document will link sections via markdown anchors
- Source citations will reference file paths relative to the repository root
- No table-of-contents, index, or glossary updates are required in the existing documentation


## 0.6 Dependency Inventory

### 0.6.1 Documentation Dependencies

The documentation deliverable is a standalone Markdown file that does not require any documentation generation tools. The following dependencies are relevant only as **subject matter** — they are the runtime dependencies of the diff kitten that the documentation will explain.

| Registry | Package Name | Version | Purpose in Diff Kitten |
|---|---|---|---|
| Go module | `github.com/alecthomas/chroma/v2` | v2.14.0 | Syntax highlighting engine — lexer selection, tokenization, and style application in `kittens/diff/highlight.go` |
| Go module | `github.com/google/go-cmp` | v0.6.0 | Deep equality comparison in test infrastructure (`kittens/diff/collect_test.go`) |
| Go module | `github.com/dlclark/regexp2` | v1.11.0 | .NET-compatible regular expressions (transitive via chroma) |
| Go module | `github.com/kovidgoyal/imaging` | v1.6.3 | Image processing (used by `images.Context` for image manipulation) |
| Go module | `golang.org/x/image` | v0.17.0 | Extended image format support for inline image rendering |
| Go stdlib | `crypto/md5` | (stdlib) | MD5 hashing for rename detection in `kittens/diff/collect.go` |
| Go stdlib | `sync` | (stdlib) | `RWMutex` for thread-safe LRU caches; `WaitGroup` for parallel execution; `OnceValue` for lazy initialization |
| Go stdlib | `runtime` | (stdlib) | `NumCPU()` for determining goroutine pool size in `images.Context.Parallel()` |
| Go stdlib | `os/exec` | (stdlib) | Running external `git diff` or `diff` commands as subprocesses |

### 0.6.2 Documentation Tooling

Since the deliverable is a plain Markdown file, no documentation tooling dependencies are required. The following existing project documentation tools are noted for context but are **not used** by this task:

| Registry | Package Name | Version | Purpose |
|---|---|---|---|
| pip | sphinx | (unpinned) | Existing Sphinx documentation generator for `docs/` |
| pip | furo | (unpinned) | Sphinx theme used by existing documentation |
| pip | sphinx-copybutton | (unpinned) | Copy button for code blocks in existing docs |
| pip | sphinxext-opengraph | (unpinned) | OpenGraph metadata for existing docs |
| pip | sphinx-inline-tabs | (unpinned) | Tab rendering in existing docs |
| pip | sphinx-autobuild | (unpinned) | Live rebuild for existing docs |

### 0.6.3 Documentation Reference Updates

No link updates are required. The new document is self-contained and does not create cross-references with existing documentation files.


## 0.7 Coverage and Quality Targets

### 0.7.1 Documentation Coverage Metrics

**Current coverage analysis (prior to this task):**

| Coverage Area | Documented | Total | Percentage |
|---|---|---|---|
| Diff kitten public functions documented | 0 | ~40 | 0% |
| Diff kitten internal runtime behavior documented | 0 | 7 key flows | 0% |
| User questions addressed by existing docs | 0 | 7 | 0% |
| Caching layer documented | 0 | 7 caches | 0% |
| Parallel execution patterns documented | 0 | 3 patterns | 0% |

**Target coverage after this task:**

| Coverage Area | Target | Status After Task |
|---|---|---|
| Directory comparison lifecycle | 100% | Fully traced from entry point through `create_collection()` to `finalize()` |
| Rename detection mechanism | 100% | MD5-hash-and-verify algorithm fully explained with rationale |
| Caching architecture | 100% | All 7 LRU caches cataloged with purpose, capacity, thread-safety, and interaction patterns |
| Parallel syntax highlighting | 100% | `highlight_all()` pipeline fully documented including goroutine pool, Chroma integration, and cache population |
| Binary and image handling | 100% | Classification pipeline and all three rendering paths (text/binary/image) documented |
| Diff algorithm internals | 100% | Anchored diff explained including `tgs()`, O(n log n) guarantee, and external backend fallback |
| Async pipeline orchestration | 100% | All four stages (COLLECTION, DIFF, HIGHLIGHT, IMAGE_LOAD) plus IMAGE_RESIZE documented |

### 0.7.2 Documentation Quality Criteria

**Completeness requirements:**

- Every user question must receive a direct, code-grounded answer with source file and line citations
- Each major concept must include at least one Mermaid diagram for visual clarity
- Each algorithm explanation must include rationale for why the approach was chosen
- All seven caches must be individually described with purpose, key type, value type, capacity, and thread-safety model

**Accuracy validation:**

- All function names, type names, and line references must match the actual source code
- All descriptions of algorithmic behavior must be verified against the source implementation
- Cache capacity values (4096) must be cited from the source (`kittens/diff/collect.go:29`)
- Thread-safety claims must be verified against the `sync.RWMutex` usage in `tools/utils/cache.go`

**Clarity standards:**

- Technical accuracy with accessible language suitable for an onboarding developer
- Progressive disclosure: start with high-level runtime flow, then drill into each subsystem
- Consistent terminology throughout, using the codebase's own naming conventions
- Rationale and thinking provided for each answer, not just mechanical descriptions

**Maintainability:**

- Source citations for every major claim enable future verification when code changes
- Section structure mirrors the codebase module structure for easy cross-referencing

### 0.7.3 Example and Diagram Requirements

| Requirement | Target |
|---|---|
| Mermaid diagrams | 5 diagrams (directory lifecycle, async pipeline, cache relationships, file classification, rename detection) |
| Code snippet references | Brief inline references to key functions and types, kept to 2–3 lines each |
| Source citations | Every major section includes `Source: path/to/file.go:line` references |
| Tables | Cache catalog table, function mapping tables, coverage summary tables |


## 0.8 Scope Boundaries

### 0.8.1 Exhaustively In Scope

**New documentation files:**

- `blitzy/documentation/kitty_815df1e210e0.md` — The sole deliverable: a comprehensive technical deep-dive answering all user questions about the diff kitten's runtime behavior

**Source code analyzed for documentation (read-only):**

- `kittens/diff/main.go` — Entry point, config loading, SSH remote handling, terminal loop setup
- `kittens/diff/collect.go` — File collection, directory walking, rename detection, caching infrastructure
- `kittens/diff/diff.go` — Built-in anchored diff algorithm (Szymanski's LCS)
- `kittens/diff/patch.go` — Diff execution backends, unified diff parsing, parallel batch diffing
- `kittens/diff/highlight.go` — Chroma syntax highlighting, parallel execution, ANSI formatting
- `kittens/diff/ui.go` — Async pipeline (Handler), goroutine-channel coordination, wakeup event processing
- `kittens/diff/render.go` — Logical/screen line rendering, text/binary/image dispatch
- `kittens/diff/search.go` — Regex search across rendered lines, parallel matching
- `kittens/diff/mouse.go` — Mouse selection, wheel scrolling, clipboard integration
- `kittens/diff/main.py` — CLI definition, configuration options, keyboard shortcuts
- `kittens/diff/__init__.py` — Syntax alias helper
- `kittens/diff/collect_test.go` — Walk traversal unit test
- `tools/utils/cache.go` — Generic LRU cache with sync.RWMutex thread-safety
- `tools/utils/images/utils.go` — Parallel goroutine pool (`Context.Parallel()`)

**Topics covered in documentation:**

- Directory comparison lifecycle (traversal, set-intersection, classification)
- Rename detection algorithm (MD5 hashing, hash matching, data verification)
- Caching architecture (all seven LRU caches, capacity, eviction, thread-safety)
- Parallel syntax highlighting (Chroma integration, goroutine pool, cache population)
- Binary and image file handling (MIME classification, metadata display, graphics protocol rendering)
- Diff algorithm internals (anchored/patience diff, tgs(), O(n log n) performance)
- Async pipeline orchestration (COLLECTION → DIFF → HIGHLIGHT → IMAGE_LOAD stages)
- Character-level change detection (center-of-change computation)

### 0.8.2 Explicitly Out of Scope

- **Source code modifications:** No existing files in the repository will be modified (per user instruction: "the repository itself should remain unchanged")
- **User-facing documentation updates:** The existing `docs/kittens/diff.rst` will not be modified
- **Documentation infrastructure changes:** No changes to `docs/conf.py`, `docs/requirements.txt`, or any Sphinx configuration
- **Test file modifications:** No changes to `kittens/diff/collect_test.go` or any other test files
- **Feature additions or code refactoring:** No code changes of any kind
- **Non-diff kittens:** Other kittens (icat, ssh, transfer, etc.) are not in scope
- **Core Kitty terminal emulator internals:** The C-based core engine, GPU rendering pipeline, and VT parser are not in scope
- **Build system or deployment configuration:** No changes to `setup.py`, `Makefile`, `go.mod`, or CI configuration
- **Mouse/keyboard interaction deep-dive:** While `mouse.go` and `search.go` are referenced for completeness, detailed UX interaction documentation is not the focus of the user's questions
- **Configuration option documentation:** The user's questions are about runtime behavior, not configuration options (which are already documented in `docs/kittens/diff.rst`)


## 0.9 Execution Parameters

### 0.9.1 Documentation-Specific Instructions

- **Documentation build command:** Not applicable — the deliverable is a standalone Markdown file that requires no build step
- **Documentation preview command:** Any Markdown viewer or `cat blitzy/documentation/kitty_815df1e210e0.md`
- **Diagram generation command:** Mermaid diagrams are embedded inline in the Markdown file and can be rendered by any Mermaid-compatible viewer (GitHub, VS Code, etc.)
- **Documentation deployment command:** Not applicable — the file is placed directly in the repository
- **Default format:** Markdown with embedded Mermaid diagrams in fenced code blocks
- **Citation requirement:** Every major section must reference source files with path and line number
- **Style guide:** Plain technical prose suitable for an onboarding developer; use the codebase's own terminology; provide rationale and thinking behind each answer
- **Documentation validation:** Visual inspection of Markdown rendering; verify all source citations match actual file paths and line numbers in the repository

### 0.9.2 Output File Specification

| Attribute | Value |
|---|---|
| File path | `blitzy/documentation/kitty_815df1e210e0.md` |
| File format | Markdown (`.md`) |
| Naming convention | `<source_branch_name>.md` = `kitty_815df1e210e0.md` |
| Target directory | `blitzy/documentation/` |
| Encoding | UTF-8 |
| Line endings | LF (Unix-style) |
| Estimated size | ~4,000–6,000 words |

### 0.9.3 Temporary Script Policy

The user explicitly allows temporary scripts for observation but requires cleanup afterward. For this documentation task:

- No temporary scripts are needed — all analysis is performed through repository inspection tools
- No files in the source repository are modified
- The `blitzy/documentation/` directory is created if it does not exist
- The only file created is the final deliverable: `blitzy/documentation/kitty_815df1e210e0.md`


## 0.10 Rules for Documentation

The following rules are derived from the user's explicit instructions and the project's implementation rules:

- **Do not modify any existing files in the source repository.** The repository must remain unchanged. Only new files in `blitzy/documentation/` may be created.
- **Create a new markdown document named `kitty_815df1e210e0.md`** (matching the source branch name) and place it in the `blitzy/documentation` directory.
- **Provide thinking and rationale behind the answers.** Do not simply describe what the code does — explain *why* it works this way and what the design implications are.
- **Do not make assumptions; base all answers on the code as the truth.** Every claim must be traceable to a specific source file and line range. Do not generalize beyond what the code actually implements.
- **Temporary scripts may be used for observation but must be cleaned up afterward.** Any temporary artifacts created during analysis must be removed before the task is complete.
- **All source citations must include file path and line numbers** to enable future verification when the codebase evolves.
- **Use Mermaid diagrams for complex relationships and flows** to make the document visually accessible to an onboarding developer.
- **Maintain consistent terminology** aligned with the codebase's own naming (e.g., `Collection`, `Patch`, `Hunk`, `Chunk`, `Center`, `LogicalLine`, `ScreenLine`).
- **Progressive disclosure structure:** Start with high-level runtime flow, then drill into each subsystem, so the reader builds understanding incrementally.


## 0.11 References

### 0.11.1 Repository Files and Folders Searched

The following files and folders were retrieved and analyzed to derive the conclusions in this Agent Action Plan:

**Diff kitten source files (primary analysis):**

| File Path | Purpose in Analysis |
|---|---|
| `kittens/diff/main.go` | Entry point, config loading, SSH remote file retrieval, terminal loop setup, cache and formatter initialization |
| `kittens/diff/collect.go` | Core collection logic: directory walking, set-intersection classification, rename detection via MD5, all seven LRU cache definitions, file data/MIME/text classification |
| `kittens/diff/diff.go` | Built-in anchored diff algorithm: `Diff()`, `tgs()` (Szymanski's LCS), `lines()` function; adapted from Go standard library |
| `kittens/diff/patch.go` | Diff execution engine: external backend selection (`git`/`diff`/`builtin`/custom), `run_diff()`, parallel batch `diff()`, unified diff parsing (`parse_patch()`), character-level `changed_center()` |
| `kittens/diff/highlight.go` | Chroma-based syntax highlighting: lexer selection, style resolution, ANSI formatter, parallel `highlight_all()` |
| `kittens/diff/ui.go` | UI controller and async pipeline: `Handler` struct, `AsyncResult` channel-based coordination, `initialize()`, `generate_diff()`, `highlight_all()`, `load_all_images()`, `handle_async_result()` state machine, `render_diff()`, `draw_screen()` |
| `kittens/diff/render.go` | Display rendering: `render()` dispatch (text/binary/image), `LogicalLine`/`ScreenLine` types, `lines_for_diff()`, `binary_lines()`, `image_lines()`, `rename_lines()`, width-aware wrapping |
| `kittens/diff/search.go` | Regex search: `Search` struct, parallel `search()`, match-to-screen-coordinate mapping |
| `kittens/diff/mouse.go` | Mouse handling: wheel scrolling, click-drag selection, clipboard/primary-selection copy, kitty.conf option reading |
| `kittens/diff/main.py` | CLI definition, configuration options (syntax_aliases, num_context_lines, diff_cmd, replace_tab_by, ignore_name), keyboard shortcuts, color settings |
| `kittens/diff/__init__.py` | `syntax_aliases()` helper function |
| `kittens/diff/collect_test.go` | Unit test for `walk()` traversal with ignore patterns |

**Supporting utility files:**

| File Path | Purpose in Analysis |
|---|---|
| `tools/utils/cache.go` | Generic `LRUCache[K, V]` implementation: `sync.RWMutex` thread-safety, `container/list` LRU eviction, `Get()`, `Set()`, `GetOrCreate()`, `MustGetOrCreate()` methods |
| `tools/utils/images/utils.go` | `Context.Parallel()` goroutine pool: `runtime.NumCPU()` thread count, buffered channel work distribution, `sync.WaitGroup` synchronization |

**Documentation and configuration files:**

| File Path | Purpose in Analysis |
|---|---|
| `docs/kittens/diff.rst` | Existing user-facing documentation: features, usage, keyboard shortcuts, git integration, configuration reference |
| `docs/kittens_intro.rst` | Kittens ecosystem overview and diff kitten listing |
| `docs/conf.py` | Sphinx documentation infrastructure configuration |
| `docs/requirements.txt` | Documentation tooling dependencies (sphinx, furo, etc.) |
| `go.mod` | Go module dependencies including chroma v2.14.0, go-cmp v0.6.0 |
| `pyproject.toml` | Python version requirement (≥3.8), mypy/ruff configuration |
| `setup.py` | Build system, version extraction |

**Folders explored:**

| Folder Path | Purpose in Analysis |
|---|---|
| `` (root) | Repository structure overview, identify major source subtrees |
| `kittens/` | Kittens ecosystem structure, identify diff kitten location |
| `kittens/diff/` | Complete diff kitten source tree (12 files) |
| `tools/utils/` | Supporting utility modules |
| `tools/utils/images/` | Parallel execution infrastructure |
| `docs/` | Documentation infrastructure |
| `docs/kittens/` | Existing kitten documentation |

**Technical specification sections retrieved:**

| Section | Purpose in Analysis |
|---|---|
| 4.8 Kittens Framework Execution Flow | Kitten resolution, module loading, and built-in kitten catalog context |
| 3.1 Programming Languages | Language architecture context (Go 1.22 for kittens, Python ≥3.8 for config) |
| 3.3 Open Source Dependencies | Chroma v2.14.0, go-cmp v0.6.0, and other Go module dependency details |

### 0.11.2 Attachments

No attachments were provided for this project.

### 0.11.3 Figma Screens

No Figma screens were provided for this project.

### 0.11.4 External References Cited in Source Code

The following external references are cited within the diff kitten source code and will be included in the documentation for reader context:

- **Szymanski's Paper (1975):** "A Special Case of the Maximal Common Subsequence Problem," Princeton Technical Report #170 — cited in `kittens/diff/diff.go:191` as the theoretical basis for the `tgs()` algorithm. Available at `https://research.swtch.com/tgs170.pdf`
- **Go Standard Library Diff:** The `Diff()` function in `kittens/diff/diff.go` is adapted from `golang.org/src/internal/diff/diff.go` as noted in the file header (line 1–2)
- **Chroma Syntax Highlighter:** `github.com/alecthomas/chroma/v2` v2.14.0 — the syntax highlighting engine used in `kittens/diff/highlight.go`


