# kitty search-query parser: is `or` + spaces broken? — a grounded investigation

> **Scope:** This document investigates kitty's remote-control `--match` search-query
> parser at the checked-out commit `815df1e21` (branch `kitty_815df1e210e0`,
> HEAD *"Wire up applying of font config"*). Every behavioral claim below is backed by
> either an exact `file:line` citation into the repository or by verbatim output captured
> from running the real parser. Where the public documentation diverges from this commit,
> **the code is authoritative** (see §7).

## The user's report (verbatim)

> "When I combine multiple search terms with 'or' and spaces, the results don't match what I expect. Items that should clearly match at least one term are being excluded entirely."

---

## Section 1 — Direct answer / summary verdict

**This is not a bug. It is a usage/syntax issue.** The `or` operator works exactly as
intended: it computes the set **UNION** of the items matched by each side, so it *cannot*
"exclude items that match at least one term." I verified this by running the real parser
(see §3): `title:foo or title:bar` returns **all three** items in the test universe:

```
'title:foo or title:bar'   -> MATCHED [1, 2, 3]
```

The symptom you observed — *"items that should clearly match at least one term are being
excluded entirely"* — is produced by one (or both) of two distinct pitfalls:

1. **Space-separated adjacent terms are implicitly ANDed (intersection), not ORed.**
   If you write two terms separated only by a space (no `or` between them), the parser
   inserts an *implicit* `and`, and the two terms are intersected. Any item matching only
   *one* term is therefore dropped. This is the single most likely cause of your symptom.
   Evidence: `title:foo title:bar -> MATCHED [3]` (only the item matching *both* survives).
   Grammar: `and_expression` inserts an implicit `AndNode` at
   `kitty/search_query_parser.py:L222-L223`.

2. **Every term requires a `location:` prefix** (e.g. `title:`, `id:`). A bare, unprefixed
   word raises a parse error, because both production callers leave `allow_no_location` at
   its default of `False`. Evidence: `foo or bar -> NoLocation: No location specified before foo`.
   Grammar: `base_token` raises `NoLocation` at `kitty/search_query_parser.py:L278` (and
   `L250` for a quoted word); the callers omit the flag at
   `kitty/boss.py:L493-L495` and `kitty/boss.py:L529-L531`.

**The fix (the correct syntax):** prefix *both* operands with a `location:` and join them with
the `or` keyword — for example `title:foo or title:bar`. On a real kitty instance the whole
match expression is a single shell-quoted argument:

```sh
kitten @ ls --match 'title:foo or title:bar'
```

The remaining sections explain *why* (grounded in the grammar), *demonstrate* the behavior
with the exact probe and its verbatim output, dissect the two pitfalls, prescribe every valid
form, and corroborate against kitty's own tests and documentation.

---

## Section 2 — How the parser works (grammar, precedence, semantics)

The parser is a small recursive-descent boolean-expression evaluator. It is **pure Python**:
`kitty/__init__.py` is empty and `kitty/types.py` imports only the standard library, so
`import kitty.search_query_parser` succeeds **without** building kitty's C extension or Go
binaries. It imports only `from .types import run_once` plus stdlib (`re`, `enum`,
`functools.lru_cache`) — `kitty/search_query_parser.py:L1-L9`.

### 2.1 Entry point and caching

The public entry point is:

```python
def search(
    query: str, locations: Union[str, Tuple[str, ...]], universal_set: Set[T], get_matches: GetMatches[T],
    allow_no_location: bool = False,
) -> Set[T]:
    return build_tree(query, locations, allow_no_location).search(universal_set, get_matches)
```

— `kitty/search_query_parser.py:L292-L296`. Note that **`allow_no_location` defaults to
`False`**. `build_tree` parses the query string into a tree of nodes and is memoized with
`@lru_cache(maxsize=64)` — `kitty/search_query_parser.py:L281-L289`. Each item in
`universal_set` is a candidate; `get_matches(location, query, candidates)` is the caller-supplied
callback that decides which candidates a single leaf term matches.

### 2.2 Grammar precedence (lowest → highest binding)

The recursive-descent methods form this chain:

```
or_expression  →  and_expression  →  not_expression  →  location_expression  →  base_token
```

— `kitty/search_query_parser.py:L199-L278`. The method called *first* (`or_expression`) sits at
the **top** of the parse tree, so `or` binds **loosest** (it is applied last / outermost);
`base_token` (a single `location:query` leaf) binds tightest. Parentheses in
`location_expression` (`kitty/search_query_parser.py:L232-L242`) let you override precedence.

