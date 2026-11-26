# Tool reference

Every tool Sankshep exposes over MCP, with its arguments and a **real** request/response captured from a
running **v4.0.0** server — the published package, driven over stdio against a three-file C# sample
repository. Long absolute paths in the captures are shortened to `C:\...\sample-repo`; nothing else is
edited. Two captures come from elsewhere and say so: the budget-bound `get_context` examples use a generated
31-file C# tree, because the three-file sample is too small to show them, and one `summarize_repo`
truncation capture was taken on 2.0.0 against a larger tree and has not been re-taken.

Paths are **relative to the repo the server was started against** (`serve --repo <root>`).

!!! warning "Absolute paths are not a general escape hatch"
    On **stdio**, an absolute path inside the served root is accepted; one outside it is refused. On the
    **HTTP transport** an absolute path is refused outright, whether or not it points inside the root, and
    `index_repo` refuses one on **both** transports. Use repo-relative paths and they work everywhere. See
    [Security](security.md).

!!! tip "These fire in Agent mode, not Ask"
    MCP tools are only invoked when the client is in an agent/tool-using mode (e.g. Copilot **Agent** mode).
    In plain chat/Ask mode the model can't call them. See [Troubleshooting](troubleshooting.md).

---

## `get_context`

Token-minimized, relevance-ranked context for the given paths: strips comments, collapses non-target method
bodies, packs under a token budget, and leads with a header stating what was compressed and withheld.
Since 4.0.0, a file that does not fit what is left of the budget is delivered **by its declarations** — its
types and top-level methods or functions, with the comments, attributes and decorators that belong to them —
the pieces whose name or text matches the query first, across every ranked file. Each excerpt is cited
`// path:start-end Symbol`, contiguous pieces share one locator, and a `PARTIAL` header line names the file.

| Argument | Type | Default | Notes |
|---|---|---|---|
| `query` | string | — | What you're looking for; ranks the files under `paths` (blended semantic + lexical relevance). Required. |
| `paths` | string[] | — | Files or directories, repo-relative. Required. (An absolute path is stdio-only and must be inside the root — see the note above.) |
| `tokenBudget` | integer | — | Max tokens of context to return. Required. |
| `level` | string | `Balanced` | `Conservative` · `Balanced` · `Aggressive`. |

```jsonc
// request
{ "query": "how are numbers added", "paths": ["."],
  "tokenBudget": 400, "level": "Balanced" }
```
```text
// response — content[0].text (a single readable block)
// ---- Sankshep context ----------------------------------------------
// 329 original -> 253 delivered tokens (23.1% smaller).
// Missing something? Narrow `paths`, raise `tokenBudget`, or use level=Conservative.
// --------------------------------------------------------------------

// Calculator.cs:1-29
namespace SampleRepo.Math;

public class Calculator
{
    private double _total;

    public double Add(double a, double b)
    { /* … body elided (3 lines) */ }

    public double Subtract(double a, double b)
    { /* … body elided (3 lines) */ }

    private double Normalize(double value)
    { /* … body elided (2 lines) */ }
}


// Greeter.cs:1-16
namespace SampleRepo.Text;

public class Greeter
{
    public string Greet(string name)
    { /* … body elided (2 lines) */ }

    public static string GreetFormal(string title, string name)
    { /* … body elided (2 lines) */ }
}


// Inventory.cs:1-25
namespace SampleRepo.Stock;

public interface IInventory
{
    int CountOf(string sku);
}

public sealed class Inventory : IInventory
{
    private readonly Dictionary<string, int> _stock = new();

    public void Restock(string sku, int quantity)
    { /* … body elided (2 lines) */ }

    public int CountOf(string sku)
    { /* … body elided (2 lines) */ }
}
```

Read the header as: `original -> delivered (% smaller)` is the compression of the source actually
delivered — a whole file counts whole, a file delivered in pieces counts the pieces' own source, and a
truncated excerpt counts the source its kept lines map back to (changed in 4.0.0; a truncated excerpt used to
be credited with its whole file). `LARGER` replaces `smaller` when the locators cost more than minimization
saved, as they can on very small inputs; that is reported honestly, not hidden. The other lines appear only
when they apply:

