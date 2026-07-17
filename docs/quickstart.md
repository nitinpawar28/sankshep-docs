# Quickstart

Install → connect → index → first grounded answer → see the savings. About five minutes.

## 1. Install

```bash
dotnet tool install -g sankshep
sankshep --version
```

First run downloads the local embedding model (~127 MB, once). Fully offline afterwards. For containers,
Windows Service, systemd, air-gap, see [Deployment](deployment.md).

## 2. Connect your MCP client

Point your client at the `sankshep` command in **stdio** mode. Full per-client configs are on
[Install](install.md); the VS Code shape:

```jsonc
// .vscode/mcp.json  (or your user mcp.json)
{ "servers": { "sankshep": { "command": "sankshep", "args": ["serve", "--repo", "${workspaceFolder}"] } } }
```

`--repo` is the root every relative path resolves against. Reload the window after editing.

## 3. Confirm the tools are available — **Agent mode, not Ask**

MCP tools only fire when the client is in an **agent/tool-using** mode. In VS Code that's Copilot Chat's
**Agent** mode (not "Ask"). Ask the model to *"list the sankshep tools"* — you should see `get_context`,
`search_code`, `index_repo`, `summarize_repo`, `remember`, `recall`, `export_decisions`, `token_report`, and
the `compose_task_prompt` prompt. If not, see [Troubleshooting](troubleshooting.md).

## 4. Build the index (for semantic search)

`get_context` and `summarize_repo` work immediately — they read files directly. `search_code` needs an index:

```text
index_repo   path: "."          → index_repo: indexed <repo>; the index now holds N chunk(s).
```

A path that matches nothing, or holds no supported source, returns an **error** — not a silent success.

## 5. Ask a grounded question

In Agent mode, let the model call `get_context` for you:

> Use `get_context` on `src/` to explain how orders are validated, and cite the files.

Or force it explicitly with the tool. A good first call:

```text
get_context   query: "how does X work"   paths: ["src"]   tokenBudget: 8000   level: "Balanced"
```

The result leads with a header — `original -> delivered (% smaller)`, what it searched, what it withheld —
then the minimized code. The model answers from that.

## 6. See the savings

```text
token_report   → { "calls": …, "selectedTokens": …, "deliveredTokens": …, "compressionPct": …, "byTool": [ … ] }
```

`compressionPct` is delivered vs. the original size of the files delivered — a real, bounded number, no dollar
estimate. To measure savings **and** answer-quality together on a named repo, see [Benchmarks](benchmarks.md).

## Next

- [Tool reference](tool-reference.md) — every tool's arguments + real captured I/O.
- [Prompt composer](composer.md) — turn a task + paths into a grounded prompt.
- [Troubleshooting](troubleshooting.md) — when something doesn't behave.
- [Security & privacy](security.md) — what stays on your machine, and where.