### 2.3 The three boolean operators are set operations

Each operator node is a callable that transforms a candidate set:

- **`or` = set UNION.** `or_expression` recognizes the lowercased token `or` and builds an
  `OrNode` — `kitty/search_query_parser.py:L208-L213`. `OrNode.__call__` is:

  ```python
  def __call__(self, candidates, get_matches):
      lhs = self.lhs(candidates, get_matches)
      return lhs.union(self.rhs(candidates.difference(lhs), get_matches))
  ```

  — `kitty/search_query_parser.py:L63-L65`. It returns the **union** of the left and right
  results. **This is precisely why `or` cannot exclude an item that matches at least one term.**

- **`and` = set INTERSECTION (narrowing).** `AndNode.__call__` evaluates the left side, then
  applies the right side *to that already-narrowed set*:

  ```python
  def __call__(self, candidates, get_matches):
      lhs = self.lhs(candidates, get_matches)
      return self.rhs(lhs, get_matches)
  ```

  — `kitty/search_query_parser.py:L79-L81`. Only items matched by **both** sides survive.

- **`not` = set DIFFERENCE.** `NotNode.__call__` returns
  `candidates.difference(self.rhs(candidates, get_matches))` —
  `kitty/search_query_parser.py:L94-L95`.

- **Leaf evaluation.** `TokenNode.__call__` returns
  `get_matches(self.location, self.query, candidates)` — `kitty/search_query_parser.py:L108-L109`.

### 2.4 Root cause #1 — the implicit-AND rule

Inside `and_expression`, after parsing the left operand, the parser first honours an explicit
`and`. But even when there is **no** `and`, it *still* inserts an implicit `AndNode` whenever the
next token is a word/quoted-word or an opening parenthesis **and** is not the keyword `or`:

```python
def and_expression(self):
    lhs = self.not_expression()
    if self.lcase_token() == 'and':
        self.advance()
        return AndNode(lhs, self.and_expression())

    # Account for the optional 'and'
    if ((self.token_type() in (TokenType.WORD, TokenType.QUOTED_WORD) or self.token() == '(') and self.lcase_token() != 'or'):
        return AndNode(lhs, self.and_expression())
    return lhs
```

— `kitty/search_query_parser.py:L215-L224` (the implicit-`AndNode` at **L222-L223**). Consequently,
`title:foo title:bar` (two terms separated only by whitespace) is parsed as
`title:foo AND title:bar` and **intersected**, not unioned.

### 2.5 Root cause #2 — the mandatory `location:` prefix

`base_token` requires a recognized `location:` prefix — `kitty/search_query_parser.py:L244-L278`:

- A **quoted** word with no location raises `NoLocation` at `kitty/search_query_parser.py:L250`
  (unless `allow_no_location` is `True`, in which case it becomes location `all`).
- A **bare** word whose first colon-separated segment is not a known location raises `NoLocation`
  at `kitty/search_query_parser.py:L278` (again, only when `allow_no_location` is `False`).

The `NoLocation` class is a **subclass of `ParseException`** and emits one of two messages
depending on whether the offending token contains a colon:

```python
class NoLocation(ParseException):

    def __init__(self, tt: str):
        a, sep, b = tt.partition(':')
        if sep == ':':
            super().__init__(f'{a} is not a recognized location in {tt}')
        else:
            super().__init__(f'No location specified before {tt}')
```

— `kitty/search_query_parser.py:L136-L143`: the *no-colon* message `No location specified before {tt}`
is at **L143**; the *colon-present* message `{a} is not a recognized location in {tt}` is at **L141**.
Because `NoLocation` derives from `ParseException` (`class NoLocation(ParseException):`,
`kitty/search_query_parser.py:L136`), it is caught anywhere a `ParseException` is caught — I
confirmed `issubclass(NoLocation, ParseException) == True` at runtime (see §3.3).

### 2.6 The prefix is mandatory in real usage

Both production call sites invoke `search(...)` **without** passing `allow_no_location`, so it
takes its default value of `False`:

- `Boss.match_windows` — `kitty/boss.py:L471-L496`; the `search()` call is at
  `kitty/boss.py:L493-L495`.
- `Boss.match_tabs` — `kitty/boss.py:L505-L539`; the `search()` call is at
  `kitty/boss.py:L529-L531`.

