# Blitzy Project Guide

---

## 1. Executive Summary

### 1.1 Project Overview

This project created a comprehensive technical investigation document analyzing the Kitty terminal emulator's search query parser (`kitty/search_query_parser.py`). The document answers user questions about unexpected results when combining search terms with `or` and spaces, identifies the root cause (implicit AND semantics for space-separated terms and standard NOT > AND > OR operator precedence), demonstrates the behavior with 18 real parser test queries, and provides a complete correct syntax reference. The deliverable is a single Markdown file placed in `blitzy/documentation/kitty_815df1e210e0.md`. No existing repository source files were modified — the codebase was left completely unchanged per project constraints.

### 1.2 Completion Status

```mermaid
pie title Project Completion — 91.4% Complete
    "Completed (AI)" : 16
    "Remaining" : 1.5
```

| Metric | Value |
|--------|-------|
| **Total Project Hours** | **17.5** |
| Completed Hours (AI) | 16 |
| Remaining Hours | 1.5 |
| **Completion Percentage** | **91.4%** |

**Calculation:** 16 completed hours / 17.5 total hours = 91.4% complete.

### 1.3 Key Accomplishments

- [x] Created `blitzy/documentation/kitty_815df1e210e0.md` (577 lines, ~29 KB) — complete Q&A technical investigation document
- [x] Investigated and documented the full parser architecture: tokenization, recursive-descent parsing, AST node evaluation
- [x] Identified root cause: implicit AND in `and_expression()` (lines 215–224) and operator precedence (NOT > AND > OR)
- [x] Executed 18 query test patterns against the real `kitty.search_query_parser.search` function — all results verified
- [x] Produced 8 parse tree visualizations showing AST structure for key query patterns
- [x] Provided mathematical proof of OrNode optimization correctness (set identity: A ∪ (B \ A) = A ∪ B)
- [x] Documented correct syntax reference with common pitfalls table
- [x] Confirmed conclusion: behavior is by design, not a bug — consistent with standard boolean search grammar conventions
- [x] Left codebase completely unchanged — zero existing files modified (verified via `git diff`)

### 1.4 Critical Unresolved Issues

| Issue | Impact | Owner | ETA |
|-------|--------|-------|-----|
| Human review of document technical accuracy needed | Low — document content verified by running actual parser, but domain expert review is standard practice | Human Developer | 1 hour |

### 1.5 Access Issues

No access issues identified. The project required only read access to the existing repository source files and write access to the new `blitzy/documentation/` directory. All dependencies are Python standard library modules — no external credentials, API keys, or service access was required.

### 1.6 Recommended Next Steps

1. **[High]** Review `blitzy/documentation/kitty_815df1e210e0.md` for technical accuracy — verify cited line numbers match current repository state and test results align with parser behavior
2. **[Medium]** Consider adding a brief operator precedence note to the existing user-facing docs at `docs/remote-control.rst` (lines 327–345) to prevent recurring user confusion about implicit AND semantics
3. **[Low]** Evaluate whether the common pitfalls table in the document should be adapted into the Kitty FAQ or documentation site

---

## 2. Project Hours Breakdown

### 2.1 Completed Work Detail