- `Searched N tokens` appears whenever the files scanned under `paths` hold more than the source delivered,
  because ranking picked a subset. It appears with or without the semantic index, so it is no sign that the
  index was consulted. `Searched` is context looked at, **not** a saving.
- `N chunk(s) withheld to fit tokenBudget=B` counts pieces of partial files, whole files withheld, and ranked
  files never minimized alike.
- `WARNING:` lines name a path that matched nothing, a file that could not be read, or a file that was
  requested but not included, with the reason.
- `NOTE: <file> was TRUNCATED to fit tokenBudget; the cut is marked in the excerpt.` names the one file that
  was cut inside a declaration to fill what was left.
- `PARTIAL, some declarations not delivered within tokenBudget (see each excerpt's line range):` names the
  files that came back in part (new in 4.0.0). The line ranges on their excerpts show what is missing.
- `WITHHELD, ranked but not delivered within tokenBudget:` names the files of which **nothing** came back —
  every ranked file not delivered, including those ranked after the point where the budget filled. Changed in
  4.0.0: it used to read `did not fit`, and it omitted files that were never tried.

The `WARNING`, `NOTE`, `PARTIAL` and `WITHHELD` lines name as many paths as the budget leaves room for, then
`(+N more)` or a bare count such as `5 file(s)`. As a last resort they fold into one line, which the second
capture below shows: `// NOTE: 1 WITHHELD; raise tokenBudget to see which.`

`tokenBudget` is a ceiling on the **whole response, the header included** — not on the code alone. Changed
in 4.0.0: it now holds on calls that fill the budget too, blank lines between chunks included. Such a call is
packed again with room for the variable header lines, so it carries up to 116 fewer tokens of code than a
plain pack would; a call with room to spare is unchanged. Until 4.0.0 a budget-bound call came back 120 to
150 tokens over, because those lines were added on top.

A file that does not fit what is left of the budget comes back **in part**. Captured verbatim against the
sample repository with `tokenBudget: 250` — `Calculator.cs` fits whole, so it is unchanged; the other two
files are cut at their declarations and the line ranges say what is missing:

```text
// ---- Sankshep context ----------------------------------------------
// PARTIAL, some declarations not delivered within tokenBudget (see each excerpt's line range): Greeter.cs, Inventory.cs
// 165 original -> 124 delivered tokens (24.8% smaller).
// Searched 329 tokens to select them. Searched is NOT a baseline:
// it is not context you would otherwise have sent, and not a saving.
// 7 chunk(s) withheld to fit tokenBudget=250.
// Missing something? Narrow `paths`, raise `tokenBudget`, or use level=Conservative.
// --------------------------------------------------------------------

// Calculator.cs:1-29
namespace SampleRepo.Math;

public class Calculator
{
    private double _total;

    public double Add(double a, double b)
    { /* … body elided (3 lines) */ }

    public double Subtract(double a, double b)
    { /* … body elided (3 lines) */ }

    private double Normalize(double value)
    { /* … body elided (2 lines) */ }
}


// Greeter.cs:1-5
namespace SampleRepo.Text;

public class Greeter
{


// Inventory.cs:1-2
namespace SampleRepo.Stock;
```

The relevant piece comes first *within* the budget and is rendered in source order, under its own locator
with the symbol's name. Captured against a generated 31-file C# tree (the sample repository's files are too
small to show this), asking for `Ship` in a ten-method class with `tokenBudget: 300`:

```text
// ---- Sankshep context ----------------------------------------------
// PARTIAL, some declarations not delivered within tokenBudget (see each excerpt's line range): OrderService.cs
// 268 original -> 147 delivered tokens (45.1% smaller).
// Searched 818 tokens to select them. Searched is NOT a baseline:
// it is not context you would otherwise have sent, and not a saving.
// 7 chunk(s) withheld to fit tokenBudget=300.
// Missing something? Narrow `paths`, raise `tokenBudget`, or use level=Conservative.
// --------------------------------------------------------------------

// OrderService.cs:1-31
namespace Shop.Orders;

public class OrderService
{
    private readonly List<string> _log = new();

    public string Create(string orderId, int quantity)
    { /* … body elided (7 lines) */ }

    public string Validate(string orderId, int quantity)
    { /* … body elided (7 lines) */ }



// OrderService.cs:68-79 Ship
    public string Ship(string orderId, int quantity)
    {
        _log.Add("ship:" + orderId);
        if (quantity <= 0)
        {
            throw new ArgumentOutOfRangeException(nameof(quantity));
        }
        return orderId + ":" + quantity;
    }
```

An **empty** delivery returns **`isError: true`**, never a silent 0-token "success". Captured verbatim with
`tokenBudget: 150` against the sample repository, then against the generated tree — the second shows the
names trimmed to what fits:

```text
// ---- Sankshep context ----------------------------------------------
// WITHHELD, ranked but not delivered within tokenBudget: Calculator.cs, Greeter.cs, Inventory.cs
// NO CONTEXT was returned: tokenBudget=150 was too small to admit any chunk.
// Re-run with corrected `paths` or a larger `tokenBudget` -- do not answer from guesswork.
// --------------------------------------------------------------------
```

```text
// ---- Sankshep context ----------------------------------------------
// WITHHELD, ranked but not delivered within tokenBudget: OrderService.cs, Service01.cs, Service02.cs, Service03.cs, Service04.cs (+26 more)
// NO CONTEXT was returned: tokenBudget=150 was too small to admit any chunk.
// Re-run with corrected `paths` or a larger `tokenBudget` -- do not answer from guesswork.
// --------------------------------------------------------------------
```

Note the cliff: against the sample repository `tokenBudget: 250` delivers `Calculator.cs` whole and
`tokenBudget: 200` delivers nothing, because the second pack reserves room for the header lines the first
pack turned out to need. If a call comes back empty, raising the budget by a few hundred tokens is usually
enough.

---

## `search_code`

Semantic (vector KNN) search over the local embedding index — pure nearest-neighbour similarity, no lexical
blend. Run `index_repo` first.

| Argument | Type | Default | Notes |
|---|---|---|---|
| `query` | string | — | Natural-language description of what you want. Required. |
| `topK` | integer | `10` | Max chunks to return, **1–100**. Above 100 is refused with an error naming the bound — for a broader read use `get_context` or `summarize_repo`. Changed in 4.0.0. |

!!! info "Changed in 2.0.0 — the result is text, not JSON"
    Hits used to come back as one JSON line with every newline, quote and tab escaped. They are now the
    same shape `get_context` uses: a header, then `// path:start-end (Kind Symbol)` above each block of
    real code. **Breaking for any caller that parsed the result as JSON.** Everything a hit carried is
    still present. See [Upgrading to 2.0.0](upgrading-2.0.md).

!!! info "Changed in 4.0.0 — a bounded refresh, a stale-file note, and `#` locators"
    Before answering, `search_code` re-indexes files that changed since they were indexed — but now **at
    most 25 of them** per search, so one search after a branch switch no longer spends minutes embedding.
    When it leaves changed files behind, the header gains a line saying how many; `index_repo` (or the
    `SANKSHEP_WATCH` watcher) brings them current. Captured against a generated 31-file tree after touching
    30 of its files:

    ```text
    // ---- Sankshep search: 2 hits, best first ----
    // Pass any path below to get_context for the whole minimized file.
    // NOTE: 5 indexed file(s) changed on disk and were not re-indexed before this search, so hits from them may be stale. Run index_repo to bring them current.
    ```

    Hits from Python and Ruby files are introduced by `# path:start-end (Kind Symbol)` rather than `//`,
    because in those languages `//` reads as code. And `topK` above 100 is refused:

    ```text
    search_code: topK must be at most 100 (got 101). For more of the repository than that, call get_context with the paths these hits name, or summarize_repo. (Parameter 'topK')
    ```