That is why, in real `kitten @ ... --match` usage, every term **must** carry a `location:` prefix.
(Both callers also short-circuit the literal value `all` *before* parsing —
`kitty/boss.py:L472-L474` for windows and `kitty/boss.py:L506-L508` for tabs.)

---

## Section 3 — Demonstration (the exact probe + verbatim output)

**Methodology (run first, then write).** Following kitty's own read-only investigation
discipline, I did **not** modify the repository. I wrote an ephemeral probe *outside* the repo
(under `/tmp/sqp_probe_dir/`), ran the **real** `kitty.search_query_parser.search`, and captured
its output verbatim. The probe mirrors `Boss.match_windows` exactly: it passes the window
`locations` tuple copied verbatim from `kitty/boss.py:L493-L494`, leaves `allow_no_location` at
its default (`False`), and uses a `get_matches` callback that performs an **unanchored**
`re.search` on each item's title — mirroring how a window's title is matched via
`compile_match_query` → `pat.search(...)` (`kitty/window.py:L213-L222`, `kitty/window.py:L774`).

The test universe is three items keyed by id, with these titles:

- `1 = 'foo terminal'`
- `2 = 'bar editor'`
- `3 = 'foobar shell'`

### 3.1 The probe (`/tmp/sqp_probe_dir/probe.py`)

```python
#!/usr/bin/env python3
# Ephemeral probe OUTSIDE the kitty repo. Mirrors Boss.match_windows.
import re, sys
REPO = "/tmp/blitzy/kitty/blitzy-74323c75-5d44-473e-bbde-c40e81287895_84020f"
sys.path.insert(0, REPO)
from kitty.search_query_parser import search, ParseException

TITLES = {1: "foo terminal", 2: "bar editor", 3: "foobar shell"}
UNIVERSE = set(TITLES)
# Window location fields, copied verbatim from boss.py match_windows (L493-494)
LOCATIONS = ("id", "title", "pid", "cwd", "cmdline", "num", "env", "var", "recent", "state", "neighbor")

def get_matches(location, query, candidates):
    # Mirror textual window matching: unanchored regex search on the title.
    return {i for i in candidates if re.search(query, TITLES[i]) is not None}

QUERIES = [
    "title:foo or title:bar",
    "title:foo title:bar",
    "title:foo or bar",
    "foo or bar",
    "title:foo and title:bar",
    "not title:foo",
    "(title:foo or title:bar)",
    "title:foo",
]
width = max(len(repr(q)) for q in QUERIES)
for q in QUERIES:
    label = repr(q).ljust(width)
    try:
        result = search(q, LOCATIONS, set(UNIVERSE), get_matches)  # allow_no_location defaults False
        print(f"{label} -> MATCHED {sorted(result)}")
    except ParseException as e:
        print(f"{label} -> {type(e).__name__}: {e.msg}")
```

### 3.2 The command and its verbatim output

Command:

```sh
python3 /tmp/sqp_probe_dir/probe.py
```

Captured output (verbatim; produced on **Python 3.13.7**):

```
'title:foo or title:bar'   -> MATCHED [1, 2, 3]
'title:foo title:bar'      -> MATCHED [3]
'title:foo or bar'         -> NoLocation: No location specified before bar
'foo or bar'               -> NoLocation: No location specified before foo
'title:foo and title:bar'  -> MATCHED [3]
'not title:foo'            -> MATCHED [2]
'(title:foo or title:bar)' -> MATCHED [1, 2, 3]
'title:foo'                -> MATCHED [1, 3]
```

### 3.3 Per-query interpretation

| Query | Observed result | Interpretation |
|-------|-----------------|----------------|
| `title:foo or title:bar` | `MATCHED [1, 2, 3]` | **`or` works** — union of both terms; every item matching *either* term is included. |
| `title:foo title:bar` | `MATCHED [3]` | Space-separated ⇒ **implicit AND**; only item 3 (`foobar shell`) matches *both*. Items 1 and 2 are excluded — **this reproduces your symptom.** |
| `title:foo or bar` | `NoLocation: No location specified before bar` | Right operand `bar` lacks a `location:` prefix ⇒ parse error. |
| `foo or bar` | `NoLocation: No location specified before foo` | *Neither* operand has a prefix ⇒ parse error on the first bare word. |
| `title:foo and title:bar` | `MATCHED [3]` | Explicit `and` equals the implicit-AND result — confirms the space is an implicit `and`. |
| `not title:foo` | `MATCHED [2]` | Negation (set difference): everything except the items matching `title:foo` (1 and 3), leaving 2. |
| `(title:foo or title:bar)` | `MATCHED [1, 2, 3]` | Parenthesized group; same union as the bare `or` form. |
| `title:foo` | `MATCHED [1, 3]` | Single-term baseline; unanchored regex substring match (`foo` occurs in `foo terminal` and `foobar shell`). |

