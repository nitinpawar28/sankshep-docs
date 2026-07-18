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
`SANKSHEP_MODEL_OFFLINE=1` to skip the download attempt entirely. Each model file is checksum-verified against
the manifest: a corrupt or wrong side-loaded `model.onnx` / `vocab.txt` is treated as *missing or invalid* and,
offline, fails closed — re-fetch the exact files the manifest names. See
[Deployment → air-gap](deployment.md#air-gapped-zero-egress).

## "Path not found" / "nothing matched" from `get_context` / `summarize_repo` / `index_repo`

Relative paths resolve against the **served `--repo` root**, not your shell's working directory. `index_repo`
and `summarize_repo` report a bad path as `path not found: <p> (resolved to <root>/<p>)`; `get_context` instead
writes `// WARNING: nothing matched: <p>` into its header, and when nothing at all matched returns
`// NO CONTEXT was returned: ...` as an error. In each case the path is wrong relative to `--repo` — correct
it, or pass an absolute path. (In earlier versions these tools resolved against the server's own working
directory and could silently target the wrong repo; they now anchor to `--repo` and fail loudly.)

To reach a file **outside** the served repo, pass an **absolute** path: a relative `paths` value that climbs
out of `--repo` via `..` is rejected as unmatched rather than read (`get_context`).

## `index_repo` says it indexed chunks, but `search_code` returns nothing {#index-built-but-search-empty}

In rare cases `index_repo` reports a non-zero chunk count while a following `search_code` returns
`{count:0,results:[]}`. Recover with:

- Re-run `index_repo` with `force: true`.
- Delete the repo's `.sankshep/index.db` (and `-wal`/`-shm` siblings) and re-index from scratch.
- Confirm the path you indexed actually contains supported source — any language on the
  [Supported languages](usage.md#supported-languages) table, plus `.docx`/`.pdf` documents.

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
taken, set `ASPNETCORE_URLS` (e.g. `http://127.0.0.1:9000`). To accept non-loopback hosts you must also enable
authentication or opt out explicitly — see the next entry; `SANKSHEP_ALLOWED_HOSTS` is an additional
Host-header allow-list, not the start gate. Health probes: `GET /health/ready` should return `200` once warm.
See [Install → HTTP clients](install.md#connect-over-http-remote-clients) and [Deployment](deployment.md).

## HTTP mode: `Refusing to start ... NO authentication on a non-loopback address`

As of v1.8.0, a non-loopback bind (`ASPNETCORE_URLS=http://0.0.0.0:8080` — the default in the container image
and Helm chart) with no authentication **fails closed**: the server throws `InvalidOperationException` and
exits rather than serve MCP tools — and the `/dashboard` + `/api/stats` surfaces — with no authentication to the
network. Resolve it one of three ways:

- **Enable authentication** — set `SANKSHEP_API_KEYS` (clients then send `Authorization: Bearer <key>`) or
  `SANKSHEP_OAUTH_*`.
- **Bind loopback** — unset `ASPNETCORE_URLS` so it returns to `127.0.0.1:8080`.
- **Trusted, network-isolated host only** — set `SANKSHEP_ALLOW_UNAUTHENTICATED=1` to acknowledge the exposure.

`SANKSHEP_ALLOWED_HOSTS` restricts the Host header but does **not** satisfy this gate.

## Memory (`remember`/`recall`) seems empty across branches

Facts are tagged with the current git branch; `recall` returns **current-branch + global** facts, not facts
from other branches. On a non-git directory, everything is treated as global. `export_decisions` writes only
`decision`-category facts.