```jsonc
// request
{ "query": "add two numbers", "topK": 2 }
```
```text
// response — content[0].text
// ---- Sankshep search: 2 hits, best first ----
// Pass any path below to get_context for the whole minimized file.

// Calculator.cs:6-28 (Type Calculator)
public class Calculator
{
    // Running total, mutated by Add/Subtract.
    private double _total;

    public double Add(double a, double b)
    {
        _total = a + b;
        return _total;
    }

    public double Subtract(double a, double b)
    {
        _total = a - b;
        return _total;
    }

    // Internal helper — not part of the public surface, should not appear in a summary.
    private double Normalize(double value)
    {
        return value == 0 ? 0 : value;
    }
}

// Inventory.cs:3-6 (Type IInventory)
public interface IInventory
{
    int CountOf(string sku);
}
```

An **empty index is an error, not an empty success** — the change that matters most here, because the old
`{count:0,results:[]}` was indistinguishable from "this code does not exist". Captured verbatim:

```text
search_code: the index is empty - nothing has been indexed for this repo yet, or the index was purged.
This is NOT evidence that the code does not exist. Run index_repo with path "." first (relative paths
resolve against the served --repo root, C:\...\sample-repo), then retry. Only index_repo discovers new
files; verify-on-read and SANKSHEP_WATCH refresh already-indexed files only.
```

The last sentence of that message understates `SANKSHEP_WATCH`. With the watcher on, each burst of file
changes, once it settles, runs an incremental index over the served root, and that picks up new files too.
Verify-on-read, which `search_code` runs before answering, is the part that refreshes only files already in
the index.