The contrast is unmissable: `title:foo or title:bar → MATCHED [1, 2, 3]` (the `or` union is
correct) versus `title:foo title:bar → MATCHED [3]` (the implicit AND excludes items matching
only one term). The `or` operator is **not** the culprit.

I also confirmed the exception-class relationship at runtime:

```sh
python3 -c "import sys; sys.path.insert(0,'/tmp/blitzy/kitty/blitzy-74323c75-5d44-473e-bbde-c40e81287895_84020f'); from kitty.search_query_parser import NoLocation, ParseException; print('issubclass(NoLocation, ParseException) ==', issubclass(NoLocation, ParseException))"
```

```
issubclass(NoLocation, ParseException) == True
```

So the runtime error you would see is the precise subclass `NoLocation`; describing it as a
`ParseException` is equally correct (`NoLocation` **is a** `ParseException`).

---

## Section 4 — Root-cause pitfall #1: implicit AND on spaces

**Omitting `or` between two terms does not default to OR.** Whitespace between adjacent terms
triggers the *implicit* `AndNode` inserted by `and_expression` at
`kitty/search_query_parser.py:L215-L224` (specifically the condition and construction at
**L222-L223**). The two terms are therefore **intersected**.

**Direct evidence from the probe:**

```
'title:foo title:bar'      -> MATCHED [3]
```

Only item `3` (`foobar shell`) — the one whose title matches **both** `foo` *and* `bar` —
survives. Item `1` (`foo terminal`, matches only `foo`) and item `2` (`bar editor`, matches only
`bar`) are **excluded entirely**. That is exactly the phrasing in your report: *"items that
should clearly match at least one term are being excluded entirely."* They were excluded because
the query asked for items matching **every** term, not **any** term.

**Corroboration from kitty's official unit test:** `id:1 and id:2` evaluates to the empty set —
`kitty_tests/search_query_parser.py:L26` (`t('id:1 and id:2')`, whose `expected` defaults to
`set()` per `kitty_tests/search_query_parser.py:L18`). This is the classic "disjoint AND is
empty" case, and I reproduced it directly (see §7.3): `'id:1 and id:2' -> []`.


---

## Section 5 — Root-cause pitfall #2: missing `location:` prefix

Every operand needs a `location:` prefix because both production callers use the default
`allow_no_location=False` (§2.6). A bare or unprefixed word cannot be resolved to a location, so
`base_token` raises `NoLocation`.

**Direct evidence from the probe:**

```
'title:foo or bar'         -> NoLocation: No location specified before bar
'foo or bar'               -> NoLocation: No location specified before foo
```

- In `title:foo or bar`, the left operand is fine (`title:foo`) but the right operand `bar` has no
  prefix, so parsing fails with the exact message `No location specified before bar`.
- In `foo or bar`, the very first token `foo` has no prefix, so parsing fails with
  `No location specified before foo`.

These messages are emitted verbatim by the `else` branch of `NoLocation.__init__`
(`No location specified before {tt}`) at `kitty/search_query_parser.py:L143`, and the exception is
raised for a bare word at `kitty/search_query_parser.py:L278` (or, for a quoted word, at
`kitty/search_query_parser.py:L250`).

**The second message variant.** When the offending token *does* contain a colon but the segment
before the colon is not a recognized location, `NoLocation` instead emits
`{a} is not a recognized location in {tt}` — `kitty/search_query_parser.py:L141`. I triggered this
with a quoted opaque token `"id:1"` (see §7.3):

```
'"id:1"'                  -> NoLocation: id is not a recognized location in id:1
```

**Subclass relationship.** `NoLocation` is a subclass of `ParseException`
(`class NoLocation(ParseException):`, `kitty/search_query_parser.py:L136`), confirmed at runtime as
`issubclass(NoLocation, ParseException) == True` (§3.3). So whether tooling reports the error as
`NoLocation` or as `ParseException`, both labels are correct — `NoLocation` is simply the precise
subclass.