| Component | Hours | Description |
|-----------|-------|-------------|
| Parser Code Investigation & Analysis | 4 | Read and traced `kitty/search_query_parser.py` (296 lines), `kitty/boss.py`, `kitty/rc/base.py`, `kitty/window.py`, `kitty/tabs.py`, `kitty/types.py`, `docs/remote-control.rst` — identified tokenizer, recursive-descent grammar, AST node classes, operator precedence, and implicit AND behavior |
| Test Harness Creation & Execution | 2 | Created temporary Python test script, imported real parser, executed 18 query patterns against controlled dataset, captured and verified output, cleaned up temporary files |
| Document Writing — Architecture Overview | 2 | Wrote Module Location and Public API, Tokenization Stage, Recursive-Descent Parsing Stage, AST Node Evaluation Stage sections with code excerpts and explanations |
| Document Writing — Root Cause Analysis | 2 | Wrote detailed analysis of `and_expression()` method, operator precedence rules table, step-by-step trace of user's scenario query (`id:1 or id:2 id:3`) |
| Document Writing — Test Results | 1.5 | Formatted 18 test results into structured table with interpretations, documented error cases, wrote explanations for surprising rows |
| Parse Tree Visualizations | 1.5 | Created 8 text-based AST renderings and Mermaid flowchart of the recursive-descent parsing flow |
| OrNode Optimization Proof | 1 | Wrote mathematical proof that `A ∪ (B \ A) = A ∪ B`, documented correctness condition for pure-filter callbacks, cited evidence from `kitty/boss.py` |
| Correct Syntax Reference & Common Pitfalls | 1 | Wrote reference for OR, AND, NOT, grouped expressions, location prefix requirement, and common pitfalls comparison table |
| Review & Factual Accuracy Corrections | 0.5 | Second commit (`24014a5cd`) fixing 3 minor factual inaccuracies in documentation |
| Quality Validation & Final Checks | 0.5 | Verified all test results match parser output, confirmed 0 TODO/FIXME markers, validated markdown structure (45 headings, 22 balanced code blocks, 86 table rows) |
| **Total Completed** | **16** | |

### 2.2 Remaining Work Detail

| Category | Hours | Priority |
|----------|-------|----------|
| Human review of document technical accuracy | 1 | Medium |
| Potential content refinements from review | 0.5 | Low |
| **Total Remaining** | **1.5** | |

**Validation:** Section 2.1 (16h) + Section 2.2 (1.5h) = 17.5h = Total Project Hours in Section 1.2 ✓

---

## 3. Test Results

All test results originate from Blitzy's autonomous validation — tests were executed by running the actual `kitty.search_query_parser.search` function against a controlled dataset (`universal_set = {1, 2, 3, 4, 5}`, `locations = 'id'`, exact-string-match callback).

| Test Category | Framework | Total Tests | Passed | Failed | Coverage % | Notes |
|---------------|-----------|-------------|--------|--------|------------|-------|
| Parser Query Patterns | Python (kitty.search_query_parser.search) | 16 | 16 | 0 | 100% | Covers single terms, explicit OR, implicit AND, mixed OR+AND, NOT, grouping, chained OR |
| Parser Error Cases | Python (kitty.search_query_parser.search) | 2 | 2 | 0 | 100% | Bare term without location prefix, quoted location string — both correctly raise ParseException |
| Existing Unit Test Assertions | Python (kitty_tests.search_query_parser.TestSQP) | 9 | 9 | 0 | 100% | All 9 assertions from existing test suite verified: id:1, quoted id, AND same, OR, AND diff, NOT, grouped, and 2 error cases |
| Markdown Quality Checks | Static Analysis | 5 | 5 | 0 | 100% | Balanced code blocks (22), no TODO/FIXME markers (0), valid heading hierarchy (45), table row count (86), Mermaid diagram (1) |
| Codebase Integrity | git diff | 1 | 1 | 0 | 100% | Only 1 file changed from base: `blitzy/documentation/kitty_815df1e210e0.md`. Zero out-of-scope modifications |
| **Totals** | | **33** | **33** | **0** | **100%** | |

---

## 4. Runtime Validation & UI Verification

### Runtime Health

- ✅ **Parser module import:** `kitty.search_query_parser` imports successfully under Python 3.12.3
- ✅ **Parser execution:** `search()` function executes correctly for all 18 tested query patterns
- ✅ **Test output accuracy:** All documented results in `kitty_815df1e210e0.md` match actual parser output (verified by re-running all 18 patterns)
- ✅ **Codebase integrity:** Working tree clean, 0 uncommitted changes, 0 existing files modified

### UI Verification