If it returns nothing right after a successful `index_repo`, see
[Troubleshooting → search returns nothing after indexing](troubleshooting.md#index-built-but-search-empty).

---

## `index_repo`

Builds or refreshes the semantic index (chunks + local embeddings) that `search_code` uses. Downloads the
embedding model (~127 MiB, once) on first run.

| Argument | Type | Default | Notes |
|---|---|---|---|
| `path` | string | `"."` | Directory to index. **Optional** — defaults to the whole served repo. Must be a directory, not a file. Repo-relative only: an absolute path is refused on both transports. |
| `force` | boolean | `false` | Re-embed every chunk even if unchanged. |

```jsonc
// request
{}
```
```text
// response — content[0].text
index_repo: indexed 3 file(s) under C:\...\sample-repo; the index now holds 4 chunk(s) in total.
```

The message names **what this walk did** and, separately, what the whole index holds. Those two numbers
differ whenever you index a subdirectory, and the old message conflated them — it reported the index's
total as if the walk had produced it.

When the client sends a `progressToken`, `index_repo` emits a progress notification per file, so a first
index of a large repository is distinguishable from a hang.

A path that doesn't exist, that names a file, or that holds no supported source files returns
**`isError: true`** with a clear message rather than a silent success.

!!! warning "Every index rebuilds once, on the first 2.0.0 start"
    Chunks are now sized by what the embedding model can actually read, so `ChunkerVersion` moved and an
    existing `index.db` is discarded and rebuilt. It is derived data, so this is always safe — it just
    takes minutes on a large repository. See [Upgrading to 2.0.0](upgrading-2.0.md).

---

## `summarize_repo`

Walks a directory and returns namespaces, type names and public method signatures from **every language
Sankshep parses**, not just C#.

| Argument | Type | Default | Notes |
|---|---|---|---|
| `path` | string | — | Directory to summarize. Required. |
| `maxTokens` | integer | `20000` | Ceiling for the whole summary. The walk stops at a file boundary once reached. Minimum `500`. |

```jsonc
// request
{ "path": "." }
```
```text
// response — content[0].text
Repo summary: C:\...\sample-repo

## Calculator.cs
namespace SampleRepo.Math
  type class Calculator
  method public double Add(double a, double b)
  method public double Subtract(double a, double b)

## Greeter.cs
namespace SampleRepo.Text
  type class Greeter
  method public string Greet(string name)
  method public static string GreetFormal(string title, string name)

## Inventory.cs
namespace SampleRepo.Stock
  type interface IInventory
  type class Inventory
  method public void Restock(string sku, int quantity)
  method public int CountOf(string sku)
```

**A truncated summary says so, on the second line.** Before 2.0.0 this tool appended every declaration of
every file with no budget and no cap, which on a real repository could consume the whole context window.
Captured on 2.0.0 with `maxTokens: 500` against a larger, 28-file tree (the three-file sample cannot fill
500 tokens, so this capture has not been re-taken; the wording is unchanged in 4.0.0):

```text
Repo summary: C:\...\src\Sankshep.Memory
TRUNCATED: 12 of 28 files shown, under a 500-token budget. This is NOT the whole repository. Raise
maxTokens, summarize a subdirectory, or use search_code to find what you are looking for directly.
```

A `maxTokens` below the floor is refused rather than silently raised:

```text
summarize_repo: maxTokens must be at least 500 (got 60). (Parameter 'maxTokens')
```

A not-found path, or a tree with no supported source files, returns **`isError: true`**. So does a path that
names a file — since 4.0.0 the error says so, instead of reporting the file as not found:

```text
summarize_repo: 'Calculator.cs' is a file, not a directory - summarize_repo walks a tree, so pass the directory that contains it. To read that one file, call get_context with it in `paths`.
```

Also since 4.0.0, a file the walk listed but could not read — locked, deleted after the walk, or denied — no
longer fails the whole call. It is skipped, and the answer gains a `WARNING: N file(s) could not be read and
are NOT summarized: …` line under the `Repo summary:` line, naming up to five of them. (Stated from the
release notes, not captured: this sample has no unreadable file.)

---

## `remember`

Persists a fact (decision, convention, or note) to this repo's memory, tagged with the current git branch.

| Argument | Type | Default | Notes |
|---|---|---|---|
| `category` | string | — | e.g. `decision`, `convention`, `gotcha`. At most **64** characters after trimming. Changed in 4.0.0. |
| `text` | string | — | The fact to remember. At most **4,000** characters. Changed in 4.0.0. |
| `source` | string \| null | — | Optional provenance (a file path or URL). At most **2,048** characters; a blank value is stored as no source (4.0.0). |

```jsonc
// request
{ "category": "convention", "text": "All money values use decimal, never double.",
  "source": "src/Calculator.cs" }
```
```jsonc
// response — content[0].text
{"id":1,"category":"convention","branch":"main"}
```

The current git `branch` is echoed back so you can see where the fact was tagged; off a git repo it is
recorded (and returned) as `"global"`.

!!! warning "Changed in 4.0.0 — oversized input is refused before anything is stored"
    A fact cannot be deleted, so an oversized one would inflate every later `recall`, `DECISIONS.md` and
    composed prompt for good. Input over a bound is refused with an error naming the argument and the bound,
    and facts already stored are untouched. Captured verbatim (the doubled `remember: remember:` prefix is
    what the server prints):

    ```text
    remember: remember: text must be at most 4000 characters (got 4001) - a fact is stored permanently and returned by every recall that matches it. There is no tool that deletes a fact, so an oversized one would be permanent. (Parameter 'text')
    ```

    ```text
    remember: remember: category must be at most 64 characters (got 65) - it is a label other tools match on, not prose. There is no tool that deletes a fact, so an oversized one would be permanent. (Parameter 'category')
    ```

---

## `recall`

Recalls remembered facts (current branch + global) whose text matches a query, optionally filtered by category.

| Argument | Type | Default | Notes |
|---|---|---|---|
| `query` | string | — | Words that must **all** appear in the fact's text, in any order, ignoring case. Required. `%` and `_` are matched literally. |
| `category` | string \| null | — | Optional filter, trimmed and matched case-insensitively. |

```jsonc
// request
{ "query": "money units" }
```
```jsonc
// response — content[0].text
{"count":1,"facts":[{"id":4,"category":"convention","text":"Money is stored in minor units (cents) everywhere.","source":"docs/adr/0003-money.md","branch":"main","createdAt":"2026-10-14T04:58:55.4729143+00:00"}],"matched":1,"truncated":false}
```

Two fields are new in 2.0.0. **`matched`** is how many facts the query matched; **`count`** is how many were
returned. The result set is capped at **50**, and when the cap bites, `truncated` is `true` and the two
numbers differ — so a caller can tell "these are all of them" from "these are the first fifty". Narrow the
query or pass a `category` to see the rest.

Matching is per-word, not substring: every whitespace-separated word in `query` has to appear somewhere in
the fact — every word, since 4.0.0; words past the sixteenth distinct one used to be ignored. A fact stored
with a blank `source` comes back without a `source` field at all:

```jsonc
{"count":1,"facts":[{"id":5,"category":"gotcha","text":"Greeter.GreetFormal is static; Greet is not.","branch":"main","createdAt":"2026-10-14T04:58:55.4750875+00:00"}],"matched":1,"truncated":false}
```

---

## `export_decisions`

Writes all `decision`-category facts to a `DECISIONS.md` at the repo root, grouped by branch.

| Argument | Type | Default | Notes |
|---|---|---|---|
| _(none)_ | | | Operates on the served repo. |

```jsonc
// request
{}
```
```jsonc
// response — content[0].text
{"path":"C:\\...\\sample-repo\\DECISIONS.md","count":2}
```

`count` is the number of **decision-category** facts exported. The category label is matched exactly — a
`convention` fact is not a decision, and neither is one filed under `decisions` or `architecture-decision`.
With none stored, the call is an error rather than a file with an empty section:

```text
export_decisions: no facts are stored in the "decision" category, so nothing was exported and DECISIONS.md
was left untouched. Remember a fact with category "decision" first. Note the label is matched exactly:
"decisions" and "architecture-decision" are different categories.
```

!!! warning "Changed in 2.0.0 — it will not overwrite a file it did not write"
    The generated file now carries a marker on its third line:

    ```markdown
    # Decisions

    <!-- Generated by sankshep export_decisions. Edits to this file will be overwritten. -->
    ```

    A `DECISIONS.md` **without** that marker is refused rather than clobbered — which means the first
    export after upgrading fails once if you already had one. Move or rename it and re-run.

    ```text
    export_decisions: DECISIONS.md already exists and was not written by export_decisions, so it was left
    untouched rather than overwritten. Move or rename it first if you want it replaced.
    ```

    Re-running over a file it *did* write succeeds and rewrites it, as it always did.

---

## `token_report`

Cumulative token accounting for this repo — how much minimization compressed, per tool and in total. No dollar
figure: Sankshep is never told which model you use or your rate, so any total would be an assumption, not a
measurement.

| Argument | Type | Default | Notes |
|---|---|---|---|
| _(none)_ | | | Reads the local `stats.db`. |

```jsonc
// request
{}
```
```jsonc
// response — content[0].text
{"calls":8,"scannedTokens":2632,"selectedTokens":823,"deliveredTokens":634,"compressedTokens":189,"compressionPct":0.22964763061968407,"byTool":[{"tool":"compose_task_prompt","calls":1,"scannedTokens":329,"selectedTokens":329,"deliveredTokens":257},{"tool":"get_context","calls":7,"scannedTokens":2303,"selectedTokens":494,"deliveredTokens":377}]}
```

On a fresh repo every number is `0` and `byTool` is `[]`. (The eight calls above are the seven `get_context`
calls made while capturing this page, five of which delivered nothing, plus the composed prompt below.)

`compressionPct` is `1 − delivered/selected` — never against `scanned`. Since 4.0.0, `selected` is the
original size of the **source actually delivered**: a whole file counts whole, a file delivered in part counts
the declarations that came back (capped at the file), and a truncated excerpt counts the source its kept lines
map back to. Before 4.0.0 a truncated excerpt was credited with its whole file, so budget-bound calls read
higher than they should have; totals recorded before 4.0.0 were credited that way and are not recomputed, so
a long-lived `stats.db` mixes the two. On tiny inputs the figure can go negative (the locator headers cost
more than minimization saves); that is reported honestly. See [Benchmarks](benchmarks.md) for how savings and
quality are measured together.

---

## `compose_task_prompt` (an MCP prompt, not a tool)

Composes a grounded, ready-to-use prompt from token-minimized code (`get_context`'s engine) + remembered
conventions (`recall`). It returns a **prompt to act on — never the answer itself**.

| Argument | Type | Default | Notes |
|---|---|---|---|
| `task` | string | — | The coding task. |
| `paths` | string | — | Comma- or newline-separated paths to draw code from. |
| `tokenBudget` | string | `4000` | Bounds the **code**. Conventions are additive (600 tokens), so the prompt is larger than this. Changed in 3.0.0. |

```jsonc
// request (prompts/get)
{ "task": "Add a Multiply method to the calculator", "paths": ".", "tokenBudget": "1500" }
```
```text
// response — the composed prompt (4 labelled sections)
# Task
Add a Multiply method to the calculator

# Relevant code (minimized)
// Calculator.cs:1-29
namespace SampleRepo.Math;

public class Calculator
{
    private double _total;

    public double Add(double a, double b)
    {
        _total = a + b;
        return _total;
    }

    public double Subtract(double a, double b)
    { /* … body elided (3 lines) */ }

    private double Normalize(double value)
    { /* … body elided (2 lines) */ }
}


// Inventory.cs:1-25
namespace SampleRepo.Stock;

public interface IInventory
{
    int CountOf(string sku);
}

public sealed class Inventory : IInventory
{
    private readonly Dictionary<string, int> _stock = new();

    public void Restock(string sku, int quantity)
    { /* … body elided (2 lines) */ }

    public int CountOf(string sku)
    { /* … body elided (2 lines) */ }
}


// Greeter.cs:1-16
namespace SampleRepo.Text;

public class Greeter
{
    public string Greet(string name)
    { /* … body elided (2 lines) */ }

    public static string GreetFormal(string title, string name)
    { /* … body elided (2 lines) */ }
}

# Project conventions
- Money is stored in minor units (cents) everywhere.
- All money values use decimal, never double.

# Constraints
- Work within the code shown above; do not invent new packages or patterns.
- Follow the recalled conventions exactly. If information is missing, say so rather than guessing.
```

Note which body survived: `Add` is kept in full because the task names it, while `Subtract` is collapsed.
That is query-targeted body collapse, and it is the same engine `get_context` uses — including, since 4.0.0,
declaration-level delivery of a file that does not fit the budget. The composed prompt carries **no header**,
though: a file delivered in part shows up only as excerpts whose `// path:start-end Symbol` line ranges do not
cover the whole file, and nothing in the prompt names withheld files.

**"Project conventions" reads the `convention` category only.** A fact remembered under `decision`,
`gotcha` or anything else never appears here, however relevant it is. The section says so rather than
looking merely empty — with nothing stored at all it reads `(none recorded)`, and when facts exist under
other labels it names them:

```text
# Project conventions
(none recorded -- 1 fact(s) are stored under other categories (decision); compose_task_prompt injects only
facts remembered with category "convention")
```

See [Prompt composer](composer.md) for the full anatomy and the prompt-not-answer boundary.

---

## Resources

Alongside its tools, Sankshep exposes one MCP **resource** — data a client pulls in-context by URI rather
than invoking with arguments.

### `sankshep://stats`

Cumulative token accounting for this repo as `application/json`: how far minimization compressed the
delivered code, how much context was searched to find it, and a per-tool breakdown. It is the same
`StatsSnapshot` the dashboard feed serves, and is registered on both stdio and HTTP transports. Captured with
`resources/read` immediately after the `token_report` call above:

```jsonc
// request (resources/read)
{"uri":"sankshep://stats"}
```
```jsonc
// response — contents[0], mimeType application/json
{"calls":8,"scannedTokens":2632,"selectedTokens":823,"deliveredTokens":634,"compressedTokens":189,"compressionPct":0.22964763061968407,"byTool":[{"tool":"compose_task_prompt","calls":1,"scannedTokens":329,"selectedTokens":329,"deliveredTokens":257},{"tool":"get_context","calls":7,"scannedTokens":2303,"selectedTokens":494,"deliveredTokens":377}]}
```