---

## Section 6 — Correct syntax

**The canonical fix:** prefix **both** operands with a `location:` and join them with the `or`
keyword. This matches items matching **either** term:

```
'title:foo or title:bar'   -> MATCHED [1, 2, 3]
```

On a real kitty instance, the entire match expression is passed as a **single shell-quoted
argument**:

```sh
kitten @ ls --match 'title:foo or title:bar'
```

### 6.1 All valid forms (each backed by observed output)

| Form | Example | Observed | Meaning |
|------|---------|----------|---------|
| **OR** (union) | `title:foo or title:bar` | `MATCHED [1, 2, 3]` | Matches items matching *either* term. |
| **AND** (intersection, explicit) | `title:foo and title:bar` | `MATCHED [3]` | Matches items matching *both* terms (identical to the implicit-AND space form). |
| **NOT** (difference) | `not title:foo` | `MATCHED [2]` | Matches items *not* matching the term. |
| **Grouping** | `(title:foo or title:bar)` | `MATCHED [1, 2, 3]` | Parentheses override precedence; here identical to the bare `or`. |
| **Single term** | `title:foo` | `MATCHED [1, 3]` | Baseline; unanchored regex substring match. |

To combine precedence explicitly, group with parentheses, e.g.
`(id:2 or id:3) and title:something` (one of kitty's own documented examples — see §7.2).

### 6.2 Matching semantics: regex vs numbers

- **Textual fields** (e.g. `title`, `cwd`, `cmdline`): the `query` is compiled as an
  **unanchored** Python regular expression via `re.compile` in `compile_match_query`
  (`kitty/window.py:L213-L222`) and matched with `re.search` — for a window's title,
  `pat.search(self.override_title or self.title)` at `kitty/window.py:L774`; for a tab's title,
  `re.search(query, self.effective_title)` at `kitty/tabs.py:L801-L802`. Because matching is
  unanchored, `title:foo` matches any title *containing* `foo` (hence `foo terminal` **and**
  `foobar shell` both match).
- **Numeric fields** (`id`, `pid`, `num`, `recent`): the expression is interpreted as a **number**,
  not a regex — help text `kitty/rc/base.py:L97-L99`. In code, `id`/`window_id`/`pid` compare the
  pattern text to the numeric string (`kitty/window.py:L769-L772`) and `num`/`recent` call
  `int(query)` (`kitty/window.py:L785-L795`).

Two practical corollaries for writing regex queries: (a) because matching is unanchored, use
anchors (`^`, `$`) if you need an exact title; and (b) regex metacharacters in a title
(e.g. `.`, `(`) are interpreted as regex, so escape them if you mean them literally.

---

## Section 7 — Windows-vs-tabs location reference + corroboration

### 7.1 Valid location fields (from the code — authoritative for commit `815df1e21`)

The valid locations differ between windows and tabs. These tuples are copied verbatim from the
`search(...)` calls in `kitty/boss.py`:

- **Windows** — `kitty/boss.py:L493-L494`:
  `id, title, pid, cwd, cmdline, num, env, var, recent, state, neighbor`
- **Tabs** — `kitty/boss.py:L529-L530`:
  `id, index, title, window_id, window_title, pid, cwd, env, var, cmdline, recent, state`

### 7.2 Help text and user-doc corroboration

The per-option help text agrees with the grammar:

- `kitty/rc/base.py:L89`: "Match specifications are of the form: `field:query`."
- `kitty/rc/base.py:L92-L93`: "Expressions can be either a number or a regular expression, and can
  be combined using Boolean operators".
- `kitty/rc/base.py:L95`: "The special value `all` matches all windows."

The user documentation (`docs/remote-control.rst`) defines the `search_syntax` anchor at
`docs/remote-control.rst:L327`, describes the `field:query` form at `docs/remote-control.rst:L335`,
notes terms "combined using Boolean operators" at `docs/remote-control.rst:L337`, and gives four
examples verbatim at `docs/remote-control.rst:L340-L343`:

```
title:"My special window" or id:43
title:bash and env:USER=kovid
not id:1
(id:2 or id:3) and title:something
```

**Key corroborating insight.** The public documentation shows **only** explicit `and` / `or`
operators — it never documents the implicit-AND-on-spaces behavior. I confirmed this against the
official kitty remote-control docs online (which likewise list only the four explicit-operator
examples above). The implicit-AND rule is real in the code (§2.4) but **undocumented publicly**,
which explains why a new user would not expect two space-separated terms to be intersected. This
reinforces the verdict: you hit an **under-documented usage quirk**, not a defect.

### 7.3 Official-test corroboration (reproduced verbatim)

The official unit test `kitty_tests/search_query_parser.py` encodes the expected results with
`locations = 'id'` (`kitty_tests/search_query_parser.py:L12`), universe `{1, 2, 3, 4, 5}`
(`L13`), and a `get_matches` that uses **exact equality** `query == str(x)` (`L15-L16`). I mirrored
this exact contract in a second ephemeral probe and captured, verbatim:

```
'id:1'                    -> [1]
'id:"1"'                  -> [1]
'id:1 and id:1'           -> [1]
'id:1 or id:2'            -> [1, 2]
'id:1 and id:2'           -> []
'not id:1'                -> [2, 3, 4, 5]
'(id:1 or id:2) and id:1' -> [1]
'1'                       -> NoLocation: No location specified before 1
'"id:1"'                  -> NoLocation: id is not a recognized location in id:1
```

This matches the test's assertions at `kitty_tests/search_query_parser.py:L22-L30` exactly:
`id:1 or id:2 → {1, 2}` (`L25`), `id:1 and id:2 → set()` (`L26`), `not id:1 → universal_set - {1}`
(`L27`), `(id:1 or id:2) and id:1 → {1}` (`L28`), and the bare `'1'` (`L29`) and quoted `'"id:1"'`
(`L30`) both raise `ParseException` (here surfaced as its `NoLocation` subclass). Note that
`id:1 or id:2 → [1, 2]` again demonstrates that **`or` includes items matching either term**.

### 7.4 Commit-fidelity note

Where the public kitty documentation lists **newer** location fields that are **not** present in
this commit's tuples — for example `session` (windows and tabs) and `focused_os_window` (a tab
`state` value) — the **code is authoritative** for this investigation. Those fields do not appear
in `kitty/boss.py:L493-L494` or `kitty/boss.py:L529-L530` at commit `815df1e21`, so they are not
valid locations here even though they appear in the online docs for later kitty versions.

