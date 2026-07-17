# Tool reference

Every tool Sankshep exposes over MCP, with its arguments and a **real** request/response captured from a
running server. Paths are **relative to the repo the server was started against** (`serve --repo <root>`);
absolute paths are used as-is.

!!! tip "These fire in Agent mode, not Ask"
    MCP tools are only invoked when the client is in an agent/tool-using mode (e.g. Copilot **Agent** mode).
    In plain chat/Ask mode the model can't call them. See [Troubleshooting](troubleshooting.md).

---

## `get_context`

Token-minimized, relevance-ranked context for the given paths: strips comments, collapses non-target method
bodies, packs under a token budget, and leads with a header stating what was compressed and withheld.

| Argument | Type | Default | Notes |
|---|---|---|---|
| `query` | string | — | What you're looking for; ranks the files under `paths`. |
| `paths` | string[] | — | Files or directories (repo-relative or absolute). Required. |
| `tokenBudget` | integer | — | Max tokens of context to return. |
| `level` | string | `Balanced` | `Conservative` · `Balanced` · `Aggressive`. |

```jsonc
// request
{ "query": "how are orders routed to a broker",
  "paths": ["Backend/TradingPlatform.Application/Services/Orders"],
  "tokenBudget": 4000, "level": "Balanced" }
```
```text
// response — content[0].text (a single readable block)
// ---- Sankshep context ----------------------------------------------
// 5,278 original -> 3,538 delivered tokens (33.0% smaller).
// Searched 45,061 tokens to select them. Searched is NOT a baseline:
// it is not context you would otherwise have sent, and not a saving.
// 7 chunk(s) withheld to fit tokenBudget=4000.
// Missing something? Narrow `paths`, raise `tokenBudget`, or use level=Conservative.
// --------------------------------------------------------------------

// Backend/TradingPlatform.Application/Services/Orders/OrderStatusSyncService.cs:1-305
...minimized code...
```

Read the header as: `original -> delivered (% smaller)` is the compression of the files actually delivered;
`Searched` is context looked at, **not** a saving; `withheld` chunks were dropped to fit the budget — raise
`tokenBudget` or narrow `paths` if the answer feels incomplete. On very small inputs the locator header can
make a file *larger*; that is reported honestly, not hidden.

---

## `search_code`

Semantic search over the local embedding index, blended with lexical ranking. Run `index_repo` first.

| Argument | Type | Default | Notes |
|---|---|---|---|
| `query` | string | — | Natural-language description of what you want. |
| `topK` | integer | `10` | Max chunks to return. |

```jsonc
// request
{ "query": "add two numbers", "topK": 2 }
```
```jsonc
// response — content[0].text
{ "count": 0, "results": [] }
```

An **empty** index returns `{count:0,results:[]}` cleanly (no error, no model download). If it returns nothing
right after a successful `index_repo`, see [Troubleshooting → search returns nothing after indexing](troubleshooting.md#index-built-but-search-empty).

---

## `index_repo`

Builds or refreshes the semantic index (chunks + local embeddings) that `search_code` uses. Downloads the
embedding model (~127 MB, once) on first run.

| Argument | Type | Default | Notes |
|---|---|---|---|
| `path` | string | — | Directory to index (repo-relative or absolute). |
| `force` | boolean | `false` | Re-embed every chunk even if unchanged. |

```jsonc
// request
{ "path": "src" }
```
```text
// response — content[0].text
index_repo: indexed C:\...\src; the index now holds 2 chunk(s).
```

A path that doesn't exist, or that holds no supported source files, returns **`isError: true`** with a clear
message rather than a silent success.

---

## `summarize_repo`

Walks a directory and returns namespaces, public type names, and public method signatures from its `.cs`
files (tree-sitter, with a regex fallback).

| Argument | Type | Default | Notes |
|---|---|---|---|
| `path` | string | — | Directory to summarize (repo-relative or absolute). |

```jsonc
// request
{ "path": "src" }
```
```text
// response — content[0].text
Repo summary: C:\...\src

## Calculator.cs
namespace Shop.Math
  type class Calculator
  method public double Add(double a, double b)
  method public double Subtract(double a, double b)

## Greeter.cs
namespace Shop.Ui
  type class Greeter
  method public string Greet(string name)
```

A not-found path returns **`isError: true`**.

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
{ "id": 1, "category": "convention" }
```

---

## `recall`

Recalls remembered facts (current branch + global) whose text matches a query, optionally filtered by category.

| Argument | Type | Default | Notes |
|---|---|---|---|
| `query` | string | — | Text to match against remembered facts. |
| `category` | string \| null | — | Optional category filter. |

```jsonc
// request
{ "query": "money", "category": null }
```
```jsonc
// response — content[0].text
{ "count": 1, "facts": [
  { "id": 1, "category": "convention",
    "text": "All money values use decimal, never double.",
    "source": "src/Calculator.cs", "createdAt": "2026-07-17T07:11:08+05:30" } ] }
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
{ "path": "C:\\...\\DECISIONS.md", "count": 0 }
```

`count` is the number of **decision-category** facts exported (a `convention` fact is not a decision, hence `0`
above). Remember facts with `category: "decision"` to populate it.

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
{ "calls": 2, "scannedTokens": 198, "selectedTokens": 198,
  "deliveredTokens": 226, "compressedTokens": -28, "compressionPct": -0.1414,
  "byTool": [
    { "tool": "compose_task_prompt", "calls": 1, "scannedTokens": 99, "selectedTokens": 99, "deliveredTokens": 113 },
    { "tool": "get_context", "calls": 1, "scannedTokens": 99, "selectedTokens": 99, "deliveredTokens": 113 } ] }
```

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
{ "task": "Add a Multiply method to the calculator", "paths": "src" }
```
```text
// response — the composed prompt (4 labelled sections)
# Task
Add a Multiply method to the calculator

# Relevant code (minimized)
// src/Calculator.cs:1-8
namespace Shop.Math;
public sealed class Calculator { ...minimized... }

# Project conventions
- All money values use decimal, never double.

# Constraints
- Work within the code shown above; do not invent new packages or patterns.
- Follow the recalled conventions exactly. If information is missing, say so rather than guessing.
```

See [Prompt composer](composer.md) for the full anatomy and the prompt-not-answer boundary.
