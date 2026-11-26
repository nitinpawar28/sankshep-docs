# Troubleshooting

Common failure modes and how to resolve them. If something here doesn't cover your case, reach the maintainer
via [Feedback](feedback.md).

## The sankshep tools don't appear in the chat

MCP tools only fire when the client is in an **agent/tool-using mode**. In VS Code Copilot that's **Agent**
mode, not **Ask**. Also check:

- The MCP server config is loaded (VS Code: reload the window after editing `mcp.json`).
- `sankshep --version` runs from the same environment the client launches it in.
- The client's MCP/log panel shows the server started, `tools/list` returned 7 tools and `prompts/list`
  returned 4 prompts.
- **Upgraded from 4.x?** 5.0.0 renamed every tool and kept no aliases, so a client, prompt file or agent
  that still calls `get_context`, `search_code` or another 4.x name gets an error. Use the new names — see
  [Upgrading to 5.0.0](upgrading-5.0.md) — or run `sankshep init` to write current [agent files](agents.md).

## First run is slow, or fails to start offline

The first `index` / semantic `find` downloads the embedding model (~127 MiB, once). If the machine is
air-gapped, that download fails. Side-load the model and set `SANKSHEP_MODEL_DIR` to its folder, and
`SANKSHEP_MODEL_OFFLINE=1` to skip the download attempt entirely. Each model file is checksum-verified against
the manifest: a corrupt or wrong side-loaded `model.onnx` / `vocab.txt` is treated as *missing or invalid* and,
offline, fails closed — re-fetch the exact files the manifest names. See
[Deployment → air-gap](deployment.md#air-gapped-zero-egress).

## "Path not found" / "nothing matched" from `read` / `outline` / `index`

Relative paths resolve against the **served `--repo` root**, not your shell's working directory. `index`
and `outline` report a bad path as `path not found: <p> (resolved to <root>/<p>)`; `read` instead
writes `// WARNING: nothing matched: <p>` into its header, and when nothing at all matched returns
`// NO CONTEXT was returned: ...` as an error. Captured from 5.0.0:

```text
// ---- Sankshep read -------------------------------------------------
// WARNING: nothing matched: Nope.cs
// Relative `paths` resolve against the repo root given to `sankshep serve --repo`.
// NO CONTEXT was returned: nothing readable matched `paths`.
// Re-run with corrected `paths` or a larger `tokenBudget` -- do not answer from guesswork.
// "N|" = source line N. "{ /* L12-40 elided */ }" = a body elided at those lines; read path:12-40 for it.
// --------------------------------------------------------------------
```
 In each case the path is wrong relative to `--repo` — correct
it. (In earlier versions these tools resolved against the server's own working directory and could silently
target the wrong repo; they now anchor to `--repo` and fail loudly.)

**Do not reach for an absolute path — in 2.0.0 it usually will not help.** A relative value that climbs out
of `--repo` via `..` is refused on both transports, and an absolute path is accepted only by the **stdio**
transport and only when it points **inside** the served root. The **HTTP** transport refuses every absolute
path, and `index` refuses one on both transports. To work with a tree outside the repository, serve a
root that contains it.

## `index` says it indexed chunks, but a semantic `find` returns nothing {#index-built-but-search-empty}

**Before v2.0.0 this was not rare and re-indexing did not fix it.** Indexing a *subdirectory* keyed
every chunk relative to that directory rather than to the served repo root, so nothing could resolve the
keys afterwards — and a forced re-index rewrote exactly the same unusable keys. The only thing that
worked was indexing from the repo root. That is fixed: keys are always relative to `--repo`, and an index
written by an older build is pruned and rebuilt on first use.

On v2.0.0 and later, if `index` reports chunks and a semantic `find` returns nothing:

- **Re-run `index`.** Unless `SANKSHEP_WATCH=1` is set, only `index` discovers new files —
  verify-on-read refreshes files that are *already* indexed, so a file added since the last index is
  invisible to search until you run it again. This is the common cause. (With the watcher on, a new file
  is picked up once the burst of changes that created it settles.)
- Confirm the path you indexed actually contains supported source — any language on the
  [Supported languages](usage.md#supported-languages) table, plus `.docx`/`.pdf` documents.
- If the header carries `// NOTE: N indexed file(s) changed on disk and were not re-indexed before this
  search …`, run `index`. Since 4.0.0 a search re-embeds at most 25 changed files before answering, so
  after a large change (a branch switch, say) some hits may be stale until you do.
- Delete the repo's `.sankshep/index.db` (and its `-wal`/`-shm` siblings) and re-index. The index is
  derived data, so this is always safe; it is also what an older index does for itself on first use. Since
  4.0.0 an `index.db` that cannot be opened at all — a directory of that name, a read-only or locked file —
  no longer stops the server: `index` and semantic `find` return an error naming the file and what to
  check, and the other tools keep working.

`read` and `outline` are unaffected — they read files directly and never depend on the index — and so are
`find`'s `exact`, `regex` and `symbol` modes, which scan the working tree.

## A semantic `find` returns nothing on a fresh repo

Expected until you run `index` — and since 2.0.0 the tool **says so instead of returning an empty
success**. It is an error whose message states that the index is empty, that this is not evidence the code is
absent, and what to do instead. Captured from 5.0.0 (`C:\...\sample-repo` stands for the served root):

```text
find: the index is empty - nothing has been indexed for this repo yet, or the index was purged. This is NOT evidence that the code does not exist. Run index with path "." first (relative paths resolve against the served --repo root, C:\...\sample-repo), then retry - or use mode=exact, regex or symbol, which need no index. Only index discovers new files; verify-on-read and SANKSHEP_WATCH refresh already-indexed files only.
```

The old `{count:0,results:[]}` was indistinguishable from a genuine no-match, which is how a model came to
answer "that code does not exist" about code that did. No model download happens either way.

## `report` shows negative compression

Honest, not a bug: on very small inputs the per-file locator headers Sankshep adds can cost more tokens than
minimization removes, so "compression" (`compressionPct` under `format=json`) goes negative. On real files it is comfortably positive. There is no
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

## `find` or `remember` refuses an argument

Both as `isError` results, before anything is searched or stored. `find` refuses a `maxResults` above 100
for `mode=semantic` or above 500 for the other modes (it was `topK`, at most 100, before 5.0.0), and points
you at `read` with the paths the hits name, or `outline`, for more of the repository than that.

`remember` refuses `text` over 4,000 characters, `category` over 64 and `source` over 2,048, naming the
argument and the bound. Facts already stored are untouched.

## Memory (`remember`/`recall`) seems empty across branches

Facts are tagged with the current git branch; `recall` returns **current-branch + global** facts, not facts
from other branches. On a non-git directory, everything is treated as global. `recall format=markdown`
returns `decision`-category facts unless you pass another `category`, as `DECISIONS.md` text in its `markdown`
field; nothing is written to your repository, so save that text yourself.

## `read` shows `[MASKED len=N]` instead of a value

Since 5.0.0 `read` and `find` mask secret values in configuration by default; the header lists each one by
file, line and key, and the rest of a connection string is kept. Captured from 5.0.0:

```text
// SECRETS MASKED: appsettings.json L3 Default.Password - pass secrets="reveal" to see them
```

When troubleshooting needs the value, pass `secrets="reveal"` on the call over stdio. Over HTTP a per-call
reveal is refused unless the server itself sets `SANKSHEP_SECRETS=reveal`, which reveals values on every
`read` and `find` — see [Environment variables](environment.md).

## `find` mode=exact or regex lists `.sankshep/` files as "could not be read, NOT searched"

**Fixed in 5.0.1**, which never walks the state directory. On 5.0.0: the state directory, `.sankshep/`, sits
inside the repository by default, and the server holds its databases open while it runs, so a working-tree
scan reached those files and could not read them. That is why they appear on the "could not be read" line;
the rest of the tree was searched normally.

Upgrade to 5.0.1, or add `.sankshep/` to your `.gitignore`, as
[Security & privacy](security.md#where-state-lives-on-disk) recommends anyway: it is local state, not
source.