---

## Final coverage pass

Confirming that every part of the question is answered:

- **(a) What is happening?** Your items are excluded because your query is asking for items
  matching *every* term (intersection), or your query fails to parse. Concretely: two
  space-separated terms are **implicitly ANDed** (§4), and any term without a `location:` prefix
  raises a `NoLocation` parse error (§5).
- **(b) Why?** The grammar. `and_expression` inserts an implicit `AndNode` for adjacent terms at
  `kitty/search_query_parser.py:L222-L223` (§2.4), and `base_token` requires a `location:` prefix,
  raising `NoLocation` at `kitty/search_query_parser.py:L278`/`L250` because both callers leave
  `allow_no_location=False` (`kitty/boss.py:L493-L495`, `kitty/boss.py:L529-L531`) (§2.5–2.6).
  Meanwhile `or` is a genuine set **UNION** (`OrNode.__call__`,
  `kitty/search_query_parser.py:L63-L65`) and cannot exclude an item that matches either term.
- **(c) Demonstrated?** Yes — with the exact ephemeral probe, the command that ran it, and its
  verbatim output (§3), plus the official-test contract reproduced verbatim (§7.3). Key contrast:
  `title:foo or title:bar → MATCHED [1, 2, 3]` versus `title:foo title:bar → MATCHED [3]`.
- **(d) Correct syntax?** Prefix both operands and join with `or`:
  `title:foo or title:bar` (e.g. `kitten @ ls --match 'title:foo or title:bar'`); plus the full set
  of valid forms — explicit `and`, `not`, and parenthesized grouping (§6).

**Verdict restated:** This is **not a bug**. The `or` operator works correctly (set union). The
symptom is caused by (1) implicit-AND on space-separated terms and/or (2) a missing mandatory
`location:` prefix. Writing `title:foo or title:bar` resolves it.

---

*Grounding note:* All code citations are to the repository at commit `815df1e21`. All quoted
program output was produced by running `kitty.search_query_parser.search` via ephemeral probes
under `/tmp/sqp_probe_dir/` on Python 3.13.7; those probes live outside the repository and were
deleted after the investigation, leaving the repository byte-for-byte unchanged apart from this
document.

