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

The first `index_repo` / `search_code` downloads the embedding model (~127 MiB, once). If the machine is
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
it. (In earlier versions these tools resolved against the server's own working directory and could silently
target the wrong repo; they now anchor to `--repo` and fail loudly.)

**Do not reach for an absolute path — in 2.0.0 it usually will not help.** A relative value that climbs out
of `--repo` via `..` is refused on both transports, and an absolute path is accepted only by the **stdio**
transport and only when it points **inside** the served root. The **HTTP** transport refuses every absolute
path, and `index_repo` refuses one on both transports. To work with a tree outside the repository, serve a
root that contains it.

## `index_repo` says it indexed chunks, but `search_code` returns nothing {#index-built-but-search-empty}

**Before v2.0.0 this was not rare and re-indexing did not fix it.** Indexing a *subdirectory* keyed
every chunk relative to that directory rather than to the served repo root, so nothing could resolve the
keys afterwards — and `index_repo --force` rewrote exactly the same unusable keys. The only thing that
worked was indexing from the repo root. That is fixed: keys are always relative to `--repo`, and an index
written by an older build is pruned and rebuilt on first use.

On v2.0.0 and later, if `index_repo` reports chunks and `search_code` returns nothing:

- **Re-run `index_repo`.** Unless `SANKSHEP_WATCH=1` is set, only `index_repo` discovers new files —
  verify-on-read refreshes files that are *already* indexed, so a file added since the last index is
  invisible to search until you run it again. This is the common cause. (With the watcher on, a new file
  is picked up once the burst of changes that created it settles.)
