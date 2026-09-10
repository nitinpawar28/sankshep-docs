# Tool reference

Every tool Sankshep exposes over MCP, with its arguments and a **real** request/response captured from a
running v2.0.0 server. Long absolute paths in the captures are shortened to `C:\...\sample-repo`; nothing
else is edited.

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

| Argument | Type | Default | Notes |
|---|---|---|---|
| `query` | string | — | What you're looking for; ranks the files under `paths` (blended semantic + lexical relevance). Required. |
| `paths` | string[] | — | Files or directories (repo-relative or absolute). Required. |
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
// 329 original -> 255 delivered tokens (22.5% smaller).
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
}
```

Read the header as: `original -> delivered (% smaller)` is the compression of the files actually delivered.
Two more lines appear when they apply — a `Searched N tokens` line when the semantic index was consulted,
and a `WITHHELD` line naming the files that did not fit. `Searched` is context looked at, **not** a saving.
On very small inputs the locator header can make a file *larger*; that is reported honestly, not hidden.

`tokenBudget` is a ceiling on the **whole response, the header included** — not on the code alone.

An **empty** delivery returns **`isError: true`**, never a silent 0-token "success". Captured verbatim with
`tokenBudget: 150` against a large tree:

```text
// ---- Sankshep context ----------------------------------------------
// WITHHELD, did not fit tokenBudget: tests/…/ChunkQueryCoverageTests.cs, tests/…/IndexToolsTests.cs (+11 more)
// NO CONTEXT was returned: tokenBudget=150 was too small to admit any chunk.
// Re-run with corrected `paths` or a larger `tokenBudget` -- do not answer from guesswork.
// --------------------------------------------------------------------
```

---

## `search_code`

Semantic (vector KNN) search over the local embedding index — pure nearest-neighbour similarity, no lexical
blend. Run `index_repo` first.

| Argument | Type | Default | Notes |
|---|---|---|---|
| `query` | string | — | Natural-language description of what you want. Required. |
| `topK` | integer | `10` | Max chunks to return. |

!!! info "Changed in 2.0.0 — the result is text, not JSON"
    Hits used to come back as one JSON line with every newline, quote and tab escaped. They are now the
    same shape `get_context` uses: a header, then `// path:start-end (Kind Symbol)` above each block of
    real code. **Breaking for any caller that parsed the result as JSON.** Everything a hit carried is
    still present. See [Upgrading to 2.0.0](upgrading-2.0.md).

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

If it returns nothing right after a successful `index_repo`, see
[Troubleshooting → search returns nothing after indexing](troubleshooting.md#index-built-but-search-empty).

---

## `index_repo`

Builds or refreshes the semantic index (chunks + local embeddings) that `search_code` uses. Downloads the
embedding model (~127 MB, once) on first run.

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
Captured with `maxTokens: 500` against `src/Sankshep.Memory`:

```text
Repo summary: C:\...\src\Sankshep.Memory
TRUNCATED: 12 of 28 files shown, under a 500-token budget. This is NOT the whole repository. Raise
maxTokens, summarize a subdirectory, or use search_code to find what you are looking for directly.
```

A `maxTokens` below the floor is refused rather than silently raised:

```text
summarize_repo: maxTokens must be at least 500 (got 60). (Parameter 'maxTokens')
```

A not-found path, or a tree with no supported source files, returns **`isError: true`**.

---

## `remember`

Persists a fact (decision, convention, or note) to this repo's memory, tagged with the current git branch.

| Argument | Type | Default | Notes |
|---|---|---|---|
| `category` | string | — | e.g. `decision`, `convention`, `gotcha`. |
| `text` | string | — | The fact to remember. |
| `source` | string \| null | — | Optional provenance (a file path or URL). |

```jsonc
// request
{ "category": "convention", "text": "All money values use decimal, never double.",
  "source": "src/Calculator.cs" }
```
```jsonc
// response — content[0].text
{ "id": 1, "category": "convention", "branch": "main" }
```

The current git `branch` is echoed back so you can see where the fact was tagged; off a git repo it is
recorded (and returned) as `"global"`.

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
{ "count": 1, "matched": 1, "truncated": false, "facts": [
  { "id": 3, "category": "convention",
    "text": "Money is stored in minor units (cents) everywhere.",
    "source": "docs/adr/0003-money.md", "createdAt": "2026-09-10T05:05:45+00:00" } ] }
```

Two fields are new in 2.0.0. **`matched`** is how many facts the query matched; **`count`** is how many were
returned. The result set is capped at **50**, and when the cap bites, `truncated` is `true` and the two
numbers differ — so a caller can tell "these are all of them" from "these are the first fifty". Narrow the
query or pass a `category` to see the rest.

Matching is per-word, not substring: every whitespace-separated word in `query` has to appear somewhere in
the fact.

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
{ "path": "C:\\...\\sample-repo\\DECISIONS.md", "count": 2 }
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
{ "calls": 3, "scannedTokens": 987, "selectedTokens": 987,
  "deliveredTokens": 769, "compressedTokens": 218, "compressionPct": 0.2208713272543060,
  "byTool": [
    { "tool": "compose_task_prompt", "calls": 1, "scannedTokens": 329, "selectedTokens": 329, "deliveredTokens": 259 },
    { "tool": "get_context", "calls": 2, "scannedTokens": 658, "selectedTokens": 658, "deliveredTokens": 510 } ] }
```

On a fresh repo every number is `0` and `byTool` is `[]`.

`compressionPct` is `1 − delivered/selected` — measured against the **original size of the files actually
delivered**, never against `scanned`. On tiny inputs it can go negative (the locator headers cost more than
minimization saves); that is reported honestly. See [Benchmarks](benchmarks.md) for how savings and quality
are measured together.

---

## `compose_task_prompt` (an MCP prompt, not a tool)

Composes a grounded, ready-to-use prompt from token-minimized code (`get_context`'s engine) + remembered
conventions (`recall`). It returns a **prompt to act on — never the answer itself**.

| Argument | Type | Default | Notes |
|---|---|---|---|
| `task` | string | — | The coding task. |
| `paths` | string | — | Comma- or newline-separated paths to draw code from. |
| `tokenBudget` | string | `4000` | Split ~70/30 between code and conventions. |

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
}

# Project conventions
- Money is stored in minor units (cents) everywhere.

# Constraints
- Work within the code shown above; do not invent new packages or patterns.
- Follow the recalled conventions exactly. If information is missing, say so rather than guessing.
```

Note which body survived: `Add` is kept in full because the task names it, while `Subtract` is collapsed.
That is query-targeted body collapse, and it is the same engine `get_context` uses.

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
`StatsSnapshot` the dashboard feed serves, and is registered on both stdio and HTTP transports.
