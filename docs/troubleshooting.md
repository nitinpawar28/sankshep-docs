# Troubleshooting

Common failure modes and how to resolve them. If something here doesn't cover your case, reach the maintainer
via [Feedback](feedback.md).

## The sankshep tools don't appear in the chat

MCP tools only fire when the client is in an **agent/tool-using mode**. In VS Code Copilot that's **Agent**
mode, not **Ask**. Also check:

- The MCP server config is loaded (VS Code: reload the window after editing `mcp.json`).
- `sankshep --version` runs from the same environment the client launches it in.
- The client's MCP/log panel shows the server started and `tools/list` returned 8 tools + 1 prompt.

## First run is slow, or fails to start offline

The first `index_repo` / `search_code` downloads the embedding model (~127 MB, once). If the machine is
air-gapped, that download fails. Side-load the model and set `SANKSHEP_MODEL_DIR` to its folder, and
`SANKSHEP_MODEL_OFFLINE=1` to skip the download attempt entirely. See [Deployment → air-gap](deployment.md).

## "Path not found" from `get_context` / `summarize_repo` / `index_repo`

Relative paths resolve against the **served `--repo` root**, not your shell's working directory. If you see
`path not found: <p> (resolved to <root>/<p>)`, the path is wrong relative to `--repo` — correct it, or pass an
absolute path. (Before v1.4.1 these tools resolved against the server's own working directory and could
silently target the wrong repo; they now anchor to `--repo` and fail loudly.)

## `index_repo` says it indexed chunks, but `search_code` returns nothing {#index-built-but-search-empty}

A known issue under investigation: in some cases `index_repo` reports a non-zero chunk count while a following
`search_code` returns `{count:0,results:[]}`. Workarounds:

- Re-run `index_repo` with `force: true`.
- Delete the repo's `.sankshep/index.db` (and `-wal`/`-shm` siblings) and re-index from scratch.
- Confirm the path you indexed actually contains supported source (`.cs/.js/.mjs/.cjs/.jsx/.ts/.tsx/.py`).

`get_context` and `summarize_repo` are unaffected — they read files directly and never depend on the index.

## `search_code` returns nothing on a fresh repo

Expected until you run `index_repo`. An empty index returns `{count:0,results:[]}` cleanly (no error, no model
download). Build the index first.

## `token_report` shows a negative `compressionPct`

Honest, not a bug: on very small inputs the per-file locator headers Sankshep adds can cost more tokens than
minimization removes, so "compression" goes negative. On real files it is comfortably positive. There is no
dollar figure by design — see [Benchmarks](benchmarks.md).

## HTTP mode: connection refused / port already in use

`sankshep --http` binds `http://127.0.0.1:8080` by default (loopback, a DNS-rebinding defense). If the port is
taken, set `ASPNETCORE_URLS` (e.g. `http://127.0.0.1:9000`). To accept non-loopback hosts you must also set
`SANKSHEP_ALLOWED_HOSTS`. Health probes: `GET /health/ready` should return `200` once warm. See
[Install → HTTP clients](install.md) and [Deployment](deployment.md).

## Memory (`remember`/`recall`) seems empty across branches

Facts are tagged with the current git branch; `recall` returns **current-branch + global** facts, not facts
from other branches. On a non-git directory, everything is treated as global. `export_decisions` writes only
`decision`-category facts.
