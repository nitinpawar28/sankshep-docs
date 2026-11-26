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

## 3. Confirm the tools are available — **select Agent, not Ask**

MCP tools only fire when the client is in an **agent/tool-using** mode. In VS Code, select **Agent** from
the agent picker in the chat input (not Ask). Then ask the model to *"list the sankshep tools"* — you
should see `get_context`, `search_code`, `index_repo`, `summarize_repo`, `remember`, `recall`,
`export_decisions`, `token_report`, and the `compose_task_prompt` prompt. If not, see
[Troubleshooting](troubleshooting.md).

## 4. Build the index (for semantic search)

`get_context` and `summarize_repo` work immediately — they read files directly. `search_code` needs an index:

```text
index_repo   (no arguments)     → index_repo: indexed N file(s) under <repo>; the index now holds M chunk(s) in total.
```

The first `index_repo` downloads the local embedding model (~127 MiB, once) — expect a short one-time
delay; fully offline afterwards. When your client sends a progress token, `index_repo` reports one
notification per file, so a first index of a large repository is distinguishable from a hang.

A path that matches nothing, holds no supported source, or names a file rather than a directory returns an
**error** — not a silent success. So does `search_code` against an index that has never been built: it says
the index is empty and tells you to run `index_repo`, rather than returning an empty result you might read
as "that code does not exist".

!!! note "Upgrading?"
    From 1.x or 2.0.0, your existing index is discarded and rebuilt once, on first start. See
    [Upgrading to 2.0.0](upgrading-2.0.md), then [Upgrading to 3.0.0](upgrading-3.0.md), then
    [Upgrading to 4.0.0](upgrading-4.0.md). From 3.0.0 to 4.0.0 nothing rebuilds.

## 5. Ask a grounded question

With **Agent** selected, let the model call `get_context` for you:

> Use `get_context` on `src/` to explain how orders are validated, and cite the files.

Or force it explicitly with the tool. A good first call:

```text
get_context   query: "how does X work"   paths: ["src"]   tokenBudget: 8000   level: "Balanced"
```

The result leads with a header — `original -> delivered (% smaller)`, what it searched, which files came back
only in part (`PARTIAL`) and which not at all (`WITHHELD`) — then the minimized code. A file that does not fit
the budget arrives as its most relevant declarations, each cited `// path:start-end Symbol`. The model
answers from that.

## 6. See the savings

```text
token_report   → { "calls": …, "selectedTokens": …, "deliveredTokens": …, "compressionPct": …, "byTool": [ … ] }
```

`compressionPct` is delivered vs. the original size of the source that came back — a whole file, or the
declarations delivered from a file that did not fit — a real, bounded number, no dollar estimate. To measure savings **and** answer-quality together on a named repo, see [Benchmarks](benchmarks.md).

## Next

- [Tool reference](tool-reference.md) — every tool's arguments + real captured I/O.
- [Prompt composer](composer.md) — turn a task + paths into a grounded prompt.
- [Troubleshooting](troubleshooting.md) — when something doesn't behave.
- [Security & privacy](security.md) — what stays on your machine, and where.