- ⚠️ **Not applicable** — This is a documentation-only project. No UI components were created or modified. The Kitty terminal emulator UI was not part of the AAP scope.

### API / Integration Verification

- ✅ **Parser public API:** `build_tree()` and `search()` functions confirmed functional
- ✅ **Callback pattern:** `get_matches(location, query, candidates)` callback interface documented and tested
- ✅ **Error handling:** `ParseException` and `NoLocation` exceptions correctly raised for invalid queries

---

## 5. Compliance & Quality Review

| AAP Requirement | Status | Evidence | Notes |
|----------------|--------|----------|-------|
| Create `blitzy/documentation/kitty_815df1e210e0.md` | ✅ Pass | File exists (577 lines, 28,947 bytes) | Named per SWE-AtlasQnA-Repo rule |
| Q&A format answering user's questions | ✅ Pass | 4 questions listed at top, each answered in dedicated sections | Includes thinking/rationale per rule |
| Investigate parser implementation | ✅ Pass | Architecture overview section (lines 46–178) covers all key methods | Cites 8 source files with line numbers |
| Explain root cause (implicit AND, precedence) | ✅ Pass | Root Cause Analysis section (lines 182–263) with step-by-step trace | Quotes `and_expression()` code verbatim |
| Run test queries (≥15 patterns) | ✅ Pass | 18 test patterns executed against real parser | All results captured from actual execution |
| Parse tree visualizations (≥8 trees) | ✅ Pass | 8 text-based AST renderings (lines 319–385) | Covers simple, OR, AND, mixed, NOT, grouped |
| OrNode optimization proof | ✅ Pass | Mathematical proof (lines 409–434) | Set identity A ∪ (B \ A) = A ∪ B proven |
| Correct syntax reference | ✅ Pass | Complete reference (lines 447–510) | Multi-term OR, AND, NOT, grouped, locations |
| Common pitfalls table | ✅ Pass | 5-row comparison table (lines 502–508) | Maps user intent to parser behavior |
| Source citations with file paths and line numbers | ✅ Pass | Source References table (lines 562–577) + inline citations | All citations verified against current repo |
| Mermaid diagram | ✅ Pass | Recursive descent flow diagram (lines 129–140) | Embedded in markdown |
| Error case documentation | ✅ Pass | Error cases table (lines 310–315) with explanation | ParseException triggers documented |
| Codebase left unchanged | ✅ Pass | `git diff --stat origin/kitty_815df1e210e0...HEAD -- ':!blitzy/'` returns empty | Zero existing files modified |
| No TODO/FIXME/placeholder markers | ✅ Pass | `grep` for TODO/FIXME returns 0 matches | Document is complete |
| Evidence-based answers (no assumptions) | ✅ Pass | Every claim cites source file and line number | Per SWE-AtlasQnA-Repo rule |
| Include thinking/rationale | ✅ Pass | "Thinking / Rationale" blocks included throughout | Explains WHY, not just WHAT |

### Fixes Applied During Autonomous Validation

| Fix | Commit | Description |
|-----|--------|-------------|
| Factual accuracy corrections | `24014a5cd` | Fixed 3 minor factual inaccuracies in the documentation (line number references, method descriptions) |

---

## 6. Risk Assessment

