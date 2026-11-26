# Tool reference

Every tool, prompt and resource Sankshep exposes over MCP, with its arguments and **real** request/response
pairs captured from the published **v5.0.0** package, driven over stdio. The captures were taken against the
three-file C# sample repository, plus three files added so that v5's text, manifest and secret handling has
something to show: `SampleRepo.csproj`, an `appsettings.json` holding an obviously fake password, and a
`README.md`. The repository's `.gitignore` lists `.sankshep/`, as [Security](security.md) recommends. Long
absolute paths are shortened to `C:\...\sample-repo`; nothing else is edited.

Upgrading from 4.x? Every tool was renamed in 5.0.0 and none of the old names is kept as an alias. See
[Upgrading to 5.0](upgrading-5.0.md) for the mapping.

| Tool | What it is for |
|---|---|
| [`outline`](#outline) | Orient: the projects, frameworks, packages and pipelines, then the declarations, each with its line |
| [`find`](#find) | Locate: exact text, a regex, a declaration by name, or code by meaning |
| [`read`](#read) | Read: minimized code, exact line ranges, or any text file, numbered by line |
| [`index`](#index) | Build the semantic index `find mode=semantic` searches |
| [`remember`](#remember) / [`recall`](#recall) | A per-repo, branch-scoped memory of decisions and conventions |
| [`report`](#report) | What each tool returned, and how far `read` compressed the code it delivered |

Paths are **relative to the repo the server was started against** (`serve --repo <root>`).

!!! warning "Absolute paths are not a general escape hatch"
    On **stdio**, an absolute path inside the served root is accepted; one outside it is refused. On the
    **HTTP transport** an absolute path is refused outright, whether or not it points inside the root, and
    `index` refuses one on **both** transports. Use repo-relative paths and they work everywhere. See
    [Security](security.md).

!!! tip "These fire in Agent mode, not Ask"
    MCP tools are only invoked when the client is in an agent/tool-using mode (e.g. Copilot **Agent** mode).
    In plain chat/Ask mode the model can't call them. See [Troubleshooting](troubleshooting.md).

**Every line is numbered.** `read` and `find` prefix each line with its source line number, `N|`, so an answer
can cite `path:N` without opening the file again. `find` marks context lines `N:` instead.

**The server tells the client how to use it.** Since 5.0.0, `initialize` returns instructions: prefer `find`
to grep, `read path:start-end` before an edit, cite from the `N|` prefixes, and act on what the header says was
not delivered. Clients that use server instructions, such as Claude Code, load them at session start.

---

## `outline`

What a directory holds. First an **inventory** of its projects and manifests: target frameworks, packages,
scripts, container images and CI jobs, from MSBuild projects, `package.json`, `angular.json`, solutions,
Dockerfiles, `go.mod` and CI pipelines, each item cited by line. Then the **declarations** in its source files,
in every language Sankshep parses, each with its line. Pipelines are read by a heuristic and are labelled so.

| Argument | Default | |
|---|---|---|
| `path` | `"."` | Directory to outline |
| `maxTokens` | `4000` | A ceiling on the **whole** answer, header included. Over it, the outline is cut at a line and its second line says so |
| `include` | `all` | `all`, `projects` (the inventory only) or `code` (the declarations only) |

```jsonc
// request (tools/call)
{"name":"outline","arguments":{}}
```
```text
// ---- Sankshep outline: . ----
// Projects and manifests (1), each item cited by line:
SampleRepo.csproj (msbuild): sdk Microsoft.NET.Sdk L1; framework net10.0 L3

Repo summary: C:\...\sample-repo

## Calculator.cs
namespace SampleRepo.Math
  type class Calculator  L6
  method public double Add(double a, double b)  L11
  method public double Subtract(double a, double b)  L17

## Greeter.cs
namespace SampleRepo.Text
  type class Greeter  L4
  method public string Greet(string name)  L6
  method public static string GreetFormal(string title, string name)  L11

## Inventory.cs
namespace SampleRepo.Stock
  type interface IInventory  L3
  type class Inventory  L11
  method public void Restock(string sku, int quantity)  L15
  method public int CountOf(string sku)  L20
```

A manifest that does not parse is named with its error, not dropped. Private members are left out of the
declarations, as `Calculator.Normalize` is here.

---

## `find`

Finds code and text, every hit cited as `path:N`.

| `mode` | What it searches | Needs `index`? |
|---|---|---|
| `semantic` (default) | Code by meaning, over the semantic index | yes |
| `exact` | A literal, line by line, in the working tree as it is now | no |
| `regex` | A .NET regular expression, one line at a time, in the working tree | no |
| `symbol` | Where a type or method is **declared** | no |

| Argument | Default | |
|---|---|---|
| `query` | required | A description, a literal, a pattern or a name, by mode |
| `mode` | `semantic` | See above |
| `paths` | whole repo | Limit `exact`, `regex` and `symbol` to these files or directories |
| `glob` | none | Limit `exact`, `regex` and `symbol` to matching paths, e.g. `**/*.cs` |
| `caseSensitive` | `true` | For `exact` and `regex` |
| `maxResults` | `20` | Up to 100 for `semantic`, 500 otherwise. (Was `topK` before 5.0.0) |
| `contextLines` | `0` | Lines of context around each `exact`, `regex` or `symbol` hit |
| `secrets` | `mask` | As for [`read`](#read) |

Use `exact` and `regex` where you would use grep: they read the working tree, so an edit made a second ago is
found, and every matching line is masked for secrets before it is returned. A regex runs on .NET's
non-backtracking engine when it can, and otherwise under a per-match timeout. A search stopped by a cap (time,
bytes read, `maxResults`, line length) says so in its header.

```jsonc
// request
{"name":"find","arguments":{"query":"public double","mode":"exact"}}
```
```text
// ---- Sankshep find (exact): "public double", 2 hits in 1 file(s) ----
// "N|" = the hit on source line N; "N:" = context. Cite as path:N; read path:start-end for more.
// Scanned 7 file(s), 2 KB of the working tree; engine: exact.

// Calculator.cs
11|    public double Add(double a, double b)
17|    public double Subtract(double a, double b)
```

```jsonc
// request
{"name":"find","arguments":{"query":"Add\\w*\\(","mode":"regex"}}
```
```text
// ---- Sankshep find (regex): "Add\w*\(", 1 hit in 1 file(s) ----
// "N|" = the hit on source line N; "N:" = context. Cite as path:N; read path:start-end for more.
// Scanned 7 file(s), 2 KB of the working tree; engine: non-backtracking.

// Calculator.cs
11|    public double Add(double a, double b)
```

`symbol` cites the declaration's own line and says what it is and which lines it spans:

```jsonc
// request
{"name":"find","arguments":{"query":"Inventory","mode":"symbol"}}
```
```text
// ---- Sankshep find (symbol): "Inventory", 1 hit in 1 file(s) ----
// "N|" = the hit on source line N; "N:" = context. Cite as path:N; read path:start-end for more.
// Declarations parsed in 1 candidate file(s) that mention the name.

// Inventory.cs
11|public sealed class Inventory : IInventory    // type, lines 11-24
```

`semantic` on an index that holds nothing is an **error**, never an empty success: no results would read as
"this code does not exist".

```jsonc
// request (before index has run)
{"name":"find","arguments":{"query":"greet a person by name"}}
```
```text
find: the index is empty - nothing has been indexed for this repo yet, or the index was purged. This is NOT evidence that the code does not exist. Run index with path "." first (relative paths resolve against the served --repo root, C:\...\sample-repo), then retry - or use mode=exact, regex or symbol, which need no index. Only index discovers new files; verify-on-read and SANKSHEP_WATCH refresh already-indexed files only.
```

After [`index`](#index), the same question finds the code by meaning, best first:

```jsonc
// request
{"name":"find","arguments":{"query":"greet a person by name","maxResults":2}}
```
```text
// ---- Sankshep find (semantic): 2 hits, best first ----
// Pass any path below to read for the whole minimized file, or path:start-end for the exact lines.

// Greeter.cs:4-15 (Type Greeter)
4|public class Greeter
5|{
6|    public string Greet(string name)
7|    {
8|        return $"Hello, {name}!";
9|    }
10|
11|    public static string GreetFormal(string title, string name)
12|    {
13|        return $"Good day, {title} {name}.";
14|    }
15|}

// Inventory.cs:3-6 (Type IInventory)
3|public interface IInventory
4|{
5|    int CountOf(string sku);
6|}
```

Before answering, a semantic search re-embeds at most 25 indexed files that changed since they were indexed, so
recent edits are searched as they are now.

---

## `read`

Reads code and text, numbered by source line. A path may be a file, a directory, or `path:start-end`. There are
three kinds of item, and the header says which you got:

- **Code, no range**: minimized. Comments are stripped, and method bodies unrelated to the `query` are elided
  as `{ /* L18-21 elided */ }`, citing the lines they came from. This is what the compression figure counts.
- **Code with a range** (`Calculator.cs:11-15`): those exact source lines, never minimized. Read the range
  before you edit it.
- **Any other text file**, such as Markdown, `.csproj`, `package.json`, YAML or `appsettings.json`: as written,
  with secrets masked.

| Argument | Default | |
|---|---|---|
| `paths` | required | Files, directories, or `path:start-end` ranges, repo-relative. Ranges and text files are delivered first; code fills what remains |
| `query` | none | What you are looking for. Ranks a directory's files, and decides which bodies Balanced keeps |
| `tokenBudget` | `6000` | A ceiling on the **whole** response, header included, counted with `o200k_base` |
| `level` | `Balanced` | `Conservative`, `Balanced` or `Aggressive` |
| `symbols` | none | Method or type names whose bodies are kept whole at every level |
| `secrets` | `mask` | `mask` or `reveal`. Over HTTP, `reveal` is refused unless the server sets `SANKSHEP_SECRETS=reveal` |
| `lineNumbers` | `true` | Prefix every line with `N|` |

**Balanced keeps the bodies the query is most about.** Bodies are scored against the query's words, rarer words
counting for more, and the top few per file are kept (at most twelve); the rest are elided. The header says
how many were kept and why.

```jsonc
// request
{"name":"read","arguments":{"paths":["Calculator.cs"],"query":"add two numbers"}}
```
```text
// ---- Sankshep read -------------------------------------------------
// Bodies: 1 of 3 kept (query-relevant 1, requested 0), 2 elided. Name a method in `symbols` to keep its body.
// minimized code: 141 original -> 126 delivered tokens (10.6% smaller).
// Missing something? Narrow `paths`, raise `tokenBudget`, name a range path:start-end, or use level=Conservative.
// "N|" = source line N. "{ /* L12-40 elided */ }" = a body elided at those lines; read path:12-40 for it.
// --------------------------------------------------------------------

// Calculator.cs:1-29
1|namespace SampleRepo.Math;
2|
6|public class Calculator
7|{
9|    private double _total;
10|
11|    public double Add(double a, double b)
12|    {
13|        _total = a + b;
14|        return _total;
15|    }
16|
17|    public double Subtract(double a, double b)
18|    { /* L18-21 elided */ }
22|
24|    private double Normalize(double value)
25|    { /* L25-27 elided */ }
28|}
```

The gaps in the numbering (3-5, 8, 23) are the comments that minimization removed. The numbers are always the
source file's own.

An exact range:

```jsonc
// request
{"name":"read","arguments":{"paths":["Calculator.cs:11-15"]}}
```
```text
// ---- Sankshep read -------------------------------------------------
// Exact lines and text files are delivered as written and are not counted as compression.
// Missing something? Narrow `paths`, raise `tokenBudget`, name a range path:start-end, or use level=Conservative.
// "N|" = source line N. "{ /* L12-40 elided */ }" = a body elided at those lines; read path:12-40 for it.
// --------------------------------------------------------------------

// Calculator.cs:11-15 exact
11|    public double Add(double a, double b)
12|    {
13|        _total = a + b;
14|        return _total;
15|    }
```

### Secrets

Values that look like secrets are **masked by default** in every text `read` and `find` return: values under
secret-named keys in JSON, YAML, INI, `.env` and XML configuration, the credential parts of a connection string
(the server and database stay visible), URI passwords, `Authorization` headers, private-key blocks, and
well-known vendor token shapes. The header lists every masked value by file, line and key, so an over-mask or
a miss is visible.

```jsonc
// request
{"name":"read","arguments":{"paths":["appsettings.json"]}}
```
```text
// ---- Sankshep read -------------------------------------------------
// SECRETS MASKED: appsettings.json L3 Default.Password - pass secrets="reveal" to see them
// Exact lines and text files are delivered as written and are not counted as compression.
// Missing something? Narrow `paths`, raise `tokenBudget`, name a range path:start-end, or use level=Conservative.
// "N|" = source line N. "{ /* L12-40 elided */ }" = a body elided at those lines; read path:12-40 for it.
// --------------------------------------------------------------------

// appsettings.json:1-6 text
1|{
2|  "ConnectionStrings": {
3|    "Default": "Server=db.internal;Database=shop;User Id=app;Password=[MASKED len=15]"
4|  },
5|  "Logging": { "LogLevel": { "Default": "Information" } }
6|}
```

When troubleshooting needs the value, ask for it:

```jsonc
// request
{"name":"read","arguments":{"paths":["appsettings.json"],"secrets":"reveal"}}
```
```text
// ---- Sankshep read -------------------------------------------------
// SECRETS REVEALED: appsettings.json L3 Default.Password (because secrets="reveal")
// Exact lines and text files are delivered as written and are not counted as compression.
// Missing something? Narrow `paths`, raise `tokenBudget`, name a range path:start-end, or use level=Conservative.
// "N|" = source line N. "{ /* L12-40 elided */ }" = a body elided at those lines; read path:12-40 for it.
// --------------------------------------------------------------------

// appsettings.json:1-6 text
1|{
2|  "ConnectionStrings": {
3|    "Default": "Server=db.internal;Database=shop;User Id=app;Password=example-only-pw"
4|  },
5|  "Logging": { "LogLevel": { "Default": "Information" } }
6|}
```

Set `SANKSHEP_SECRETS=reveal` on the server to reveal by default (see [Environment](environment.md)). Masking
never adds or removes a line, so line numbers stay true either way. The semantic index stores masked text only.

### What the header discloses

The header always says what did **not** come back, so a partial answer never looks complete: paths that matched
nothing, files that could not be read, files skipped, a file truncated to fit (the cut is marked, with the range
to continue from), files delivered in part by their declarations, and files that ranked but did not fit
(**WITHHELD**). A delivery with nothing in it is **`isError: true`**, never a silent empty success:

```jsonc
// request
{"name":"read","arguments":{"paths":["Nope.cs"]}}
```
```text
// ---- Sankshep read -------------------------------------------------
// WARNING: nothing matched: Nope.cs
// Relative `paths` resolve against the repo root given to `sankshep serve --repo`.
// NO CONTEXT was returned: nothing readable matched `paths`.
// Re-run with corrected `paths` or a larger `tokenBudget` -- do not answer from guesswork.
// "N|" = source line N. "{ /* L12-40 elided */ }" = a body elided at those lines; read path:12-40 for it.
// --------------------------------------------------------------------
```

A file too large for what is left of the budget is delivered **by its declarations**: its types and top-level
methods or functions, the pieces that match the query first, each under a locator naming its symbol and source
lines, and the file is listed as **PARTIAL**.

---

## `index`

Builds or refreshes the semantic index that `find mode=semantic` searches. Incremental: only new or changed
chunks are re-embedded, so a re-run over an unchanged tree is cheap. The first run downloads the ~127 MB
embedding model, pinned and checked by SHA-256 (see [Install](install.md) for offline use). Since 5.0.0 the
index covers Markdown and manifests as well as code, stores masked text only, and never indexes `.env*` files,
key and certificate files, or lockfiles.

| Argument | Default | |
|---|---|---|
| `path` | `"."` | Directory to index; a directory, not a file |
| `force` | `false` | Re-embed every chunk even if unchanged |

```jsonc
// request
{"name":"index","arguments":{}}
```
```text
index: indexed 5 file(s) under C:\...\sample-repo; the index now holds 6 chunk(s) in total.
```

A client that sends a `progressToken` gets one progress notification per file. Upgrading from 4.x rebuilds the
index once on the first start.

---

## `remember`

Persists a fact to this repository's memory, tagged with the current git branch.

| Argument | | |
|---|---|---|
| `category` | required | Stored trimmed and lowercased, at most 64 characters. Two labels are wired to features and must be spelled exactly: **`decision`** (what `recall format=markdown` exports) and **`convention`** (what the [`implement`](#implement) prompt injects) |
| `text` | required | At most 4,000 characters |
| `source` | optional | Provenance, e.g. a file path or URL |

```jsonc
// request
{"name":"remember","arguments":{"text":"Calculator methods never throw on overflow; they wrap.","category":"decision","source":"design review"}}
```
```json
{"id":1,"category":"decision","branch":"master"}
```

---

## `recall`

Recalls facts for the current branch plus the branch-less ones. A fact matches when **every** word of the query
appears in its text, in any order, ignoring case; an empty query returns every fact.

| Argument | Default | |
|---|---|---|
| `query` | required | Words that must all appear; `""` for all |
| `category` | none | Optional filter |
| `format` | `list` | `list`, or `markdown`: a `DECISIONS.md` document of the matching facts (category `decision` unless another is given) |

```jsonc
// request
{"name":"recall","arguments":{"query":"Calculator"}}
```
```json
{"count":1,"facts":[{"id":1,"category":"decision","text":"Calculator methods never throw on overflow; they wrap.","source":"design review","branch":"master","createdAt":"2026-11-26T16:53:31.1826612+00:00"}],"matched":1,"truncated":false}
```

`format=markdown` returns the document and writes nothing. It replaces 4.x's `export_decisions`, which wrote
the file into your repository; write it where you want it:

```jsonc
// request
{"name":"recall","arguments":{"query":"","format":"markdown"}}
```
```json
{"count":1,"facts":[],"matched":1,"truncated":false,"markdown":"# Decisions\r\n\r\n\u003C!-- Generated by sankshep (recall format=markdown). Regenerate rather than edit. --\u003E\r\n\r\n## Branch: master\r\n\r\n- Calculator methods never throw on overflow; they wrap. _(2026-11-26, source: design review)_\r\n\r\n"}
```

(This capture ran on Windows, so the document's line endings are `\r\n`.) With nothing to export, `markdown`
is an error that names the category it looked in.

---

## `report`

What Sankshep's tools cost and saved in this repository. Every tool and prompt call is recorded, one row per
call, with its outcome, time and the tokens it returned; no path or query is stored. **Compression is reported
apart from usage**, and only over the calls that minimize code (`read` and `implement`): searched tokens are
never a baseline, and no dollar figure is ever shown.

| Argument | Default | |
|---|---|---|
| `window` | `session` | `session`, `24h`, `7d` or `all` |
| `format` | `text` | `text`, or `json` for a program |

```jsonc
// request
{"name":"report","arguments":{}}
```
```text
// ---- Sankshep report: this session, 19 call(s), 2 error(s) ----
// tool                     calls  errors  returned tokens   avg ms
// explore                       1       0              224        0
// find                          5       1              597        8
// implement                     1       0              159       19
// index                         1       0               83      591
// outline                       1       0              264      133
// read                          5       1              879       50
// recall                        2       0              165        8
// remember                      2       0               27       11
// review                        1       0              195        0
// Returned in total: 2,593 tokens - what these calls added to the caller's context.
// Compression, over the calls that minimize code: 282 original -> 218 delivered (22.7% smaller).
```

An unreadable `stats.db` is an error from `report`, the `sankshep://stats` resource and `/api/stats` (503),
never a row of zeros. Totals recorded by 4.x are kept and shown on their own line, not added in.

The same report needs no MCP client: `sankshep report --repo . [--window session|24h|7d|all] [--json]` reads a
copy of `stats.db`, so nothing in the state directory changes.

---

## Prompts

Four MCP prompts. Clients that offer prompts show them as slash commands; Claude Code lists them as
`/sankshep:<name>`. VS Code's Copilot sessions do not expose MCP prompts, so there the same workflows come from
the agent files [`sankshep init`](agents.md) writes. A prompt returns a prompt, never an answer.

| Prompt | Arguments | Returns |
|---|---|---|
| [`implement`](#implement) | `task`, `paths`, `tokenBudget` (default 4000) | A grounded prompt: the task, minimized code, the recorded conventions, constraints |
| `explore` | `question` | The Explorer workflow, then the question |
| `architect` | `scope` (default: the whole repository) | The Architect workflow, then the scope |
| `review` | `base` (default `HEAD`) | The Reviewer workflow against that commit or branch |

### `implement`

Replaces 4.x's `compose_task_prompt`, with the same arguments. It reads the code through the same engine as
[`read`](#read), adds every fact remembered with category `convention`, and lists anything it could not
include: files withheld, cut, unreadable or skipped, named one by one, and conventions over their budget.
`paths` is comma- or newline-separated; `tokenBudget` bounds the code, and conventions are added on top. A
composition with no code in it is an error, never a prompt grounded in nothing. See
[Prompt composer](composer.md).

```jsonc
// request (prompts/get)
{"name":"implement","arguments":{"task":"add a Multiply method to the calculator","paths":"Calculator.cs"}}
```
```markdown
# Task
add a Multiply method to the calculator

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
    { /* L18-21 elided */ }

    private double Normalize(double value)
    { /* L25-27 elided */ }
}

# Project conventions
- Public arithmetic methods take and return double.

# Constraints
- Work within the code shown above; do not invent new packages or patterns.
- Follow the recalled conventions exactly. If information is missing, say so rather than guessing.
```

The composed code carries no `N|` prefixes; read the file with [`read`](#read) for citable line numbers.

### `explore`, `architect`, `review`

Each returns its role's workflow, which is the same text the agent files carry, followed by the argument:

```jsonc
// request (prompts/get)
{"name":"explore","arguments":{"question":"what does Inventory track?"}}
```
```markdown
You answer questions about this repository. You change nothing.

1. Orient with `outline` on the directory the question is about (path=. for the whole repository). Use include=projects when the question is about frameworks, packages, images or pipelines.
2. Locate with `find`: mode=symbol for a named type or method, mode=exact or mode=regex for an identifier, a string, an attribute or a configuration key, mode=semantic for a concept.
3. Read with `read`, passing the files `find` named and the question as query. Read path:start-end for the exact lines you will quote.
4. Answer with every claim cited as path:N, taken from the N| prefix of the lines you read. Say what you could not see: anything the header lists as WITHHELD, PARTIAL, TRUNCATED or not matched.

Read nothing through a terminal or the host's own file tools: `read` covers code, Markdown, configuration and manifests, numbered by line. Secrets stay masked unless the question is about their values.

# Question
what does Inventory track?
```

`review` refuses a `base` that is not shaped like a git revision (`HEAD`, `main`, `HEAD~3`), because the
workflow asks the agent to run `git diff <base>` in a terminal.

---

## Resources

### `sankshep://stats`

Token accounting for this repo as `application/json`: calls, errors and returned tokens, and the compression
claim over the calls that minimize code, with a per-tool breakdown. It is the feed the dashboard serves, and is
registered on both stdio and HTTP. Captured with `resources/read` after the calls above:

```jsonc
// request (resources/read)
{"uri":"sankshep://stats"}
```
```jsonc
// response - contents[0], mimeType application/json
{"calls":21,"scannedTokens":282,"selectedTokens":282,"deliveredTokens":218,"compressedTokens":64,"compressionPct":0.22695035460992907,"byTool":[{"tool":"explore","calls":1,"scannedTokens":0,"selectedTokens":0,"deliveredTokens":0},{"tool":"find","calls":5,"scannedTokens":0,"selectedTokens":0,"deliveredTokens":0},{"tool":"implement","calls":1,"scannedTokens":141,"selectedTokens":141,"deliveredTokens":92},{"tool":"index","calls":1,"scannedTokens":0,"selectedTokens":0,"deliveredTokens":0},{"tool":"outline","calls":1,"scannedTokens":0,"selectedTokens":0,"deliveredTokens":0},{"tool":"read","calls":5,"scannedTokens":141,"selectedTokens":141,"deliveredTokens":126},{"tool":"recall","calls":2,"scannedTokens":0,"selectedTokens":0,"deliveredTokens":0},{"tool":"remember","calls":2,"scannedTokens":0,"selectedTokens":0,"deliveredTokens":0},{"tool":"report","calls":2,"scannedTokens":0,"selectedTokens":0,"deliveredTokens":0},{"tool":"review","calls":1,"scannedTokens":0,"selectedTokens":0,"deliveredTokens":0}],"errors":2,"returnedTokens":3590}
```

## Annotations

Every tool declares MCP annotations. `outline`, `read`, `recall` and `report` are read-only. `find` is not,
because a semantic search may re-embed changed files first. No tool is destructive: since 5.0.0 nothing
overwrites a file in your repository.