- Confirm the path you indexed actually contains supported source — any language on the
  [Supported languages](usage.md#supported-languages) table, plus `.docx`/`.pdf` documents.
- If the header carries `// NOTE: N indexed file(s) changed on disk and were not re-indexed before this
  search …`, run `index_repo`. Since 4.0.0 a search re-embeds at most 25 changed files before answering, so
  after a large change (a branch switch, say) some hits may be stale until you do.
- Delete the repo's `.sankshep/index.db` (and its `-wal`/`-shm` siblings) and re-index. The index is
  derived data, so this is always safe; it is also what an older index does for itself on first use. Since
  4.0.0 an `index.db` that cannot be opened at all — a directory of that name, a read-only or locked file —
  no longer stops the server: `index_repo` and `search_code` return an error naming the file and what to
  check, and the other tools keep working.

`get_context` and `summarize_repo` are unaffected — they read files directly and never depend on the index.

## `search_code` returns nothing on a fresh repo

Expected until you run `index_repo` — and since 2.0.0 the tool **says so instead of returning an empty
success**. It is an error whose message states that the index is empty, that this is not evidence the code is
absent, and which command to run:

```text
search_code: the index is empty - nothing has been indexed for this repo yet, or the index was purged.
This is NOT evidence that the code does not exist. Run index_repo with path "." first ...
```

The old `{count:0,results:[]}` was indistinguishable from a genuine no-match, which is how a model came to
answer "that code does not exist" about code that did. No model download happens either way.

## `token_report` shows a negative `compressionPct`

Honest, not a bug: on very small inputs the per-file locator headers Sankshep adds can cost more tokens than
minimization removes, so "compression" goes negative. On real files it is comfortably positive. There is no
dollar figure by design — see [Benchmarks](benchmarks.md).

## HTTP mode: connection refused / port already in use

`sankshep --http` binds `http://127.0.0.1:8080` by default (loopback, a DNS-rebinding defense). If the port is
taken, set `ASPNETCORE_URLS` (e.g. `http://127.0.0.1:9000`). To accept non-loopback hosts you must also enable
authentication or opt out explicitly — see the next entry — and, since 4.0.0, set `SANKSHEP_ALLOWED_HOSTS`
to the host names clients use; on a non-loopback bind that allow-list is a start gate of its own (see the
entries below), though it never satisfies the authentication one. Health probes: `GET /health/ready` should
return `200` once warm.
See [Install → HTTP clients](install.md#connect-over-http-remote-clients) and [Deployment](deployment.md).

## HTTP mode: `Refusing to start ... NO authentication on a non-loopback address`

A non-loopback bind (`ASPNETCORE_URLS=http://0.0.0.0:8080` — the default in the container image and Helm
chart) with no authentication **fails closed** rather than serve MCP tools, `/dashboard` and `/api/stats` to
the network unauthenticated.

Since 2.0.0 the refusal is **one line on stderr and exit code 1**. It used to surface as an unhandled
`InvalidOperationException`: a .NET stack trace and exit code `0xE0434352`, which reads like a crash to a
service manager and to anyone watching a container restart. Anything that parsed the old exit code needs
updating. Resolve it one of three ways:

- **Enable authentication** — set `SANKSHEP_API_KEYS` (clients then send `Authorization: Bearer <key>`; since
  4.0.0 each key must be at least 32 characters, `openssl rand -hex 32`) or both `SANKSHEP_OAUTH_AUTHORITY`
  and `SANKSHEP_OAUTH_AUDIENCE`.
- **Bind loopback** — unset `ASPNETCORE_URLS` so it returns to `127.0.0.1:8080`.
- **Trusted, network-isolated host only** — set `SANKSHEP_ALLOW_UNAUTHENTICATED=1` to acknowledge the exposure.

`SANKSHEP_ALLOWED_HOSTS` restricts the Host header but does **not** satisfy this gate. On the container and
Helm defaults (`0.0.0.0`) you also need it set, or the next refusal follows — the Helm chart sets it for you.

## HTTP mode: `Refusing to start: the server binds a non-loopback address and SANKSHEP_ALLOWED_HOSTS is not set`

New in 4.0.0. A routable bind with the allow-list unset, blank, or naming no host (`,`) refuses to start,
because with it empty any `Host` header was accepted and the DNS-rebinding defence was off. One line on
stderr, exit code 1, captured from 4.0.0:

```text
Refusing to start: the server binds a non-loopback address and SANKSHEP_ALLOWED_HOSTS is not set, so any Host header would be accepted and the DNS-rebinding defence would be off. Set it to the host names clients use to reach this server (comma-separated, e.g. "sankshep.example.com,sankshep"), or to "*" to accept any Host deliberately.
```

Set it to the names clients send (a port is ignored), or to `*` alone. Loopback binds keep their built-in
list and are unaffected.

## HTTP mode: `Refusing to start: SANKSHEP_ALLOWED_HOSTS combines "*" with host names`

New in 4.0.0, and checked on every bind, loopback included:

```text
Refusing to start: SANKSHEP_ALLOWED_HOSTS combines "*" with host names. "*" accepts any Host, so the names beside it would be ignored. Set either "*" alone or the list of names alone.
```

## HTTP mode: `Refusing to start: N of M API key(s) in SANKSHEP_API_KEYS is shorter than 32 characters`

New in 4.0.0, and checked on every `--http` start, loopback included. The message names the variable the
keys came from (`SANKSHEP_API_KEY` when you used the singular alias) and how many are short; it never prints
a key:

```text
Refusing to start: 1 of 1 API key(s) in SANKSHEP_API_KEYS is shorter than 32 characters. A key must be at least 32 characters - generate one with `openssl rand -hex 32`.
```

Generate a new key, set it, and re-issue it to your clients.

## HTTP mode: `Refusing to start: SANKSHEP_OAUTH_AUTHORITY is set but SANKSHEP_OAUTH_AUDIENCE is not`

New in 4.0.0, on any `--http` start. One OAuth variable alone used to fall back silently to API-key mode or
to no authentication; it is now refused, naming the missing one:

```text
Refusing to start: SANKSHEP_OAUTH_AUTHORITY is set but SANKSHEP_OAUTH_AUDIENCE is not. OAuth needs both SANKSHEP_OAUTH_AUTHORITY and SANKSHEP_OAUTH_AUDIENCE; with one of them the server would fall back to API-key mode or to no authentication at all. Set the other, or unset both.
```

## HTTP mode: every request gets `403 Forbidden: Host '…' is not allowed`

The `Host` and `Origin` headers are validated on every non-health request, and since 4.0.0 every routable
deployment has an allow-list, so a hostname the list does not name — an Ingress hostname, a Prometheus
scrape target, a client on another machine — is refused with a JSON-RPC error that names the variable to
set. Captured from 4.0.0 on a loopback bind, with a forged `Host`:

```json
{"jsonrpc":"2.0","error":{"code":-32600,"message":"Forbidden: Host 'evil.example' is not allowed. Set SANKSHEP_ALLOWED_HOSTS to permit it."},"id":null}
```

The `Origin` variant reads `Forbidden: Origin '…' is not allowed. Set SANKSHEP_ALLOWED_ORIGINS to permit it.`
Health probes are exempt: `/health/live` and `/health/ready` answer whatever the headers say, so a 403 here
never takes a pod down.

## `search_code` or `remember` refuses an argument

New in 4.0.0, both as `isError` results. `topK` above 100 and an oversized fact are refused before anything
is searched or stored; the [tool reference](tool-reference.md) has the captured texts:

```text
search_code: topK must be at most 100 (got 101). For more of the repository than that, call get_context with the paths these hits name, or summarize_repo. (Parameter 'topK')
```

`remember` refuses `text` over 4,000 characters, `category` over 64 and `source` over 2,048, naming the
argument and the bound. Facts already stored are untouched.

## Memory (`remember`/`recall`) seems empty across branches

Facts are tagged with the current git branch; `recall` returns **current-branch + global** facts, not facts
from other branches. On a non-git directory, everything is treated as global. `export_decisions` writes only
`decision`-category facts.