| Risk | Category | Severity | Probability | Mitigation | Status |
|------|----------|----------|-------------|------------|--------|
| Line number citations may drift if upstream parser code changes | Technical | Low | Medium | Document references specific commit; reviewers should re-verify line numbers against current HEAD | Open — inherent to code-referencing documentation |
| Human reviewer may disagree with "not a bug" conclusion | Operational | Low | Low | Conclusion is supported by 4 independent evidence sources (design comment, standard grammar, unit tests, existing docs) | Mitigated |
| Mermaid diagram may not render in all Markdown viewers | Technical | Low | Medium | Diagram uses standard Mermaid syntax; fallback is the text-based grammar notation also included | Mitigated |
| Test results were captured against Python 3.12.3 — may differ on older Python | Technical | Very Low | Very Low | Parser uses only stdlib features available since Python 3.8 (project's minimum); behavior is deterministic | Mitigated |

---

## 7. Visual Project Status

```mermaid
pie title Project Hours Breakdown
    "Completed Work" : 16
    "Remaining Work" : 1.5
```

**Integrity check:** "Remaining Work" (1.5h) matches Section 1.2 Remaining Hours (1.5h) and Section 2.2 total (1.5h) ✓

---

## 8. Summary & Recommendations

### Achievements

The project successfully delivered a comprehensive 577-line technical investigation document that fully addresses all user questions about unexpected search query parser behavior in the Kitty terminal emulator. The document identifies the root cause (implicit AND semantics in the `and_expression()` method combined with standard NOT > AND > OR operator precedence), demonstrates the behavior with 18 verified test queries against the real parser, provides 8 parse tree visualizations, and includes a complete correct syntax reference. The codebase was left completely unchanged — zero existing files were modified, fulfilling the project's critical read-only constraint.

### Remaining Gaps

The project is 91.4% complete (16 hours completed out of 17.5 total hours). The remaining 1.5 hours consist of human review of the document's technical accuracy (1 hour) and potential content refinements arising from that review (0.5 hours). No code changes, deployment steps, or infrastructure setup are required — the deliverable is a standalone Markdown document ready for use.

### Critical Path to Production

This is a documentation-only project with no deployment pipeline. The document is immediately usable upon merge. The only production-readiness gate is human review of technical accuracy.

### Production Readiness Assessment

The document is production-ready. All content is evidence-based, all test results are captured from actual parser execution, all source citations have been verified against the current repository state, and the document contains zero TODO/FIXME/placeholder markers. The deliverable can be merged as-is.

---

## 9. Development Guide

### System Prerequisites

| Requirement | Version | Purpose |
|-------------|---------|---------|
| Python | ≥ 3.8 (3.12.3 recommended) | Required to run parser tests and verify document claims |
| Git | Any recent version | Repository management and diff verification |
| Markdown viewer | Any (VS Code, GitHub, etc.) | View the deliverable document with rendered tables and Mermaid diagrams |

### Environment Setup

```bash
# Clone the repository and switch to the feature branch
git clone <repository-url>
cd kitty
git checkout blitzy-512701a3-32c3-41b5-9a09-7f1d73aa4080
```

No virtual environment, package installation, or environment variables are needed. The parser uses only Python standard library modules (`re`, `enum`, `functools`).

### Viewing the Deliverable

```bash
# View the document
cat blitzy/documentation/kitty_815df1e210e0.md

# Or open in any Markdown viewer for rendered tables and Mermaid diagrams
# VS Code: code blitzy/documentation/kitty_815df1e210e0.md
```

### Verifying Parser Behavior

To independently verify any test result documented in the file:

```bash
cd /path/to/kitty/repo

python3 -c "
import sys
sys.path.insert(0, '.')
from kitty.search_query_parser import search

universal_set = {1, 2, 3, 4, 5}
def get_matches(location, query, candidates):
    return {x for x in candidates if query == str(x)}

# Example: Verify the user's scenario (Row 9 in the document)
result = search('id:1 or id:2 id:3', 'id', universal_set, get_matches)
print(f'Result: {sorted(result)}')  # Expected: [1]

# Example: Verify correct multi-term OR syntax (Row 4)
result = search('id:1 or id:2 or id:3', 'id', universal_set, get_matches)
print(f'Result: {sorted(result)}')  # Expected: [1, 2, 3]
"
```

**Expected output:**
```
Result: [1]
Result: [1, 2, 3]
```

### Verifying Codebase Integrity

```bash
# Confirm only the documentation file was added
git diff --name-status origin/kitty_815df1e210e0...HEAD
# Expected output:
# A    blitzy/documentation/kitty_815df1e210e0.md

# Confirm no existing files were modified
git diff --stat origin/kitty_815df1e210e0...HEAD -- ':!blitzy/'
# Expected: no output (empty diff)

# Confirm working tree is clean
git status --porcelain
# Expected: no output
```

### Troubleshooting

| Issue | Cause | Resolution |
|-------|-------|------------|
| `ModuleNotFoundError: No module named 'kitty.fast_data_types'` | Running the full test suite requires compiled C extensions | Use the direct import method shown above (`sys.path.insert(0, '.'); from kitty.search_query_parser import search`) — the parser module has no native dependencies |
| Mermaid diagram not rendering | Markdown viewer does not support Mermaid | Use a Mermaid-capable viewer (GitHub, VS Code with Mermaid extension, or https://mermaid.live) |
| `ParseException: No location specified` | Query term lacks `location:` prefix | All terms must use `location:query` format (e.g., `id:1` not `1`) |

---

## 10. Appendices

### A. Command Reference

| Command | Purpose |
|---------|---------|
| `cat blitzy/documentation/kitty_815df1e210e0.md` | View the deliverable document |
| `git diff --name-status origin/kitty_815df1e210e0...HEAD` | Verify only the doc file was changed |
| `git diff --stat origin/kitty_815df1e210e0...HEAD -- ':!blitzy/'` | Confirm zero existing file modifications |
| `python3 -c "import sys; sys.path.insert(0,'.'); from kitty.search_query_parser import search; print('OK')"` | Verify parser module is importable |
| `git log --oneline HEAD --not origin/kitty_815df1e210e0` | List commits on the feature branch |

### B. Port Reference

Not applicable — this is a documentation-only project with no services or ports.

### C. Key File Locations

| File | Purpose |
|------|---------|
| `blitzy/documentation/kitty_815df1e210e0.md` | **Deliverable** — Complete Q&A investigation document (577 lines) |
| `kitty/search_query_parser.py` | Parser implementation — primary investigation target (296 lines) |
| `kitty_tests/search_query_parser.py` | Existing unit tests — behavioral specification (31 lines) |
| `kitty/boss.py` (lines 471–531) | Parser consumer — `match_windows()` and `match_tabs()` methods |
| `kitty/rc/base.py` (lines 87–165) | Canonical match option definitions (`MATCH_WINDOW_OPTION`, `MATCH_TAB_OPTION`) |
| `kitty/window.py` (line 784) | `Window.matches_query()` — per-field matching |
| `kitty/tabs.py` (line 800) | `Tab.matches_query()` — per-field matching |
| `docs/remote-control.rst` (lines 327–345) | Existing user-facing matching documentation |

### D. Technology Versions

| Technology | Version | Role |
|------------|---------|------|
| Python | ≥ 3.8 (3.12.3 on build system) | Parser runtime and test execution |
| Git | 2.x | Version control and diff verification |
| Markdown | CommonMark / GFM | Document format |
| Mermaid | Standard syntax | Embedded diagram in document |

### E. Environment Variable Reference

Not applicable — no environment variables are required for this documentation-only project.

### G. Glossary

| Term | Definition |
|------|------------|
| **AST** | Abstract Syntax Tree — the tree data structure produced by the parser representing the logical structure of a query |
| **Implicit AND** | The parser's behavior of treating space-separated terms (without an explicit operator) as AND (intersection) — the root cause of the user's confusion |
| **Operator precedence** | The order in which operators bind: NOT (highest) > AND (middle) > OR (lowest) |
| **Recursive descent** | A parsing technique where each grammar rule is implemented as a separate function that calls other functions for sub-rules |
| **OrNode / AndNode / NotNode / TokenNode** | The four AST node types in the parser, representing OR, AND, NOT operations and leaf terms respectively |
| **ParseException** | Exception raised when a query string cannot be parsed (e.g., missing location prefix, unmatched parentheses) |
| **Pure filter callback** | A `get_matches` function that only selects from (never adds to) its candidate set — required for OrNode optimization correctness |