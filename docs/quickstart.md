# Quickstart

Install → connect → index → first grounded answer → see the savings. About five minutes.

## 1. Install

```bash
dotnet tool install -g sankshep
sankshep --version
```

For containers, Windows Service, systemd, air-gap, see [Deployment](deployment.md).

## 2. Connect your MCP client

Point your client at the `sankshep` command in **stdio** mode. Full per-client configs are on
[Install](install.md); the shape VS Code now prefers is a `.mcp.json` at the workspace root, which Claude
Code reads too:

```jsonc
// .mcp.json  (at the workspace root)
{ "mcpServers": { "sankshep": { "command": "sankshep", "args": ["serve", "--repo", "."] } } }
```

`--repo` is the root every relative path resolves against; VS Code starts the server in the workspace
folder, so `.` is the repository you opened. The older `.vscode/mcp.json` form, keyed `servers`, still
works — see [Install](install.md). Reload the window after editing.

!!! tip "Or let `sankshep init` write it"
    Run `sankshep init` from the repository root: it writes the `.mcp.json` registration and four agents
    (Explorer, Architect, Implementer, Reviewer) for VS Code Copilot, Claude Code and Cursor, and never
    overwrites a file you edited. `--dry-run` shows what it would write. See [Agents](agents.md).

## 3. Confirm the tools are available — **select Agent, not Ask**

MCP tools only fire when the client is in an **agent/tool-using** mode. In VS Code, select **Agent** from
the agent picker in the chat input (not Ask). Then ask the model to *"list the sankshep tools"* — you
should see `read`, `find`, `outline`, `index`, `remember`, `recall` and `report`, and the prompts `explore`,
`architect`, `review` and `implement`. If not, see [Troubleshooting](troubleshooting.md).

## 4. Build the index (for semantic search)

`read` and `outline` work immediately — they read files directly — and so do `find`'s `exact`, `regex` and
`symbol` modes, which scan the working tree. `find`'s default `semantic` mode needs an index:

```text
index   (no arguments)     → index: indexed N file(s) under <repo>; the index now holds M chunk(s) in total.
```

The first `index` downloads the local embedding model (~127 MiB, once) — expect a short one-time
delay; fully offline afterwards. When your client sends a progress token, `index` reports one
notification per file, so a first index of a large repository is distinguishable from a hang.

A path that matches nothing, holds no supported source, or names a file rather than a directory returns an
**error** — not a silent success. So does a semantic `find` against an index that has never been built: it
says the index is empty and tells you to run `index` (or to use `mode=exact`, `regex` or `symbol`), rather
than returning an empty result you might read as "that code does not exist".

!!! note "Upgrading?"
    **From 4.x, every tool is renamed in 5.0.0 and none of the old names is kept as an alias**, and
    `index.db` is rebuilt once on the first start, so allow for one full re-index. See
    [Upgrading to 5.0.0](upgrading-5.0.md). From older releases, read [Upgrading to 2.0.0](upgrading-2.0.md),
    [Upgrading to 3.0.0](upgrading-3.0.md) and [Upgrading to 4.0.0](upgrading-4.0.md) first.

## 5. Ask a grounded question

With **Agent** selected, let the model call `read` for you:

> Use `read` on `src/` to explain how orders are validated, and cite `path:N` for every claim.

Or force it explicitly with the tool. A good first call:

```text
read   query: "how does X work"   paths: ["src"]   tokenBudget: 8000   level: "Balanced"
```

The result leads with a header — `original -> delivered (% smaller)`, which files came back only in part
(`PARTIAL`) and which not at all (`WITHHELD`) — then the minimized code, every line prefixed `N|` with its
source line number, so the model cites `path:N` without counting. A file that does not fit the budget arrives
as its most relevant declarations, each cited `// path:start-end Symbol`. The model answers from that.

To see exact lines before an edit, name a range. Captured from 5.0.0 on a small sample repository,
`read` with `paths: ["Calculator.cs:11-15"]`:

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

## 6. See the savings

`report` lists every tool and prompt call this session. Captured from 5.0.0 after a short session on the
same sample repository:

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

The compression line is delivered vs. the original size of the source that came back — a real, bounded
number, no dollar estimate. Pass `window` (`session`, `24h`, `7d` or `all`) for a longer view and
`format=json` for a program; `sankshep report --repo .` prints the report from a terminal, with no MCP
client. The terminal command defaults to `--window all`, and its `session` means the most recent session
recorded. To measure savings **and** answer-quality together on a named repo, see [Benchmarks](benchmarks.md).

## Next

- [Tool reference](tool-reference.md) — every tool's arguments + real captured I/O.
- [Agents](agents.md) — what `sankshep init` writes, per client.
- [Prompt composer](composer.md) — turn a task + paths into a grounded prompt.
- [Troubleshooting](troubleshooting.md) — when something doesn't behave.
- [Security & privacy](security.md) — what stays on your machine, and where.
